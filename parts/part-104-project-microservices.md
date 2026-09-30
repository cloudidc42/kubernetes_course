# Part 104: โปรเจค Microservices Architecture บน Kubernetes

## บทนำ

บทนี้นำเสนอการออกแบบและ Implement Microservices Architecture ที่สมบูรณ์บน Kubernetes รวมถึง Service Mesh (Istio), Distributed Tracing, Event-Driven Architecture, และ API Gateway

## สถาปัตยกรรม

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Applications                       │
│              Web | Mobile | Partner API                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                     API Gateway (Kong)                       │
│           Rate Limiting | Auth | SSL Termination            │
└─────┬──────────┬──────────┬──────────┬──────────────────────┘
      │          │          │          │
┌─────▼──┐ ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
│ Auth   │ │Payment │ │Catalog │ │ User   │
│ Service│ │Service │ │Service │ │Service │
└─────┬──┘ └────┬───┘ └────┬───┘ └────┬───┘
      │          │          │          │
      └──────────┴──────────┴──────────┘
                      │
         ┌────────────▼────────────┐
         │    Service Mesh (Istio) │
         │  mTLS | Circuit Breaker │
         │  Retry | Load Balancing │
         └─────────────────────────┘
                      │
    ┌─────────────────┼─────────────────┐
    │                 │                 │
┌───▼────┐  ┌─────────▼──────┐  ┌──────▼─────┐
│Event   │  │ Distributed    │  │Config      │
│Bus     │  │ Cache (Redis)  │  │Service     │
│(Kafka) │  └────────────────┘  └────────────┘
└───┬────┘
    │
┌───▼────────────────────────────────────┐
│          Event Consumers               │
│  Notification | Analytics | Audit      │
└────────────────────────────────────────┘

Observability Stack:
┌──────────────────────────────────────────────────┐
│  Jaeger (Tracing) | Prometheus | Grafana | Loki  │
└──────────────────────────────────────────────────┘
```

---

## Step 1: ติดตั้ง Service Mesh (Istio)

```bash
# ติดตั้ง istioctl
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/istio-1.20.0/bin:$PATH

# Install Istio
istioctl install --set profile=production -y

# Enable Injection สำหรับ Namespace
kubectl label namespace backend istio-injection=enabled

# ตรวจสอบ
istioctl verify-install
kubectl get pods -n istio-system
```

### 1.1 Istio Traffic Management

```yaml
# VirtualService - Traffic Routing
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: catalog-service
  namespace: backend
spec:
  hosts:
  - catalog-service
  http:
  - match:
    - headers:
        x-version:
          exact: v2
    route:
    - destination:
        host: catalog-service
        subset: v2
  - route:
    - destination:
        host: catalog-service
        subset: v1
      weight: 90
    - destination:
        host: catalog-service
        subset: v2
      weight: 10
---
# DestinationRule - Subsets และ Load Balancing
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: catalog-service
  namespace: backend
spec:
  host: catalog-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http2MaxRequests: 1000
        maxRequestsPerConnection: 10
    loadBalancer:
      simple: LEAST_REQUEST
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
  - name: v1
    labels:
      version: v1
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v2
    labels:
      version: v2
```

### 1.2 Circuit Breaker

```yaml
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-circuit-breaker
  namespace: backend
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 50
      http:
        http1MaxPendingRequests: 100
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 100
      minHealthPercent: 0
```

### 1.3 Retry Policy

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-vs
spec:
  hosts:
  - payment-service
  http:
  - retries:
      attempts: 3
      perTryTimeout: 10s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"
      retryRemoteLocalities: true
    timeout: 30s
    route:
    - destination:
        host: payment-service
```

### 1.4 mTLS Policy

```yaml
# Require mTLS ทั้ง Namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: backend
spec:
  mtls:
    mode: STRICT
---
# Allow specific service with PERMISSIVE
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: payment-service-policy
  namespace: backend
spec:
  selector:
    matchLabels:
      app: payment-service
  mtls:
    mode: PERMISSIVE
  portLevelMtls:
    8443:
      mode: STRICT
```

