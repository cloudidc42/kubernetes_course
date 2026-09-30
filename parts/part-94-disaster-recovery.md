# Part 94: Disaster Recovery สำหรับ Kubernetes

## บทนำ

Disaster Recovery (DR) คือกระบวนการที่ช่วยให้ระบบสามารถกลับมาทำงานได้หลังจากเกิดเหตุการณ์ที่ไม่คาดคิด บทนี้ครอบคลุมกลยุทธ์ DR สำหรับ Kubernetes Cluster ตั้งแต่การ Backup ไปจนถึงการ Restore ในกรณีฉุกเฉิน

## สารบัญ

1. DR Strategies Overview
2. RTO และ RPO Planning
3. etcd Backup และ Restore
4. Application Data Backup
5. Cross-region Recovery
6. Velero สำหรับ Backup
7. DR Testing
8. Workshop: DR Drill

---

## 1. DR Strategies Overview

### 1.1 DR Tier Levels

```
Tier 1: Active-Active (RTO: near-zero, RPO: near-zero)
- หลาย Clusters ใน หลาย Regions ทำงานพร้อมกัน
- Traffic กระจายผ่าน Global Load Balancer
- Data sync real-time

Tier 2: Active-Passive Hot Standby (RTO: < 15 min, RPO: < 15 min)
- Secondary Cluster พร้อมใช้งาน แต่ไม่รับ Traffic ปกติ
- Data replication near-real-time
- Failover อัตโนมัติหรือ Manual

Tier 3: Warm Standby (RTO: < 2 hours, RPO: < 1 hour)
- Secondary Cluster มี Infra แต่ Scale down
- Regular Backup ทุก 1 ชั่วโมง
- Failover ต้องการ Scale up

Tier 4: Cold Standby (RTO: < 24 hours, RPO: < 24 hours)
- Backup ไว้บน Object Storage
- ต้องสร้าง Cluster ใหม่เมื่อเกิด Disaster
- ประหยัดค่าใช้จ่ายสูงสุด
```

### 1.2 Disaster Scenarios

```
ระดับ 1: Pod/Deployment Failure
- สาเหตุ: Bug ใน Code, OOM
- Impact: Partial service degradation
- Recovery: Rolling restart, rollback

ระดับ 2: Node Failure
- สาเหตุ: Hardware failure, OS crash
- Impact: Pods บน node นั้นไม่ available
- Recovery: Pod reschedule อัตโนมัติ

ระดับ 3: Control Plane Failure
- สาเหตุ: etcd corruption, multiple CP failure
- Impact: ไม่สามารถ manage cluster
- Recovery: etcd restore

ระดับ 4: Region/Datacenter Failure
- สาเหตุ: Network outage, power failure, natural disaster
- Impact: ทั้ง Cluster ไม่ available
- Recovery: Failover ไป Secondary Region

ระดับ 5: Data Corruption
- สาเหตุ: Storage failure, ransomware
- Impact: Data loss
- Recovery: Restore จาก Backup
```

---

## 2. RTO และ RPO Planning

### 2.1 คำนิยาม

```
RTO (Recovery Time Objective):
- เวลาสูงสุดที่ระบบ Downtime ได้รับอนุญาต
- ตัวอย่าง: "ระบบต้องกลับมาทำงานได้ภายใน 2 ชั่วโมง"

RPO (Recovery Point Objective):
- ข้อมูลที่สูญเสียได้รับอนุญาต (วัดเป็นเวลา)
- ตัวอย่าง: "ข้อมูลไม่เกิน 15 นาทีก่อน disaster"
```

### 2.2 DR Planning Matrix

```
Service Type    | RTO Target  | RPO Target  | DR Strategy      | Backup Frequency
----------------|-------------|-------------|------------------|------------------
Core API        | < 5 min     | < 1 min     | Active-Active    | Continuous
Payment         | < 15 min    | < 5 min     | Hot Standby      | Every 5 min
User Data       | < 2 hours   | < 1 hour    | Warm Standby     | Hourly
Analytics       | < 24 hours  | < 24 hours  | Cold Standby     | Daily
Logs/Metrics    | < 48 hours  | < 24 hours  | Archive          | Daily
```

### 2.3 Cost vs Recovery Time

```
Active-Active:
  Cost: $$$$$
  RTO: Seconds
  Best for: Critical financial systems

Hot Standby:
  Cost: $$$$
  RTO: Minutes
  Best for: Core business systems

Warm Standby:
  Cost: $$$
  RTO: Hours
  Best for: Internal tools, dev environments

Cold Standby:
  Cost: $$
  RTO: Hours-Days
  Best for: Non-critical, development

Backup Only:
  Cost: $
  RTO: Days
  Best for: Archives, logs
```

