# Part 72: Jenkins บน Kubernetes

## บทนำ

Jenkins เป็นหนึ่งใน Open Source CI/CD Tools ที่ได้รับความนิยมมากที่สุดในโลก มีประวัติยาวนานและ Community ขนาดใหญ่ การรัน Jenkins บน Kubernetes ช่วยให้สามารถ Scale Build Agents ได้อัตโนมัติ ประหยัดทรัพยากร และใช้งาน Container ได้อย่างเต็มที่

## 1. Jenkins Architecture บน Kubernetes

### 1.1 Components

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                       │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Jenkins Namespace                      │   │
│  │                                                     │   │
│  │  ┌──────────────┐     ┌──────────────────────────┐  │   │
│  │  │ Jenkins      │     │   Dynamic Agent Pods     │  │   │
│  │  │ Controller   │────>│                          │  │   │
│  │  │ (Master)     │     │  Pod 1: build-xxx         │  │   │
│  │  │              │     │  Pod 2: build-yyy         │  │   │
│  │  │ Port: 8080   │     │  Pod 3: build-zzz         │  │   │
│  │  │ Port: 50000  │     │                          │  │   │
│  │  └──────────────┘     └──────────────────────────┘  │   │
│  │          │                                           │   │
│  │          │ PVC                                       │   │
│  │  ┌──────────────┐                                   │   │
│  │  │ Jenkins Home │                                   │   │
│  │  │ PersistentVol│                                   │   │
│  │  └──────────────┘                                   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 ทำไมต้อง Jenkins บน Kubernetes

1. **Dynamic Scaling** - Agent Pods สร้างและลบอัตโนมัติ
2. **Resource Efficiency** - ไม่ต้องจ่าย Idle Agents
3. **Isolation** - แต่ละ Build รันใน Pod แยก
4. **Consistency** - ใช้ Container Image สำหรับ Build Environment
5. **Parallel Builds** - รัน Multiple Builds พร้อมกัน

## 2. ติดตั้ง Jenkins บน Kubernetes

### 2.1 เตรียม Namespace และ RBAC

```yaml
# jenkins-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: jenkins
  labels:
    name: jenkins

---
# jenkins-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create", "delete", "get", "list", "patch", "update", "watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
```

### 2.2 Jenkins PersistentVolumeClaim

```yaml
# jenkins-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-home
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard  # เปลี่ยนตาม StorageClass ที่มี
  resources:
    requests:
      storage: 20Gi
```

### 2.3 Jenkins Deployment

```yaml
# jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
      initContainers:
      - name: init-jenkins
        image: busybox:1.36
        command: ['sh', '-c', 'chown -R 1000:1000 /var/jenkins_home']
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        securityContext:
          runAsUser: 0
      containers:
      - name: jenkins
        image: jenkins/jenkins:lts-jdk17
        ports:
        - name: http
          containerPort: 8080
        - name: jnlp
          containerPort: 50000
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 2000m
            memory: 4Gi
        env:
        - name: JAVA_OPTS
          value: >-
            -Xmx2g
            -Xms512m
            -Djenkins.install.runSetupWizard=false
            -Dhudson.security.csrf.DefaultCrumbIssuer.EXCLUDE_SESSION_ID=true
        - name: CASC_JENKINS_CONFIG
          value: /var/jenkins_home/casc_configs
        volumeMounts:
        - name: jenkins-home
          mountPath: /var/jenkins_home
        - name: jenkins-config
          mountPath: /var/jenkins_home/casc_configs
        livenessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /login
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 3
      volumes:
      - name: jenkins-home
        persistentVolumeClaim:
          claimName: jenkins-home
      - name: jenkins-config
        configMap:
          name: jenkins-config

---
# jenkins-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: ClusterIP
  selector:
    app: jenkins
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: jnlp
    port: 50000
    targetPort: 50000

---
# jenkins-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins
  namespace: jenkins
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - jenkins.example.com
    secretName: jenkins-tls
  rules:
  - host: jenkins.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: jenkins
            port:
              number: 8080
```

### 2.4 Jenkins Configuration as Code (JCasC)

