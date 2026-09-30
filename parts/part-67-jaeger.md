# Part 67: Jaeger - Distributed Tracing

## Distributed Tracing คืออะไร

ในระบบ Microservices Architecture, Request หนึ่งอาจผ่าน Services หลายตัวก่อนที่จะได้ Response กลับมา ถ้าเกิด Latency สูงหรือ Error เราจำเป็นต้องรู้ว่าปัญหาเกิดขึ้นที่ Service ไหน นั่นคือสิ่งที่ Distributed Tracing แก้ได้

### ปัญหาที่ Distributed Tracing แก้

```
User Request -> API Gateway -> User Service -> Order Service -> Payment Service
                                                     |
                                             Database Service
                                                     |
                                              Cache Service

ถ้า Response ช้า... ปัญหาอยู่ที่ไหน?
- API Gateway?
- User Service?
- Order Service?
- Database?

Distributed Tracing ช่วยให้เห็นว่าแต่ละ Service ใช้เวลาเท่าไหร่!
```

### Trace และ Span

- **Trace**: การเดินทางทั้งหมดของ Request ผ่าน Services ต่างๆ
- **Span**: การทำงานของแต่ละ Operation ใน Service หนึ่งๆ
- **TraceID**: ID ที่ Unique สำหรับ Trace ทั้งหมด
- **SpanID**: ID ที่ Unique สำหรับแต่ละ Span
- **ParentSpanID**: SpanID ของ Parent Operation

```
Trace (TraceID: abc123)
├── Span: API Gateway (50ms)
│   ├── Span: User Service (10ms)
│   ├── Span: Order Service (30ms)
│   │   ├── Span: Database Query (15ms)
│   │   └── Span: Cache Lookup (5ms)
│   └── Span: Payment Service (5ms)
```

---

## Jaeger Architecture

Jaeger เป็น Open Source Distributed Tracing System ที่พัฒนาโดย Uber และปัจจุบันเป็น CNCF Graduated Project

### Components ของ Jaeger

```
+------------------+     Spans     +------------------+
|  Application     | ------------> |    Jaeger        |
|  (with SDK)      |               |    Collector     |
+------------------+               +--------+---------+
                                            |
                                    +-------v--------+
                                    |    Storage     |
                                    | (Cassandra/    |
                                    |  Elasticsearch)|
                                    +-------+--------+
                                            |
                                    +-------v--------+
                                    |    Jaeger      |
                                    |    Query       |
                                    +-------+--------+
                                            |
                                    +-------v--------+
                                    |    Jaeger      |
                                    |    UI          |
                                    +----------------+
```

| Component | หน้าที่ |
|-----------|---------|
| **Jaeger Agent** | Daemon ที่ทำงานบน Node รับ Spans จาก Application |
| **Jaeger Collector** | รับ Spans และบันทึกลง Storage |
| **Jaeger Query** | API Server สำหรับ Query Traces |
| **Jaeger UI** | Web Interface แสดง Traces |
| **Storage Backend** | Cassandra, Elasticsearch, Memory |

### Jaeger Deployment Modes

**1. All-in-One (สำหรับ Development)**:
```
Application -> Jaeger All-in-One (Agent + Collector + Query + UI)
```

**2. Production Architecture**:
```
Application -> Jaeger Agent (DaemonSet) -> Kafka -> Jaeger Collector -> Elasticsearch
                                                                              |
                                                                    Jaeger Query/UI
```

---

## ติดตั้ง Jaeger

### ติดตั้ง All-in-One (Development)

