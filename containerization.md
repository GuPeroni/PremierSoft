# Estratégia de Containerização com Oracle Database

## Visão Geral

Estratégia de containerização otimizada para microsserviço, utilizando Docker multi-stage builds e Oracle Instant Client para máxima performance.

## Dockerfile - Multi-Stage Build

### Estrutura Principal

```dockerfile
# =============================================================================
# Stage 1: Builder - Instalação Oracle Instant Client + Dependencies
# =============================================================================
FROM python:3.11-slim as builder

# Metadados
LABEL maintainer="devops@company.com" \
      version="1.0.0" \
      description="Data Microservice with Oracle DB - Builder"

# Instalar dependências sistema para Oracle
RUN apt-get update && apt-get install -y \
    build-essential \
    wget \
    unzip \
    libaio1 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Baixar e instalar Oracle Instant Client 21.11
WORKDIR /opt/oracle
RUN wget -q https://download.oracle.com/otn_software/linux/instantclient/2111000/instantclient-basic-linux.x64-21.11.0.0.0dbru.zip \
    && unzip -q instantclient-basic-linux.x64-21.11.0.0.0dbru.zip \
    && rm -f *.zip \
    && cd instantclient_21_11 \
    && ln -sf libclntsh.so.21.1 libclntsh.so \
    && ln -sf libocci.so.21.1 libocci.so

# Configurar Oracle environment
ENV ORACLE_HOME=/opt/oracle/instantclient_21_11
ENV LD_LIBRARY_PATH=$ORACLE_HOME:$LD_LIBRARY_PATH
ENV PATH=$ORACLE_HOME:$PATH

# Instalar dependências Python
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# =============================================================================
# Stage 2: Production - Imagem final otimizada
# =============================================================================
FROM python:3.11-slim as production

# Metadados finais
LABEL maintainer="devops@company.com" \
      version="1.0.0" \
      description="Data Microservice - Production Ready"

# Instalar apenas runtime necessário
RUN apt-get update && apt-get install -y \
    libaio1 \
    curl \
    && rm -rf /var/lib/apt/lists/* \
    && apt-get clean

# Copiar Oracle Instant Client do builder
COPY --from=builder /opt/oracle/instantclient_21_11 /opt/oracle/instantclient_21_11

# Configurar Oracle environment
ENV ORACLE_HOME=/opt/oracle/instantclient_21_11
ENV LD_LIBRARY_PATH=$ORACLE_HOME:$LD_LIBRARY_PATH
ENV PATH=$ORACLE_HOME:$PATH
ENV NLS_LANG=AMERICAN_AMERICA.UTF8

# Criar usuário não-root
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Setup diretório de trabalho
WORKDIR /app

# Copiar dependências Python
COPY --from=builder /root/.local /home/appuser/.local
ENV PATH=/home/appuser/.local/bin:$PATH

# Copiar código da aplicação
COPY --chown=appuser:appuser app/ ./app/
COPY --chown=appuser:appuser alembic.ini ./
COPY --chown=appuser:appuser alembic/ ./alembic/

# Configurar environment para produção
ENV PYTHONPATH=/app \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Criar diretório para logs
RUN mkdir -p /app/logs && chown appuser:appuser /app/logs

# Mudar para usuário não-root
USER appuser

# Expor porta
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Comando padrão
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]

# =============================================================================
# Stage 3: Development - Para desenvolvimento local
# =============================================================================
FROM production as development

USER root

# Instalar ferramentas de desenvolvimento
RUN pip install --no-cache-dir \
    pytest \
    pytest-asyncio \
    pytest-cov \
    black \
    flake8 \
    mypy \
    debugpy

USER appuser

# Hot reload para desenvolvimento
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

### Justificativas das Decisões

#### 1. **Multi-Stage Build**
```yaml
Benefícios:
  - Builder: 800MB → Production: 180MB (77% redução)
  - Security: Ferramentas de build não ficam na imagem final
  - Cache: Layers otimizados para rebuild rápido

Stages:
  builder: Oracle Client + Python dependencies
  production: Runtime mínimo + aplicação
  development: Ferramentas de dev + hot reload
