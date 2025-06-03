# CI/CD Pipeline Strategy

## Visão Geral

Pipeline CI/CD automatizado para o microsserviço de dados com Oracle Database, focado em qualidade, segurança e deployments confiáveis.

## Arquitetura do Pipeline

```
┌─────────────────────────────────────────────────────────┐
│                Source Control (GitHub)                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Feature   │  │   Develop   │  │    Main     │     │
│  │   Branch    │  │   Branch    │  │   Branch    │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────┬───────────┬───────────┬───────────────────┘
              │           │           │
              ▼           ▼           ▼
┌─────────────────────────────────────────────────────────┐
│               GitHub Actions Pipelines                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │    PR       │  │   Staging   │  │ Production  │     │
│  │  Pipeline   │  │  Pipeline   │  │  Pipeline   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────┬───────────┬───────────┬───────────────────┘
              │           │           │
              ▼           ▼           ▼
┌─────────────────────────────────────────────────────────┐
│                 Environments                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │     Dev     │  │   Staging   │  │ Production  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

## 1. Pull Request Pipeline

### Workflow Principal
```yaml
# .github/workflows/pr-pipeline.yml
name: Pull Request Pipeline

on:
  pull_request:
    branches: [develop, main]

jobs:
  # Code Quality & Security (5 min)
  quality-check:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Setup Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
        cache: 'pip'
    
    - name: Install Dependencies
      run: pip install -r requirements.txt -r requirements-dev.txt
    
    - name: Code Quality
      run: |
        black --check app/ tests/
        flake8 app/ tests/ --max-line-length=127
        mypy app/ --ignore-missing-imports
    
    - name: Security Scan
      run: |
        bandit -r app/ -f json
        safety check --json

  # Unit Tests (10 min)
  unit-tests:
    runs-on: ubuntu-latest
    services:
      oracle:
        image: gvenzl/oracle-xe:21-slim
        env:
          ORACLE_PASSWORD: TestPassword123
        ports:
          - 1521:1521
        options: >-
          --health-cmd "sqlplus -s system/TestPassword123@//localhost:1521/XE <<< 'SELECT 1 FROM DUAL;'"
          --health-interval 30s
          --health-timeout 10s
          --health-retries 10
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
    
    steps:
    - uses: actions/checkout@v4
    - name: Setup Python & Oracle Client
      run: |
        # Install Oracle Instant Client
        wget -q https://download.oracle.com/otn_software/linux/instantclient/2111000/instantclient-basic-linux.x64-21.11.0.0.0dbru.zip
        unzip -q instantclient-basic-linux.x64-21.11.0.0.0dbru.zip
        sudo mv instantclient_21_11 /opt/oracle/
        echo "/opt/oracle/instantclient_21_11" | sudo tee /etc/ld.so.conf.d/oracle.conf
        sudo ldconfig
        pip install -r requirements.txt -r requirements-dev.txt
    
    - name: Run Tests
      env:
        DATABASE_URL: oracle+cx_oracle://system:TestPassword123@localhost:1521/XE
        REDIS_URL: redis://localhost:6379/0
      run: |
        pytest tests/unit/ -v --cov=app --cov-report=xml --cov-fail-under=80
    
    - name: Upload Coverage
      uses: codecov/codecov-action@v3

  # Build Validation (5 min)
  build-test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Build Docker Image
      run: |
        docker build -f docker/Dockerfile -t test-image .
        docker run --rm test-image python -c "import app.main; print('Build OK')"
```

## 2. Staging Pipeline

### Deploy para Staging
```yaml
# .github/workflows/staging-pipeline.yml
name: Staging Pipeline

on:
  push:
    branches: [develop]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    environment: staging
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Generate Version
      id: version
      run: echo "version=$(date +%Y%m%d)-${GITHUB_SHA::8}" >> $GITHUB_OUTPUT
    
    - name: Build and Push Image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./docker/Dockerfile
        push: true
        tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Deploy to Staging
      run: |
        # Configure kubectl
        aws eks update-kubeconfig --name staging-cluster --region us-east-1
        
        # Run database migrations
        kubectl apply -f k8s/staging/migration-job.yaml
        kubectl wait --for=condition=complete job/oracle-migration --timeout=300s
        
        # Update deployment
        kubectl set image deployment/data-microservice app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }} -n staging
        kubectl rollout status deployment/data-microservice -n staging --timeout=300s
        
        # Verify deployment
        kubectl wait --for=condition=ready pod -l app=data-microservice -n staging --timeout=300s
    
    - name: Run Smoke Tests
      run: python tests/smoke/staging_smoke_tests.py
```

## 3. Production Pipeline

### Deploy para Produção
```yaml
# .github/workflows/production-pipeline.yml
name: Production Pipeline

on:
  push:
    branches: [main]
  workflow_dispatch:
    inputs:
      deployment_strategy:
        type: choice
        options: [rolling, blue-green, canary]
        default: rolling

