# Part 105: โปรเจค Data Pipeline บน Kubernetes

## บทนำ

บทสุดท้ายของคอร์สนี้นำเสนอการสร้าง Data Pipeline แบบ Production-Ready บน Kubernetes ครอบคลุม Data Ingestion, Processing, Storage, และ Visualization โดยใช้เครื่องมือ Modern Data Stack

## สถาปัตยกรรม Data Pipeline

```
┌─────────────────────────────────────────────────────────────┐
│                    Data Sources                              │
│   Database CDC | API | Kafka | S3 | Web Events | IoT       │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                 Ingestion Layer                              │
│          Kafka Connect | Airbyte | Custom ETL               │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                 Stream Processing                            │
│         Apache Flink | Kafka Streams | Spark Streaming      │
└────────────┬──────────────┬──────────────────────────────────┘
             │              │
    ┌────────▼──────┐  ┌────▼───────────┐
    │  Data Lake    │  │  Data Warehouse │
    │  (MinIO/S3)   │  │  (ClickHouse)  │
    └────────┬──────┘  └────┬───────────┘
             │              │
┌────────────▼──────────────▼──────────────────────────────────┐
│                  Analytics & BI Layer                         │
│          Apache Superset | Grafana | Jupyter                 │
└──────────────────────────────────────────────────────────────┘

Orchestration:
┌──────────────────────────────────────────────────────────────┐
│                    Apache Airflow                            │
│            DAG Management | Scheduling | Monitoring         │
└──────────────────────────────────────────────────────────────┘

Quality & Governance:
┌──────────────────────────────────────────────────────────────┐
│          Great Expectations | Apache Atlas | dbt             │
└──────────────────────────────────────────────────────────────┘
```

---

## Step 1: ติดตั้ง Infrastructure

### 1.1 Kafka (Message Broker)

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install kafka bitnami/kafka \
  -n data-platform \
  --create-namespace \
  --set replicaCount=3 \
  --set zookeeper.replicaCount=3 \
  --set persistence.size=50Gi \
  --set resources.requests.cpu=500m \
  --set resources.requests.memory=2Gi \
  --set metrics.kafka.enabled=true
```

### 1.2 MinIO (Data Lake)

```yaml
# minio-values.yaml
replicas: 4
mode: distributed

rootUser: minioadmin
rootPassword: minioadmin123

persistence:
  enabled: true
  size: 500Gi
  storageClass: fast

resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: "2"
    memory: 4Gi

metrics:
  serviceMonitor:
    enabled: true

buckets:
- name: raw-data
  policy: none
  purge: false
- name: processed-data
  policy: none
  purge: false
- name: analytics
  policy: none
  purge: false
```

```bash
helm install minio minio/minio \
  -n data-platform \
  -f minio-values.yaml
```

### 1.3 ClickHouse (Data Warehouse)

```yaml
# clickhouse/clickhouse-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: clickhouse
  namespace: data-platform
spec:
  serviceName: clickhouse
  replicas: 3
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      labels:
        app: clickhouse
    spec:
      containers:
      - name: clickhouse
        image: clickhouse/clickhouse-server:23.8
        ports:
        - containerPort: 9000
          name: tcp
        - containerPort: 8123
          name: http
        - containerPort: 9009
          name: interserver
        env:
        - name: CLICKHOUSE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: clickhouse-credentials
              key: password
        resources:
          requests:
            cpu: "1"
            memory: 4Gi
          limits:
            cpu: "4"
            memory: 16Gi
        volumeMounts:
        - name: data
          mountPath: /var/lib/clickhouse
        - name: config
          mountPath: /etc/clickhouse-server/config.d
        livenessProbe:
          httpGet:
            path: /ping
            port: 8123
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /ping
            port: 8123
          initialDelaySeconds: 20
          periodSeconds: 10
      volumes:
      - name: config
        configMap:
          name: clickhouse-config
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast
      resources:
        requests:
          storage: 500Gi
---
apiVersion: v1
kind: Service
metadata:
  name: clickhouse
  namespace: data-platform
spec:
  clusterIP: None  # Headless
  selector:
    app: clickhouse
  ports:
  - name: tcp
    port: 9000
  - name: http
    port: 8123
  - name: interserver
    port: 9009
```

---

## Step 2: Data Ingestion

### 2.1 Apache Airflow

```bash
# ติดตั้ง Airflow
helm repo add apache-airflow https://airflow.apache.org