```yaml
# jenkins-config-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-config
  namespace: jenkins
data:
  jenkins.yaml: |
    jenkins:
      systemMessage: "Jenkins configured with JCasC"
      numExecutors: 0
      mode: EXCLUSIVE
      scmCheckoutRetryCount: 3
      
      securityRealm:
        local:
          allowsSignup: false
          users:
          - id: "admin"
            password: "${JENKINS_ADMIN_PASSWORD}"
      
      authorizationStrategy:
        loggedInUsersCanDoAnything:
          allowAnonymousRead: false
      
      clouds:
      - kubernetes:
          name: "kubernetes"
          serverUrl: "https://kubernetes.default.svc"
          namespace: "jenkins"
          jenkinsUrl: "http://jenkins.jenkins.svc.cluster.local:8080"
          jenkinsTunnel: "jenkins.jenkins.svc.cluster.local:50000"
          connectionTimeout: 5
          readTimeout: 15
          containerCapStr: "10"
          maxRequestsPerHostStr: "32"
          podLabels:
          - key: "jenkins"
            value: "agent"
          templates:
          - name: "default"
            namespace: "jenkins"
            label: "jenkins-agent"
            containers:
            - name: "jnlp"
              image: "jenkins/inbound-agent:latest"
              resourceRequestCpu: "200m"
              resourceRequestMemory: "256Mi"
              resourceLimitCpu: "500m"
              resourceLimitMemory: "512Mi"
    
    credentials:
      system:
        domainCredentials:
        - credentials:
          - usernamePassword:
              scope: GLOBAL
              id: "registry-credentials"
              username: "${REGISTRY_USER}"
              password: "${REGISTRY_PASSWORD}"
              description: "Container Registry Credentials"
          - gitUsernamePassword:
              scope: GLOBAL
              id: "github-credentials"
              username: "${GITHUB_USER}"
              password: "${GITHUB_TOKEN}"
              description: "GitHub Credentials"
    
    unclassified:
      location:
        url: "https://jenkins.example.com"
        adminAddress: "jenkins@example.com"
      
      slackNotifier:
        teamDomain: "myteam"
        tokenCredentialId: "slack-token"
        room: "#ci-cd"
    
    jobs:
    - script: >
        pipelineJob('myapp-pipeline') {
          definition {
            cpsScm {
              scm {
                git {
                  remote {
                    url('https://github.com/org/myapp.git')
                    credentials('github-credentials')
                  }
                  branches('*/main')
                }
              }
              scriptPath('Jenkinsfile')
            }
          }
          triggers {
            scm('H/5 * * * *')
          }
        }
```

### 2.5 ติดตั้งด้วย Helm

```bash
# เพิ่ม Jenkins Helm Repository
helm repo add jenkins https://charts.jenkins.io
helm repo update

# สร้าง values.yaml
cat > jenkins-values.yaml << 'EOF'
controller:
  image: jenkins/jenkins
  tag: lts-jdk17
  
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
  
  serviceType: ClusterIP
  
  ingress:
    enabled: true
    ingressClassName: nginx
    hostName: jenkins.example.com
    tls:
    - secretName: jenkins-tls
      hosts:
      - jenkins.example.com
  
  installPlugins:
    - kubernetes:3900.va_dce992317b_4
    - workflow-aggregator:596.v8c21c963d92d
    - git:5.2.0
    - configuration-as-code:1775.v810dc950b_514
    - blueocean:1.27.11
    - pipeline-stage-view:2.33
    - docker-workflow:572.v950f58993843
    - credentials-binding:657.v2b_19db_7d6e6d
    - gitlab-plugin:1.8.1
    - slack:683.v96b_372b_d08a_c
    
  JCasC:
    defaultConfig: true
    
  persistence:
    enabled: true
    storageClass: standard
    size: 20Gi

agent:
  enabled: true
  defaultsProviderTemplate: "default"
  namespace: jenkins
  
  podName: "jenkins-agent"
  
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

persistence:
  enabled: true
  storageClass: standard
  size: 20Gi

serviceAccount:
  create: true
  name: jenkins

rbac:
  create: true
  readSecrets: true
EOF

# ติดตั้ง
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --create-namespace \
  -f jenkins-values.yaml

# ดู Admin Password
kubectl exec -n jenkins -it svc/jenkins -c jenkins -- \
  /bin/cat /run/secrets/additional/chart-admin-password
```

## 3. Kubernetes Plugin

### 3.1 Pod Template Configuration