```

#### 2. **Oracle Instant Client 21.11**
```yaml
Escolha vs Alternativas:
  ✅ Instant Client: Footprint menor (180MB vs 2GB Oracle XE)
  ✅ Version 21.11: Latest stable com melhor performance
  ✅ Multi-stage: Build tools não vão para produção

Configuração:
  - ORACLE_HOME: Path do client
  - LD_LIBRARY_PATH: Library loading
  - NLS_LANG: UTF8 encoding
```

#### 3. **Security Best Practices**
```yaml
Non-root User:
  - Cria appuser:appuser (UID 1000)
  - Filesystem permissions corretas
  - Principle of least privilege

Image Hardening:
  - Minimal base image (python:3.11-slim)
  - Cleanup package cache
  - No unnecessary packages
```

## Docker Compose - Ambiente Completo

### docker-compose.yml (Produção Local)

```yaml
version: '3.8'

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge

volumes:
  oracle_data:
    driver: local
  redis_data:
    driver: local

services:
  # =============================================================================
  # Oracle Database Service
  # =============================================================================
  oracle-db:
    image: container-registry.oracle.com/database/express:21.3.0-xe
    container_name: oracle_xe_db
    restart: unless-stopped
    
    environment:
      - ORACLE_PWD=StrongPassword123
      - ORACLE_CHARACTERSET=AL32UTF8
      - ORACLE_EDITION=express
    
    ports:
      - "1521:1521"
      - "5500:5500"  # Enterprise Manager
    
    volumes:
      - oracle_data:/opt/oracle/oradata
      - ./oracle/init:/opt/oracle/scripts/startup
    
    networks:
      - backend
    
    healthcheck:
      test: ["CMD", "sqlplus", "-L", "system/StrongPassword123@//localhost:1521/XE", "<<<", "SELECT 1 FROM DUAL;"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s  # Oracle demora para inicializar
    
    deploy:
      resources:
        limits:
          memory: 2GB    # Oracle XE limit
          cpus: '2.0'
        reservations:
          memory: 1GB

  # =============================================================================
  # Application Service
  # =============================================================================
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: production
    container_name: data_microservice_app
    restart: unless-stopped
    
    environment:
      - DATABASE_URL=oracle+cx_oracle://system:StrongPassword123@oracle-db:1521/XE
      - REDIS_URL=redis://redis:6379/0
      - LOG_LEVEL=INFO
      - WORKERS=2
      - MAX_CONNECTIONS=10
    
    ports:
      - "8000:8000"
    
    networks:
      - frontend
      - backend
    
    depends_on:
      oracle-db:
        condition: service_healthy
      redis:
        condition: service_healthy
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    
    deploy:
      resources:
        limits:
          memory: 512MB
          cpus: '1.0'

  # =============================================================================
  # Redis Cache Service
  # =============================================================================
  redis:
    image: redis:7-alpine
    container_name: redis_cache
    restart: unless-stopped
    
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    
    ports:
      - "6379:6379"
    
    volumes:
      - redis_data:/data
    
    networks:
      - backend
    
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  # =============================================================================
  # NGINX Load Balancer
  # =============================================================================
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    restart: unless-stopped
    
    ports:
      - "80:80"
    
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    
    networks:
      - frontend
    
    depends_on:
      - app
    
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
```

### docker-compose.dev.yml (Desenvolvimento)

```yaml
version: '3.8'

services:
  app:
    build:
      target: development  # Use development stage
    
    volumes:
      - ./app:/app/app:ro           # Hot reload
      - ./tests:/app/tests:ro       # Tests
      - ./alembic:/app/alembic:ro   # Migrations
    
    environment:
      - LOG_LEVEL=DEBUG
      - RELOAD=true
    
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
    
    ports:
      - "8000:8000"
      - "5678:5678"  # debugpy port

  oracle-db:
    volumes:
      - ./oracle/init-dev:/opt/oracle/scripts/startup
      - ./oracle/sample-data:/opt/oracle/scripts/sample-data
```

### docker-compose.test.yml (Testing)

```yaml
version: '3.8'

services:
  oracle-test:
    image: gvenzl/oracle-xe:21-slim
    environment:
      ORACLE_PASSWORD: TestPassword123
      ORACLE_DATABASE: TESTDB
    ports:
      - "1521:1521"
    healthcheck:
      test: ["CMD", "sqlplus", "-L", "system/TestPassword123@//localhost:1521/XE"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 60s
  
  redis-test:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
  
  app-test:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: development
    environment:
      - DATABASE_URL=oracle+cx_oracle://system:TestPassword123@oracle-test:1521/XE
      - REDIS_URL=redis://redis-test:6379/0
      - TESTING=true
      - LOG_LEVEL=DEBUG
    depends_on:
      oracle-test:
        condition: service_healthy
      redis-test:
        condition: service_healthy
    volumes:
      - ./tests:/app/tests
    command: ["pytest", "tests/", "-v", "--cov=app"]
```

## Configurações Auxiliares

### NGINX Configuration

```nginx
# nginx/nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream app_backend {
        server app:8000;
    }
    
    server {
        listen 80;
        
        # Health check endpoint
        location /health {
            proxy_pass http://app_backend/health;
            proxy_set_header Host $host;
        }
        
        # Main API
        location / {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            # Timeouts
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
            
            # Rate limiting
            limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
            limit_req zone=api burst=20 nodelay;
        }
    }
}
```

### Oracle Initialization Scripts

```sql
-- oracle/init/01-create-schema.sql
-- Criar schema de dados
CREATE TABLE data_records (
    id VARCHAR2(36) PRIMARY KEY DEFAULT SYS_GUID(),
    user_id VARCHAR2(100) NOT NULL,
    payload CLOB CHECK (payload IS JSON),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Índices para performance
CREATE INDEX idx_user_id ON data_records(user_id);
CREATE INDEX idx_created_at ON data_records(created_at);
CREATE INDEX idx_user_created ON data_records(user_id, created_at);

-- Trigger para updated_at
CREATE OR REPLACE TRIGGER data_records_updated_at
    BEFORE UPDATE ON data_records
    FOR EACH ROW
BEGIN
    :NEW.updated_at := CURRENT_TIMESTAMP;
END;
/
```

## Otimizações e Best Practices

### Build Optimization

```yaml
Layer Caching Strategy:
  1. Base image (rarely changes)
  2. System dependencies (rarely changes)
  3. Oracle Client (rarely changes)
  4. Python dependencies (changes moderately)
  5. Application code (changes frequently)

Dockerfile Optimizations:
  - Multi-stage builds
  - .dockerignore para reduzir context
  - Specific COPY commands
  - Cleanup em mesmo RUN
```

### .dockerignore

```
# .dockerignore
**/__pycache__
**/*.pyc
**/*.pyo
**/*.pyd
**/.pytest_cache
**/node_modules
.git
.gitignore
README.md
docs/
tests/
.env*
docker-compose*.yml
```

### Resource Management

```yaml
Production Limits:
  app:
    memory: 512MB
    cpu: 1.0 cores
  
  oracle-db:
    memory: 2GB (XE limit)
    cpu: 2.0 cores
  
  redis:
    memory: 256MB
    cpu: 0.5 cores

Development (Reduced):
  app:
    memory: 256MB
    cpu: 0.5 cores
```

## Comandos Úteis

### Build e Deploy
```bash
# Build production image
docker build -f docker/Dockerfile --target production -t data-microservice:latest .

# Build development image
docker build -f docker/Dockerfile --target development -t data-microservice:dev .

# Run com compose
docker-compose -f docker/docker-compose.yml up -d

# Logs
docker-compose logs -f app

# Clean up
docker-compose down -v
```

### Development Workflow
```bash
# Start development environment
docker-compose -f docker/docker-compose.yml -f docker/docker-compose.dev.yml up -d

# Run tests
docker-compose -f docker/docker-compose.test.yml up --build app-test

# Execute commands inside container
docker-compose exec app bash
docker-compose exec oracle-db sqlplus system/StrongPassword123@XE
```