helm install airflow apache-airflow/airflow \
  -n data-platform \
  --set executor=KubernetesExecutor \
  --set dags.gitSync.enabled=true \
  --set dags.gitSync.repo=https://github.com/company/airflow-dags \
  --set dags.gitSync.branch=main \
  --set postgresql.enabled=true \
  --set redis.enabled=true \
  --set webserver.replicas=2
```

### 2.2 Airflow DAGs

```python
# dags/pipeline_dag.py
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.providers.amazon.aws.transfers.s3_to_local import S3ToLocalFilesystemOperator
from airflow.providers.http.operators.http import SimpleHttpOperator
from datetime import datetime, timedelta
import json

default_args = {
    'owner': 'data-team',
    'depends_on_past': False,
    'start_date': datetime(2025, 1, 1),
    'email': ['data-alerts@company.com'],
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': timedelta(minutes=5),
    'sla': timedelta(hours=4),
}

def extract_data(**kwargs):
    """Extract data from source database"""
    import psycopg2
    
    conn = psycopg2.connect(
        host=kwargs['db_host'],
        dbname=kwargs['db_name'],
        user=kwargs['db_user'],
        password=kwargs['db_password']
    )
    
    cursor = conn.cursor()
    cursor.execute("""
        SELECT id, user_id, amount, status, created_at
        FROM orders
        WHERE created_at >= %s AND created_at < %s
    """, (kwargs['start_date'], kwargs['end_date']))
    
    data = cursor.fetchall()
    conn.close()
    
    # Push to XCom
    kwargs['ti'].xcom_push(key='raw_data', value=data)
    return f"Extracted {len(data)} records"

def transform_data(**kwargs):
    """Transform and clean data"""
    import pandas as pd
    
    raw_data = kwargs['ti'].xcom_pull(task_ids='extract', key='raw_data')
    df = pd.DataFrame(raw_data, columns=['id', 'user_id', 'amount', 'status', 'created_at'])
    
    # ทำ Transformations
    df['created_at'] = pd.to_datetime(df['created_at'])
    df['date'] = df['created_at'].dt.date
    df['hour'] = df['created_at'].dt.hour
    df = df[df['status'].isin(['completed', 'refunded'])]
    df['net_amount'] = df.apply(
        lambda r: r['amount'] if r['status'] == 'completed' else -r['amount'],
        axis=1
    )
    
    # เก็บผล
    output_path = f"/tmp/orders_{kwargs['ds']}.parquet"
    df.to_parquet(output_path, index=False)
    kwargs['ti'].xcom_push(key='output_path', value=output_path)
    return f"Transformed {len(df)} records"

def load_to_warehouse(**kwargs):
    """Load data to ClickHouse"""
    import clickhouse_driver
    import pandas as pd
    
    output_path = kwargs['ti'].xcom_pull(task_ids='transform', key='output_path')
    df = pd.read_parquet(output_path)
    
    client = clickhouse_driver.Client(
        host=kwargs['ch_host'],
        password=kwargs['ch_password']
    )
    
    client.execute(
        'INSERT INTO analytics.orders VALUES',
        df.to_dict('records')
    )
    
    return f"Loaded {len(df)} records to ClickHouse"

with DAG(
    'orders_pipeline',
    default_args=default_args,
    description='Daily orders data pipeline',
    schedule_interval='0 2 * * *',
    catchup=True,
    max_active_runs=3,
    tags=['production', 'orders'],
) as dag:
    
    extract = PythonOperator(
        task_id='extract',
        python_callable=extract_data,
        op_kwargs={
            'db_host': '{{ var.value.source_db_host }}',
            'db_name': 'orders_db',
            'db_user': '{{ var.value.db_user }}',
            'db_password': '{{ var.value.db_password }}',
            'start_date': '{{ ds }}',
            'end_date': '{{ next_ds }}'
        },
    )
    
    transform = PythonOperator(
        task_id='transform',
        python_callable=transform_data,
    )
    
    load = PythonOperator(
        task_id='load',
        python_callable=load_to_warehouse,
        op_kwargs={
            'ch_host': '{{ var.value.clickhouse_host }}',
            'ch_password': '{{ var.value.ch_password }}'
        },
    )
    
    validate = PythonOperator(
        task_id='validate',
        python_callable=validate_data,
    )
    
    extract >> transform >> load >> validate
