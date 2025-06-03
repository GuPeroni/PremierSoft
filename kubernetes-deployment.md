# Kubernetes Deployment Strategy with Oracle Database

## Visão Geral

Esta documentação detalha a estratégia completa de deployment no Kubernetes para o microsserviço de dados com Oracle Database, incluindo configurações de alta disponibilidade, auto-scaling e estratégias de deployment enterprise.

## Arquitetura Kubernetes

```
┌─────────────────────────────────────────────────────────┐
│  # Oracle Database credentials (base64 encoded)
  ORACLE_USERNAME: c3lzdGVt  # system
  ORACLE_PASSWORD: U3Ryb25nUGFzc3dvcmQxMjM=  # StrongPassword123
  ORACLE_HOST: b3JhY2xlLXNlcnZpY2U=  # oracle-service
  ORACLE_PORT: MTUyMQ==  # 1521
  ORACLE_SERVICE_NAME: WEU=  # XE
  DATABASE_URL: b3JhY2xlK2N4X29yYWNsZTovL3N5c3RlbTpTdHJvbmdQYXNzd29yZDEyM0BvcmFjbGUtc2VydmljZToxNTIxL1hF

---
# TLS Secret para HTTPS
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: data-microservice
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi... # Base64 encoded certificate
  tls.key: LS0tLS1CRUdJTi... # Base64 encoded private key
```

### 3. Oracle Database StatefulSet

```yaml
# k8s/oracle-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: oracle-db
  namespace: data-microservice
  labels:
    app: oracle-db
spec:
  serviceName: oracle-service
  replicas: 1  # Primary instance
  selector:
    matchLabels:
      app: oracle-db
  template:
    metadata:
      labels:
        app: oracle-db
    spec:
      serviceAccountName: data-microservice-sa
      securityContext:
        fsGroup: 54321  # Oracle user group
      containers:
      - name: oracle-db
        image: container-registry.oracle.com/database/express:21.3.0-xe
        imagePullPolicy: IfNotPresent
        
        # Resource limits para Oracle XE
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "2Gi"    # Oracle XE limit
            cpu: "2000m"
        
        # Environment variables
        env:
        - name: ORACLE_PWD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: ORACLE_PASSWORD
        - name: ORACLE_CHARACTERSET
          value: "AL32UTF8"
        - name: ORACLE_EDITION
          value: "express"
        
        # Ports
        ports:
        - containerPort: 1521
          name: oracle-db
        - containerPort: 5500
          name: oracle-em
        
        # Volume mounts
        volumeMounts:
        - name: oracle-data
          mountPath: /opt/oracle/oradata
        - name: oracle-init
          mountPath: /opt/oracle/scripts/startup
        
        # Startup probe (Oracle demora para inicializar)
        startupProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - "echo 'SELECT 1 FROM DUAL;' | sqlplus -s system/${ORACLE_PWD}@localhost:1521/XE"
          initialDelaySeconds: 120
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 10
        
        # Liveness probe
        livenessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - "echo 'SELECT 1 FROM DUAL;' | sqlplus -s system/${ORACLE_PWD}@localhost:1521/XE"
          initialDelaySeconds: 180
          periodSeconds: 60
          timeoutSeconds: 10
          failureThreshold: 3
        
        # Readiness probe
        readinessProbe:
          exec:
            command:
            - /bin/bash
            - -c
            - "echo 'SELECT 1 FROM DUAL;' | sqlplus -s system/${ORACLE_PWD}@localhost:1521/XE"
          initialDelaySeconds: 60
          periodSeconds: 30
          timeoutSeconds: 10
      
      # Init container para preparar dados
      initContainers:
      - name: oracle-init
        image: busybox:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "Preparing Oracle initialization scripts..."
          # Copiar scripts de inicialização se necessário
        volumeMounts:
        - name: oracle-init
          mountPath: /opt/oracle/scripts/startup
      
      # Volumes
      volumes:
      - name: oracle-init
        configMap:
          name: oracle-init-scripts

  # Volume Claim Templates
  volumeClaimTemplates:
  - metadata:
      name: oracle-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"  # Use SSD storage class
      resources:
        requests:
          storage: 20Gi

---
# Oracle Service
apiVersion: v1
kind: Service
metadata:
  name: oracle-service
  namespace: data-microservice
  labels:
    app: oracle-db
spec:
  selector:
    app: oracle-db
  ports:
  - name: oracle-db
    port: 1521
    targetPort: 1521
  - name: oracle-em
    port: 5500
    targetPort: 5500
  clusterIP: None  # Headless service para StatefulSet
```

