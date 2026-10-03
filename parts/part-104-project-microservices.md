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

---

## Complete YAML Manifests - ทุก Microservice

### Auth Service - YAML ครบ

```yaml
# auth-service.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: microservices
  labels:
    istio-injection: enabled
---
# Auth Service ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
  namespace: microservices
data:
  APP_PORT: "8080"
  LOG_LEVEL: "info"
  JWT_EXPIRY: "3600"
  REFRESH_TOKEN_EXPIRY: "604800"
  REDIS_HOST: "redis.microservices.svc.cluster.local"
  REDIS_PORT: "6379"
---
# Auth Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: microservices
  labels:
    app: auth-service
    version: v1
    tier: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: auth-service
      version: v1
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: auth-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: auth-service-sa
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: auth-service
              topologyKey: kubernetes.io/hostname
      containers:
      - name: auth-service
        image: myregistry/auth-service:v1.2.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        - containerPort: 9090
          name: metrics
          protocol: TCP
        envFrom:
        - configMapRef:
            name: auth-service-config
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: auth-db-secret
              key: host
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: auth-db-secret
              key: password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: auth-jwt-secret
              key: jwt_secret
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health/startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        securityContext:
          runAsNonRoot: true
          runAsUser: 1000
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir:
          medium: Memory
          sizeLimit: "50Mi"
---
# Auth Service Service
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: microservices
  labels:
    app: auth-service
spec:
  selector:
    app: auth-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  - name: metrics
    port: 9090
    targetPort: 9090
    protocol: TCP
  type: ClusterIP
---
# Auth Service ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: auth-service-sa
  namespace: microservices
automountServiceAccountToken: false
---
# Auth Service HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: auth-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: auth-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
      - type: Pods
        value: 3
        periodSeconds: 60
```

### Payment Service - YAML ครบ

```yaml
# payment-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: microservices
  labels:
    app: payment-service
    version: v1
    tier: backend
    pci-dss: "true"   # PCI DSS compliance label
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
      version: v1
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      serviceAccountName: payment-service-sa
      # Security context เข้มงวดสำหรับ PCI DSS
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      # Dedicated nodes สำหรับ payment (taint + toleration)
      tolerations:
      - key: "workload-type"
        operator: "Equal"
        value: "payment"
        effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: workload-type
                operator: In
                values:
                - payment
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: payment-service
            topologyKey: kubernetes.io/hostname
      containers:
      - name: payment-service
        image: myregistry/payment-service:v2.1.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: APP_ENV
          value: "production"
        - name: STRIPE_API_KEY
          valueFrom:
            secretKeyRef:
              name: payment-stripe-secret
              key: api_key
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: payment-db-secret
              key: url
        - name: ENCRYPTION_KEY
          valueFrom:
            secretKeyRef:
              name: payment-encryption-secret
              key: key
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "1Gi"
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: ["ALL"]
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: audit-logs
          mountPath: /var/log/audit
      # Audit log sidecar (PCI DSS requirement)
      - name: audit-log-collector
        image: fluentd:v1.16
        volumeMounts:
        - name: audit-logs
          mountPath: /var/log/audit
          readOnly: true
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "100m"
            memory: "128Mi"
      volumes:
      - name: tmp
        emptyDir:
          medium: Memory
      - name: audit-logs
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  namespace: microservices
  labels:
    app: payment-service
spec:
  selector:
    app: payment-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service-sa
  namespace: microservices
automountServiceAccountToken: false
```

### Catalog Service - YAML ครบ

```yaml
# catalog-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: microservices
  labels:
    app: catalog-service
    version: v1
spec:
  replicas: 4
  selector:
    matchLabels:
      app: catalog-service
  template:
    metadata:
      labels:
        app: catalog-service
        version: v1
    spec:
      serviceAccountName: catalog-service-sa
      containers:
      - name: catalog-service
        image: myregistry/catalog-service:v1.5.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: ELASTICSEARCH_URL
          value: "http://elasticsearch.microservices.svc.cluster.local:9200"
        - name: REDIS_URL
          value: "redis://redis.microservices.svc.cluster.local:6379"
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: catalog-db-secret
              key: url
        - name: S3_BUCKET
          value: "product-images"
        resources:
          requests:
            cpu: "100m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: catalog-service
  namespace: microservices
spec:
  selector:
    app: catalog-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: catalog-service-sa
  namespace: microservices
  annotations:
    # AWS IRSA สำหรับ access S3
    eks.amazonaws.com/role-arn: "arn:aws:iam::ACCOUNT_ID:role/CatalogServiceRole"
automountServiceAccountToken: false
```