```

---

## Step 3: Stream Processing (Apache Flink)

```yaml
# Flink Cluster
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: stream-processor
  namespace: data-platform
spec:
  image: flink:1.18
  flinkVersion: v1_18
  flinkConfiguration:
    taskmanager.numberOfTaskSlots: "4"
    state.backend: rocksdb
    state.checkpoints.dir: s3://processed-data/flink-checkpoints/
    execution.checkpointing.interval: 60000
    execution.checkpointing.min-pause: 30000
    restart-strategy: exponential-delay
    metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
    metrics.reporter.prom.port: "9249"
  serviceAccount: flink-service-account
  jobManager:
    resource:
      memory: "2g"
      cpu: 1
    replicas: 1
  taskManager:
    resource:
      memory: "4g"
      cpu: 2
    replicas: 3
  job:
    jarURI: s3://jobs/stream-processor-1.0.0.jar
    parallelism: 6
    upgradeMode: stateless
    entryClass: com.company.StreamProcessorJob
    args:
    - "--kafka.bootstrap-servers=kafka.data-platform.svc.cluster.local:9092"
    - "--kafka.consumer.group=stream-processor"
    - "--clickhouse.host=clickhouse.data-platform.svc.cluster.local"
    - "--minio.endpoint=http://minio.data-platform.svc.cluster.local:9000"
```

### 3.1 Flink Job Example

```java
// StreamProcessorJob.java
package com.company;

import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaProducer;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.windowing.time.Time;

public class StreamProcessorJob {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Enable Checkpointing
        env.enableCheckpointing(60000);
        env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
        
        // Kafka Consumer
        Properties kafkaProps = new Properties();
        kafkaProps.setProperty("bootstrap.servers", "kafka:9092");
        kafkaProps.setProperty("group.id", "stream-processor");
        
        DataStream<String> orderEvents = env.addSource(
            new FlinkKafkaConsumer<>(
                "order.created",
                new SimpleStringSchema(),
                kafkaProps
            )
        );
        
        // Process: คำนวณ Revenue per minute
        DataStream<RevenueMetric> revenueStream = orderEvents
            .map(event -> parseEvent(event))
            .filter(order -> "completed".equals(order.getStatus()))
            .keyBy(order -> order.getProductCategory())
            .timeWindow(Time.minutes(1))
            .aggregate(new RevenueAggregator());
        
        // Write to ClickHouse
        revenueStream.addSink(
            new ClickHouseSink("clickhouse:8123", "analytics", "revenue_per_minute")
        );
        
        // Write to Kafka (for downstream consumers)
        revenueStream.addSink(
            new FlinkKafkaProducer<>(
                "analytics.revenue",
                new RevenueMetricSerializer(),
                kafkaProps,
                FlinkKafkaProducer.Semantic.EXACTLY_ONCE
            )
        );
        
        env.execute("Stream Processor Job");
    }
}
```

---

## Step 4: Data Quality (dbt + Great Expectations)

### 4.1 dbt Models

```sql
-- models/staging/stg_orders.sql
{{ config(materialized='view') }}

SELECT
    id AS order_id,
    user_id,
    amount,
    status,
    created_at::timestamp AS created_at,
    DATE_TRUNC('day', created_at) AS order_date
FROM {{ source('raw', 'orders') }}
WHERE status IN ('completed', 'refunded', 'cancelled')
  AND amount > 0

-- models/marts/fct_daily_revenue.sql
{{ config(
    materialized='incremental',
    unique_key='order_date',
    on_schema_change='sync_all_columns'
) }}

WITH daily_orders AS (
    SELECT
        order_date,
        COUNT(*) AS total_orders,
        COUNT(CASE WHEN status = 'completed' THEN 1 END) AS completed_orders,
        SUM(CASE WHEN status = 'completed' THEN amount ELSE 0 END) AS gross_revenue,
        SUM(CASE WHEN status = 'refunded' THEN amount ELSE 0 END) AS refunded_amount
    FROM {{ ref('stg_orders') }}
    {% if is_incremental() %}
    WHERE order_date >= (SELECT MAX(order_date) FROM {{ this }}) - INTERVAL '3 days'
    {% endif %}
    GROUP BY order_date
)