---

## 3. etcd Backup และ Restore

### 3.1 Automated etcd Backup System

```bash
#!/bin/bash
# /usr/local/bin/etcd-backup-system.sh
# Production-grade etcd backup script

set -euo pipefail

# Configuration
BACKUP_DIR="/backup/etcd"
S3_BUCKET="s3://my-k8s-etcd-backup"
RETENTION_DAYS=30
RETENTION_HOURLY=24  # เก็บ hourly backup 24 ชั่วโมง
RETENTION_DAILY=7    # เก็บ daily backup 7 วัน
RETENTION_WEEKLY=4   # เก็บ weekly backup 4 อาทิตย์

ETCD_ENDPOINTS="https://127.0.0.1:2379"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_CERT="/etc/kubernetes/pki/etcd/healthcheck-client.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/healthcheck-client.key"

# Logging
LOG_FILE="/var/log/etcd-backup.log"
exec 1> >(tee -a $LOG_FILE)
exec 2>&1

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# สร้าง Backup Directory
mkdir -p $BACKUP_DIR/{hourly,daily,weekly}

# ตรวจสอบ etcd health ก่อน Backup
log "Checking etcd health..."
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# สร้าง Backup
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
HOUR=$(date +%H)
DOW=$(date +%u)  # 1=Monday, 7=Sunday
DOM=$(date +%d)

SNAPSHOT_FILE="$BACKUP_DIR/etcd-snapshot-${TIMESTAMP}.db"

log "Creating etcd snapshot..."
ETCDCTL_API=3 etcdctl snapshot save "$SNAPSHOT_FILE" \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Verify Snapshot
log "Verifying snapshot..."
SNAPSHOT_STATUS=$(ETCDCTL_API=3 etcdctl snapshot status "$SNAPSHOT_FILE" --write-out=json)
REVISION=$(echo $SNAPSHOT_STATUS | jq -r '.revision')
TOTAL_KEY=$(echo $SNAPSHOT_STATUS | jq -r '.totalKey')
log "Snapshot - Revision: $REVISION, Total Keys: $TOTAL_KEY"

# Compress
gzip "$SNAPSHOT_FILE"
SNAPSHOT_FILE="${SNAPSHOT_FILE}.gz"

# Categorize backup
if [ "$HOUR" = "00" ] && [ "$DOM" = "01" ]; then
  # Monthly backup on 1st at midnight
  cp "$SNAPSHOT_FILE" "$BACKUP_DIR/weekly/"
elif [ "$DOW" = "7" ] && [ "$HOUR" = "00" ]; then
  # Weekly backup on Sunday midnight
  cp "$SNAPSHOT_FILE" "$BACKUP_DIR/weekly/"
elif [ "$HOUR" = "00" ]; then
  # Daily backup at midnight
  cp "$SNAPSHOT_FILE" "$BACKUP_DIR/daily/"
fi

# Hourly backup
cp "$SNAPSHOT_FILE" "$BACKUP_DIR/hourly/"

# Upload to S3
log "Uploading to S3..."
aws s3 cp "$SNAPSHOT_FILE" "$S3_BUCKET/$(date +%Y/%m/%d)/$(basename $SNAPSHOT_FILE)"
aws s3 cp "$SNAPSHOT_FILE" "$S3_BUCKET/latest/etcd-latest.db.gz"

# Rotate old backups locally
find "$BACKUP_DIR/hourly" -mtime +1 -name "*.gz" -delete
find "$BACKUP_DIR/daily" -mtime +$RETENTION_DAILY -name "*.gz" -delete
find "$BACKUP_DIR/weekly" -mtime +$((RETENTION_WEEKLY * 7)) -name "*.gz" -delete

# Remove original
rm -f "$SNAPSHOT_FILE"

# Report
BACKUP_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log "Backup completed. Directory size: $BACKUP_SIZE"

# Send notification (optional)
# curl -X POST "$SLACK_WEBHOOK" \
#   -H 'Content-type: application/json' \
#   --data '{"text":"etcd backup completed: '"$TIMESTAMP"'"}'
```

### 3.2 etcd Restore Procedures