### User Service - YAML ครบ

```yaml
# user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
  labels:
    app: user-service
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      serviceAccountName: user-service-sa
      containers:
      - name: user-service
        image: myregistry/user-service:v1.3.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 9090
          name: grpc
        env:
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: user-db-secret
              key: url
        - name: AUTH_SERVICE_URL
          value: "http://auth-service.microservices.svc.cluster.local"
        - name: KAFKA_BROKERS
          value: "kafka.microservices.svc.cluster.local:9092"
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: microservices
spec:
  selector:
    app: user-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  - name: grpc
    port: 9090
    targetPort: 9090
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: user-service-sa
  namespace: microservices
automountServiceAccountToken: false
```

### Notification Service - YAML ครบ

```yaml
# notification-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: notification-service
  namespace: microservices
  labels:
    app: notification-service
    version: v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: notification-service
  template:
    metadata:
      labels:
        app: notification-service
        version: v1
    spec:
      serviceAccountName: notification-service-sa
      containers:
      - name: notification-service
        image: myregistry/notification-service:v1.1.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: KAFKA_BROKERS
          value: "kafka.microservices.svc.cluster.local:9092"
        - name: KAFKA_GROUP_ID
          value: "notification-service"
        - name: KAFKA_TOPICS
          value: "user.events,order.events,payment.events"
        - name: SENDGRID_API_KEY
          valueFrom:
            secretKeyRef:
              name: notification-sendgrid-secret
              key: api_key
        - name: FIREBASE_CREDENTIALS
          valueFrom:
            secretKeyRef:
              name: notification-firebase-secret
              key: credentials
        resources:
          requests:
            cpu: "50m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: notification-service
  namespace: microservices
spec:
  selector:
    app: notification-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: notification-service-sa
  namespace: microservices
automountServiceAccountToken: false
```

---

## Kong API Gateway - YAML ครบ

```yaml
# kong-deployment.yaml
# ติดตั้ง Kong Gateway ด้วย Helm ก่อน
# helm repo add kong https://charts.konghq.com
# helm install kong kong/kong --namespace kong --create-namespace

# Kong Ingress Class
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: kong
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress-controllers.konghq.com/kong
---
# Rate Limiting Plugin (Global)
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limiting-global
  namespace: microservices
  annotations:
    kubernetes.io/ingress.class: "kong"
plugin: rate-limiting
config:
  minute: 100
  hour: 1000
  day: 10000
  policy: redis
  redis_host: redis.microservices.svc.cluster.local
  redis_port: 6379
  fault_tolerant: true
  hide_client_headers: false
---
# JWT Authentication Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: jwt-auth
  namespace: microservices
plugin: jwt
config:
  uri_param_names:
  - jwt
  cookie_names:
  - jwt
  claims_to_verify:
  - exp
  key_claim_name: iss
  secret_is_base64: false
  run_on_preflight: true
---
# CORS Plugin
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: cors-plugin
  namespace: microservices
plugin: cors
config:
  origins:
  - "https://app.example.com"
  - "https://admin.example.com"
  methods:
  - GET
  - POST
  - PUT
  - DELETE
  - PATCH
  - OPTIONS
  headers:
  - Accept
  - Authorization
  - Content-Type
  - X-Request-ID
  exposed_headers:
  - X-Auth-Token
  - X-Request-ID
  credentials: true
  max_age: 3600
---
# Request Size Limiting
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: request-size-limiter
  namespace: microservices
plugin: request-size-limiting
config:
  allowed_payload_size: 10
  size_unit: megabytes
---
# IP Restriction (Whitelist สำหรับ admin)
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: admin-ip-restriction
  namespace: microservices
plugin: ip-restriction
config:
  allow:
  - 10.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16
  deny: []
---
# Ingress สำหรับ Auth Service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-service-ingress
  namespace: microservices
  annotations:
    kubernetes.io/ingress.class: "kong"
    konghq.com/plugins: "rate-limiting-global,cors-plugin,request-size-limiter"
    konghq.com/strip-path: "true"
    konghq.com/protocols: "https"
spec:
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /auth
        pathType: Prefix
        backend:
          service:
            name: auth-service
            port:
              number: 80
---
# Ingress สำหรับ Payment Service (เพิ่ม JWT Auth)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: payment-service-ingress
  namespace: microservices
  annotations:
    kubernetes.io/ingress.class: "kong"
    konghq.com/plugins: "jwt-auth,rate-limiting-global,cors-plugin,request-size-limiter"
    konghq.com/strip-path: "true"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /payments
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
---
# Ingress สำหรับ Catalog Service (Public)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: catalog-service-ingress
  namespace: microservices
  annotations:
    kubernetes.io/ingress.class: "kong"
    konghq.com/plugins: "rate-limiting-global,cors-plugin"
    konghq.com/strip-path: "true"
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /catalog
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 80
      - path: /products
        pathType: Prefix
        backend:
          service:
            name: catalog-service
            port:
              number: 80
```