```yaml
# jaeger-allinone.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "14269"
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.52
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
            - name: COLLECTOR_ZIPKIN_HOST_PORT
              value: ":9411"
          ports:
            - containerPort: 5775
              protocol: UDP
              name: zk-compact-trft
            - containerPort: 5778
              name: config-rest
            - containerPort: 6831
              protocol: UDP
              name: jg-compact-trft
            - containerPort: 6832
              protocol: UDP
              name: jg-binary-trft
            - containerPort: 9411
              name: zipkin
            - containerPort: 14268
              name: jaeger-collector-http
            - containerPort: 14269
              name: admin-http
            - containerPort: 14250
              name: grpc
            - containerPort: 16686
              name: query-http
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /
              port: 14269
            initialDelaySeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 14269
            initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: observability
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: jaeger-collector-tchannel
      port: 14267
      targetPort: 14267
      protocol: TCP
    - name: jaeger-collector-http
      port: 14268
      targetPort: 14268
      protocol: TCP
    - name: jaeger-collector-grpc
      port: 14250
      targetPort: 14250
      protocol: TCP
    - name: otlp-grpc
      port: 4317
      targetPort: 4317
    - name: otlp-http
      port: 4318
      targetPort: 4318
    - name: zipkin
      port: 9411
      targetPort: 9411
      protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: observability
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: query-http
      port: 16686
      targetPort: 16686
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-agent
  namespace: observability
  labels:
    app: jaeger
spec:
  selector:
    app: jaeger
  ports:
    - name: zk-compact-trft
      port: 5775
      protocol: UDP
    - name: jg-compact-trft
      port: 6831
      protocol: UDP
    - name: jg-binary-trft
      port: 6832
      protocol: UDP
    - name: config-rest
      port: 5778
  clusterIP: None
```

### ติดตั้งด้วย Jaeger Operator

```bash
# ติดตั้ง Jaeger Operator
kubectl create namespace observability

# ติดตั้ง cert-manager (ถ้ายังไม่มี)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

# ติดตั้ง Jaeger Operator
kubectl apply -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.52.0/jaeger-operator.yaml \
  -n observability

# รอให้ Operator Ready
kubectl wait --for=condition=Available deployment/jaeger-operator \
  -n observability --timeout=300s

# สร้าง Jaeger Instance
kubectl apply -f - << 'EOF'
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: simple-jaeger
  namespace: observability
spec:
  strategy: allInOne
  allInOne:
    image: jaegertracing/all-in-one:latest
    options:
      log-level: debug
  storage:
    type: memory
    options:
      memory:
        max-traces: 100000
  ingress:
    enabled: false
  agent:
    strategy: DaemonSet
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ""
EOF

# รอให้ Jaeger Ready
kubectl wait --for=condition=Available deployment/simple-jaeger -n observability --timeout=300s

echo "Jaeger ติดตั้งเสร็จแล้ว!"
```

### ติดตั้งด้วย Helm

```bash
# เพิ่ม Helm Repository
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo update

# ติดตั้ง Jaeger
helm install jaeger jaegertracing/jaeger \
  --namespace observability \
  --create-namespace \
  --set allInOne.enabled=true \
  --set storage.type=memory \
  --set query.enabled=false \
  --set collector.enabled=false \
  --set agent.enabled=false

# หรือ Jaeger with Elasticsearch Backend
helm install jaeger jaegertracing/jaeger \
  --namespace observability \
  --set storage.type=elasticsearch \
  --set storage.elasticsearch.host=elasticsearch:9200 \
  --set query.enabled=true \
  --set collector.enabled=true \
  --set agent.enabled=true
```

---

## Instrument Applications

### Python Application

```python
# app.py - Python Application with OpenTelemetry

from flask import Flask, jsonify
import time
import random
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Setup OpenTelemetry
def setup_tracing(service_name: str, jaeger_endpoint: str):
    """Configure OpenTelemetry with Jaeger exporter"""
    # Create TracerProvider
    provider = TracerProvider()
    
    # Create Jaeger Exporter
    exporter = OTLPSpanExporter(endpoint=jaeger_endpoint)
    
    # Add BatchSpanProcessor
    provider.add_span_processor(BatchSpanProcessor(exporter))
    
    # Set as global TracerProvider
    trace.set_tracer_provider(provider)
    
    return trace.get_tracer(service_name)

# Initialize Flask App
app = Flask(__name__)
tracer = setup_tracing("my-service", "http://jaeger-collector:4317")

# Auto-instrument Flask
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

@app.route('/api/users/<int:user_id>')
def get_user(user_id):
    with tracer.start_as_current_span("get_user") as span:
        span.set_attribute("user.id", user_id)
        span.set_attribute("http.method", "GET")
        
        # Simulate database call
        with tracer.start_as_current_span("db.query") as db_span:
            db_span.set_attribute("db.type", "postgresql")
            db_span.set_attribute("db.statement", f"SELECT * FROM users WHERE id={user_id}")
            time.sleep(random.uniform(0.01, 0.1))  # Simulate DB latency
        
        # Simulate cache check
        with tracer.start_as_current_span("cache.get") as cache_span:
            cache_span.set_attribute("cache.type", "redis")
            cache_span.set_attribute("cache.key", f"user:{user_id}")
            time.sleep(random.uniform(0.001, 0.01))  # Simulate cache latency
        
        return jsonify({"id": user_id, "name": f"User {user_id}"})

@app.route('/api/orders')
def get_orders():
    with tracer.start_as_current_span("get_orders") as span:
        span.set_attribute("http.method", "GET")
        
        # Simulate slow query
        time.sleep(random.uniform(0.05, 0.5))
        
        return jsonify({"orders": []})

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, debug=False)
```