```bash
#!/bin/bash
# /usr/local/bin/etcd-restore.sh
# Comprehensive etcd restore procedure

set -euo pipefail

BACKUP_FILE="$1"
CLUSTER_NAME="${2:-production}"
RESTORE_DATA_DIR="/var/lib/etcd-new"
ETCD_DATA_DIR="/var/lib/etcd"

# Validate inputs
if [[ -z "$BACKUP_FILE" ]]; then
  echo "Usage: $0 <backup-file> [cluster-name]"
  echo "Examples:"
  echo "  $0 /backup/etcd/etcd-snapshot-20240101_120000.db.gz"
  echo "  $0 s3://my-backup/etcd-snapshot-20240101_120000.db.gz"
  exit 1
fi

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# ดาวน์โหลดจาก S3 ถ้าจำเป็น
if [[ "$BACKUP_FILE" == s3://* ]]; then
  log "Downloading from S3..."
  LOCAL_BACKUP="/tmp/etcd-restore-$(date +%s).db.gz"
  aws s3 cp "$BACKUP_FILE" "$LOCAL_BACKUP"
  BACKUP_FILE="$LOCAL_BACKUP"
fi

# Decompress ถ้าจำเป็น
if [[ "$BACKUP_FILE" == *.gz ]]; then
  log "Decompressing..."
  gunzip -c "$BACKUP_FILE" > "/tmp/etcd-restore.db"
  BACKUP_FILE="/tmp/etcd-restore.db"
fi

# Verify Snapshot
log "Verifying snapshot integrity..."
ETCDCTL_API=3 etcdctl snapshot status "$BACKUP_FILE" --write-out=table

# === CRITICAL: หยุด Control Plane ===
log "CRITICAL: Stopping control plane on ALL control plane nodes"
log "This MUST be done on all control plane nodes simultaneously"
echo ""
echo "Run these commands on ALL control plane nodes:"
echo "  mkdir -p /etc/kubernetes/manifests-backup"
echo "  mv /etc/kubernetes/manifests/* /etc/kubernetes/manifests-backup/"
echo ""
read -p "Press ENTER when all control planes are stopped..."

# รอให้ Static Pods หยุด
log "Waiting for static pods to stop..."
sleep 30

# ตรวจสอบว่า API Server ไม่ทำงาน
if curl -sk https://127.0.0.1:6443/healthz -o /dev/null; then
  log "ERROR: API Server is still running! Please stop it first."
  exit 1
fi

log "API Server stopped. Proceeding with restore..."

# สำรองข้อมูล etcd เก่า
log "Backing up old etcd data..."
if [ -d "$ETCD_DATA_DIR" ]; then
  mv "$ETCD_DATA_DIR" "${ETCD_DATA_DIR}-backup-$(date +%s)"
fi

# Restore etcd (ต้องรันบน ALL etcd nodes)
log "Restoring etcd data..."

# สำหรับ Single Node
if [ "$CLUSTER_NAME" = "single" ]; then
  ETCDCTL_API=3 etcdctl snapshot restore "$BACKUP_FILE" \
    --data-dir "$ETCD_DATA_DIR"
else
  # สำหรับ HA Cluster (3 nodes)
  # รันบน etcd-1
  ETCDCTL_API=3 etcdctl snapshot restore "$BACKUP_FILE" \
    --name "etcd-1" \
    --initial-cluster "etcd-1=https://10.0.0.10:2380,etcd-2=https://10.0.0.11:2380,etcd-3=https://10.0.0.12:2380" \
    --initial-cluster-token "${CLUSTER_NAME}-etcd-cluster" \
    --initial-advertise-peer-urls "https://10.0.0.10:2380" \
    --data-dir "$ETCD_DATA_DIR"
  
  # รันบน etcd-2 (ด้วย backup file เดียวกัน)
  # ETCDCTL_API=3 etcdctl snapshot restore "$BACKUP_FILE" \
  #   --name "etcd-2" \
  #   --initial-cluster "etcd-1=https://10.0.0.10:2380,etcd-2=https://10.0.0.11:2380,etcd-3=https://10.0.0.12:2380" \
  #   --initial-cluster-token "${CLUSTER_NAME}-etcd-cluster" \
  #   --initial-advertise-peer-urls "https://10.0.0.11:2380" \
  #   --data-dir "$ETCD_DATA_DIR"
fi

# ตั้งค่า Permissions
chown -R root:root "$ETCD_DATA_DIR"

log "Restore complete. Starting control plane..."

# เริ่ม Control Plane อีกครั้ง
mv /etc/kubernetes/manifests-backup/* /etc/kubernetes/manifests/

# รอให้ Control Plane ขึ้นมา
log "Waiting for control plane to start..."
for i in {1..60}; do
  if curl -sk https://127.0.0.1:6443/healthz -o /dev/null 2>&1; then
    log "API Server is up!"
    break
  fi
  log "Waiting... ($i/60)"
  sleep 10
done

# ตรวจสอบผล
log "Checking cluster status..."
kubectl get nodes
kubectl get pods -A | head -20

log "=== etcd Restore Complete ==="
```