---

## Kafka Deployment - Strimzi Operator

```yaml
# strimzi-kafka.yaml
# ติดตั้ง Strimzi Operator ก่อน
# kubectl create namespace kafka
# kubectl apply -f https://strimzi.io/install/latest?namespace=kafka -n kafka

# Kafka Cluster
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: microservices-kafka
  namespace: microservices
spec:
  kafka:
    version: 3.6.0
    replicas: 3
    listeners:
    - name: plain
      port: 9092
      type: internal
      tls: false
    - name: tls
      port: 9093
      type: internal
      tls: true
    - name: external
      port: 9094
      type: loadbalancer
      tls: true
    config:
      offsets.topic.replication.factor: 3
      transaction.state.log.replication.factor: 3
      transaction.state.log.min.isr: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      inter.broker.protocol.version: "3.6"
      log.retention.hours: 168        # 7 วัน
      log.retention.bytes: 1073741824 # 1GB per partition
      log.segment.bytes: 536870912    # 512MB per segment
    storage:
      type: persistent-claim
      size: 50Gi
      class: fast-ssd
      deleteClaim: false
    resources:
      requests:
        memory: "2Gi"
        cpu: "500m"
      limits:
        memory: "4Gi"
        cpu: "2000m"
    jvmOptions:
      -Xms: "1g"
      -Xmx: "2g"
    rack:
      topologyKey: topology.kubernetes.io/zone
    template:
      pod:
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  strimzi.io/name: microservices-kafka-kafka
              topologyKey: kubernetes.io/hostname
  zookeeper:
    replicas: 3
    storage:
      type: persistent-claim
      size: 10Gi
      class: fast-ssd
    resources:
      requests:
        memory: "512Mi"
        cpu: "200m"
      limits:
        memory: "1Gi"
        cpu: "500m"
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
# Kafka Topics
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: user-events
  namespace: microservices
  labels:
    strimzi.io/cluster: microservices-kafka
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 604800000     # 7 วัน
    cleanup.policy: delete
    min.insync.replicas: "2"
    max.message.bytes: "1048576" # 1MB
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: order-events
  namespace: microservices
  labels:
    strimzi.io/cluster: microservices-kafka
spec:
  partitions: 24    # order traffic สูงกว่า
  replicas: 3
  config:
    retention.ms: 2592000000    # 30 วัน
    cleanup.policy: delete
    min.insync.replicas: "2"
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: payment-events
  namespace: microservices
  labels:
    strimzi.io/cluster: microservices-kafka
spec:
  partitions: 12
  replicas: 3
  config:
    retention.ms: 7776000000    # 90 วัน (compliance)
    cleanup.policy: delete
    min.insync.replicas: "2"
---
# Kafka User สำหรับ Services
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: notification-service-user
  namespace: microservices
  labels:
    strimzi.io/cluster: microservices-kafka
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
    - resource:
        type: topic
        name: user-events
        patternType: literal
      operation: Read
      host: "*"
    - resource:
        type: topic
        name: order-events
        patternType: literal
      operation: Read
      host: "*"
    - resource:
        type: topic
        name: payment-events
        patternType: literal
      operation: Read
      host: "*"
    - resource:
        type: group
        name: notification-service
        patternType: literal
      operation: Read
      host: "*"
---
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: order-service-user
  namespace: microservices
  labels:
    strimzi.io/cluster: microservices-kafka
spec:
  authentication:
    type: scram-sha-512
  authorization:
    type: simple
    acls:
    - resource:
        type: topic
        name: order-events
        patternType: literal
      operation: Write
      host: "*"
    - resource:
        type: topic
        name: order-events
        patternType: literal
      operation: Read
      host: "*"
    - resource:
        type: group
        name: order-service
        patternType: literal
      operation: Read
      host: "*"
```