### 4. Application Deployment

```yaml
# k8s/app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-microservice
  namespace: data-microservice
  labels:
    app: data-microservice
    version: v1.0.0
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero downtime deployment
  selector:
    matchLabels:
      app: data-microservice
  template:
    metadata:
      labels:
        app: data-microservice
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: data-microservice-sa
      
      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      # Init container para aguardar dependências
      initContainers:
      - name: wait-for-oracle
        image: busybox:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "Waiting for Oracle Database..."
          until nc -z oracle-service 1521; do
            echo "Oracle not ready, waiting..."
            sleep 5
          done
          echo "Oracle is ready!"
      
      - name: wait-for-redis
        image: busybox:latest
        command:
        - /bin/sh
        - -c
        - |
          echo "Waiting for Redis..."
          until nc -z redis-service 6379; do
            echo "Redis not ready, waiting..."
            sleep 2
          done
          echo "Redis is ready!"
      
      containers:
      - name: app
        image: data-microservice:latest
        imagePullPolicy: Always
        
        # Resource requirements
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Ports
        ports:
        - containerPort: 8000
          name: http
          protocol: TCP
        
        # Environment variables
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DATABASE_URL
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
        - name: WORKERS
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: WORKERS
        - name: MAX_CONNECTIONS
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: MAX_CONNECTIONS
        - name: REDIS_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_URL
        - name: NLS_LANG
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: NLS_LANG
        
        # Health checks
        startupProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 10
        
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
          timeoutSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        
        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        
        # Volume mounts
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: logs
          mountPath: /app/logs
      
      # Volumes
      volumes:
      - name: tmp
        emptyDir: {}
      - name: logs
        emptyDir: {}

---
# Application Service
apiVersion: v1
kind: Service
metadata:
  name: data-microservice-service
  namespace: data-microservice
  labels:
    app: data-microservice
spec:
  selector:
    app: data-microservice
  ports:
  - name: http
    port: 80
    targetPort: 8000
    protocol: TCP
  type: ClusterIP
```

### 5. Redis Deployment

```yaml
# k8s/redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: data-microservice
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        imagePullPolicy: IfNotPresent
        
        # Resource limits
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        
        # Ports
        ports:
        - containerPort: 6379
          name: redis
        
        # Redis configuration
        args:
        - redis-server
        - --appendonly
        - "yes"
        - --maxmemory
        - "200mb"
        - --maxmemory-policy
        - allkeys-lru
        
        # Health checks
        livenessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        
        readinessProbe:
          exec:
            command:
            - redis-cli
            - ping
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        
        # Volume mounts
        volumeMounts:
        - name: redis-data
          mountPath: /data
      
      volumes:
      - name: redis-data
        emptyDir: {}

---
# Redis Service
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: data-microservice
  labels:
    app: redis
spec:
  selector:
    app: redis
  ports:
  - name: redis
    port: 6379
    targetPort: 6379
  type: ClusterIP
```

### 6. Horizontal Pod Autoscaler

```yaml
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: data-microservice-hpa
  namespace: data-microservice
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-microservice
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
  # Custom metrics (se disponível)
  - type: Pods
    pods:
      metric:
        name: requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  
  # Behavior para controlar scaling
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      selectPolicy: Min
```

### 7. Ingress Configuration

```yaml
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: data-microservice-ingress
  namespace: data-microservice
  annotations:
    # NGINX Ingress Controller annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    
    # Request timeout
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "30"
    
    # Request size limits
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    
    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Content-Type, Authorization"
    
    # Health check
    nginx.ingress.kubernetes.io/health-check-path: "/health"
    
spec:
  tls:
  - hosts:
    - api.dataservice.com
    secretName: tls-secret
  
  rules:
  - host: api.dataservice.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: data-microservice-service
            port:
              number: 80
      
      # Health check endpoint
      - path: /health
        pathType: Exact
        backend:
          service:
            name: data-microservice-service
            port:
              number: 80
      
      # Metrics endpoint (interno apenas)
      - path: /metrics
        pathType: Exact
        backend:
          service:
            name: data-microservice-service
            port:
              number: 80
```

## Estratégias de Escalabilidade

### 1. Horizontal Pod Autoscaling (HPA)