---

## Step 2: API Gateway (Kong)

```bash
# ติดตั้ง Kong
helm repo add kong https://charts.konghq.com
helm install kong kong/kong \
  -n kong \
  --create-namespace \
  --set ingressController.installCRDs=false \
  --set postgresql.enabled=true \
  --set env.database=postgres
```

```yaml
# KongPlugin: Rate Limiting
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting
  namespace: backend
config:
  minute: 100
  hour: 1000
  policy: local
  hide_client_headers: false
  error_message: "API rate limit exceeded. Try again later."
plugin: rate-limiting
---
# KongPlugin: JWT Authentication
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-auth
  namespace: backend
config:
  key_claim_name: iss
  claims_to_verify:
  - exp
plugin: jwt
---
# KongPlugin: Request Transformer
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-transformer
config:
  add:
    headers:
    - "x-request-id:$(uuid)"
    - "x-service-version:1.0.0"
plugin: request-transformer
---
# Ingress สำหรับ API Gateway
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-gateway
  namespace: backend
  annotations:
    konghq.com/plugins: "rate-limiting,jwt-auth,request-transformer"
    konghq.com/strip-path: "true"
    konghq.com/protocols: "https"
spec:
  ingressClassName: kong
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1/catalog
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 80
      - path: /v1/payment
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
      - path: /v1/auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 80
```

---

## Step 3: Event-Driven Architecture (Kafka)

```bash
# ติดตั้ง Kafka
helm repo add bitnami https://charts.bitnami.com/bitnami

helm install kafka bitnami/kafka \
  -n infrastructure \
  --set replicaCount=3 \
  --set zookeeper.replicaCount=3 \
  --set persistence.size=20Gi \
  --set metrics.kafka.enabled=true \
  --set metrics.jmx.enabled=true
```

### 3.1 Kafka Topics

```bash
# สร้าง Topics
kubectl exec -n infrastructure kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic order.created \
  --partitions 10 \
  --replication-factor 3

kubectl exec -n infrastructure kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic payment.processed \
  --partitions 10 \
  --replication-factor 3

kubectl exec -n infrastructure kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic notification.send \
  --partitions 5 \
  --replication-factor 3

kubectl exec -n infrastructure kafka-0 -- kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic analytics.events \
  --partitions 20 \
  --replication-factor 3 \
  --config retention.ms=604800000  # 7 days
```

### 3.2 Event Consumer Service

```yaml
# notification-consumer/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-consumer
  namespace: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: notification-consumer
  template:
    metadata:
      labels:
        app: notification-consumer
    spec:
      containers:
      - name: consumer
        image: company-registry.io/notification-consumer:1.0.0
        env:
        - name: KAFKA_BROKERS
          value: kafka.infrastructure.svc.cluster.local:9092
        - name: KAFKA_GROUP_ID
          value: notification-group
        - name: KAFKA_TOPICS
          value: "order.created,payment.processed"
        - name: KAFKA_AUTO_OFFSET_RESET
          value: earliest
        - name: SMTP_HOST
          value: smtp.sendgrid.net
        - name: SMTP_PASSWORD
          valueFrom:
            secretKeyRef:
              name: smtp-credentials
              key: password
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### 3.3 KEDA สำหรับ Kafka Consumer Autoscaling

```yaml
# KEDA ScaledObject สำหรับ Kafka
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-consumer-scaler
  namespace: backend
spec:
  scaleTargetRef:
    name: notification-consumer
  pollingInterval: 15
  cooldownPeriod: 30
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka.infrastructure.svc.cluster.local:9092
      consumerGroup: notification-group
      topic: order.created
      lagThreshold: "100"
      offsetResetPolicy: earliest
```

---

## Step 4: Distributed Tracing (Jaeger)

```bash
# ติดตั้ง Jaeger
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts

helm install jaeger jaegertracing/jaeger \
  -n monitoring \
  --set provisionDataStore.cassandra=false \
  --set allInOne.enabled=true \
  --set storage.type=elasticsearch \
  --set elasticsearch.externalDatabase.host=elasticsearch-master.infrastructure
```

### 4.1 Istio Tracing Config

```yaml
# เปิด Tracing ใน Istio
apiVersion: v1
kind: ConfigMap
metadata:
  name: istio-mesh-config
  namespace: istio-system
data:
  mesh: |
    enableTracing: true
    defaultConfig:
      tracing:
        sampling: 100.0
        zipkin:
          address: jaeger-collector.monitoring.svc.cluster.local:9411
```

### 4.2 OpenTelemetry Instrumentation

```javascript
// Node.js Service - Tracing Setup
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-http');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');
const { PgInstrumentation } = require('@opentelemetry/instrumentation-pg');

const sdk = new NodeSDK({
  resource: new Resource({
    [SemanticResourceAttributes.SERVICE_NAME]: 'catalog-service',
    [SemanticResourceAttributes.SERVICE_VERSION]: '1.0.0',
    [SemanticResourceAttributes.DEPLOYMENT_ENVIRONMENT]: 'production',
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT || 'http://jaeger-collector:4318/v1/traces',
  }),
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
    new PgInstrumentation(),
  ],
});

sdk.start();
```

---

## Step 5: Service-to-Service Communication Patterns

### 5.1 Synchronous (REST/gRPC)

```yaml
# gRPC Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: backend
spec:
  template:
    spec:
      containers:
      - name: catalog
        image: company-registry.io/catalog-service:1.0.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: grpc
        - containerPort: 9091
          name: metrics
---
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
  namespace: backend
spec:
  selector:
    app: catalog-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090
    appProtocol: grpc
  - name: metrics
    port: 9091
    targetPort: 9091
```

### 5.2 Async (Event-Driven)

```python
# Python Kafka Producer Example
import json
import time
from kafka import KafkaProducer

class OrderEventProducer:
    def __init__(self, bootstrap_servers):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8'),
            acks='all',
            retries=3,
            max_in_flight_requests_per_connection=1
        )
    
    def publish_order_created(self, order):
        event = {
            'event_type': 'order.created',
            'event_id': str(uuid.uuid4()),
            'timestamp': time.time(),
            'data': {
                'order_id': order['id'],
                'user_id': order['user_id'],
                'total_amount': order['total'],
                'items': order['items'],
                'status': 'pending'
            }
        }
        
        future = self.producer.send(
            'order.created',
            key=str(order['id']),
            value=event
        )
        result = future.get(timeout=10)
        return result
```

---

## Step 6: Config Service

```yaml
# Config Service สำหรับ Dynamic Configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: config-service
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: config-service
  template:
    spec:
      containers:
      - name: config-service
        image: company-registry.io/config-service:1.0.0
        ports:
        - containerPort: 8888
        env:
        - name: SPRING_CLOUD_CONFIG_SERVER_GIT_URI
          value: https://github.com/company/config-repo
        - name: SPRING_CLOUD_CONFIG_SERVER_GIT_SEARCHPATHS
          value: "microservices/{application}"
        resources:
          requests:
            cpu: 200m
            memory: 512Mi
          limits:
            cpu: 500m
            memory: 1Gi
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8888
          initialDelaySeconds: 30
          periodSeconds: 10