---

## KEDA ScaledObject สำหรับ Kafka Consumers

```yaml
# keda-scaledobjects.yaml
# ติดตั้ง KEDA ก่อน
# helm repo add kedacore https://kedacore.github.io/charts
# helm install keda kedacore/keda --namespace keda --create-namespace

# ScaledObject สำหรับ Notification Service (Consumer ของ Kafka)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: notification-service-scaler
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: notification-service
  pollingInterval: 15          # ตรวจสอบทุก 15 วินาที
  cooldownPeriod: 60           # รอ 60 วินาทีก่อน scale down
  idleReplicaCount: 1          # minimum 1 เมื่อ idle
  minReplicaCount: 2           # minimum 2 เมื่อมี messages
  maxReplicaCount: 20          # maximum 20
  advanced:
    restoreToOriginalReplicaCount: false
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
  triggers:
  # Scale ตาม lag ของ user-events topic
  - type: kafka
    metadata:
      bootstrapServers: "microservices-kafka-kafka-bootstrap.microservices.svc.cluster.local:9092"
      consumerGroup: notification-service
      topic: user-events
      lagThreshold: "100"          # Scale เมื่อ lag > 100 messages
      offsetResetPolicy: latest
  # Scale ตาม lag ของ order-events topic
  - type: kafka
    metadata:
      bootstrapServers: "microservices-kafka-kafka-bootstrap.microservices.svc.cluster.local:9092"
      consumerGroup: notification-service
      topic: order-events
      lagThreshold: "50"
      offsetResetPolicy: latest
  # Scale ตาม lag ของ payment-events topic
  - type: kafka
    metadata:
      bootstrapServers: "microservices-kafka-kafka-bootstrap.microservices.svc.cluster.local:9092"
      consumerGroup: notification-service
      topic: payment-events
      lagThreshold: "20"           # payment เร่งด่วนกว่า
      offsetResetPolicy: latest
---
# ScaledObject สำหรับ Order Processor
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-processor
  minReplicaCount: 3
  maxReplicaCount: 50
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: "microservices-kafka-kafka-bootstrap.microservices.svc.cluster.local:9092"
      consumerGroup: order-processor
      topic: order-events
      lagThreshold: "10"    # Strict threshold สำหรับ orders
  - type: prometheus
    metadata:
      serverAddress: http://prometheus.monitoring.svc.cluster.local:9090
      metricName: order_queue_depth
      query: |
        sum(kafka_consumer_group_lag{topic="order-events", consumergroup="order-processor"})
      threshold: "10"
---
# TriggerAuthentication สำหรับ SCRAM-SHA-512
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: kafka-auth
  namespace: microservices
spec:
  secretTargetRef:
  - parameter: sasl
    name: keda-kafka-secret
    key: sasl
  - parameter: username
    name: keda-kafka-secret
    key: username
  - parameter: password
    name: keda-kafka-secret
    key: password
---
# ScaledJob สำหรับ Batch Processing
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: report-generator
  namespace: microservices
spec:
  jobTargetRef:
    template:
      spec:
        containers:
        - name: report-generator
          image: myregistry/report-generator:v1.0.0
          env:
          - name: KAFKA_BOOTSTRAP_SERVERS
            value: "microservices-kafka-kafka-bootstrap.microservices.svc.cluster.local:9092"
        restartPolicy: Never
  pollingInterval: 30
  maxReplicaCount: 10
  triggers:
  - type: kafka
    metadata:
      bootstrapServers: "microservices-kafka-kafka-bootstrap.microservices.svc.cluster.local:9092"
      consumerGroup: report-generator
      topic: report-requests
      lagThreshold: "1"
```