SELECT
    order_date,
    total_orders,
    completed_orders,
    gross_revenue,
    refunded_amount,
    gross_revenue - refunded_amount AS net_revenue,
    completed_orders::float / NULLIF(total_orders, 0) AS completion_rate,
    CURRENT_TIMESTAMP AS updated_at
FROM daily_orders
```

### 4.2 dbt Tests

```yaml
# models/staging/schema.yml
version: 2

sources:
- name: raw
  database: analytics
  schema: raw
  tables:
  - name: orders
    loaded_at_field: created_at
    freshness:
      warn_after:
        count: 1
        period: hour
      error_after:
        count: 4
        period: hour

models:
- name: stg_orders
  description: "Staged orders data"
  columns:
  - name: order_id
    description: "Unique order identifier"
    tests:
    - unique
    - not_null
  - name: amount
    description: "Order amount"
    tests:
    - not_null
    - dbt_expectations.expect_column_values_to_be_between:
        min_value: 0
        max_value: 100000
  - name: status
    description: "Order status"
    tests:
    - accepted_values:
        values: ["completed", "refunded", "cancelled", "pending"]
```

---

## Step 5: Analytics Dashboard (Apache Superset)

```bash
helm repo add superset https://apache.github.io/superset

helm install superset superset/superset \
  -n data-platform \
  --set supersetNode.replicaCount=2 \
  --set postgresql.enabled=true \
  --set redis.enabled=true \
  --set ingress.enabled=true \
  --set ingress.hostname=superset.example.com
```

```yaml
# Superset ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: superset-config
  namespace: data-platform
data:
  superset_config.py: |
    import os
    
    # Database Connection
    SQLALCHEMY_DATABASE_URI = os.environ.get('SUPERSET_DB_URI')
    
    # Security
    SECRET_KEY = os.environ.get('SUPERSET_SECRET_KEY')
    WTF_CSRF_ENABLED = True
    
    # Feature Flags
    FEATURE_FLAGS = {
        "ENABLE_TEMPLATE_PROCESSING": True,
        "DASHBOARD_RBAC": True,
        "ALERT_REPORTS": True,
        "EMBEDDED_SUPERSET": True,
    }
    
    # Cache
    CACHE_CONFIG = {
        'CACHE_TYPE': 'RedisCache',
        'CACHE_DEFAULT_TIMEOUT': 300,
        'CACHE_KEY_PREFIX': 'superset_',
        'CACHE_REDIS_URL': os.environ.get('REDIS_URL'),
    }
    
    # Email Reports
    SMTP_HOST = 'smtp.sendgrid.net'
    SMTP_PORT = 587
    SMTP_USER = os.environ.get('SMTP_USER')
    SMTP_PASSWORD = os.environ.get('SMTP_PASSWORD')
    SMTP_MAIL_FROM = 'analytics@company.com'
```

---

## Step 6: Kubernetes Jobs สำหรับ Batch Processing

### 6.1 Spark Job

```yaml
# spark-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: daily-aggregation
  namespace: data-platform
spec:
  ttlSecondsAfterFinished: 86400  # ลบหลัง 24 ชั่วโมง
  template:
    spec:
      restartPolicy: OnFailure
      serviceAccountName: spark-job-sa
      containers:
      - name: spark-submit
        image: apache/spark:3.4.0
        command:
        - /opt/spark/bin/spark-submit
        - --master
        - k8s://https://kubernetes.default.svc:443
        - --deploy-mode
        - cluster
        - --name
        - daily-aggregation
        - --conf
        - spark.executor.instances=5
        - --conf
        - spark.executor.memory=4g
        - --conf
        - spark.executor.cores=2
        - --conf
        - spark.driver.memory=2g
        - --conf
        - spark.kubernetes.container.image=company-registry.io/spark-app:1.0.0
        - --conf
        - spark.kubernetes.namespace=data-platform
        - --conf
        - spark.kubernetes.authenticate.driver.serviceAccountName=spark-job-sa
        - --conf
        - spark.hadoop.fs.s3a.endpoint=http://minio.data-platform.svc.cluster.local:9000
        - --conf
        - spark.hadoop.fs.s3a.access.key=$(MINIO_ACCESS_KEY)
        - --conf
        - spark.hadoop.fs.s3a.secret.key=$(MINIO_SECRET_KEY)
        - s3a://processed-data/jobs/daily_aggregation.py
        - --date
        - "$(DATE)"
        env:
        - name: DATE
          value: "$(date -d 'yesterday' +%Y-%m-%d)"
        - name: MINIO_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: access-key
        - name: MINIO_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: minio-credentials
              key: secret-key
        resources:
          requests:
            cpu: 500m
            memory: 2Gi
          limits:
            cpu: "2"
            memory: 4Gi