---

## 4. Application Data Backup

### 4.1 Velero Installation

```bash
#!/bin/bash
# install-velero.sh

VELERO_VERSION="1.12.0"
S3_BUCKET="my-k8s-backup"
S3_REGION="ap-southeast-1"

# ดาวน์โหลด Velero CLI
wget https://github.com/vmware-tanzu/velero/releases/download/v${VELERO_VERSION}/velero-v${VELERO_VERSION}-linux-amd64.tar.gz
tar xzvf velero-v${VELERO_VERSION}-linux-amd64.tar.gz
mv velero-v${VELERO_VERSION}-linux-amd64/velero /usr/local/bin/

# สร้าง IAM credentials file (สำหรับ AWS)
cat > /tmp/velero-credentials << EOF
[default]
aws_access_key_id=YOUR_ACCESS_KEY
aws_secret_access_key=YOUR_SECRET_KEY
EOF

# ติดตั้ง Velero บน Cluster
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket $S3_BUCKET \
  --backup-location-config region=$S3_REGION \
  --snapshot-location-config region=$S3_REGION \
  --secret-file /tmp/velero-credentials \
  --use-volume-snapshots=true \
  --default-volumes-to-restic=false

# ตรวจสอบ
kubectl get pods -n velero
velero backup-location get

# ลบ credentials file
rm /tmp/velero-credentials
```

### 4.2 Backup Policies

```bash
#!/bin/bash
# velero-backup-policies.sh

# Backup ทุกอย่างใน Namespace production
velero backup create production-backup \
  --include-namespaces production \
  --snapshot-volumes=true \
  --storage-location default \
  --ttl 720h  # 30 วัน

# Backup เฉพาะ Resources บางประเภท
velero backup create config-backup \
  --include-resources configmaps,secrets,deployments \
  --storage-location default \
  --ttl 168h  # 7 วัน

# Backup ด้วย Label Selector
velero backup create critical-services-backup \
  --selector "tier=critical" \
  --snapshot-volumes=true \
  --ttl 720h

# Schedule - ทุกวัน
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production,staging \
  --snapshot-volumes=true \
  --ttl 720h \
  --storage-location default

# Schedule - ทุกชั่วโมง
velero schedule create hourly-backup \
  --schedule="0 * * * *" \
  --include-namespaces production \
  --exclude-resources pods \
  --ttl 24h
```

### 4.3 Velero Restore

```bash
#!/bin/bash
# velero-restore.sh

# ดู Backups ที่มี
velero backup get

# ดู Backups จาก Schedule
velero backup get --selector velero.io/schedule-name=daily-backup

# Restore ทั้งหมดจาก Backup
velero restore create \
  --from-backup production-backup

# Restore เฉพาะ Namespace
velero restore create \
  --from-backup production-backup \
  --include-namespaces production \
  --namespace-mappings production:production-restored

# Restore เฉพาะ Resources
velero restore create \
  --from-backup production-backup \
  --include-resources deployments,services \
  --namespace-mappings production:production-restored

# ดู Status ของ Restore
velero restore get
velero restore describe <restore-name> --details

# ดู Logs ของ Restore
velero restore logs <restore-name>
```

### 4.4 Database Backup

```yaml
# postgres-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 */6 * * *"  # ทุก 6 ชั่วโมง
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: backup-sa
          containers:
          - name: postgres-backup
            image: postgres:14
            env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: S3_BUCKET
              value: "my-db-backup"
            command:
            - /bin/bash
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d_%H%M%S)
              BACKUP_FILE="/tmp/backup-${TIMESTAMP}.sql.gz"
              
              echo "Starting backup..."
              pg_dump \
                -h postgres-service \
                -U postgres \
                -d mydb \
                --no-password \
                | gzip > $BACKUP_FILE
              
              echo "Uploading to S3..."
              aws s3 cp $BACKUP_FILE s3://${S3_BUCKET}/postgres/$(date +%Y/%m)/
              aws s3 cp $BACKUP_FILE s3://${S3_BUCKET}/postgres/latest.sql.gz
              
              rm $BACKUP_FILE
              echo "Backup complete"
            resources:
              requests:
                cpu: "500m"
                memory: "256Mi"
              limits:
                cpu: "1000m"
                memory: "512Mi"
          restartPolicy: OnFailure
```

---

## 5. Cross-region Recovery