```groovy
// Jenkinsfile ที่ใช้ Pod Template
pipeline {
    agent {
        kubernetes {
            yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
  - name: golang
    image: golang:1.21-alpine
    command: [cat]
    tty: true
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
      limits:
        cpu: 1000m
        memory: 1Gi
  - name: docker
    image: docker:24-dind
    securityContext:
      privileged: true
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
  - name: kubectl
    image: bitnami/kubectl:latest
    command: [cat]
    tty: true
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
  volumes:
  - name: docker-socket
    emptyDir: {}
'''
        }
    }
    
    stages {
        stage('Build') {
            steps {
                container('golang') {
                    sh 'go build ./...'
                }
            }
        }
        
        stage('Test') {
            steps {
                container('golang') {
                    sh 'go test ./... -v'
                }
            }
        }
        
        stage('Docker Build') {
            steps {
                container('docker') {
                    sh 'docker build -t myapp:latest .'
                }
            }
        }
        
        stage('Deploy') {
            steps {
                container('kubectl') {
                    sh 'kubectl apply -f k8s/'
                }
            }
        }
    }
}
```

### 3.2 Shared Pod Templates

```yaml
# shared-agent-templates.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-agent-templates
  namespace: jenkins
data:
  go-agent.yaml: |
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: go
        image: golang:1.21-alpine
        command: [cat]
        tty: true
        volumeMounts:
        - name: go-cache
          mountPath: /go/pkg/mod
      volumes:
      - name: go-cache
        emptyDir: {}
  
  node-agent.yaml: |
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: node
        image: node:18-alpine
        command: [cat]
        tty: true
        volumeMounts:
        - name: npm-cache
          mountPath: /root/.npm
      volumes:
      - name: npm-cache
        emptyDir: {}
  
  python-agent.yaml: |
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: python
        image: python:3.11-slim
        command: [cat]
        tty: true
```

## 4. Jenkinsfile - Pipeline as Code

### 4.1 Declarative Pipeline