**Configuração Multi-Métrica:**
```yaml
Metrics:
  CPU: 70% threshold
  Memory: 80% threshold
  Custom: requests_per_second > 100
  
Scaling Behavior:
  Scale Up: Max 100% ou 2 pods por minuto
  Scale Down: Max 10% por minuto (gradual)
  
Stabilization:
  Scale Up: 60 seconds
  Scale Down: 300 seconds (5 minutos)
```

**Justificativas:**
- **CPU 70%**: Margem para spikes sem degradação
- **Memory 80%**: Oracle client pode usar bastante memória
- **Custom metrics**: Business-specific scaling
- **Conservative scale-down**: Evita thrashing

### 2. Vertical Pod Autoscaling (VPA)

```yaml
# k8s/vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: data-microservice-vpa
  namespace: data-microservice
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: data-microservice
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 1000m
        memory: 1Gi
      controlledResources: ["cpu", "memory"]
```

### 3. Cluster Autoscaling

**Node Pool Configuration:**
```yaml
Node Pools:
  General Purpose:
    Instance Type: t3.medium - t3.large
    Min Nodes: 3
    Max Nodes: 10
    
  Oracle Database:
    Instance Type: m5.xlarge - m5.2xlarge
    Min Nodes: 1
    Max Nodes: 2
    Dedicated: true (para Oracle licensing)
    
  Burst Capacity:
    Instance Type: t3.micro - t3.small
    Min Nodes: 0
    Max Nodes: 5
    Spot Instances: true
```

### 4. Database Scaling Strategy

**Oracle Database Scaling:**
```yaml
# Read Replicas via Oracle Data Guard
Primary Database:
  Role: Read/Write
  Resources: 4 CPU, 16GB RAM
  
Standby Database:
  Role: Read-Only
  Resources: 2 CPU, 8GB RAM
  Sync: Real-time via Data Guard
  
Connection Routing:
  Write Operations: Primary only
  Read Operations: Load balance (Primary + Standby)
  Failover: Automatic via Data Guard
```

## Deployment Strategies

### 1. Blue/Green Deployment

```yaml
# Blue/Green strategy
Blue Environment:
  Deployment: data-microservice-blue
  Service: data-microservice-blue-service
  Traffic: 100% (current)
  
Green Environment:
  Deployment: data-microservice-green
  Service: data-microservice-green-service
  Traffic: 0% (new version)
  
Switch Process:
  1. Deploy green environment
  2. Health checks + smoke tests
  3. Switch ingress to green
  4. Monitor for 10 minutes
  5. Terminate blue environment
```

### 2. Canary Deployment

```yaml
# Canary strategy com Istio
Traffic Split:
  Stable Version: 90%
  Canary Version: 10%
  
Success Criteria:
  Error Rate: < 0.1%
  Latency P95: < 500ms
  Duration: 30 minutes
  
Auto Rollback:
  Error Rate: > 1%
  Latency P95: > 1000ms
  Custom Metrics: Business KPIs
```

### 3. Rolling Update (Default)

```yaml
Rolling Update Strategy:
  Max Surge: 1 pod (33% do total)
  Max Unavailable: 0 pods (zero downtime)
  
Process:
  1. Start new pod
  2. Wait for readiness probe
  3. Terminate old pod
  4. Repeat for all replicas
  
Rollback:
  Automatic: Health check failures
  Manual: kubectl rollout undo
```

## Monitoramento e Observabilidade

### 1. ServiceMonitor para Prometheus

```yaml
# k8s/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: data-microservice-metrics
  namespace: data-microservice
  labels:
    app: data-microservice
spec:
  selector:
    matchLabels:
      app: data-microservice
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
```

### 2. Oracle Database Monitoring

```yaml
# Oracle Exporter para Prometheus
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oracle-exporter
  namespace: data-microservice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oracle-exporter
  template:
    metadata:
      labels:
        app: oracle-exporter
    spec:
      containers:
      - name: oracle-exporter
        image: iamseth/oracledb_exporter:latest
        env:
        - name: DATA_SOURCE_NAME
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DATABASE_URL
        ports:
        - containerPort: 9161
          name: metrics
```

### 3. Alerting Rules

```yaml
# Prometheus AlertManager rules
groups:
- name: data-microservice
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
    for: 5m
    annotations:
      summary: "High error rate detected"
      
  - alert: OracleConnectionPoolExhausted
    expr: oracle_connection_pool_active / oracle_connection_pool_max > 0.9
    for: 2m
    annotations:
      summary: "Oracle connection pool near capacity"
      
  - alert: HighMemoryUsage
    expr: container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
    for: 5m
    annotations:
      summary: "Pod memory usage > 90%"
```