```

### 6.2 Spark Python Script

```python
# daily_aggregation.py
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window
import argparse
import sys

def create_spark_session():
    return SparkSession.builder \
        .appName("DailyAggregation") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()

def process_orders(spark, date):
    # อ่านข้อมูลจาก Data Lake
    df = spark.read.parquet(f"s3a://raw-data/orders/date={date}/")
    
    # Aggregate
    daily_summary = df \
        .filter(F.col("status") == "completed") \
        .groupBy("product_category", "region") \
        .agg(
            F.count("*").alias("order_count"),
            F.sum("amount").alias("revenue"),
            F.avg("amount").alias("avg_order_value"),
            F.countDistinct("user_id").alias("unique_customers"),
            F.min("created_at").alias("first_order"),
            F.max("created_at").alias("last_order")
        ) \
        .withColumn("date", F.lit(date)) \
        .withColumn("revenue_rank", F.rank().over(
            Window.partitionBy("date").orderBy(F.desc("revenue"))
        ))
    
    # เขียนลง Data Warehouse format
    daily_summary.write \
        .mode("overwrite") \
        .partitionBy("date") \
        .parquet(f"s3a://processed-data/daily_summary/")
    
    print(f"Processed {daily_summary.count()} category-region combinations for {date}")
    return daily_summary

def calculate_cohort_analysis(spark, date):
    """คำนวณ User Cohort Analysis"""
    # อ่าน Orders
    orders = spark.read.parquet("s3a://raw-data/orders/")
    
    # หา First Order Date สำหรับแต่ละ User
    first_orders = orders.groupBy("user_id") \
        .agg(F.min("created_at").alias("cohort_date"))
    
    first_orders = first_orders.withColumn(
        "cohort_month",
        F.date_trunc("month", "cohort_date")
    )
    
    # Join กับ Orders
    cohort_data = orders.join(first_orders, "user_id") \
        .withColumn("months_since_first", 
            F.months_between(F.col("created_at"), F.col("cohort_date")).cast("int")
        )
    
    # คำนวณ Retention
    retention = cohort_data.groupBy("cohort_month", "months_since_first") \
        .agg(F.countDistinct("user_id").alias("active_users"))
    
    # เขียนผล
    retention.write \
        .mode("overwrite") \
        .parquet(f"s3a://processed-data/cohort_analysis/")
    
    return retention

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--date", required=True, help="Processing date YYYY-MM-DD")
    args = parser.parse_args()
    
    spark = create_spark_session()
    
    print(f"Starting daily aggregation for {args.date}")
    
    orders_summary = process_orders(spark, args.date)
    cohort_data = calculate_cohort_analysis(spark, args.date)
    
    spark.stop()
    print("Done!")
```

---

## Step 7: Data Monitoring

### 7.1 Data Quality Alerts

```yaml
# PrometheusRule สำหรับ Data Pipeline
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: data-pipeline-alerts
  namespace: monitoring
spec:
  groups:
  - name: data.quality
    rules:
    - alert: PipelineJobFailed
      expr: |
        kube_job_status_failed{
          namespace="data-platform",
          job_name=~"daily-.*"
        } > 0
      for: 0m
      labels:
        severity: critical
        team: data-engineering
      annotations:
        summary: "Data Pipeline Job Failed: {{ $labels.job_name }}"
        description: "Job {{ $labels.job_name }} has failed"
        runbook_url: "https://wiki.company.com/runbooks/data-pipeline"

    - alert: KafkaConsumerLag
      expr: |
        kafka_consumergroup_lag_sum{
          namespace="data-platform",
          consumergroup=~".*processor.*"
        } > 100000
      for: 15m
      labels:
        severity: warning
      annotations:
        summary: "High Kafka Consumer Lag: {{ $labels.consumergroup }}"

    - alert: DataFreshnessViolation
      expr: |
        time() - max(data_last_processed_timestamp_seconds) > 7200
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Data Freshness Violation"
        description: "Data has not been processed for more than 2 hours"

    - alert: FlinkJobNotRunning
      expr: |
        flink_jobmanager_numRunningJobs < 1
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "No Flink Jobs Running"
```

### 7.2 Data Pipeline Dashboard

```python
# grafana_dashboard.py
# Dashboard JSON สำหรับ Data Pipeline