```groovy
// Jenkinsfile - Complete CI/CD Pipeline
pipeline {
    agent none
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE_NAME = 'myapp'
        DOCKER_CREDENTIALS = credentials('registry-credentials')
        GITHUB_CREDENTIALS = credentials('github-credentials')
        SONAR_TOKEN = credentials('sonar-token')
        SLACK_CHANNEL = '#ci-cd'
    }
    
    stages {
        stage('Checkout') {
            agent {
                kubernetes {
                    defaultContainer 'jnlp'
                }
            }
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT_SHORT}"
                }
                stash name: 'source', includes: '**/*'
            }
        }
        
        stage('Lint & Static Analysis') {
            parallel {
                stage('Go Lint') {
                    agent {
                        kubernetes {
                            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: golangci
    image: golangci/golangci-lint:v1.55-alpine
    command: [cat]
    tty: true
"""
                        }
                    }
                    steps {
                        unstash 'source'
                        container('golangci') {
                            sh 'golangci-lint run ./... --timeout 5m'
                        }
                    }
                }
                
                stage('Security Scan') {
                    agent {
                        kubernetes {
                            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: gosec
    image: securego/gosec:latest
    command: [cat]
    tty: true
"""
                        }
                    }
                    steps {
                        unstash 'source'
                        container('gosec') {
                            sh 'gosec -severity high ./...'
                        }
                    }
                }
            }
        }
        
        stage('Test') {
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: go
    image: golang:1.21-alpine
    command: [cat]
    tty: true
    env:
    - name: DATABASE_URL
      value: "postgres://test:test@localhost:5432/testdb"
  - name: postgres
    image: postgres:15-alpine
    env:
    - name: POSTGRES_DB
      value: testdb
    - name: POSTGRES_USER
      value: test
    - name: POSTGRES_PASSWORD
      value: test
"""
                }
            }
            steps {
                unstash 'source'
                container('go') {
                    sh '''
                        # รอ PostgreSQL พร้อม
                        while ! pg_isready -h localhost -p 5432; do
                          sleep 2
                        done
                        
                        # รัน Migrations
                        go run cmd/migrate/main.go up
                        
                        # รัน Tests
                        go test ./... -v -coverprofile=coverage.out
                        go tool cover -func=coverage.out | tee coverage-summary.txt
                        
                        # ตรวจสอบ Coverage
                        COVERAGE=$(grep "total:" coverage-summary.txt | awk '{print $3}' | tr -d '%')
                        if (( $(echo "$COVERAGE < 80" | bc -l) )); then
                            echo "Coverage $COVERAGE% is below 80% threshold"
                            exit 1
                        fi
                        echo "Coverage: $COVERAGE% ✓"
                    '''
                }
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'coverage.html',
                    reportName: 'Coverage Report'
                ])
            }
            post {
                always {
                    junit 'test-results/*.xml'
                }
            }
        }
        
        stage('Build & Push Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: [/busybox/cat]
    tty: true
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  volumes:
  - name: docker-config
    secret:
      secretName: registry-credentials
      items:
      - key: .dockerconfigjson
        path: config.json
"""
                }
            }
            steps {
                unstash 'source'
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                          --context=. \
                          --dockerfile=Dockerfile \
                          --destination=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                          --destination=${REGISTRY}/${IMAGE_NAME}:latest \
                          --cache=true \
                          --cache-repo=${REGISTRY}/${IMAGE_NAME}/cache
                    """
                }
            }
        }
        
        stage('Trivy Scan') {
            when {
                anyOf {
                    branch 'main'
                    branch 'develop'
                }
            }
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: trivy
    image: aquasec/trivy:latest
    command: [cat]
    tty: true
"""
                }
            }
            steps {
                container('trivy') {
                    sh """
                        trivy image \
                          --format json \
                          --output trivy-report.json \
                          ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                        
                        # Fail on HIGH/CRITICAL
                        trivy image \
                          --exit-code 1 \
                          --severity HIGH,CRITICAL \
                          ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
                archiveArtifacts artifacts: 'trivy-report.json'
            }
        }
        
        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: [cat]
    tty: true
"""
                }
            }
            steps {
                container('kubectl') {
                    sh """
                        kubectl set image deployment/myapp \
                          myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                          -n development
                        kubectl rollout status deployment/myapp \
                          -n development \
                          --timeout=5m
                    """
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kubectl
    image: bitnami/kubectl:latest
    command: [cat]
    tty: true
"""
                }
            }
            steps {
                container('kubectl') {
                    sh """
                        kubectl set image deployment/myapp \
                          myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                          -n staging
                        kubectl rollout status deployment/myapp \
                          -n staging \
                          --timeout=5m
                    """
                }
                
                // Integration Tests
                sh 'curl -f https://staging.example.com/health || exit 1'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            agent {
                kubernetes {
                    defaultContainer 'jnlp'
                }
            }
            options {
                timeout(time: 24, unit: 'HOURS')
            }
            input {
                message "Deploy to Production?"
                ok "Deploy"
                submitter "admin,ops-team"
                parameters {
                    string(
                        name: 'DEPLOY_NOTE',
                        defaultValue: '',
                        description: 'Deployment note'
                    )
                }
            }
            steps {
                container('kubectl') {
                    sh """
                        kubectl set image deployment/myapp \
                          myapp=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                          -n production
                        kubectl rollout status deployment/myapp \
                          -n production \
                          --timeout=10m
                    """
                }
            }
        }
    }
    
    post {
        always {
            script {
                def buildStatus = currentBuild.result ?: 'SUCCESS'
                def color = buildStatus == 'SUCCESS' ? 'good' : 'danger'
                def emoji = buildStatus == 'SUCCESS' ? '✅' : '❌'
                
                slackSend(
                    channel: SLACK_CHANNEL,
                    color: color,
                    message: """
${emoji} *Build ${buildStatus}*
Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Branch: ${env.BRANCH_NAME}
Commit: ${env.GIT_COMMIT_SHORT}
Duration: ${currentBuild.durationString}
URL: ${env.BUILD_URL}
                    """
                )
            }
        }
        
        failure {
            emailext(
                subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """
Build failed!

Job: ${env.JOB_NAME}
Build: #${env.BUILD_NUMBER}
Branch: ${env.BRANCH_NAME}
URL: ${env.BUILD_URL}

Please investigate.
                """,
                to: 'devops-team@example.com'
            )
        }
    }
}
```

### 4.2 Scripted Pipeline