```dockerfile
# Dockerfile สำหรับ Python App
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 8080
CMD ["python", "app.py"]
```

```text
# requirements.txt
flask==3.0.0
opentelemetry-api==1.21.0
opentelemetry-sdk==1.21.0
opentelemetry-exporter-otlp-proto-grpc==1.21.0
opentelemetry-instrumentation-flask==0.42b0
opentelemetry-instrumentation-requests==0.42b0
```

### Go Application

```go
// main.go - Go Application with OpenTelemetry

package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/sdk/resource"
	sdktrace "go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
	"go.opentelemetry.io/otel/trace"
)

var tracer trace.Tracer

func initTracer(serviceName, jaegerEndpoint string) func(context.Context) error {
	ctx := context.Background()

	res, err := resource.New(ctx,
		resource.WithAttributes(
			semconv.ServiceName(serviceName),
			semconv.ServiceVersion("1.0.0"),
		),
	)
	if err != nil {
		log.Fatal(err)
	}

	exporter, err := otlptracegrpc.New(ctx,
		otlptracegrpc.WithEndpoint(jaegerEndpoint),
		otlptracegrpc.WithInsecure(),
	)
	if err != nil {
		log.Fatal(err)
	}

	tp := sdktrace.NewTracerProvider(
		sdktrace.WithBatcher(exporter),
		sdktrace.WithResource(res),
		sdktrace.WithSampler(sdktrace.AlwaysSample()),
	)

	otel.SetTracerProvider(tp)
	tracer = otel.Tracer(serviceName)

	return tp.Shutdown
}

func handleGetUser(w http.ResponseWriter, r *http.Request) {
	ctx, span := tracer.Start(r.Context(), "handleGetUser")
	defer span.End()

	userID := r.URL.Query().Get("id")
	span.SetAttributes(attribute.String("user.id", userID))

	// Database query
	_, dbSpan := tracer.Start(ctx, "database.query")
	dbSpan.SetAttributes(
		attribute.String("db.type", "postgresql"),
		attribute.String("db.statement", fmt.Sprintf("SELECT * FROM users WHERE id=%s", userID)),
	)
	time.Sleep(time.Duration(rand.Intn(100)+10) * time.Millisecond)
	dbSpan.End()

	// Cache lookup
	_, cacheSpan := tracer.Start(ctx, "cache.get")
	cacheSpan.SetAttributes(
		attribute.String("cache.type", "redis"),
		attribute.String("cache.key", fmt.Sprintf("user:%s", userID)),
	)
	time.Sleep(time.Duration(rand.Intn(10)+1) * time.Millisecond)
	cacheSpan.End()

	response := map[string]interface{}{
		"id":   userID,
		"name": fmt.Sprintf("User %s", userID),
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(response)
}

func main() {
	shutdown := initTracer("user-service", "jaeger-collector:4317")
	defer shutdown(context.Background())

	http.HandleFunc("/api/users", handleGetUser)
	http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		json.NewEncoder(w).Encode(map[string]string{"status": "healthy"})
	})

	log.Println("Server starting on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### Node.js Application

```javascript
// app.js - Node.js Application with OpenTelemetry

'use strict';

// Setup OpenTelemetry FIRST (before other imports)
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'order-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
  }),
  traceExporter: new OTLPTraceExporter({
    url: 'http://jaeger-collector:4317',
  }),
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

// Application Code
const express = require('express');
const { trace, context } = require('@opentelemetry/api');

const app = express();
const tracer = trace.getTracer('order-service');