---

## Istio VirtualService และ DestinationRule - ทุก Service

```yaml
# istio-config.yaml

# ========== Auth Service ==========
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: auth-service-vs
  namespace: microservices
spec:
  hosts:
  - auth-service
  http:
  - match:
    - headers:
        x-api-version:
          exact: "v2"
    route:
    - destination:
        host: auth-service
        subset: v2
      weight: 100
  - route:
    - destination:
        host: auth-service
        subset: v1
      weight: 100
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
      retryOn: "5xx,reset,connect-failure,retriable-4xx"
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: auth-service-dr
  namespace: microservices
spec:
  host: auth-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
        maxRetries: 3
      tcp:
        maxConnections: 100
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
    loadBalancer:
      simple: LEAST_CONN
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

# ========== Payment Service ==========
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service-vs
  namespace: microservices
spec:
  hosts:
  - payment-service
  http:
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 100
    timeout: 30s    # payment ต้องการ timeout นานกว่า
    retries:
      attempts: 2   # payment ไม่ควร retry เยอะ (idempotency)
      perTryTimeout: 15s
      retryOn: "connect-failure,retriable-4xx"
    fault:
      # สำหรับ testing - inject delay 10% ของ requests
      # delay:
      #   percentage:
      #     value: 10
      #   fixedDelay: 2s
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: payment-service-dr
  namespace: microservices
spec:
  host: payment-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 50
        http2MaxRequests: 500
      tcp:
        maxConnections: 50
        connectTimeout: 5s
    outlierDetection:
      consecutive5xxErrors: 3
      interval: 5s
      baseEjectionTime: 60s    # eject นานกว่าสำหรับ payment
      maxEjectionPercent: 30
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1

# ========== Catalog Service ==========
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: catalog-service-vs
  namespace: microservices
spec:
  hosts:
  - catalog-service
  http:
  # Canary deployment: 10% ไป v2
  - route:
    - destination:
        host: catalog-service
        subset: v1
      weight: 90
    - destination:
        host: catalog-service
        subset: v2
      weight: 10
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: catalog-service-dr
  namespace: microservices
spec:
  host: catalog-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 200
        http2MaxRequests: 2000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2

# ========== User Service ==========
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: user-service-vs
  namespace: microservices
spec:
  hosts:
  - user-service
  http:
  - route:
    - destination:
        host: user-service
        subset: v1
      weight: 100
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: user-service-dr
  namespace: microservices
spec:
  host: user-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
        http2MaxRequests: 1000
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1

# ========== Notification Service ==========
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: notification-service-vs
  namespace: microservices
spec:
  hosts:
  - notification-service
  http:
  - route:
    - destination:
        host: notification-service
        subset: v1
      weight: 100
    timeout: 5s
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: notification-service-dr
  namespace: microservices
spec:
  host: notification-service
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 50
    tls:
      mode: ISTIO_MUTUAL
  subsets:
  - name: v1
    labels:
      version: v1

# ========== PeerAuthentication (mTLS ทั้ง namespace) ==========
---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: microservices
spec:
  mtls:
    mode: STRICT   # บังคับ mTLS ทุก connections
```

---

## Monitoring: ServiceMonitor + PrometheusRule