```groovy
// Scripted Pipeline (ยืดหยุ่นกว่า)
node('jenkins-agent') {
    def imageTag = ""
    
    try {
        stage('Checkout') {
            checkout scm
            imageTag = sh(
                script: 'git rev-parse --short HEAD',
                returnStdout: true
            ).trim()
        }
        
        stage('Test') {
            docker.image('golang:1.21').inside {
                sh 'go test ./...'
            }
        }
        
        stage('Build') {
            def image = docker.build("myapp:${imageTag}")
            
            docker.withRegistry('https://registry.example.com', 'registry-credentials') {
                image.push(imageTag)
                image.push('latest')
            }
        }
        
    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        stage('Notify') {
            def status = currentBuild.result ?: 'SUCCESS'
            slackSend channel: '#ci-cd', message: "Build ${status}: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
    }
}
```

### 4.3 Shared Libraries

```groovy
// vars/buildDockerImage.groovy (Shared Library)
def call(Map config = [:]) {
    def registry = config.registry ?: 'registry.example.com'
    def imageName = config.imageName ?: error('imageName is required')
    def imageTag = config.imageTag ?: 'latest'
    
    container('kaniko') {
        sh """
            /kaniko/executor \
              --context=. \
              --dockerfile=${config.dockerfile ?: 'Dockerfile'} \
              --destination=${registry}/${imageName}:${imageTag} \
              --cache=true
        """
    }
}

// vars/deployToKubernetes.groovy
def call(Map config = [:]) {
    def namespace = config.namespace ?: 'default'
    def deployment = config.deployment ?: error('deployment is required')
    def image = config.image ?: error('image is required')
    def timeout = config.timeout ?: '5m'
    
    container('kubectl') {
        sh """
            kubectl set image deployment/${deployment} \
              ${deployment}=${image} \
              -n ${namespace}
            
            kubectl rollout status deployment/${deployment} \
              -n ${namespace} \
              --timeout=${timeout}
        """
    }
}

// Jenkinsfile ที่ใช้ Shared Library
@Library('my-shared-library') _

pipeline {
    agent {
        kubernetes {
            yaml libraryResource('pod-templates/full-stack.yaml')
        }
    }
    
    stages {
        stage('Build') {
            steps {
                buildDockerImage(
                    imageName: 'myapp',
                    imageTag: env.BUILD_NUMBER
                )
            }
        }
        
        stage('Deploy') {
            steps {
                deployToKubernetes(
                    namespace: 'production',
                    deployment: 'myapp',
                    image: "registry.example.com/myapp:${env.BUILD_NUMBER}"
                )
            }
        }
    }
}
```

## 5. Jenkins Plugins สำคัญ

### 5.1 ติดตั้ง Plugins

```groovy
// JCasC Plugin Installation
jenkins:
  installPlugins:
    - kubernetes:3900.va_dce992317b_4
    - workflow-aggregator:596.v8c21c963d92d
    - docker-workflow:572.v950f58993843
    - git:5.2.0
    - credentials-binding:657.v2b_19db_7d6e6d
    - blueocean:1.27.11
    - sonar:2.15
    - slack:683.v96b_372b_d08a_c
    - junit:1.64
    - htmlpublisher:1.32
    - build-timeout:1.31
    - timestamper:1.26
    - pipeline-stage-view:2.33
    - prometheus:722.vd09519c75cfa
```

### 5.2 Blue Ocean Pipeline View

Blue Ocean ให้ UI ที่สวยงามกว่า Jenkins Classic สำหรับดู Pipeline:

```bash
# URL สำหรับเข้า Blue Ocean
https://jenkins.example.com/blue
```

### 5.3 Prometheus Metrics

```yaml
# prometheus-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: jenkins
  namespace: monitoring
  labels:
    release: prometheus
spec:
  endpoints:
  - interval: 30s
    path: /prometheus
    port: http
  namespaceSelector:
    matchNames:
    - jenkins
  selector:
    matchLabels:
      app: jenkins
```

## 6. Jenkins Security

### 6.1 Matrix-based Security

```yaml
# JCasC Security Configuration
jenkins:
  authorizationStrategy:
    projectMatrix:
      permissions:
      - "Overall/Administer:admin"
      - "Overall/Read:authenticated"
      - "Job/Build:developers"
      - "Job/Read:authenticated"
      - "Job/Workspace:developers"
```

### 6.2 Credentials Management