## Security Considerations

### 1. Network Policies

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: data-microservice-netpol
  namespace: data-microservice
spec:
  podSelector:
    matchLabels:
      app: data-microservice
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8000
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: oracle-db
    ports:
    - protocol: TCP
      port: 1521
  - to:
    - podSelector:
        matchLabels:
          app: redis
    ports:
    - protocol: TCP
      port: 6379
```

### 2. Pod Security Policy

```yaml
# k8s/pod-security-policy.yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: data-microservice-psp
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'emptyDir'
    - 'projected'
    - 'secret'
    - 'downwardAPI'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

## Backup e Disaster Recovery

### 1. Oracle Database Backup

```yaml
# CronJob para backup Oracle
apiVersion: batch/v1
kind: CronJob
metadata:
  name: oracle-backup
  namespace: data-microservice
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: oracle-backup
            image: oracle/database:19.3.0-ee
            command:
            - /bin/bash
            - -c
            - |
              rman target / <<EOF
              CONFIGURE RETENTION POLICY TO REDUNDANCY 3;
              BACKUP DATABASE PLUS ARCHIVELOG;
              DELETE NOPROMPT OBSOLETE;
              EOF
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
          restartPolicy: OnFailure
```

### 2. Application State Backup

```yaml
# Velero backup para aplicação
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: data-microservice-backup
  namespace: velero
spec:
  includedNamespaces:
  - data-microservice
  storageLocation: default
  ttl: 720h0m0s  # 30 days
  schedule: "0 2 * * *"  # Daily at 2 AM
```

Esta estratégia de deployment Kubernetes foi projetada para fornecer alta disponibilidade, escalabilidade e operação enterprise-grade com Oracle Database, incluindo todas as considerações de segurança, monitoramento e disaster recovery necessárias para um ambiente de produção.                  Internet                             │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                 Ingress Controller                      │
│               (NGINX/HAProxy)                           │
│  ┌─────────────────────────────────────────────────┐   │
│  │  SSL Termination + Load Balancing             │   │
│  │  - TLS 1.3                                     │   │
│  │  - Rate Limiting                               │   │
│  │  - Path-based Routing                          │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│              Kubernetes Services                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ App Service  │  │Oracle Service│  │Redis Service │ │
│  │ (ClusterIP)  │  │ (ClusterIP)  │  │ (ClusterIP)  │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
└─────────────────┬───────────────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────────────┐
│                Pod Layer                                │
│  ┌─────────────────────────────────────────────────┐   │
│  │          Application Pods (HPA)                 │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────┐ │   │
│  │  │    App      │  │    App      │  │   App   │ │   │
│  │  │   Pod 1     │  │   Pod 2     │  │  Pod 3  │ │   │
│  │  │             │  │             │  │         │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │          Database Pods (StatefulSet)            │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────┐ │   │
│  │  │   Oracle    │  │   Oracle    │  │  Redis  │ │   │
│  │  │  Primary    │  │  Standby    │  │  Cache  │ │   │
│  │  │             │  │ (Data Guard)│  │         │ │   │
│  │  └─────────────┘  └─────────────┘  └─────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────┬───────────────────────────────────────┘
```

## Manifestos Kubernetes

### 1. Namespace e RBAC

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: data-microservice
  labels:
    name: data-microservice
    app: data-service

---
# Service Account para aplicação
apiVersion: v1
kind: ServiceAccount
metadata:
  name: data-microservice-sa
  namespace: data-microservice

---
# ClusterRole para recursos necessários
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: data-microservice-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: data-microservice-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: data-microservice-role
subjects:
- kind: ServiceAccount
  name: data-microservice-sa
  namespace: data-microservice
```

### 2. ConfigMaps e Secrets

```yaml
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: data-microservice
data:
  LOG_LEVEL: "INFO"
  WORKERS: "2"
  MAX_CONNECTIONS: "20"
  REDIS_URL: "redis://redis-service:6379/0"
  ORACLE_CLIENT_VERSION: "21.11"
  NLS_LANG: "AMERICAN_AMERICA.UTF8"
  # Oracle connection pool settings
  POOL_SIZE: "10"
  POOL_MAX_OVERFLOW: "20"
  POOL_TIMEOUT: "30"
  POOL_RECYCLE: "3600"

---
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: data-microservice
type: Opaque
data:
  # Kubernetes Deployment Strategy