dashboard = {
    "title": "Data Pipeline Overview",
    "refresh": "30s",
    "panels": [
        {
            "title": "Kafka Consumer Lag",
            "type": "timeseries",
            "targets": [
                {
                    "expr": "kafka_consumergroup_lag_sum",
                    "legendFormat": "{{consumergroup}}"
                }
            ]
        },
        {
            "title": "Records Processed per Second",
            "type": "stat",
            "targets": [
                {
                    "expr": "rate(flink_taskmanager_job_task_numRecordsOut[5m])",
                    "legendFormat": "Records/s"
                }
            ]
        },
        {
            "title": "Airflow DAG Success Rate",
            "type": "gauge",
            "targets": [
                {
                    "expr": """
                        rate(airflow_dag_success_total[24h]) /
                        (rate(airflow_dag_success_total[24h]) + rate(airflow_dag_failed_total[24h]))
                    """
                }
            ],
            "fieldConfig": {
                "defaults": {
                    "thresholds": {
                        "steps": [
                            {"color": "red", "value": 0},
                            {"color": "yellow", "value": 0.9},
                            {"color": "green", "value": 0.95}
                        ]
                    }
                }
            }
        },
        {
            "title": "ClickHouse Query Performance",
            "type": "timeseries",
            "targets": [
                {
                    "expr": "histogram_quantile(0.99, rate(clickhouse_query_duration_seconds_bucket[5m]))",
                    "legendFormat": "P99 Query Duration"
                }
            ]
        }
    ]
}
```

---

## Step 8: Airflow DAG สำหรับ ML Pipeline

```python
# dags/ml_pipeline_dag.py
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import KubernetesPodOperator
from kubernetes.client import models as k8s
from datetime import datetime, timedelta

default_args = {
    'owner': 'ml-team',
    'start_date': datetime(2025, 1, 1),
    'retries': 2,
    'retry_delay': timedelta(minutes=10),
}

with DAG(
    'ml_training_pipeline',
    default_args=default_args,
    schedule_interval='0 3 * * 0',  # ทุกอาทิตย์
    catchup=False,
    tags=['ml', 'training'],
) as dag:
    
    # Feature Engineering
    feature_engineering = KubernetesPodOperator(
        task_id='feature_engineering',
        name='feature-engineering',
        namespace='data-platform',
        image='company-registry.io/ml-feature-engineering:latest',
        cmds=['python', 'feature_engineering.py'],
        arguments=[
            '--start-date', '{{ ds }}',
            '--end-date', '{{ next_ds }}'
        ],
        env_vars=[
            k8s.V1EnvVar(
                name='MINIO_ENDPOINT',
                value='minio.data-platform.svc.cluster.local:9000'
            )
        ],
        resources=k8s.V1ResourceRequirements(
            requests={'cpu': '2', 'memory': '8Gi'},
            limits={'cpu': '4', 'memory': '16Gi'}
        ),
        is_delete_operator_pod=True,
    )
    
    # Model Training
    model_training = KubernetesPodOperator(
        task_id='model_training',
        name='model-training',
        namespace='data-platform',
        image='company-registry.io/ml-training:latest',
        cmds=['python', 'train.py'],
        arguments=[
            '--experiment', 'churn_prediction',
            '--date', '{{ ds }}'
        ],
        resources=k8s.V1ResourceRequirements(
            requests={'cpu': '4', 'memory': '16Gi', 'nvidia.com/gpu': '1'},
            limits={'cpu': '8', 'memory': '32Gi', 'nvidia.com/gpu': '1'}
        ),
        tolerations=[
            k8s.V1Toleration(
                key='nvidia.com/gpu',
                operator='Exists',
                effect='NoSchedule'
            )
        ],
        is_delete_operator_pod=True,
    )
    
    # Model Validation
    model_validation = KubernetesPodOperator(
        task_id='model_validation',
        name='model-validation',
        namespace='data-platform',
        image='company-registry.io/ml-validation:latest',
        cmds=['python', 'validate.py'],
        arguments=['--model-version', '{{ ds }}'],
        is_delete_operator_pod=True,
    )
    
    # Deploy Model (if validation passes)
    deploy_model = KubernetesPodOperator(
        task_id='deploy_model',
        name='deploy-model',
        namespace='data-platform',
        image='company-registry.io/ml-deployer:latest',
        cmds=['python', 'deploy.py'],
        arguments=['--model-version', '{{ ds }}'],
        is_delete_operator_pod=True,
    )
    
    feature_engineering >> model_training >> model_validation >> deploy_model