app.get('/api/orders', async (req, res) => {
  const span = tracer.startSpan('getOrders');
  
  try {
    // Simulate database call
    const dbSpan = tracer.startSpan('database.query', {
      attributes: {
        'db.type': 'mongodb',
        'db.statement': 'db.orders.find({})',
      },
    }, trace.setSpan(context.active(), span));
    
    await new Promise(resolve => setTimeout(resolve, Math.random() * 100 + 20));
    dbSpan.end();
    
    span.setAttributes({
      'http.method': 'GET',
      'http.url': '/api/orders',
      'order.count': 5,
    });
    
    res.json({ orders: [], count: 5 });
  } catch (error) {
    span.recordException(error);
    span.setStatus({ code: 2, message: error.message });
    res.status(500).json({ error: error.message });
  } finally {
    span.end();
  }
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy' });
});

app.listen(8080, () => {
  console.log('Order Service running on port 8080');
});

process.on('SIGTERM', async () => {
  await sdk.shutdown();
  process.exit(0);
});
```

### Kubernetes Deployment ของ Applications

```yaml
# microservices-with-tracing.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: default
  labels:
    app: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
        - name: app
          image: myregistry/user-service:1.0
          ports:
            - containerPort: 8080
          env:
            - name: JAEGER_ENDPOINT
              value: "http://jaeger-collector.observability:4317"
            - name: SERVICE_NAME
              value: "user-service"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://jaeger-collector.observability:4317"
            - name: OTEL_SERVICE_NAME
              value: "user-service"
            - name: OTEL_TRACES_SAMPLER
              value: "always_on"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: default
  labels:
    app: order-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: app
          image: myregistry/order-service:1.0
          ports:
            - containerPort: 8080
          env:
            - name: USER_SERVICE_URL
              value: "http://user-service:8080"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://jaeger-collector.observability:4317"
            - name: OTEL_SERVICE_NAME
              value: "order-service"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

---

## Workshop: Trace Requests ผ่าน Microservices

### เป้าหมาย

ในส่วนนี้เราจะ:
1. Deploy Jaeger บน Kubernetes
2. Deploy Application ที่มี Tracing
3. ส่ง Requests และดู Traces
4. วิเคราะห์ Latency ด้วย Jaeger UI
5. ค้นหา Bottlenecks

### ขั้นตอนที่ 1: Deploy Jaeger All-in-One

```bash
# สร้าง Namespace
kubectl create namespace observability

# Deploy Jaeger All-in-One
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
  labels:
    app: jaeger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/all-in-one:1.52
          env:
            - name: COLLECTOR_OTLP_ENABLED
              value: "true"
          ports:
            - containerPort: 16686
              name: query-http
            - containerPort: 14268
              name: collector-http
            - containerPort: 4317
              name: otlp-grpc
            - containerPort: 4318
              name: otlp-http
            - containerPort: 6831
              protocol: UDP
              name: agent-compact
          resources:
            requests:
              memory: "256Mi"
              cpu: "200m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-collector
  namespace: observability
spec:
  selector:
    app: jaeger
  ports:
    - name: otlp-grpc
      port: 4317
    - name: otlp-http
      port: 4318
    - name: collector-http
      port: 14268
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger-query
  namespace: observability
spec:
  selector:
    app: jaeger
  ports:
    - name: query-http
      port: 16686
  type: ClusterIP
EOF

# รอ Jaeger Ready
kubectl wait --for=condition=Ready pod -l app=jaeger -n observability --timeout=120s
echo "Jaeger is ready!"
```

### ขั้นตอนที่ 2: Deploy Demo Microservices

```yaml
# hotrod-demo.yaml - Jaeger HotROD Demo Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hotrod
  namespace: observability
  labels:
    app: hotrod
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hotrod
  template:
    metadata:
      labels:
        app: hotrod
    spec:
      containers:
        - name: hotrod
          image: jaegertracing/example-hotrod:1.52
          args:
            - all
          env:
            - name: JAEGER_AGENT_HOST
              value: "jaeger-collector.observability"
            - name: JAEGER_AGENT_PORT
              value: "6831"
          ports:
            - containerPort: 8080
              name: frontend
            - containerPort: 8081
              name: customer
            - containerPort: 8082
              name: driver
            - containerPort: 8083
              name: route
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: hotrod
  namespace: observability
spec:
  selector:
    app: hotrod
  ports:
    - name: frontend
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

```bash
# Apply HotROD Demo
kubectl apply -f hotrod-demo.yaml