```groovy
// ใช้ withCredentials ใน Pipeline
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'registry-credentials',
                        usernameVariable: 'REGISTRY_USER',
                        passwordVariable: 'REGISTRY_PASSWORD'
                    ),
                    file(
                        credentialsId: 'kubeconfig-prod',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    sh '''
                        docker login -u $REGISTRY_USER -p $REGISTRY_PASSWORD registry.example.com
                        kubectl --kubeconfig=$KUBECONFIG apply -f k8s/
                    '''
                }
            }
        }
    }
}
```

## 7. Workshop: Build และ Deploy ด้วย Jenkins

### Workshop Overview

ในการ Workshop นี้เราจะ:
1. ติดตั้ง Jenkins บน Kubernetes
2. Configure Kubernetes Plugin
3. สร้าง Jenkinsfile สำหรับ Go Application
4. Build Docker Image
5. Deploy ไปยัง Kubernetes

### Step 1: เตรียม Application

```bash
# สร้าง Application Directory
mkdir -p jenkins-workshop/cmd/server
cd jenkins-workshop

# สร้าง main.go
cat > cmd/server/main.go << 'EOF'
package main

import (
    "encoding/json"
    "log"
    "net/http"
    "os"
)

type Response struct {
    Message string `json:"message"`
    Version string `json:"version"`
    Env     string `json:"env"`
}

func main() {
    version := os.Getenv("APP_VERSION")
    if version == "" {
        version = "development"
    }
    
    env := os.Getenv("ENVIRONMENT")
    if env == "" {
        env = "local"
    }
    
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        resp := Response{
            Message: "Hello from Jenkins Workshop!",
            Version: version,
            Env:     env,
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(resp)
    })
    
    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })
    
    log.Printf("Server starting on :8080 (version: %s, env: %s)", version, env)
    if err := http.ListenAndServe(":8080", nil); err != nil {
        log.Fatal(err)
    }
}
EOF

# สร้าง go.mod
cat > go.mod << 'EOF'
module github.com/myorg/jenkins-workshop

go 1.21
EOF

# สร้าง main_test.go
cat > cmd/server/main_test.go << 'EOF'
package main

import (
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestHealthEndpoint(t *testing.T) {
    req, err := http.NewRequest("GET", "/health", nil)
    if err != nil {
        t.Fatal(err)
    }
    
    rr := httptest.NewRecorder()
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })
    
    handler.ServeHTTP(rr, req)
    
    if rr.Code != http.StatusOK {
        t.Errorf("Expected status 200, got %d", rr.Code)
    }
    
    if rr.Body.String() != "OK" {
        t.Errorf("Expected body 'OK', got '%s'", rr.Body.String())
    }
}

func TestMainEndpoint(t *testing.T) {
    req, err := http.NewRequest("GET", "/", nil)
    if err != nil {
        t.Fatal(err)
    }
    
    rr := httptest.NewRecorder()
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        resp := Response{
            Message: "Hello from Jenkins Workshop!",
            Version: "test",
            Env:     "test",
        }
        w.Header().Set("Content-Type", "application/json")
        json.NewEncoder(w).Encode(resp)
    })
    
    handler.ServeHTTP(rr, req)
    
    if rr.Code != http.StatusOK {
        t.Errorf("Expected status 200, got %d", rr.Code)
    }
    
    var resp Response
    if err := json.NewDecoder(rr.Body).Decode(&resp); err != nil {
        t.Fatal(err)
    }
    
    if resp.Message != "Hello from Jenkins Workshop!" {
        t.Errorf("Unexpected message: %s", resp.Message)
    }
}
EOF
```

### Step 2: สร้าง Dockerfile

```dockerfile
# Dockerfile
FROM golang:1.21-alpine AS builder

WORKDIR /app

# ดาวน์โหลด dependencies ก่อน (layer caching)
COPY go.mod go.sum* ./
RUN go mod download

# Copy source code
COPY . .

# Build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-w -s" \
    -o server ./cmd/server/

# Final image
FROM scratch

# Copy binary
COPY --from=builder /app/server /server

# Copy SSL certs
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 8080

ENTRYPOINT ["/server"]
```

### Step 3: สร้าง Kubernetes Manifests

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: workshop