### 5.1 Multi-Region Architecture

```yaml
# Primary Region: ap-southeast-1 (Singapore)
# Secondary Region: ap-southeast-2 (Sydney)
# Tertiary Region: ap-east-1 (Hong Kong) - optional

# Global Load Balancer ส่ง Traffic ไปยัง:
# 1. Primary Region (ปกติ)
# 2. Secondary Region (เมื่อ Primary ล้มเหลว)
```

### 5.2 Cross-Region Replication

```bash
#!/bin/bash
# cross-region-sync.sh
# Sync etcd backup ไปยัง Secondary Region

PRIMARY_BUCKET="s3://k8s-backup-primary"
SECONDARY_BUCKET="s3://k8s-backup-secondary"

# Sync Velero backups
aws s3 sync \
  $PRIMARY_BUCKET \
  $SECONDARY_BUCKET \
  --region ap-southeast-2

# Sync etcd backups
aws s3 sync \
  s3://k8s-etcd-backup-primary \
  s3://k8s-etcd-backup-secondary \
  --region ap-southeast-2

# Sync Container Images ไปยัง Secondary Registry
aws ecr get-login-password --region ap-southeast-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-southeast-1.amazonaws.com

aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.ap-southeast-2.amazonaws.com

# Sync critical images
for IMAGE in myapp:latest api:v1.2 worker:v2.0; do
  docker pull 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/$IMAGE
  docker tag 123456789.dkr.ecr.ap-southeast-1.amazonaws.com/$IMAGE \
    123456789.dkr.ecr.ap-southeast-2.amazonaws.com/$IMAGE
  docker push 123456789.dkr.ecr.ap-southeast-2.amazonaws.com/$IMAGE
done
```

### 5.3 Failover Procedure

```bash
#!/bin/bash
# failover-to-secondary.sh
# Automated/Manual failover script

set -euo pipefail

SECONDARY_CLUSTER_NAME="k8s-secondary"
SECONDARY_REGION="ap-southeast-2"
DNS_ZONE_ID="your-route53-zone-id"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# === Phase 1: Verification ===
log "Phase 1: Verifying primary cluster failure..."

PRIMARY_HEALTHY=$(kubectl --context=primary get nodes --no-headers 2>/dev/null | grep -c Ready || echo "0")
if [ "$PRIMARY_HEALTHY" -gt "0" ]; then
  log "WARNING: Primary cluster appears healthy. Manual override required."
  read -p "Are you sure you want to failover? (yes/no): " CONFIRM
  if [ "$CONFIRM" != "yes" ]; then
    log "Failover cancelled."
    exit 0
  fi
fi

# === Phase 2: Prepare Secondary ===
log "Phase 2: Preparing secondary cluster..."

# Switch context to secondary
kubectl config use-context secondary

# ตรวจสอบ Secondary Cluster
kubectl get nodes
kubectl get pods -A | grep -v Running | grep -v Completed || true

# === Phase 3: Restore Latest Backup ===
log "Phase 3: Restoring from latest backup..."

# ดาวน์โหลด Latest Backup
aws s3 cp \
  s3://k8s-backup-secondary/latest/backup-manifest.json \
  /tmp/latest-backup.json \
  --region $SECONDARY_REGION

LATEST_BACKUP=$(jq -r '.latestBackup' /tmp/latest-backup.json)
log "Latest backup: $LATEST_BACKUP"

# Restore ด้วย Velero
velero restore create failover-restore \
  --from-backup $LATEST_BACKUP \
  --wait

# ตรวจสอบ Restore
velero restore describe failover-restore
kubectl get pods -A

# === Phase 4: Update DNS ===
log "Phase 4: Updating DNS..."

SECONDARY_LB=$(kubectl --context=secondary get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# อัพเดท Route53
aws route53 change-resource-record-sets \
  --hosted-zone-id $DNS_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "CNAME",
        "TTL": 60,
        "ResourceRecords": [{"Value": "'"$SECONDARY_LB"'"}]
      }
    }]
  }'

log "DNS updated. Failover complete!"
log "New endpoint: $SECONDARY_LB"

# === Phase 5: Notify Team ===
log "Phase 5: Sending notifications..."
# curl -X POST "$SLACK_WEBHOOK_URL" \
#   -H 'Content-type: application/json' \
#   --data '{"text":"⚠️ FAILOVER EXECUTED: Production traffic now routing to secondary cluster in '"$SECONDARY_REGION"'"}'

log "=== FAILOVER COMPLETE ==="
log "Monitor: kubectl --context=secondary get pods -A -w"
```