jobs:
  # Security & Approval
  pre-production:
    runs-on: ubuntu-latest
    steps:
    - name: Security Scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:staging-latest'
        format: 'table'
        exit-code: '1'
        severity: 'CRITICAL,HIGH'
    
    - name: Manual Approval
      if: github.event_name == 'workflow_dispatch'
      uses: trstringer/manual-approval@v1
      with:
        secret: ${{ github.TOKEN }}
        approvers: devops-team,tech-leads
        minimum-approvals: 2

  # Production Deployment
  production-deploy:
    needs: pre-production
    runs-on: ubuntu-latest
    environment: production
    
    steps:
    - uses: actions/checkout@v4
    
    - name: Generate Production Version
      id: version
      run: |
        if [[ "${{ github.event_name }}" == "release" ]]; then
          echo "version=${{ github.event.release.tag_name }}" >> $GITHUB_OUTPUT
        else
          echo "version=$(date +%Y%m%d)-${GITHUB_SHA::8}" >> $GITHUB_OUTPUT
        fi
    
    - name: Build Production Image
      uses: docker/build-push-action@v5
      with:
        context: .
        file: ./docker/Dockerfile
        push: true
        tags: |
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }}
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:production-latest
    
    - name: Database Migration
      run: |
        # Configure kubectl for production
        aws eks update-kubeconfig --name production-cluster --region us-east-1
        
        # Backup database
        kubectl create job --from=cronjob/oracle-backup oracle-backup-pre-migration -n production
        kubectl wait --for=condition=complete job/oracle-backup-pre-migration --timeout=1800s
        
        # Run migrations
        envsubst < k8s/production/migration-job.yaml | kubectl apply -f -
        kubectl wait --for=condition=complete job/oracle-migration-${{ steps.version.outputs.version }} --timeout=1800s
    
    - name: Deploy Application
      env:
        STRATEGY: ${{ github.event.inputs.deployment_strategy || 'rolling' }}
      run: |
        case $STRATEGY in
          "blue-green")
            ./scripts/deploy-blue-green.sh ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }}
            ;;
          "canary")
            ./scripts/deploy-canary.sh ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }}
            ;;
          *)
            kubectl set image deployment/data-microservice app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }} -n production
            kubectl rollout status deployment/data-microservice -n production --timeout=600s
            ;;
        esac
    
    - name: Post-Deploy Validation
      run: |
        # Health checks
        kubectl wait --for=condition=ready pod -l app=data-microservice -n production --timeout=300s
        
        # Application tests
        python tests/smoke/production_smoke_tests.py
        
        # Performance check
        python scripts/check_performance.py --threshold 500ms
    
    - name: Rollback on Failure
      if: failure()
      run: |
        kubectl rollout undo deployment/data-microservice -n production
        kubectl rollout status deployment/data-microservice -n production --timeout=300s
```

## 4. Deployment Scripts

### Blue/Green Deployment
```bash
#!/bin/bash
# scripts/deploy-blue-green.sh
IMAGE_TAG=$1
NAMESPACE="production"

# Determine current/new environment
CURRENT=$(kubectl get service data-microservice-service -n $NAMESPACE -o jsonpath='{.spec.selector.version}')
NEW=$([ "$CURRENT" = "blue" ] && echo "green" || echo "blue")

echo "Deploying to $NEW environment..."

# Deploy new version
kubectl set image deployment/data-microservice-$NEW app=$IMAGE_TAG -n $NAMESPACE
kubectl rollout status deployment/data-microservice-$NEW -n $NAMESPACE --timeout=600s

# Health check
kubectl wait --for=condition=ready pod -l app=data-microservice,version=$NEW -n $NAMESPACE --timeout=300s

# Switch traffic
kubectl patch service data-microservice-service -n $NAMESPACE -p '{"spec":{"selector":{"version":"'$NEW'"}}}'

# Wait and validate
sleep 60
python tests/smoke/production_smoke_tests.py

# Scale down old environment
kubectl scale deployment data-microservice-$CURRENT --replicas=0 -n $NAMESPACE
```

## 5. Monitoramento e Alertas

### Pipeline Metrics
```yaml
# Prometheus metrics
pipeline_duration_seconds{pipeline="production"}
pipeline_success_rate{pipeline="production"}
deployment_frequency{environment="production"}

# Alerting rules
- alert: PipelineFailureRate
  expr: rate(pipeline_success_total[1h]) < 0.9
  for: 5m

- alert: DeploymentDurationHigh  
  expr: pipeline_duration_seconds > 1800
  annotations:
    summary: "Deployment taking too long"
```

### Rollback Strategy
```yaml
Automatic Rollback Triggers:
  - Health check failures
  - Error rate > 1%
  - Response time > 1000ms (P95)
  - Manual trigger via Slack: /rollback production

Rollback Process:
  1. kubectl rollout undo deployment/data-microservice -n production
  2. Verify health checks
  3. Notify incident response team
```

## 7. Otimizações e Best Practices

### Build Optimization
```yaml
Caching Strategy:
  - Docker layer caching
  - Dependency caching (pip, npm)
  - Oracle Client caching

Parallel Execution:
  - Matrix builds for different environments
  - Parallel test execution
  - Concurrent deployment validations
```