```

---

## Step 7: Advanced Deployment Patterns

### 7.1 Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

DEPLOYMENT=$1
IMAGE=$2
NAMESPACE=${3:-backend}

# ตรวจสอบ Active Version
CURRENT=$(kubectl get svc $DEPLOYMENT -n $NAMESPACE -o jsonpath='{.spec.selector.version}')
echo "Current active version: $CURRENT"

if [ "$CURRENT" == "blue" ]; then
  NEW_VERSION="green"
  OLD_VERSION="blue"
else
  NEW_VERSION="blue"
  OLD_VERSION="green"
fi

echo "Deploying $NEW_VERSION..."

# Deploy ไปยัง Inactive Version
kubectl set image deployment/$DEPLOYMENT-$NEW_VERSION \
  app=$IMAGE \
  -n $NAMESPACE

# Wait for Rollout
kubectl rollout status deployment/$DEPLOYMENT-$NEW_VERSION \
  -n $NAMESPACE \
  --timeout=5m

# Run Smoke Tests
echo "Running smoke tests..."
kubectl run smoke-test \
  --image=curlimages/curl \
  --rm \
  -it \
  -n $NAMESPACE \
  -- curl -f http://$DEPLOYMENT-$NEW_VERSION/health

if [ $? -ne 0 ]; then
  echo "Smoke tests failed! Aborting."
  exit 1
fi

# Switch Traffic
echo "Switching traffic to $NEW_VERSION..."
kubectl patch svc $DEPLOYMENT \
  -n $NAMESPACE \
  -p "{\"spec\":{\"selector\":{\"version\":\"$NEW_VERSION\"}}}"

echo "Traffic switched to $NEW_VERSION"
echo "Previous version ($OLD_VERSION) still running for rollback"
echo "To rollback: kubectl patch svc $DEPLOYMENT -n $NAMESPACE -p '{\"spec\":{\"selector\":{\"version\":\"$OLD_VERSION\"}}}'"
```

### 7.2 Feature Flags

```yaml
# ConfigMap สำหรับ Feature Flags
apiVersion: v1
kind: ConfigMap
metadata:
  name: feature-flags
  namespace: backend
data:
  NEW_CHECKOUT_FLOW: "true"
  ENHANCED_SEARCH: "false"
  PAYMENT_V2: "true"
  LOYALTY_POINTS: "false"
  DARK_MODE: "true"
---
# Pod ที่ Mount Feature Flags
spec:
  containers:
  - name: app
    volumeMounts:
    - name: feature-flags
      mountPath: /etc/feature-flags
  volumes:
  - name: feature-flags
    configMap:
      name: feature-flags
```

---

## Step 8: Observability Stack

### 8.1 Distributed Tracing Dashboard

```yaml
# Grafana Dashboard สำหรับ Jaeger
apiVersion: v1
kind: ConfigMap
metadata:
  name: jaeger-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  jaeger.json: |
    {
      "title": "Distributed Tracing",
      "panels": [
        {
          "title": "Request Duration P99",
          "type": "graph",
          "targets": [
            {
              "datasource": "Jaeger",
              "query": "service=catalog-service"
            }
          ]
        }
      ]
    }
```

### 8.2 Service Mesh Metrics

```yaml
# PrometheusRule สำหรับ Istio
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: istio-service-rules
  namespace: monitoring
spec:
  groups:
  - name: istio.service
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(istio_requests_total{
          reporter="destination",
          response_code=~"5.*"
        }[5m])) by (destination_service_name)
        /
        sum(rate(istio_requests_total{
          reporter="destination"
        }[5m])) by (destination_service_name)
        > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate in {{ $labels.destination_service_name }}"
        description: "Error rate is {{ $value | humanizePercentage }}"

    - alert: HighLatency
      expr: |
        histogram_quantile(0.99,
          sum(rate(istio_request_duration_milliseconds_bucket{
            reporter="destination"
          }[5m])) by (le, destination_service_name)
        ) > 2000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High latency in {{ $labels.destination_service_name }}"
        description: "P99 latency is {{ $value }}ms"

    - alert: CircuitBreakerOpen
      expr: |
        istio_tcp_connections_closed_total{
          response_flags="UO"
        } > 0
      for: 1m
      labels:
        severity: warning
      annotations:
        summary: "Circuit breaker opened for a service"
```

---

## Step 9: Testing Microservices

### 9.1 Contract Testing