```yaml
# monitoring.yaml

# ========== ServiceMonitors ==========
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: auth-service-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
    - microservices
  selector:
    matchLabels:
      app: auth-service
  endpoints:
  - port: metrics
    path: /metrics
    interval: 15s
    scrapeTimeout: 10s
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-service-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
    - microservices
  selector:
    matchLabels:
      app: payment-service
  endpoints:
  - port: metrics
    path: /metrics
    interval: 15s
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: catalog-service-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
    - microservices
  selector:
    matchLabels:
      app: catalog-service
  endpoints:
  - port: metrics
    path: /metrics
    interval: 15s
---
# ========== PrometheusRules ==========
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: microservices-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
  - name: microservices.availability
    interval: 30s
    rules:
    # Auth Service Availability
    - alert: AuthServiceDown
      expr: |
        sum(up{job="auth-service"}) == 0
      for: 1m
      labels:
        severity: critical
        service: auth-service
        team: platform
      annotations:
        summary: "Auth Service is down"
        description: "All Auth Service pods are down for more than 1 minute"
        runbook: "https://runbooks.example.com/auth-service-down"
    
    # Payment Service Error Rate
    - alert: PaymentServiceHighErrorRate
      expr: |
        sum(rate(http_requests_total{job="payment-service", status=~"5.."}[5m])) /
        sum(rate(http_requests_total{job="payment-service"}[5m])) > 0.05
      for: 2m
      labels:
        severity: critical
        service: payment-service
      annotations:
        summary: "Payment Service error rate > 5%"
        description: "Payment Service is experiencing {{ $value | humanizePercentage }} error rate"
    
    # Payment Service Latency
    - alert: PaymentServiceHighLatency
      expr: |
        histogram_quantile(0.99, 
          sum(rate(http_request_duration_seconds_bucket{job="payment-service"}[5m])) by (le)
        ) > 3
      for: 5m
      labels:
        severity: warning
        service: payment-service
      annotations:
        summary: "Payment Service P99 latency > 3s"
    
    # Catalog Service Latency
    - alert: CatalogServiceHighLatency
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket{job="catalog-service"}[5m])) by (le)
        ) > 1
      for: 5m
      labels:
        severity: warning
        service: catalog-service
      annotations:
        summary: "Catalog Service P95 latency > 1s"
  
  - name: microservices.kafka
    rules:
    # Kafka Consumer Lag
    - alert: KafkaConsumerLagHigh
      expr: |
        sum(kafka_consumer_group_lag{consumergroup=~"notification.*"}) > 1000
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Kafka consumer lag is high"
        description: "Notification service consumer lag: {{ $value }}"
    
    # Kafka Consumer Lag Critical
    - alert: KafkaConsumerLagCritical
      expr: |
        sum(kafka_consumer_group_lag{consumergroup="payment-processor"}) > 100
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "Payment Kafka consumer lag is critically high"
  
  - name: microservices.slo
    rules:
    # Auth Service SLO: 99.9% availability
    - record: auth_service:availability:5m
      expr: |
        sum(rate(http_requests_total{job="auth-service", status!~"5.."}[5m])) /
        sum(rate(http_requests_total{job="auth-service"}[5m]))
    
    - alert: AuthServiceSLOBreach
      expr: auth_service:availability:5m < 0.999
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Auth Service SLO breach: availability {{ $value | humanizePercentage }}"
    
    # Payment Service SLO: 99.99% success rate
    - record: payment_service:success_rate:5m
      expr: |
        sum(rate(payment_transactions_total{status="success"}[5m])) /
        sum(rate(payment_transactions_total[5m]))
    
    - alert: PaymentSLOBreach
      expr: payment_service:success_rate:5m < 0.9999
      for: 2m
      labels:
        severity: critical
        pci_dss: "true"
      annotations:
        summary: "Payment Service SLO breach: success rate {{ $value | humanizePercentage }}"
```

---

## Workshop: Deploy ทีละขั้นตอนพร้อม Verification

### Phase 1: ติดตั้ง Infrastructure

```bash
#!/bin/bash
# deploy-phase1-infrastructure.sh
set -euo pipefail

NAMESPACE="microservices"
echo "=== Phase 1: Setting up Infrastructure ==="

# Create namespace
kubectl create namespace $NAMESPACE || true
kubectl label namespace $NAMESPACE istio-injection=enabled --overwrite

# ติดตั้ง operators
echo "Installing Strimzi Kafka Operator..."
kubectl create namespace kafka || true
kubectl apply -f https://strimzi.io/install/latest?namespace=kafka -n kafka
kubectl wait --for=condition=Ready pods \
  -l name=strimzi-cluster-operator \
  -n kafka --timeout=120s

echo "Installing KEDA..."
helm repo add kedacore https://kedacore.github.io/charts
helm upgrade --install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --wait

echo "Installing Kong..."
helm repo add kong https://charts.konghq.com
helm upgrade --install kong kong/kong \
  --namespace kong \
  --create-namespace \
  --set ingressController.installCRDs=false \
  --wait

echo "=== Phase 1 Complete ==="
```