```

---

## Workshop: Deploy Data Platform ทั้งหมด

```bash
#!/bin/bash
set -e

echo "=== Deploying Data Platform ==="

# 1. Namespace
kubectl create namespace data-platform --dry-run=client -o yaml | kubectl apply -f -

# 2. Storage
echo "Deploying storage layer..."
helm upgrade --install kafka bitnami/kafka -n data-platform -f kafka-values.yaml --wait
helm upgrade --install minio minio/minio -n data-platform -f minio-values.yaml --wait
kubectl apply -f clickhouse/ -n data-platform

# 3. Processing
echo "Deploying processing layer..."
kubectl apply -f flink/ -n data-platform

# 4. Orchestration
echo "Deploying Airflow..."
helm upgrade --install airflow apache-airflow/airflow \
  -n data-platform \
  -f airflow-values.yaml \
  --wait

# 5. Analytics
echo "Deploying Analytics..."
helm upgrade --install superset superset/superset \
  -n data-platform \
  --wait

# 6. Monitoring
echo "Deploying monitoring..."
kubectl apply -f monitoring/

echo "=== Data Platform Deployed ==="

# Status
kubectl get pods -n data-platform
```

### Verification

```bash
# ทดสอบ Kafka
kubectl exec -n data-platform kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --list

# ทดสอบ MinIO
kubectl port-forward -n data-platform svc/minio 9001:9001
# เปิด http://localhost:9001

# ทดสอบ ClickHouse
kubectl exec -n data-platform clickhouse-0 -- clickhouse-client \
  --query "SELECT version()"

# ดู Airflow
kubectl port-forward -n data-platform svc/airflow-webserver 8080:8080
# เปิด http://localhost:8080

# ดู Superset
kubectl port-forward -n data-platform svc/superset 8088:8088
# เปิด http://localhost:8088
```

---

## สรุปคอร์สทั้งหมด

ยินดีด้วย! คุณได้เรียนจบคอร์ส Kubernetes ครบ 105 บท ครอบคลุม:

### ภาพรวมที่เรียนมา

```
Parts 1-30:   Kubernetes Fundamentals
             - Architecture, Pods, Deployments, Services
             - Storage, ConfigMaps, Secrets
             - RBAC, Network Policies

Parts 31-60:  Advanced Kubernetes
             - Helm, Operators, CRDs
             - Multi-cluster, Federation
             - GitOps with ArgoCD/Flux

Parts 61-90:  Enterprise Kubernetes
             - CI/CD Pipelines
             - Service Mesh (Istio)
             - Observability Stack

Parts 91-99:  Production Operations
             - AKS, Production Setup, HA
             - Disaster Recovery, Performance Tuning
             - Cost Optimization, Compliance
             - Troubleshooting, Migration

Parts 100-102: Certifications
             - CKA (Cluster Administration)
             - CKAD (Application Development)
             - CKS (Security Specialist)

Parts 103-105: Capstone Projects
             - E-Commerce Platform
             - Microservices Architecture
             - Data Pipeline
```

### Next Steps

1. **สอบ CKA, CKAD, CKS** - ฝึกผ่าน KillerCoda และ killer.sh
2. **Deploy โปรเจคจริง** - เริ่มจาก E-Commerce Project
3. **เรียนรู้ต่อ**:
   - OpenTelemetry
   - Platform Engineering
   - FinOps
   - Green/Sustainable Computing

## References

- [Apache Airflow](https://airflow.apache.org/)
- [Apache Flink](https://flink.apache.org/)
- [Apache Spark](https://spark.apache.org/)
- [ClickHouse](https://clickhouse.com/)
- [Apache Superset](https://superset.apache.org/)
- [dbt Documentation](https://docs.getdbt.com/)
- [CNCF Landscape](https://landscape.cncf.io/)