# รอ HotROD Ready
kubectl wait --for=condition=Ready pod -l app=hotrod -n observability --timeout=120s
```

### ขั้นตอนที่ 3: ส่ง Requests เพื่อสร้าง Traces

```bash
# Port Forward HotROD
kubectl port-forward -n observability svc/hotrod 8080:8080 &
sleep 2

# Port Forward Jaeger UI
kubectl port-forward -n observability svc/jaeger-query 16686:16686 &
sleep 2

echo "HotROD Demo: http://localhost:8080"
echo "Jaeger UI: http://localhost:16686"

# ส่ง Requests ไปยัง HotROD
for i in {1..20}; do
    # ส่ง Request ไปขอรถ
    curl -s "http://localhost:8080/dispatch?customer=123&nonse=$RANDOM" &
    sleep 0.5
done

wait
echo "ส่ง Requests เสร็จแล้ว!"
```

### ขั้นตอนที่ 4: ดู Traces ใน Jaeger UI

```bash
# เปิด Jaeger UI
echo "เปิด Jaeger UI ที่ http://localhost:16686"

# ดู Services ที่ Trace
curl -s "http://localhost:16686/api/services" | python3 -m json.tool

# ดู Traces ล่าสุด
curl -s "http://localhost:16686/api/traces?service=frontend&limit=5" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for trace in data.get('data', []):
    trace_id = trace.get('traceID')
    spans = trace.get('spans', [])
    duration = max(s.get('duration', 0) for s in spans) / 1000  # microseconds to ms
    print(f'TraceID: {trace_id[:16]}... | Spans: {len(spans)} | Duration: {duration:.2f}ms')
"
```

### ขั้นตอนที่ 5: Query Traces ด้วย API

```bash
# ค้นหา Slow Traces (มากกว่า 500ms)
curl -s "http://localhost:16686/api/traces?service=frontend&minDuration=500ms&limit=10" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
traces = data.get('data', [])
print(f'Found {len(traces)} slow traces')
for trace in traces:
    spans = trace.get('spans', [])
    duration = max(s.get('duration', 0) for s in spans) / 1000
    trace_id = trace.get('traceID')
    print(f'  TraceID: {trace_id} | Duration: {duration:.2f}ms')
"

# ดู Operations ที่มีอยู่
curl -s "http://localhost:16686/api/operations?service=frontend" | python3 -m json.tool

# ดู Trace ที่ Specific
TRACE_ID="your-trace-id"
curl -s "http://localhost:16686/api/traces/$TRACE_ID" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
trace = data.get('data', [{}])[0]
spans = trace.get('spans', [])
print(f'Trace has {len(spans)} spans:')
for span in sorted(spans, key=lambda x: x.get('startTime', 0)):
    name = span.get('operationName', 'unknown')
    duration = span.get('duration', 0) / 1000
    tags = {t['key']: t['value'] for t in span.get('tags', [])}
    print(f'  {name}: {duration:.2f}ms')