```javascript
// Consumer Contract Test (Node.js + Pact)
const { Pact } = require('@pact-foundation/pact');
const { like, term } = require('@pact-foundation/pact').Matchers;

const provider = new Pact({
  consumer: 'order-service',
  provider: 'catalog-service',
  port: 1234,
  log: path.resolve(process.cwd(), 'logs', 'pact.log'),
  dir: path.resolve(process.cwd(), 'pacts'),
});

describe('Catalog Service - Get Product', () => {
  before(() => provider.setup());
  after(() => provider.finalize());

  it('returns a product by id', () => {
    provider.addInteraction({
      state: 'product exists',
      uponReceiving: 'a request for product',
      withRequest: {
        method: 'GET',
        path: term({
          generate: '/v1/products/123',
          matcher: '/v1/products/\\d+'
        }),
      },
      willRespondWith: {
        status: 200,
        body: {
          id: like('123'),
          name: like('Product Name'),
          price: like(99.99),
          stock: like(100),
        },
      },
    });
  });
});
```

### 9.2 Chaos Engineering

```yaml
# Chaos Mesh - Network Delay
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: catalog-delay
  namespace: backend
spec:
  action: delay
  mode: random
  selector:
    namespaces: ["backend"]
    labelSelectors:
      app: catalog-service
  delay:
    latency: "200ms"
    correlation: "100"
    jitter: "50ms"
  direction: to
  duration: "1m"
---
# Chaos Mesh - Pod Kill
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: payment-pod-kill
  namespace: backend
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces: ["backend"]
    labelSelectors:
      app: payment-service
  duration: "30s"
  scheduler:
    cron: "@every 10m"
```

---

## Workshop: Deploy Complete Microservices System

```bash
#!/bin/bash
set -e

echo "=== Deploying Microservices Platform ==="

# 1. Namespaces
kubectl create namespace backend --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace infrastructure --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

# 2. Enable Istio
kubectl label namespace backend istio-injection=enabled

# 3. Infrastructure
helm upgrade --install kafka bitnami/kafka -n infrastructure --wait
helm upgrade --install redis bitnami/redis -n infrastructure --wait
helm upgrade --install postgresql bitnami/postgresql -n infrastructure --wait

# 4. Services
for svc in catalog auth payment order user notification; do
  kubectl apply -f services/$svc/ -n backend
done

# 5. API Gateway
kubectl apply -f api-gateway/

# 6. Istio Config
kubectl apply -f istio/

# 7. Monitoring
kubectl apply -f monitoring/

echo "=== Deployment Complete ==="

# Status
kubectl get pods -n backend
kubectl get vs,dr -n backend  # Virtual Services & Destination Rules
```

### Verification Steps

```bash
# ตรวจสอบ mTLS
istioctl x describe service catalog-service.backend.svc.cluster.local

# ตรวจสอบ Traffic
kubectl exec -n backend -it deployment/catalog-service -- curl -v http://payment-service/health

# ดู Jaeger Traces
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686

# ดู Kiali Dashboard
kubectl port-forward -n istio-system svc/kiali 20001:20001
```

---

## สรุป

Microservices Architecture นี้ประกอบด้วย:

1. **Service Mesh (Istio)** - Traffic Management, mTLS, Circuit Breaking
2. **API Gateway (Kong)** - Rate Limiting, Authentication, Routing
3. **Event Bus (Kafka)** - Async Communication, Event Sourcing
4. **Distributed Tracing (Jaeger)** - End-to-End Visibility
5. **Advanced Deployments** - Blue-Green, Canary, Feature Flags
6. **Chaos Engineering** - Testing Resilience

## References

- [Istio Documentation](https://istio.io/docs/)
- [Kong Documentation](https://docs.konghq.com/)
- [Kafka on Kubernetes](https://kafka.apache.org/documentation/)
- [Jaeger Tracing](https://www.jaegertracing.io/)
- [Chaos Mesh](https://chaos-mesh.org/)