---

## 6. DR Testing

### 6.1 DR Test Plan

```markdown
## DR Test Schedule

**Quarterly Tests:**
- [ ] etcd backup/restore test (single node)
- [ ] Application restore ด้วย Velero
- [ ] Node failure simulation

**Semi-annual Tests:**
- [ ] Full cluster restore
- [ ] Failover to secondary region
- [ ] DNS failover test

**Annual Tests:**
- [ ] Complete DR drill (simulate real disaster)
- [ ] All team members walk through DR procedures
- [ ] Update and verify runbooks

## Test Criteria
- RTO: ต้องสำเร็จภายในเวลาที่กำหนด
- RPO: ข้อมูลสูญหายต้องน้อยกว่าที่กำหนด
- Completeness: ทุก critical service ต้องทำงาน
- Data Integrity: ข้อมูลถูกต้องหลัง restore
```

### 6.2 DR Test Script

```bash
#!/bin/bash
# dr-test.sh - Automated DR Test

set -euo pipefail

TEST_NAMESPACE="dr-test"
TEST_DEPLOYMENT="dr-test-app"
TEST_DATA_SECRET="dr-test-data"
BACKUP_NAME="dr-test-$(date +%Y%m%d-%H%M%S)"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [DR-TEST] $1"
}

pass() {
  echo "[PASS] $1"
}

fail() {
  echo "[FAIL] $1"
  FAILED_TESTS+=("$1")
}

FAILED_TESTS=()

# === Setup Test Environment ===
log "Setting up test environment..."
kubectl create namespace $TEST_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# สร้าง Test Data
kubectl create secret generic $TEST_DATA_SECRET \
  --from-literal=test-key="test-value-$(date +%s)" \
  -n $TEST_NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl create deployment $TEST_DEPLOYMENT \
  --image=nginx \
  --replicas=3 \
  -n $TEST_NAMESPACE

kubectl wait --for=condition=available deployment/$TEST_DEPLOYMENT \
  -n $TEST_NAMESPACE \
  --timeout=120s

# บันทึก Original State
ORIGINAL_SECRET_VALUE=$(kubectl get secret $TEST_DATA_SECRET \
  -n $TEST_NAMESPACE \
  -o jsonpath='{.data.test-key}' | base64 -d)

log "Original secret value: $ORIGINAL_SECRET_VALUE"

# === Test 1: Create Backup ===
log "Test 1: Creating Velero backup..."
velero backup create $BACKUP_NAME \
  --include-namespaces $TEST_NAMESPACE \
  --wait

BACKUP_STATUS=$(velero backup get $BACKUP_NAME -o json | jq -r '.status.phase')
if [ "$BACKUP_STATUS" = "Completed" ]; then
  pass "Backup created successfully"
else
  fail "Backup failed (status: $BACKUP_STATUS)"
fi

# === Test 2: Simulate Disaster ===
log "Test 2: Simulating disaster (deleting namespace)..."
kubectl delete namespace $TEST_NAMESPACE --wait=true

if ! kubectl get namespace $TEST_NAMESPACE 2>/dev/null; then
  pass "Namespace deleted (disaster simulated)"
else
  fail "Namespace deletion failed"
fi

# === Test 3: Measure Restore Time ===
log "Test 3: Measuring restore time..."
RESTORE_START=$(date +%s)

velero restore create "${BACKUP_NAME}-restore" \
  --from-backup $BACKUP_NAME \
  --wait

RESTORE_END=$(date +%s)
RESTORE_TIME=$((RESTORE_END - RESTORE_START))

RESTORE_STATUS=$(velero restore get "${BACKUP_NAME}-restore" -o json | jq -r '.status.phase')
if [ "$RESTORE_STATUS" = "Completed" ]; then
  pass "Restore completed in ${RESTORE_TIME} seconds"
else
  fail "Restore failed (status: $RESTORE_STATUS)"
fi

# === Test 4: Verify Data Integrity ===
log "Test 4: Verifying data integrity..."
kubectl wait --for=condition=available deployment/$TEST_DEPLOYMENT \
  -n $TEST_NAMESPACE \
  --timeout=120s

RESTORED_SECRET_VALUE=$(kubectl get secret $TEST_DATA_SECRET \
  -n $TEST_NAMESPACE \
  -o jsonpath='{.data.test-key}' | base64 -d)

if [ "$ORIGINAL_SECRET_VALUE" = "$RESTORED_SECRET_VALUE" ]; then
  pass "Data integrity verified (secret value matches)"
else
  fail "Data integrity check FAILED (original: $ORIGINAL_SECRET_VALUE, restored: $RESTORED_SECRET_VALUE)"
fi

# === Test 5: Application Functionality ===
log "Test 5: Testing application functionality..."
REPLICAS=$(kubectl get deployment $TEST_DEPLOYMENT -n $TEST_NAMESPACE \
  -o jsonpath='{.status.readyReplicas}')

if [ "$REPLICAS" = "3" ]; then
  pass "Application running with correct replicas ($REPLICAS/3)"
else
  fail "Application not fully restored (ready replicas: $REPLICAS/3)"
fi

# === Cleanup ===
log "Cleaning up test environment..."
kubectl delete namespace $TEST_NAMESPACE
velero backup delete $BACKUP_NAME --confirm
velero restore delete "${BACKUP_NAME}-restore" --confirm

# === Test Report ===
echo ""
echo "=== DR TEST REPORT ==="
echo "Date: $(date)"
echo "Restore Time: ${RESTORE_TIME} seconds"
echo ""

if [ ${#FAILED_TESTS[@]} -eq 0 ]; then
  echo "RESULT: ALL TESTS PASSED"
  echo ""
  echo "RTO Achieved: ${RESTORE_TIME} seconds"
else
  echo "RESULT: ${#FAILED_TESTS[@]} TESTS FAILED"
  for test in "${FAILED_TESTS[@]}"; do
    echo "  - FAILED: $test"
  done
  exit 1
fi
```