"
```

### ขั้นตอนที่ 6: สร้าง Simple Tracing Demo

```yaml
# simple-tracing-demo.yaml
# สร้าง Application ง่ายๆ ที่ส่ง Traces ไปยัง Jaeger
apiVersion: apps/v1
kind: Deployment
metadata:
  name: trace-demo
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: trace-demo
  template:
    metadata:
      labels:
        app: trace-demo
    spec:
      containers:
        - name: app
          image: python:3.11-slim
          command:
            - sh
            - -c
            - |
              pip install -q opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-grpc
              
              python3 << 'PYTHON_EOF'
              import time
              import random
              from opentelemetry import trace
              from opentelemetry.sdk.trace import TracerProvider
              from opentelemetry.sdk.trace.export import BatchSpanProcessor
              from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
              from opentelemetry.sdk.resources import Resource
              
              # Setup
              resource = Resource.create({"service.name": "trace-demo-service"})
              provider = TracerProvider(resource=resource)
              exporter = OTLPSpanExporter(endpoint="http://jaeger-collector:4317", insecure=True)
              provider.add_span_processor(BatchSpanProcessor(exporter))
              trace.set_tracer_provider(provider)
              tracer = trace.get_tracer("trace-demo")
              
              print("Starting trace generation...")
              
              while True:
                  # Simulate a request
                  with tracer.start_as_current_span("handle-request") as span:
                      span.set_attribute("request.id", random.randint(1000, 9999))
                      
                      # DB query
                      with tracer.start_as_current_span("db-query") as db_span:
                          db_span.set_attribute("db.system", "postgresql")
                          db_span.set_attribute("db.name", "mydb")
                          time.sleep(random.uniform(0.01, 0.1))
                      
                      # Cache
                      with tracer.start_as_current_span("cache-lookup") as cache_span:
                          cache_span.set_attribute("cache.system", "redis")
                          time.sleep(random.uniform(0.001, 0.01))
                      
                      # External API
                      with tracer.start_as_current_span("external-api") as api_span:
                          api_span.set_attribute("http.method", "GET")
                          api_span.set_attribute("http.url", "https://api.example.com/data")
                          
                          # Random error
                          if random.random() < 0.1:
                              api_span.set_attribute("error", True)
                              api_span.set_attribute("error.message", "Connection timeout")
                          
                          time.sleep(random.uniform(0.05, 0.3))
                      
                      time.sleep(random.uniform(0.01, 0.05))
                  
                  time.sleep(1)
              PYTHON_EOF
          env:
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: "http://jaeger-collector:4317"
          resources:
            requests:
              memory: "128Mi"
              cpu: "50m"
            limits:
              memory: "256Mi"
              cpu: "200m"
```

```bash
# Apply Demo
kubectl apply -f simple-tracing-demo.yaml

# ดู Logs
kubectl logs -n observability deployment/trace-demo -f

# รอ 30 วินาที แล้วดู Traces ใน Jaeger UI
echo "ดู Traces ที่ http://localhost:16686"
echo "Service: trace-demo-service"
```

### ขั้นตอนที่ 7: วิเคราะห์ Traces

```bash
# Script วิเคราะห์ Traces
cat > /tmp/analyze-traces.sh << 'ANALYSIS_EOF'
#!/bin/bash
JAEGER_URL="${1:-http://localhost:16686}"
SERVICE="${2:-trace-demo-service}"

echo "=== Trace Analysis for Service: $SERVICE ==="
echo ""

# ดู Traces ล่าสุด
echo "--- Recent Traces ---"
curl -s "$JAEGER_URL/api/traces?service=$SERVICE&limit=20" | \
  python3 -c "
import sys, json

data = json.load(sys.stdin)
traces = data.get('data', [])
print(f'Total traces: {len(traces)}')
print()

durations = []
for trace in traces:
    spans = trace.get('spans', [])
    if spans:
        max_duration = max(s.get('duration', 0) for s in spans)
        durations.append(max_duration / 1000)  # microseconds to ms

if durations:
    durations.sort()
    n = len(durations)
    print(f'Latency Statistics:')
    print(f'  Min: {min(durations):.2f}ms')
    print(f'  Max: {max(durations):.2f}ms')
    print(f'  Avg: {sum(durations)/n:.2f}ms')
    print(f'  P50: {durations[int(n*0.5)]:.2f}ms')
    print(f'  P95: {durations[int(n*0.95)]:.2f}ms')
    print(f'  P99: {durations[int(n*0.99)]:.2f}ms')
"

echo ""
echo "--- Operations ---"
curl -s "$JAEGER_URL/api/operations?service=$SERVICE" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
ops = data.get('data', [])
print(f'Operations ({len(ops)}):')
for op in ops:
    print(f'  - {op}')
"
ANALYSIS_EOF

chmod +x /tmp/analyze-traces.sh
/tmp/analyze-traces.sh
```

### ขั้นตอนที่ 8: Integration กับ Prometheus

```yaml
# jaeger-prometheus-scrape.yaml
# ตั้งค่าให้ Prometheus Scrape Metrics จาก Jaeger
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: jaeger
  namespace: observability
  labels:
    release: prometheus
spec:
  namespaceSelector:
    matchNames:
      - observability
  selector:
    matchLabels:
      app: jaeger
  endpoints:
    - port: admin-http
      path: /metrics
      interval: 30s