### Phase 2: Deploy Kafka

```bash
# deploy-phase2-kafka.sh
echo "=== Phase 2: Deploying Kafka ==="

# Apply Kafka cluster
kubectl apply -f strimzi-kafka.yaml

# รอ Kafka พร้อม
echo "Waiting for Kafka cluster to be ready..."
kubectl wait kafka/microservices-kafka \
  --for=condition=Ready \
  --timeout=300s \
  -n microservices

# Verify Kafka topics
echo "Checking Kafka topics..."
kubectl get kafkatopics -n microservices

# ทดสอบ Kafka ด้วย test producer/consumer
echo "Testing Kafka..."
kubectl run kafka-test \
  --image=quay.io/strimzi/kafka:latest-kafka-3.6.0 \
  --rm -it \
  --restart=Never \
  -n microservices \
  -- bin/kafka-topics.sh \
    --bootstrap-server microservices-kafka-kafka-bootstrap:9092 \
    --list

echo "=== Phase 2 Complete ==="
```

### Phase 3: Deploy Services

```bash
# deploy-phase3-services.sh
echo "=== Phase 3: Deploying Microservices ==="

SERVICES=("auth-service" "user-service" "catalog-service" "payment-service" "notification-service")

# Apply all service manifests
for service in "${SERVICES[@]}"; do
    echo "Deploying $service..."
    kubectl apply -f "${service}.yaml"
done

# รอทุก services พร้อม
echo "Waiting for all services to be ready..."
for service in "${SERVICES[@]}"; do
    kubectl wait deployment/$service \
      --for=condition=Available \
      --timeout=120s \
      -n microservices
    echo "✅ $service is ready"
done

# ตรวจสอบ pods
kubectl get pods -n microservices -o wide

echo "=== Phase 3 Complete ==="
```

### Phase 4: Apply Istio Configuration

```bash
# deploy-phase4-istio.sh
echo "=== Phase 4: Configuring Istio ==="

# Apply VirtualServices และ DestinationRules
kubectl apply -f istio-config.yaml

# Verify Istio sidecars
echo "Checking Istio sidecar injection..."
kubectl get pods -n microservices \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[*].name}{"\n"}{end}' | \
  grep "istio-proxy"

# ตรวจสอบ mTLS
kubectl exec -n microservices \
  $(kubectl get pod -n microservices -l app=auth-service -o jsonpath='{.items[0].metadata.name}') \
  -- curl -sv http://user-service.microservices.svc.cluster.local/health 2>&1 | \
  grep -E "(SSL|TLS|ALPN|HTTP)"

echo "=== Phase 4 Complete ==="
```

### Phase 5: Setup Monitoring

```bash
# deploy-phase5-monitoring.sh
echo "=== Phase 5: Setting up Monitoring ==="

# Apply ServiceMonitors and PrometheusRules
kubectl apply -f monitoring.yaml

# Verify metrics are being scraped
sleep 30
echo "Checking Prometheus targets..."
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &
PROM_PID=$!
sleep 3

curl -s "http://localhost:9090/api/v1/targets" | \
  jq -r '.data.activeTargets[] | select(.labels.namespace == "microservices") | 
    "\(.labels.job): \(.health)"' | sort

kill $PROM_PID 2>/dev/null
echo "=== Phase 5 Complete ==="
```

### Phase 6: End-to-End Verification