---

## 7. Workshop: DR Drill

### Workshop Overview

ในบทนี้เราจะทำ DR Drill แบบสมบูรณ์:
1. สร้าง Production Environment
2. ทำ Full Backup
3. Simulate Disaster
4. Execute Recovery
5. Verify Recovery
6. Document Lessons Learned

### Step 1: Setup Production Environment

```bash
# สร้าง Production Namespace และ Application
kubectl create namespace production

# Deploy Multi-tier Application
cat <<'EOF' > /tmp/production-app.yaml
---
# PostgreSQL Database
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: "SuperSecret123!"
        - name: POSTGRES_DB
          value: "appdb"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: production
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
---
# API Application
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api
  namespace: production
spec:
  selector:
    app: api
  ports:
  - port: 80
---
# Important Config
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  DATABASE_URL: "postgres://postgres:SuperSecret123!@postgres:5432/appdb"
  API_VERSION: "v2.1.0"
  FEATURE_FLAGS: "new_ui=true,beta_features=false"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  jwt-secret: "ThisIsAVerySecretJWTKey123!"
  api-key: "sk-prod-abcdef123456"
EOF

kubectl apply -f /tmp/production-app.yaml
kubectl wait --for=condition=available deployment/api -n production --timeout=120s
echo "Production environment ready!"
```

### Step 2: ทำ Full Backup

```bash
# Backup ด้วย Velero
velero backup create pre-drill-backup \
  --include-namespaces production \
  --snapshot-volumes=true \
  --wait

# Backup etcd ด้วย
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd/pre-drill-$(date +%s).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

echo "Full backup complete!"
velero backup describe pre-drill-backup
```

### Step 3: Simulate Disaster

```bash
# === DISASTER SIMULATION ===
echo "!!! SIMULATING DISASTER !!!"
echo "Deleting production namespace..."

# บันทึก state ก่อน
kubectl get all -n production > /tmp/pre-disaster-state.txt
kubectl get configmaps -n production >> /tmp/pre-disaster-state.txt

# Delete namespace (simulate disaster)
kubectl delete namespace production
echo "Production namespace deleted. DISASTER IN PROGRESS!"
sleep 5

# ยืนยันว่า namespace หายแล้ว
kubectl get namespace production 2>/dev/null || echo "Confirmed: namespace does not exist"
```

### Step 4: Execute Recovery

```bash
# === RECOVERY BEGINS ===
RECOVERY_START=$(date +%s)
echo "[$(date)] Recovery started"

# Restore ด้วย Velero
velero restore create drill-recovery \
  --from-backup pre-drill-backup \
  --wait

# ตรวจสอบ Restore Status
velero restore describe drill-recovery

# รอให้ Pods ขึ้นมา
kubectl wait --for=condition=available deployment/api -n production --timeout=300s

RECOVERY_END=$(date +%s)
RTO=$((RECOVERY_END - RECOVERY_START))
echo "[$(date)] Recovery completed in ${RTO} seconds"
```

### Step 5: Verify Recovery

