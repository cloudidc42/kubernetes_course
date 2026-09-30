# Part 23: Jobs และ CronJobs - Batch Processing ใน Kubernetes

## สารบัญ
1. [Jobs คืออะไร](#jobs-คืออะไร)
2. [Job YAML ละเอียด](#job-yaml-ละเอียด)
3. [Job Types](#job-types)
4. [Job Parallelism](#job-parallelism)
5. [CronJobs คืออะไร](#cronjobs-คืออะไร)
6. [CronJob YAML ละเอียด](#cronjob-yaml-ละเอียด)
7. [Workshop: Batch Data Processing](#workshop-batch-data-processing)
8. [Workshop: Database Backup CronJob](#workshop-database-backup-cronjob)
9. [Workshop: Report Generation](#workshop-report-generation)
10. [Troubleshooting Jobs](#troubleshooting-jobs)
11. [Best Practices](#best-practices)

---

## Jobs คืออะไร

**Job** ใน Kubernetes คือ workload resource ที่สร้าง Pod หนึ่งตัวหรือมากกว่าและรับประกันว่า Pod จะ**รันจนสำเร็จ** (complete successfully):

- ต่างจาก Deployment ที่รัน Pod ตลอดเวลา
- Job รัน Pod จนกว่างานจะเสร็จ แล้ว Pod จะหยุด
- ถ้า Pod ล้มเหลว Job จะสร้าง Pod ใหม่และรันใหม่

### เปรียบเทียบ Workload Types

```
┌───────────────┬──────────────────┬────────────────────────────┐
│ Workload Type │ Duration         │ Use Case                   │
├───────────────┼──────────────────┼────────────────────────────┤
│ Deployment    │ ตลอดเวลา          │ Web server, API service     │
│ StatefulSet   │ ตลอดเวลา          │ Databases, stateful apps   │
│ DaemonSet     │ ตลอดเวลา          │ Monitoring, logging agents  │
│ Job           │ รันครั้งเดียว      │ Data migration, batch job  │
│ CronJob       │ รันตามเวลา        │ Backup, report generation  │
└───────────────┴──────────────────┴────────────────────────────┘
```

### ตัวอย่าง Use Cases

```
┌────────────────────────────────────────────────────────────┐
│                     Job Use Cases                          │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  One-time Tasks:                                           │
│  - Database migration (schema changes)                     │
│  - Data import/export                                      │
│  - Seed initial data                                       │
│  - System initialization                                   │
│                                                            │
│  Batch Processing:                                         │
│  - Process large files                                     │
│  - Image/video transcoding                                 │
│  - Send bulk emails                                        │
│  - Machine learning training                               │
│                                                            │
│  Testing:                                                  │
│  - Run integration tests                                   │
│  - Load testing                                            │
│  - Data validation                                         │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Job YAML ละเอียด

### Basic Job

```yaml
# basic-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello-job
  namespace: default
  labels:
    app: hello-job
spec:
  # จำนวน Pod completions ที่ต้องการ
  completions: 1
  
  # จำนวน Pod ที่รัน parallel กัน
  parallelism: 1
  
  # Backoff limit: retry ได้กี่ครั้งก่อน fail
  backoffLimit: 3
  
  # Active deadline: Job ต้องเสร็จภายใน X วินาที
  activeDeadlineSeconds: 300
  
  # TTL หลัง Job เสร็จ: ลบ Job หลัง X วินาที (ต้องเปิด TTLAfterFinished)
  ttlSecondsAfterFinished: 100
  
  # Pod template
  template:
    metadata:
      labels:
        app: hello-job
    spec:
      # restartPolicy ต้องเป็น Never หรือ OnFailure (ไม่ใช่ Always)
      restartPolicy: Never
      
      containers:
      - name: hello
        image: busybox:1.35
        command: ["sh", "-c"]
        args: ["echo 'Hello Kubernetes Job!' && sleep 5 && echo 'Job completed!'"]
        
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "200m"
            memory: "100Mi"
```

### Job ที่ซับซ้อนกว่า

```yaml
# complex-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
  namespace: batch
  labels:
    app: data-processor
    job-type: batch
  annotations:
    description: "Process customer data from S3"
spec:
  completions: 10       # ต้องการ Pod สำเร็จ 10 ตัว
  parallelism: 3        # รัน 3 Pod พร้อมกัน
  backoffLimit: 6       # retry ได้ 6 ครั้ง
  activeDeadlineSeconds: 3600  # ต้องเสร็จใน 1 ชม.
  ttlSecondsAfterFinished: 300  # ลบหลัง 5 นาที
  
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      restartPolicy: OnFailure
      
      initContainers:
      - name: wait-for-db
        image: busybox:1.35
        command: ['sh', '-c', 'until nc -z postgres 5432; do sleep 2; done']
      
      containers:
      - name: processor
        image: python:3.11
        command: ["python", "/app/process_data.py"]
        
        env:
        - name: BATCH_SIZE
          value: "1000"
        - name: DB_HOST
          value: "postgres.default.svc.cluster.local"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        - name: JOB_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        
        volumeMounts:
        - name: scripts
          mountPath: /app
        - name: output
          mountPath: /output
      
      volumes:
      - name: scripts
        configMap:
          name: processor-scripts
      - name: output
        persistentVolumeClaim:
          claimName: job-output-pvc
```

---

## Job Types

### 1. Non-parallel Job (Default)

```yaml
spec:
  completions: 1     # รับ default เป็น 1
  parallelism: 1     # รับ default เป็น 1
  # รัน Pod เดียว จนสำเร็จ
```

### 2. Parallel Job with Fixed Completion Count

```yaml
spec:
  completions: 5    # ต้องการ 5 completions
  parallelism: 2    # รัน 2 Pod พร้อมกัน
  # รัน 2 Pod พร้อมกัน จนได้ 5 completions
  # Pattern: [p1, p2] -> p1 done -> [p3, p2] -> p2 done -> [p3, p4] -> ...
```

### 3. Parallel Job with Work Queue

```yaml
spec:
  parallelism: 3     # รัน 3 Pod พร้อมกัน
  # ไม่ระบุ completions
  # Pod อ่าน work จาก queue
  # Job เสร็จเมื่อ Pod ใด Pod หนึ่ง exit 0 และ Pod อื่น exit 0 ด้วย
```

### 4. Indexed Job (Kubernetes 1.21+)

```yaml
spec:
  completions: 5
  parallelism: 2
  completionMode: Indexed  # แต่ละ Pod มี index (0, 1, 2, 3, 4)
  # env var JOB_COMPLETION_INDEX จะถูก set ให้แต่ละ Pod
```

```bash
# ตรวจสอบ index ของแต่ละ Pod
kubectl get pods -l batch.kubernetes.io/job-name=my-job \
  -o custom-columns='NAME:.metadata.name,INDEX:.metadata.annotations.batch\.kubernetes\.io/job-completion-index'
```

---

## Job Parallelism

### ตัวอย่าง: Process หลาย Files พร้อมกัน

```yaml
# parallel-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: file-processor
  namespace: batch
spec:
  completions: 6      # ต้องประมวลผล 6 files
  parallelism: 3      # รัน 3 workers พร้อมกัน
  completionMode: Indexed
  backoffLimit: 3
  
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: processor
        image: python:3.11-slim
        command:
        - python
        - -c
        - |
          import os
          import time
          
          # รับ file index จาก environment variable
          index = int(os.environ.get('JOB_COMPLETION_INDEX', '0'))
          
          files = [
            'data-2024-01.csv',
            'data-2024-02.csv',
            'data-2024-03.csv',
            'data-2024-04.csv',
            'data-2024-05.csv',
            'data-2024-06.csv',
          ]
          
          file = files[index]
          print(f"Processing file: {file} (index: {index})")
          
          # Simulate processing
          time.sleep(10)
          
          print(f"Completed processing: {file}")
        
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        
        resources:
          requests:
            cpu: "200m"
            memory: "128Mi"
```

### Work Queue Pattern

```yaml
# work-queue-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue-processor
spec:
  parallelism: 4  # 4 workers
  # ไม่ระบุ completions - workers pull from queue
  
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: worker
        image: python:3.11
        command:
        - python
        - -c
        - |
          import redis
          import json
          import time
          import sys
          
          r = redis.Redis(host='redis', port=6379)
          
          while True:
            # ดึงงานจาก queue
            work = r.blpop('work-queue', timeout=10)
            
            if work is None:
              # ไม่มีงาน - exit
              print("Queue empty, exiting")
              sys.exit(0)
            
            job_data = json.loads(work[1])
            print(f"Processing job: {job_data['id']}")
            
            # ทำงาน
            time.sleep(job_data.get('duration', 5))
            
            print(f"Completed job: {job_data['id']}")
        
        env:
        - name: REDIS_HOST
          value: "redis.default.svc.cluster.local"
```

---

## CronJobs คืออะไร

**CronJob** สร้าง Job ตามตาราง schedule (เหมือน cron ใน Linux):

- ใช้ **cron syntax** สำหรับ schedule
- สร้าง Job ใหม่ทุกครั้งที่ถึง schedule
- ควบคุม history ของ Job ที่เสร็จแล้ว

### Cron Syntax

```
┌──────────── minute (0 - 59)
│ ┌────────── hour (0 - 23)
│ │ ┌──────── day of month (1 - 31)
│ │ │ ┌────── month (1 - 12)
│ │ │ │ ┌──── day of week (0 - 7) (0 and 7 are Sunday)
│ │ │ │ │
* * * * *

ตัวอย่าง:
0 * * * *      → ทุกชั่วโมง (นาทีที่ 0)
*/5 * * * *    → ทุก 5 นาที
0 2 * * *      → ทุกวัน 2am
0 2 * * 0      → ทุกวันอาทิตย์ 2am
0 2 1 * *      → วันที่ 1 ของทุกเดือน 2am
0 0 * * 1-5    → ทุกวัน weekday เที่ยงคืน
@hourly        → ทุกชั่วโมง (= 0 * * * *)
@daily         → ทุกวัน (= 0 0 * * *)
@weekly        → ทุกสัปดาห์ (= 0 0 * * 0)
@monthly       → ทุกเดือน (= 0 0 1 * *)
@yearly        → ทุกปี (= 0 0 1 1 *)
```

---

## CronJob YAML ละเอียด

### Basic CronJob

```yaml
# basic-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cronjob
  namespace: default
spec:
  # Schedule: ทุก 5 นาที
  schedule: "*/5 * * * *"
  
  # Timezone (Kubernetes 1.25+)
  timeZone: "Asia/Bangkok"
  
  # Concurrency Policy: กรณี Job ก่อนยังไม่เสร็จเมื่อถึง schedule ใหม่
  concurrencyPolicy: Allow  # Allow, Forbid, Replace
  
  # เก็บ history ของ successful jobs ไว้กี่ Job
  successfulJobsHistoryLimit: 3
  
  # เก็บ history ของ failed jobs ไว้กี่ Job
  failedJobsHistoryLimit: 1
  
  # หยุดทำงาน CronJob (ไม่ trigger Job ใหม่)
  suspend: false
  
  # ถ้า schedule ถูก miss ไปนานเท่าไร (วินาที) ยังคง trigger
  startingDeadlineSeconds: 200
  
  # Job template
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 300
      
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.35
            command:
            - /bin/sh
            - -c
            - date; echo "Hello from CronJob!"
            
            resources:
              requests:
                cpu: "100m"
                memory: "50Mi"
              limits:
                cpu: "200m"
                memory: "100Mi"
```

### CronJob ที่สมบูรณ์

```yaml
# full-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
  namespace: production
  labels:
    app: database-backup
    type: cronjob
spec:
  # ทุกวัน 2am (Bangkok time)
  schedule: "0 2 * * *"
  timeZone: "Asia/Bangkok"
  
  # ไม่อนุญาตให้ Job ซ้อนกัน
  concurrencyPolicy: Forbid
  
  successfulJobsHistoryLimit: 7   # เก็บ history 7 วัน
  failedJobsHistoryLimit: 3       # เก็บ failed history 3 ครั้ง
  
  startingDeadlineSeconds: 3600   # ถ้า miss ไปไม่เกิน 1 ชม. ยัง trigger
  
  jobTemplate:
    metadata:
      labels:
        app: database-backup
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 7200  # ต้องเสร็จใน 2 ชม.
      
      template:
        metadata:
          labels:
            app: database-backup
        spec:
          restartPolicy: OnFailure
          
          serviceAccountName: backup-sa
          
          initContainers:
          - name: check-db
            image: postgres:14
            command:
            - sh
            - -c
            - until pg_isready -h $POSTGRES_HOST -U postgres; do sleep 5; done
            env:
            - name: POSTGRES_HOST
              value: "postgres.production.svc.cluster.local"
          
          containers:
          - name: backup
            image: postgres:14
            command:
            - /bin/bash
            - -c
            - |
              set -e
              
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              BACKUP_FILE="/backup/db_backup_${TIMESTAMP}.sql.gz"
              
              echo "Starting backup at $(date)"
              echo "Backup file: ${BACKUP_FILE}"
              
              # pg_dump with compression
              PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
                -h $POSTGRES_HOST \
                -U postgres \
                -d $POSTGRES_DB \
                --format=custom \
                --no-acl \
                --no-owner \
                | gzip > ${BACKUP_FILE}
              
              # ตรวจสอบขนาดไฟล์
              BACKUP_SIZE=$(du -h ${BACKUP_FILE} | cut -f1)
              echo "Backup size: ${BACKUP_SIZE}"
              
              # Upload ไป S3 (ถ้ามี aws cli)
              # aws s3 cp ${BACKUP_FILE} s3://$BACKUP_BUCKET/
              
              # ลบ backup เก่ากว่า 30 วัน
              find /backup -name "db_backup_*.sql.gz" -mtime +30 -delete
              
              echo "Backup completed successfully at $(date)"
            
            env:
            - name: POSTGRES_HOST
              value: "postgres.production.svc.cluster.local"
            - name: POSTGRES_DB
              value: "myapp"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: BACKUP_BUCKET
              valueFrom:
                configMapKeyRef:
                  name: backup-config
                  key: s3-bucket
            
            resources:
              requests:
                cpu: "500m"
                memory: "256Mi"
              limits:
                cpu: "1000m"
                memory: "512Mi"
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

---

## Workshop: Batch Data Processing

เป้าหมาย: ประมวลผล CSV files แบบ batch ด้วย Job

### 1. สร้าง Namespace และ Resources

```bash
kubectl create namespace batch-processing
```

### 2. สร้าง ConfigMap สำหรับ Processing Script

```yaml
# processor-script.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: processor-scripts
  namespace: batch-processing
data:
  process_batch.py: |
    #!/usr/bin/env python3
    """
    Batch data processor
    ประมวลผล CSV records และบันทึกผลลัพธ์
    """
    import os
    import csv
    import json
    import time
    import logging
    import sys
    from datetime import datetime
    
    # Setup logging
    logging.basicConfig(
        level=logging.INFO,
        format='%(asctime)s - %(levelname)s - %(message)s'
    )
    logger = logging.getLogger(__name__)
    
    def process_record(record):
        """ประมวลผล record แต่ละอัน"""
        # simulate processing
        time.sleep(0.1)
        
        return {
            'id': record.get('id'),
            'name': record.get('name', '').upper(),
            'amount': float(record.get('amount', 0)) * 1.07,  # บวก VAT
            'processed_at': datetime.now().isoformat()
        }
    
    def main():
        batch_index = int(os.environ.get('JOB_COMPLETION_INDEX', '0'))
        input_dir = os.environ.get('INPUT_DIR', '/data/input')
        output_dir = os.environ.get('OUTPUT_DIR', '/data/output')
        batch_size = int(os.environ.get('BATCH_SIZE', '100'))
        
        logger.info(f"Starting batch processor - Index: {batch_index}")
        logger.info(f"Processing batch {batch_index} of size {batch_size}")
        
        # สร้าง output directory
        os.makedirs(output_dir, exist_ok=True)
        
        # กำหนด records ที่จะประมวลผล
        start_record = batch_index * batch_size
        end_record = start_record + batch_size
        
        logger.info(f"Processing records {start_record} to {end_record}")
        
        # simulate reading from input
        results = []
        for i in range(start_record, end_record):
            record = {
                'id': i,
                'name': f'Customer {i}',
                'amount': i * 10.50
            }
            
            try:
                processed = process_record(record)
                results.append(processed)
            except Exception as e:
                logger.error(f"Failed to process record {i}: {e}")
        
        # บันทึกผลลัพธ์
        output_file = os.path.join(output_dir, f'batch_{batch_index:04d}.json')
        with open(output_file, 'w') as f:
            json.dump(results, f, indent=2)
        
        logger.info(f"Saved {len(results)} records to {output_file}")
        logger.info(f"Batch {batch_index} completed successfully!")
    
    if __name__ == '__main__':
        main()
```

### 3. สร้าง PVC สำหรับ Output

```yaml
# batch-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: batch-output-pvc
  namespace: batch-processing
spec:
  accessModes:
  - ReadWriteMany
  storageClassName: standard
  resources:
    requests:
      storage: 5Gi
```

### 4. สร้าง Batch Job

```yaml
# batch-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor-$(date +%Y%m%d)
  namespace: batch-processing
  labels:
    app: data-processor
    date: "2024-01-15"
spec:
  completions: 10        # ประมวลผล 10 batches
  parallelism: 3         # 3 workers พร้อมกัน
  completionMode: Indexed
  backoffLimit: 3
  activeDeadlineSeconds: 7200
  ttlSecondsAfterFinished: 600
  
  template:
    metadata:
      labels:
        app: data-processor
    spec:
      restartPolicy: OnFailure
      
      containers:
      - name: processor
        image: python:3.11-slim
        command:
        - python
        - /scripts/process_batch.py
        
        env:
        - name: JOB_COMPLETION_INDEX
          valueFrom:
            fieldRef:
              fieldPath: metadata.annotations['batch.kubernetes.io/job-completion-index']
        - name: INPUT_DIR
          value: "/data/input"
        - name: OUTPUT_DIR
          value: "/data/output"
        - name: BATCH_SIZE
          value: "1000"
        
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"
        
        volumeMounts:
        - name: scripts
          mountPath: /scripts
        - name: output
          mountPath: /data/output
      
      volumes:
      - name: scripts
        configMap:
          name: processor-scripts
      - name: output
        persistentVolumeClaim:
          claimName: batch-output-pvc
```

### 5. รัน Job และ Monitor

```bash
# Apply resources
kubectl apply -f processor-script.yaml
kubectl apply -f batch-storage.yaml

# สร้าง Job ด้วย unique name
kubectl create job data-processor-$(date +%Y%m%d%H%M) \
  --from=cronjob/data-processor-cron \
  -n batch-processing

# หรือ apply job yaml โดยตรง
kubectl apply -f batch-job.yaml

# ดู Job status
kubectl get jobs -n batch-processing

# ดู Pod ที่รันอยู่
kubectl get pods -n batch-processing -l app=data-processor

# ดู logs ของ Pod แต่ละตัว
kubectl logs -n batch-processing -l app=data-processor --all-containers

# ดู progress แบบ real-time
kubectl get pods -n batch-processing -w

# ดู Job events
kubectl describe job data-processor-20240115 -n batch-processing

# รอจน Job เสร็จ
kubectl wait --for=condition=complete job/data-processor-20240115 \
  -n batch-processing --timeout=7200s

# ตรวจสอบผลลัพธ์
kubectl run -it --rm result-checker \
  --image=busybox \
  --restart=Never \
  -n batch-processing \
  -- ls /data/output
```

---

## Workshop: Database Backup CronJob

### 1. สร้าง ServiceAccount และ Secret

```bash
# สร้าง namespace
kubectl create namespace production

# สร้าง secret สำหรับ database
kubectl create secret generic postgres-secret \
  --from-literal=password=supersecret \
  -n production

# สร้าง secret สำหรับ S3
kubectl create secret generic aws-credentials \
  --from-literal=access-key=AKIAIOSFODNN7EXAMPLE \
  --from-literal=secret-key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY \
  -n production
```

### 2. สร้าง ConfigMap

```yaml
# backup-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-config
  namespace: production
data:
  s3-bucket: "my-db-backups"
  retention-days: "30"
  databases: "myapp,analytics,logs"
  
  backup.sh: |
    #!/bin/bash
    set -euo pipefail
    
    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
    BACKUP_DIR="/backup/${TIMESTAMP}"
    mkdir -p "${BACKUP_DIR}"
    
    echo "=== Database Backup Started at $(date) ==="
    
    # Loop through databases
    for DB in $(echo $DATABASES | tr ',' ' '); do
      echo "Backing up database: ${DB}"
      BACKUP_FILE="${BACKUP_DIR}/${DB}.sql.gz"
      
      PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
        -h $POSTGRES_HOST \
        -U postgres \
        -d $DB \
        --format=plain \
        --no-owner \
        --no-acl \
        | gzip > "${BACKUP_FILE}"
      
      SIZE=$(du -h "${BACKUP_FILE}" | cut -f1)
      echo "  ✓ ${DB}: ${SIZE}"
    done
    
    # สร้าง manifest file
    echo "{" > "${BACKUP_DIR}/manifest.json"
    echo "  \"timestamp\": \"${TIMESTAMP}\"," >> "${BACKUP_DIR}/manifest.json"
    echo "  \"host\": \"${POSTGRES_HOST}\"," >> "${BACKUP_DIR}/manifest.json"
    echo "  \"databases\": [$(echo $DATABASES | sed 's/,/\", \"/g' | sed 's/^/\"/' | sed 's/$/\"/')]" >> "${BACKUP_DIR}/manifest.json"
    echo "}" >> "${BACKUP_DIR}/manifest.json"
    
    # Upload ไป S3
    if [ ! -z "${AWS_ACCESS_KEY_ID}" ]; then
      echo "Uploading to S3..."
      aws s3 sync "${BACKUP_DIR}" "s3://${S3_BUCKET}/backups/${TIMESTAMP}/" \
        --storage-class STANDARD_IA
      echo "  ✓ Uploaded to s3://${S3_BUCKET}/backups/${TIMESTAMP}/"
    fi
    
    # ลบ backup เก่า
    echo "Cleaning up old backups (> ${RETENTION_DAYS} days)..."
    find /backup -maxdepth 1 -type d -mtime "+${RETENTION_DAYS}" -exec rm -rf {} \;
    
    echo "=== Backup Completed at $(date) ==="
```

### 3. สร้าง CronJob

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
  labels:
    app: postgres-backup
    team: platform
spec:
  schedule: "0 2 * * *"       # ทุกวัน 2am
  timeZone: "Asia/Bangkok"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  startingDeadlineSeconds: 3600
  suspend: false
  
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 7200
      
      template:
        metadata:
          labels:
            app: postgres-backup
          annotations:
            "cluster-autoscaler.kubernetes.io/safe-to-evict": "false"
        spec:
          restartPolicy: OnFailure
          
          containers:
          - name: backup
            image: postgres:14
            command: ["/bin/bash", "/scripts/backup.sh"]
            
            env:
            - name: POSTGRES_HOST
              value: "postgres.production.svc.cluster.local"
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: DATABASES
              valueFrom:
                configMapKeyRef:
                  name: backup-config
                  key: databases
            - name: S3_BUCKET
              valueFrom:
                configMapKeyRef:
                  name: backup-config
                  key: s3-bucket
            - name: RETENTION_DAYS
              valueFrom:
                configMapKeyRef:
                  name: backup-config
                  key: retention-days
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access-key
                  optional: true
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret-key
                  optional: true
            
            resources:
              requests:
                cpu: "500m"
                memory: "256Mi"
              limits:
                cpu: "1000m"
                memory: "512Mi"
            
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            - name: scripts
              mountPath: /scripts
          
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          - name: scripts
            configMap:
              name: backup-config
              items:
              - key: backup.sh
                path: backup.sh
                mode: 0755
```

### 4. ทดสอบ CronJob

```bash
# Apply CronJob
kubectl apply -f backup-config.yaml
kubectl apply -f backup-cronjob.yaml

# ดู CronJob status
kubectl get cronjob -n production

# Trigger manual run
kubectl create job --from=cronjob/postgres-backup manual-backup-$(date +%s) -n production

# ดู Job ที่สร้าง
kubectl get jobs -n production

# ดู Pod logs
kubectl logs -l app=postgres-backup -n production --tail=100

# ดู Job history
kubectl get jobs -n production --sort-by=.status.startTime

# Suspend CronJob (หยุดชั่วคราว)
kubectl patch cronjob postgres-backup -n production -p '{"spec":{"suspend":true}}'

# Resume CronJob
kubectl patch cronjob postgres-backup -n production -p '{"spec":{"suspend":false}}'

# ลบ Job history เก่า
kubectl delete jobs -n production -l app=postgres-backup \
  --field-selector=status.successful=1
```

---

## Workshop: Report Generation

```yaml
# report-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-report
  namespace: analytics
spec:
  schedule: "0 8 * * MON"    # ทุกวันจันทร์ 8am
  timeZone: "Asia/Bangkok"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 4
  failedJobsHistoryLimit: 2
  
  jobTemplate:
    spec:
      backoffLimit: 1
      activeDeadlineSeconds: 3600
      
      template:
        spec:
          restartPolicy: OnFailure
          
          containers:
          - name: report-generator
            image: python:3.11-slim
            command:
            - python
            - -c
            - |
              import datetime
              import json
              import os
              
              print(f"Generating weekly report: {datetime.date.today()}")
              
              # สร้าง report data
              report = {
                  "period": "weekly",
                  "generated_at": datetime.datetime.now().isoformat(),
                  "metrics": {
                      "total_users": 1234,
                      "new_users": 45,
                      "total_revenue": 98765.50,
                      "top_products": ["Product A", "Product B"]
                  }
              }
              
              # บันทึก report
              report_path = f"/reports/weekly_{datetime.date.today()}.json"
              with open(report_path, 'w') as f:
                  json.dump(report, f, indent=2)
              
              print(f"Report saved to: {report_path}")
              
              # ส่ง email notification
              print("Sending email notification...")
              # ส่ง email ด้วย SMTP
              
              print("Report generation completed!")
            
            env:
            - name: DB_HOST
              value: "postgres.analytics.svc.cluster.local"
            - name: SMTP_HOST
              value: "smtp.company.com"
            
            resources:
              requests:
                cpu: "200m"
                memory: "256Mi"
              limits:
                cpu: "500m"
                memory: "512Mi"
            
            volumeMounts:
            - name: reports
              mountPath: /reports
          
          volumes:
          - name: reports
            persistentVolumeClaim:
              claimName: reports-pvc
```

---

## Troubleshooting Jobs

### ปัญหา: Job ไม่เสร็จ (Stuck)

```bash
# ดู Job status
kubectl describe job my-job -n batch

# ดู Pod ที่สร้าง
kubectl get pods -l job-name=my-job -n batch

# ดู logs
kubectl logs -l job-name=my-job -n batch

# ดูว่า Pod ติด CrashLoopBackOff หรือไม่
kubectl get pods -l job-name=my-job -n batch

# ดู events
kubectl get events -n batch --sort-by='.lastTimestamp' | grep my-job
```

### ปัญหา: Job รัน Pod มากเกินไป

```bash
# ตรวจสอบ backoffLimit
kubectl get job my-job -n batch -o yaml | grep backoffLimit

# ดู retry history
kubectl get pods -l job-name=my-job -n batch \
  -o custom-columns='NAME:.metadata.name,STATUS:.status.phase,RESTARTS:.status.containerStatuses[0].restartCount'

# ลด backoffLimit เพื่อหยุด retry
kubectl patch job my-job -n batch -p '{"spec":{"backoffLimit":0}}'
```

### ปัญหา: CronJob ไม่ Trigger

```bash
# ดู CronJob status
kubectl describe cronjob my-cronjob -n production

# ดู next schedule
kubectl get cronjob my-cronjob -n production -o jsonpath='{.status.nextScheduleTime}'

# ดูว่า suspend หรือไม่
kubectl get cronjob my-cronjob -n production -o jsonpath='{.spec.suspend}'

# ดู events
kubectl get events -n production | grep my-cronjob

# ตรวจสอบ timezone
kubectl get cronjob my-cronjob -n production -o yaml | grep timeZone
```

### คำสั่ง Job/CronJob ที่ใช้บ่อย

```bash
# ดู Jobs ทั้งหมด
kubectl get jobs -A

# ดู CronJobs ทั้งหมด
kubectl get cronjobs -A

# สร้าง Job จาก CronJob template
kubectl create job test-run --from=cronjob/my-cronjob -n production

# ดู logs ของ Job ทั้งหมด
kubectl logs -l job-name=my-job --all-containers -n batch

# รอ Job เสร็จ
kubectl wait --for=condition=complete job/my-job --timeout=300s -n batch

# รอ Job ล้มเหลว
kubectl wait --for=condition=failed job/my-job --timeout=300s -n batch || true

# ลบ Jobs เก่า
kubectl delete jobs -n batch --field-selector status.successful=1

# ลบ Job และ Pods ที่เกี่ยวข้อง
kubectl delete job my-job -n batch

# ดู Job ที่ running
kubectl get jobs -n batch -o custom-columns=\
'NAME:.metadata.name,COMPLETIONS:.status.completionTime,ACTIVE:.status.active,SUCCEEDED:.status.succeeded,FAILED:.status.failed'
```

---

## Best Practices

### 1. ตั้งค่า activeDeadlineSeconds เสมอ

```yaml
spec:
  activeDeadlineSeconds: 3600  # Job ต้องเสร็จใน 1 ชั่วโมง
```
ป้องกัน Job ค้างทำงานนานเกินไปและใช้ทรัพยากรโดยไม่จำเป็น

### 2. ตั้งค่า ttlSecondsAfterFinished

```yaml
spec:
  ttlSecondsAfterFinished: 3600  # ลบ Job หลัง 1 ชั่วโมง
```
ทำความสะอาด completed Jobs อัตโนมัติ

### 3. ใช้ Idempotent Operations

Job อาจรัน Pod ซ้ำกรณีล้มเหลว ดังนั้น operation ต้องทนต่อการรันซ้ำ:

```python
# ไม่ดี: INSERT ซ้ำอาจเกิด error
cursor.execute("INSERT INTO processed_records VALUES (%s)", record_id)

# ดี: INSERT OR IGNORE
cursor.execute("INSERT OR IGNORE INTO processed_records VALUES (%s)", record_id)

# หรือ: ตรวจสอบก่อน
cursor.execute("SELECT 1 FROM processed_records WHERE id = %s", record_id)
if not cursor.fetchone():
    cursor.execute("INSERT INTO processed_records VALUES (%s)", record_id)
```

### 4. ใช้ restartPolicy ที่เหมาะสม

```yaml
restartPolicy: OnFailure  # สร้าง container ใหม่บน Pod เดิม (retry ภายใน Pod)
restartPolicy: Never      # สร้าง Pod ใหม่ถ้าล้มเหลว (retry ที่ระดับ Job)
```
- **OnFailure**: เหมาะกับงานที่ idempotent และ container มีขนาดใหญ่ (ไม่ต้องการ Pod ใหม่)
- **Never**: เหมาะเมื่อต้องการ Pod ใหม่ทุกครั้ง (clean state)

### 5. CronJob Concurrency Policy

```yaml
concurrencyPolicy: Forbid   # ปลอดภัยกว่า - ป้องกัน job ซ้อนกัน
concurrencyPolicy: Allow    # อนุญาต (อาจทำให้ทรัพยากรหมด)
concurrencyPolicy: Replace  # แทนที่ job เก่าด้วย job ใหม่
```

---

## สรุป

Jobs และ CronJobs เป็นเครื่องมือสำคัญสำหรับ batch processing:

| ประเภท | ใช้เมื่อ |
|--------|---------|
| Job (completions=1) | งานครั้งเดียว, migration |
| Job (parallel) | ประมวลผลหลาย items พร้อมกัน |
| Indexed Job | แต่ละ worker ประมวลผล index เฉพาะ |
| CronJob | งานที่ต้องทำตามเวลา, backup, report |

ในบทต่อไป เราจะเรียนรู้เรื่อง **Horizontal Pod Autoscaler (HPA)** - การ scale แอปพลิเคชันอัตโนมัติ