```

```promql
# PromQL Queries สำหรับ Jaeger Metrics

# จำนวน Spans ที่รับต่อวินาที
rate(jaeger_collector_spans_received_total[5m])

# จำนวน Traces ที่เก็บ
jaeger_collector_traces_saved_by_svc_total

# Span Processing Latency
histogram_quantile(0.99, rate(jaeger_collector_queue_length[5m]))

# Error Spans
rate(jaeger_collector_spans_rejected_total[5m])
```

### ขั้นตอนที่ 9: Production Jaeger Setup

```yaml
# jaeger-production.yaml
# Production Setup ด้วย Elasticsearch Backend
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: production-jaeger
  namespace: observability
spec:
  strategy: production
  
  collector:
    replicas: 2
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
      limits:
        memory: "1Gi"
        cpu: "500m"
    options:
      collector:
        num-workers: 50
        queue-size: 2000
  
  query:
    replicas: 2
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "512Mi"
        cpu: "300m"
  
  agent:
    strategy: DaemonSet
    resources:
      requests:
        memory: "64Mi"
        cpu: "10m"
      limits:
        memory: "128Mi"
        cpu: "100m"
  
  storage:
    type: elasticsearch
    options:
      es:
        server-urls: http://elasticsearch:9200
        index-prefix: jaeger
        num-shards: 5
        num-replicas: 1
    esIndexCleaner:
      enabled: true
      numberOfDays: 7
      schedule: "55 23 * * *"
    
  sampling:
    options:
      default_strategy:
        type: probabilistic
        param: 0.1  # 10% Sampling Rate
      service_strategies:
        - service: critical-service
          type: probabilistic
          param: 1.0  # 100% Sampling สำหรับ Critical Service
```

### ขั้นตอนที่ 10: ทำความสะอาด

```bash
# ลบ Resources
kubectl delete -f hotrod-demo.yaml
kubectl delete -f simple-tracing-demo.yaml
kubectl delete namespace observability

# Stop Port Forwards
kill $(lsof -ti:16686) 2>/dev/null || true
kill $(lsof -ti:8080) 2>/dev/null || true

echo "Cleanup เสร็จสิ้น"
```

---

## Tips และ Best Practices

### 1. Sampling Strategy

```yaml
# สำหรับ Production ไม่ควร Sample 100%
# ใช้ Sampling Rate ที่เหมาะสม:
- Development: 100% (Always sample)
- Staging: 50%
- Production: 10% หรือน้อยกว่า

# ใช้ Adaptive Sampling สำหรับ Services ที่ Traffic สูง
default_strategy:
  type: adaptive
  max_traces_per_second: 100
```

### 2. Context Propagation

```python
# ส่ง TraceID ผ่าน HTTP Headers
headers = {}
inject(headers)  # OpenTelemetry จะเพิ่ม Trace Headers
# Headers: traceparent, tracestate

# ตัวอย่าง Header: 
# traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

### 3. Resource Attributes

```python
# เพิ่ม Resource Attributes ที่ Useful
from opentelemetry.sdk.resources import Resource, SCHEMA_URL

resource = Resource.create({
    "service.name": "my-service",
    "service.version": "1.2.3",
    "service.environment": "production",
    "k8s.namespace.name": "production",
    "k8s.pod.name": os.environ.get("POD_NAME", "unknown"),
    "k8s.node.name": os.environ.get("NODE_NAME", "unknown"),
})
```

---

## สรุป

Distributed Tracing ด้วย Jaeger ช่วยให้เราเข้าใจการทำงานของ Microservices:

1. **Traces และ Spans**: เข้าใจ Flow ของ Request ผ่าน Services
2. **Jaeger UI**: Visualize Traces และหา Bottlenecks
3. **OpenTelemetry**: Standard SDK สำหรับ Instrument Applications
4. **Sampling**: เลือก Sampling Rate ที่เหมาะสม
5. **Integration**: ทำงานร่วมกับ Prometheus และ Grafana

ในส่วนต่อไป เราจะเรียนรู้เกี่ยวกับ Liveness Probes ซึ่งเป็นกลไกสำคัญที่ Kubernetes ใช้ตรวจสอบสุขภาพของ Containers