```bash
#!/bin/bash
# e2e-verification.sh
echo "=== End-to-End Verification ==="

NAMESPACE="microservices"

# 1. Check all pods running
echo -e "\n[1] Pod Status:"
kubectl get pods -n $NAMESPACE \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,RESTARTS:.status.containerStatuses[0].restartCount'

# 2. Check services
echo -e "\n[2] Services:"
kubectl get svc -n $NAMESPACE

# 3. Check Kafka topics
echo -e "\n[3] Kafka Topics:"
kubectl get kafkatopics -n $NAMESPACE \
  -o custom-columns='NAME:.metadata.name,PARTITIONS:.spec.partitions,REPLICAS:.spec.replicas'

# 4. Check KEDA scalers
echo -e "\n[4] KEDA ScaledObjects:"
kubectl get scaledobjects -n $NAMESPACE

# 5. Test auth service
echo -e "\n[5] Auth Service Health:"
kubectl exec -n $NAMESPACE \
  $(kubectl get pod -n $NAMESPACE -l app=auth-service -o jsonpath='{.items[0].metadata.name}') \
  -- wget -qO- http://localhost:8080/health/ready

# 6. Check Istio config
echo -e "\n[6] Istio VirtualServices:"
kubectl get virtualservices -n $NAMESPACE

echo -e "\n[7] Istio DestinationRules:"
kubectl get destinationrules -n $NAMESPACE

# 7. Check mTLS
echo -e "\n[8] mTLS Status:"
kubectl get peerauthentication -n $NAMESPACE

# 8. Check HPA
echo -e "\n[9] HPA Status:"
kubectl get hpa -n $NAMESPACE

# 9. Load test (optional)
echo -e "\n[10] Quick Load Test (10 requests):"
kubectl run load-test \
  --image=busybox:latest \
  --rm -it \
  --restart=Never \
  -n $NAMESPACE \
  -- sh -c '
    for i in $(seq 1 10); do
      wget -qO- http://auth-service.microservices.svc.cluster.local/health/ready
      echo ""
    done
  ' 2>/dev/null

echo -e "\n=== Verification Complete ==="
```

---

## สรุปเพิ่มเติม

### Microservices Architecture Summary

```
┌───────────────────────────────────────────────────────────────┐
│                     Internet / Users                           │
└─────────────────────────┬─────────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────────┐
│              Kong API Gateway (kong namespace)                 │
│  Rate Limiting | JWT Auth | CORS | SSL Termination            │
└──────┬──────────────┬──────────────┬────────────┬─────────────┘
       │              │              │            │
       ▼              ▼              ▼            ▼
┌──────────┐  ┌──────────────┐  ┌────────┐  ┌──────────┐
│   Auth   │  │   Catalog    │  │  User  │  │ Payment  │
│ Service  │  │   Service    │  │Service │  │ Service  │
│  (3 pods)│  │   (4 pods)   │  │(3 pods)│  │ (3 pods) │
└──────────┘  └──────────────┘  └────────┘  └──────────┘
                                                  │
                   Kafka Events                   │
       ┌──────────────────────────────────────────┘
       │       (Strimzi Kafka, 3 brokers)
       ▼
┌──────────────┐
│ Notification │◄── KEDA (scale ตาม Kafka lag)
│   Service    │
│   (2-20 pods)│
└──────────────┘

Observability Stack:
├── Prometheus + Grafana (metrics)
├── Loki (logs)
├── Jaeger (traces)
└── Kiali (service mesh topology)
```

### Quick Reference Commands

```bash
# ดู status ทุกอย่าง
kubectl get all -n microservices

# ดู Istio topology
kubectl port-forward -n istio-system svc/kiali 20001:20001
# เปิด http://localhost:20001

# ดู traces
kubectl port-forward -n monitoring svc/jaeger-query 16686:16686
# เปิด http://localhost:16686

# ดู metrics
kubectl port-forward -n monitoring svc/grafana 3000:3000
# เปิด http://localhost:3000

# Scale service manually
kubectl scale deployment/catalog-service \
  --replicas=6 \
  -n microservices

# Rolling restart
kubectl rollout restart deployment/auth-service -n microservices
kubectl rollout status deployment/auth-service -n microservices

# Canary: ส่ง 20% traffic ไป v2
kubectl patch virtualservice catalog-service-vs -n microservices \
  --type json \
  -p '[
    {"op":"replace","path":"/spec/http/0/route/0/weight","value":80},
    {"op":"replace","path":"/spec/http/0/route/1/weight","value":20}
  ]'
```