---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins-workshop
  namespace: workshop
  labels:
    app: jenkins-workshop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: jenkins-workshop
  template:
    metadata:
      labels:
        app: jenkins-workshop
    spec:
      containers:
      - name: app
        image: registry.example.com/jenkins-workshop:latest
        ports:
        - containerPort: 8080
        env:
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: environment
        - name: APP_VERSION
          value: "REPLACE_ME"  # Jenkins จะ replace
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5

---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins-workshop
  namespace: workshop
spec:
  selector:
    app: jenkins-workshop
  ports:
  - port: 80
    targetPort: 8080

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: workshop
data:
  environment: workshop
```

### Step 4: สร้าง Jenkinsfile

```groovy
// Jenkinsfile
pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: jenkins-agent
spec:
  serviceAccountName: jenkins
  containers:
  - name: go
    image: golang:1.21-alpine
    command: [cat]
    tty: true
    resources:
      requests:
        cpu: 500m
        memory: 512Mi
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: [/busybox/cat]
    tty: true
    volumeMounts:
    - name: registry-credentials
      mountPath: /kaniko/.docker
  - name: kubectl
    image: bitnami/kubectl:1.28
    command: [cat]
    tty: true
  volumes:
  - name: registry-credentials
    secret:
      secretName: registry-credentials
      items:
      - key: .dockerconfigjson
        path: config.json