```bash
#!/bin/bash
# verify-recovery.sh

echo "=== Recovery Verification ==="
FAILED=0

# 1. Namespace exists
echo -n "1. Namespace exists: "
kubectl get namespace production &>/dev/null && echo "PASS" || { echo "FAIL"; FAILED=$((FAILED+1)); }

# 2. Pods running
echo -n "2. Pods running: "
READY_PODS=$(kubectl get pods -n production --field-selector=status.phase=Running --no-headers | wc -l)
[ "$READY_PODS" -gt "0" ] && echo "PASS ($READY_PODS pods)" || { echo "FAIL (0 pods)"; FAILED=$((FAILED+1)); }

# 3. API Deployment
echo -n "3. API Deployment available: "
kubectl wait --for=condition=available deployment/api -n production --timeout=60s &>/dev/null && echo "PASS" || { echo "FAIL"; FAILED=$((FAILED+1)); }

# 4. Database running
echo -n "4. Database running: "
kubectl get pods -n production -l app=postgres --field-selector=status.phase=Running --no-headers | grep -q postgres && echo "PASS" || { echo "FAIL"; FAILED=$((FAILED+1)); }

# 5. ConfigMap exists
echo -n "5. ConfigMap restored: "
kubectl get configmap app-config -n production &>/dev/null && echo "PASS" || { echo "FAIL"; FAILED=$((FAILED+1)); }

# 6. Secrets exist
echo -n "6. Secrets restored: "
kubectl get secret app-secrets -n production &>/dev/null && echo "PASS" || { echo "FAIL"; FAILED=$((FAILED+1)); }

# 7. Services exist
echo -n "7. Services restored: "
kubectl get svc -n production --no-headers | wc -l | xargs | grep -qv "^0$" && echo "PASS" || { echo "FAIL"; FAILED=$((FAILED+1)); }

# 8. Verify Config Values
echo -n "8. Config values correct: "
API_VER=$(kubectl get configmap app-config -n production -o jsonpath='{.data.API_VERSION}')
[ "$API_VER" = "v2.1.0" ] && echo "PASS ($API_VER)" || { echo "FAIL (got: $API_VER)"; FAILED=$((FAILED+1)); }

echo ""
if [ "$FAILED" -eq "0" ]; then
  echo "=== ALL TESTS PASSED ==="
else
  echo "=== $FAILED TESTS FAILED ==="
fi

# Cleanup
echo "Cleaning up DR drill environment..."
kubectl delete namespace production
velero backup delete pre-drill-backup --confirm
velero restore delete drill-recovery --confirm
```

### Step 6: หลักจาก DR Drill - Lessons Learned Template

```markdown
## DR Drill Report - [Date]

### Summary
- Drill Type: Full Production DR
- Trigger: Scheduled Drill
- Start Time: [time]
- End Time: [time]
- Total Duration: [X minutes]

### Results
- RTO Achieved: [X minutes] (Target: [Y minutes])
- RPO Achieved: [X minutes] (Target: [Y minutes])
- Tests Passed: [X/Y]
- Data Loss: None/[description]

### Timeline
| Time | Event |
|------|-------|
| T+0  | Disaster detected |
| T+5  | Recovery team assembled |
| T+10 | Backup identified and validated |
| T+15 | Restore initiated |
| T+30 | Services restored |
| T+45 | Verification complete |

### Issues Found
1. Issue: [description]
   - Impact: [low/medium/high]
   - Action: [what to do]

### Improvements
1. [ ] Update backup frequency from 6h to 4h
2. [ ] Add automated notification to Slack
3. [ ] Create runbook for database restore

### Sign-off
- Tested by: [name]
- Approved by: [name]
- Next drill: [date]
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **DR Strategies** - Active-Active, Hot Standby, Warm, Cold
2. **RTO/RPO Planning** - การกำหนดเป้าหมาย Recovery
3. **etcd Backup/Restore** - Automated backup และ restore procedures
4. **Application Backup** - Velero สำหรับ Application data
5. **Cross-region Recovery** - Multi-region architecture
6. **DR Testing** - Automated test scripts
7. **Workshop** - Full DR Drill

## แบบฝึกหัด

1. ติดตั้ง Velero และสร้าง Backup Policy สำหรับ Namespace
2. ทำ etcd Backup และ Restore ให้สำเร็จ
3. สร้าง Automated DR Test Script
4. วางแผน Multi-Region DR สำหรับ Application
5. ทำ DR Drill และเขียน Lessons Learned Report

## References

- [Velero Documentation](https://velero.io/docs/)
- [etcd Disaster Recovery](https://etcd.io/docs/v3.5/op-guide/recovery/)
- [Kubernetes Backup Best Practices](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/)