"""
        }
    }
    
    environment {
        REGISTRY = 'registry.example.com'
        IMAGE = "${REGISTRY}/jenkins-workshop"
        GIT_SHA = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        IMAGE_TAG = "${env.BUILD_NUMBER}-${GIT_SHA}"
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
    }
    
    stages {
        stage('Lint') {
            steps {
                container('go') {
                    sh '''
                        # ติดตั้ง golangci-lint
                        wget -O golangci-lint.sh https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh
                        sh golangci-lint.sh -b /usr/local/bin v1.55.2
                        
                        # รัน lint
                        golangci-lint run ./...
                    '''
                }
            }
        }
        
        stage('Test') {
            steps {
                container('go') {
                    sh '''
                        go test ./... -v -coverprofile=coverage.out 2>&1 | tee test-output.txt
                        
                        # แสดง Coverage
                        go tool cover -func=coverage.out | tail -1
                        
                        # แปลงเป็น HTML
                        go tool cover -html=coverage.out -o coverage.html
                    '''
                }
            }
            post {
                always {
                    publishHTML([
                        reportDir: '.',
                        reportFiles: 'coverage.html',
                        reportName: 'Go Coverage Report'
                    ])
                }
            }
        }
        
        stage('Build Image') {
            steps {
                container('kaniko') {
                    sh """
                        /kaniko/executor \
                          --context=. \
                          --dockerfile=Dockerfile \
                          --destination=${IMAGE}:${IMAGE_TAG} \
                          --destination=${IMAGE}:latest \
                          --cache=true
                    """
                }
            }
        }
        
        stage('Deploy to Workshop') {
            steps {
                container('kubectl') {
                    sh """
                        # Apply namespace and config
                        kubectl apply -f k8s/namespace.yaml
                        kubectl apply -f k8s/configmap.yaml
                        
                        # Update image and version
                        sed -i 's|image: .*jenkins-workshop:.*|image: ${IMAGE}:${IMAGE_TAG}|g' k8s/deployment.yaml
                        sed -i 's|value: "REPLACE_ME"|value: "${IMAGE_TAG}"|g' k8s/deployment.yaml
                        
                        # Apply
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        
                        # รอ Rollout
                        kubectl rollout status deployment/jenkins-workshop -n workshop --timeout=5m
                        
                        # แสดง Pod status
                        kubectl get pods -n workshop
                    """
                }
            }
        }
        
        stage('Smoke Test') {
            steps {
                container('go') {
                    sh '''
                        # รอให้ Service พร้อม
                        sleep 10
                        
                        # ทดสอบ Endpoint
                        kubectl port-forward svc/jenkins-workshop 8080:80 -n workshop &
                        sleep 3
                        
                        # Test health endpoint
                        curl -f http://localhost:8080/health || exit 1
                        echo "Health check passed!"
                        
                        # Test main endpoint
                        RESPONSE=$(curl -s http://localhost:8080/)
                        echo "Response: $RESPONSE"
                        
                        # Verify response
                        echo $RESPONSE | grep "Jenkins Workshop" || exit 1
                        echo "Smoke test passed!"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo "Pipeline completed successfully!"
            echo "Image: ${IMAGE}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed!"
        }
        always {
            cleanWs()
        }
    }
}
```

### Step 5: รัน Workshop

```bash
# 1. ติดตั้ง Jenkins
kubectl apply -f jenkins-namespace.yaml
kubectl apply -f jenkins-rbac.yaml
kubectl apply -f jenkins-pvc.yaml
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f jenkins-service.yaml

# 2. รอ Jenkins พร้อม
kubectl wait --for=condition=ready pod -l app=jenkins -n jenkins --timeout=120s

# 3. ดู Initial Password
kubectl exec -n jenkins -it \
  $(kubectl get pods -n jenkins -l app=jenkins -o jsonpath='{.items[0].metadata.name}') \
  -- cat /var/jenkins_home/secrets/initialAdminPassword

# 4. เข้า Jenkins UI
kubectl port-forward svc/jenkins 8080:8080 -n jenkins

# 5. ใน Browser ไปที่ http://localhost:8080
# - ใส่ Initial Password
# - ติดตั้ง Suggested Plugins
# - สร้าง Admin User

# 6. Configure Kubernetes Cloud
# Manage Jenkins -> Nodes and Clouds -> Clouds -> Add Cloud -> Kubernetes

# 7. สร้าง Pipeline Job
# New Item -> Pipeline -> จาก SCM

# 8. Push Code และ Trigger Build
git init
git add .
git commit -m "Initial commit"
git remote add origin https://github.com/myorg/jenkins-workshop.git
git push -u origin main

# Jenkins จะ Auto-trigger Pipeline
```

### Step 6: ตรวจสอบผลลัพธ์

```bash
# ดู Pipeline Status
kubectl get pods -n jenkins -l jenkins=agent

# ดู Application
kubectl get pods -n workshop
kubectl get svc -n workshop

# ทดสอบ Application
kubectl port-forward svc/jenkins-workshop 8080:80 -n workshop
curl http://localhost:8080/
# Response: {"message":"Hello from Jenkins Workshop!","version":"5-abc1234","env":"workshop"}

# ดู Logs
kubectl logs -l app=jenkins-workshop -n workshop
```

## 8. Troubleshooting

### 8.1 ปัญหา: Agent Pod ไม่สร้าง

```bash
# ดู Jenkins Logs
kubectl logs -l app=jenkins -n jenkins --tail=100

# ดู RBAC ว่าถูกต้องไหม
kubectl auth can-i create pods --as system:serviceaccount:jenkins:jenkins -n jenkins

# ดู Pod Events
kubectl describe pod -l jenkins=agent -n jenkins
```

### 8.2 ปัญหา: Connection Timeout

```yaml
# เพิ่ม JVM Options
env:
- name: JAVA_OPTS
  value: >-
    -Xmx2g
    -Xms512m
    -Dorg.csanchez.jenkins.plugins.kubernetes.PodTemplate.connectTimeout=300
```

### 8.3 ปัญหา: Image Pull Error

```bash
# ตรวจสอบ Image Pull Secret
kubectl get secrets -n jenkins | grep registry

# สร้าง Secret ใหม่
kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.example.com \
  --docker-username=user \
  --docker-password=password \
  -n jenkins
```

## สรุป

Jenkins บน Kubernetes ให้:
1. **Dynamic Agents** - Pod สร้าง/ลบอัตโนมัติตาม Build
2. **Resource Efficiency** - ไม่ต้องจ่าย Idle Agents
3. **Pipeline as Code** - Jenkinsfile ใน Git
4. **Parallel Execution** - Build หลายอย่างพร้อมกัน
5. **Plugin Ecosystem** - Plugin มากมาย

ในบทต่อไปจะเรียนรู้เรื่อง GitHub Actions ซึ่งเป็น CI/CD ที่รวมอยู่กับ GitHub

## แบบฝึกหัด

1. ติดตั้ง Jenkins บน Kubernetes ด้วย Helm
2. สร้าง Declarative Pipeline สำหรับ Application ของคุณ
3. ทดลองใช้ Parallel Stages
4. ติดตั้ง Blue Ocean และทดลองดู Pipeline UI
5. สร้าง Shared Library สำหรับ Reusable Pipeline Functions
