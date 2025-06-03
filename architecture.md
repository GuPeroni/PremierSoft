# Arquitetura do Microsserviço de Dados

## Visão Geral da Arquitetura

### Padrão Arquitetural: Clean Architecture + Domain-Driven Design

A arquitetura foi projetada seguindo os princípios de Clean Architecture, com separação clara de responsabilidades e baixo acoplamento entre camadas.

```
┌─────────────────────────────────────────────────────────┐
│                 External Systems                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Client    │  │   K8s API   │  │ Monitoring  │     │
│  │   Apps      │  │   Server    │  │   Systems   │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                Infrastructure Layer                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │    NGINX    │  │  Kubernetes │  │ Prometheus  │     │
│  │ Load Balancer│  │  Ingress    │  │  + Grafana  │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                 Presentation Layer                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │              FastAPI Application                │   │
│  │                                                 │   │
│  │  ┌──────────────┐  ┌─────────────────────────┐ │   │
│  │  │   API        │  │     Middleware          │ │   │
│  │  │ Controllers  │  │ - Authentication        │ │   │
│  │  │              │  │ - Rate Limiting         │ │   │
│  │  │ - POST /data │  │ - Request Logging       │ │   │
│  │  │ - GET /data  │  │ - Error Handling        │ │   │
│  │  │ - Health     │  │ - CORS                  │ │   │
│  │  └──────────────┘  └─────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                 Application Layer                       │
│  ┌─────────────────────────────────────────────────┐   │
│  │                Service Layer                    │   │
│  │                                                 │   │
│  │  ┌──────────────┐  ┌─────────────────────────┐ │   │
│  │  │ Data Service │  │    Business Logic       │ │   │
│  │  │              │  │ - Validation Rules      │ │   │
│  │  │ - Create     │  │ - Rate Limiting Logic   │ │   │
│  │  │ - Retrieve   │  │ - Cache Strategy        │ │   │
│  │  │ - Validate   │  │ - Error Handling        │ │   │
│  │  │ - Cache      │  │ - Data Transformation   │ │   │
│  │  └──────────────┘  └─────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                 Domain Layer                            │
│  ┌─────────────────────────────────────────────────┐   │
│  │                 Domain Models                   │   │
│  │                                                 │   │
│  │  ┌──────────────┐  ┌─────────────────────────┐ │   │
│  │  │   Entities   │  │       Value Objects     │ │   │
│  │  │              │  │                         │ │   │
│  │  │ - DataRecord │  │ - UserId               │ │   │
│  │  │ - User       │  │ - Payload              │ │   │
│  │  │              │  │ - Timestamp            │ │   │
│  │  └──────────────┘  └─────────────────────────┘ │   │
│  │                                                 │   │
│  │  ┌─────────────────────────────────────────────┐ │   │
│  │  │            Repository Interfaces           │ │   │
│  │  │ - IDataRepository                          │ │   │
│  │  │ - ICacheRepository                         │ │   │
│  │  └─────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│               Infrastructure Layer                      │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Repository Layer                   │   │
│  │                                                 │   │
│  │  ┌──────────────┐  ┌─────────────────────────┐ │   │
│  │  │ Oracle       │      │        Redis        │ │   │
│  │  │ Repository   │  │    Cache Repository     │ │   │
│  │  │              │  │                         │ │   │
│  │  │ - CRUD Ops   │  │ - Get/Set/Delete       │ │   │
│  │  │ - Queries    │  │ - Rate Limiting        │ │   │
│  │  │ - Migrations │  │ - Session Storage      │ │   │
│  │  └──────────────┘  └─────────────────────────┘ │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                 Data Layer                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │   Oracle    │  │    Redis    │  │  File       │     │
│  │   Primary   │  │    Cache    │  │  Storage    │     │
│  │  Database   │  │             │  │  (Logs)     │     │
│  └─────────────┘  └─────────────┘  └─────────────┘     │
└─────────────────────────────────────────────────────────┘
```

## Componentes Detalhados

### 1. API Layer (Presentation)

**Responsabilidades:**
- Recebimento e validação de requests HTTP
- Serialização/deserialização de dados
- Autenticação e autorização
- Rate limiting
- Error handling e logging

**Tecnologias:**
- **FastAPI**: Framework principal
- **Pydantic**: Validação e serialização
- **Uvicorn**: ASGI server
- **Middleware customizado**: Rate limiting, CORS, logging

**Endpoints Principais:**
```
POST /data
├── Input: JSON payload + user_id
├── Validation: Pydantic schema
├── Rate limiting: 100 req/min per user
└── Output: Created data record

GET /data
├── Input: user_id (query param) + pagination
├── Cache check: Redis first
├── Database fallback: PostgreSQL
└── Output: Paginated data list

GET /health
├── Liveness probe
└── Output: {"status": "healthy"}

GET /ready
├── Readiness probe
├── Checks: DB connection + Redis connection
└── Output: {"status": "ready", "checks": {...}}
```

### 2. Service Layer (Application)

**Responsabilidades:**
- Orquestração de operações de negócio
- Implementação de regras de negócio
- Coordenação entre repositories
- Cache management
- Error handling específico do domínio

**Componentes Principais:**

#### DataService
```python
class DataService:
    - create_data(request: DataCreateRequest) -> DataResponse
    - get_user_data(user_id, page, page_size) -> (List[DataResponse], int)
    - validate_business_rules(data) -> bool
    - manage_cache_strategy() -> None
```

**Regras de Negócio Implementadas:**
- Validação de payload size (max 1MB)
- Rate limiting por usuário (100 req/min)
- Cache invalidation strategy
- Data consistency validation

### 3. Repository Layer (Infrastructure)

**Responsabilidades:**
- Abstração do acesso a dados
- Implementação de queries otimizadas
- Connection pooling
- Transaction management

#### DataRepository (Oracle Database)
```python
class DataRepository:
    - create(user_id: str, payload: dict) -> DataRecord
    - get_by_id(record_id: str) -> Optional[DataRecord]
    - get_by_user(user_id: str, page: int, page_size: int) -> (List[DataRecord], int)
    - get_by_date_range(start: datetime, end: datetime) -> List[DataRecord]
    - execute_plsql_procedure(proc_name: str, params: dict) -> Any
```

#### CacheRepository (Redis)
```python
class CacheRepository:
    - get(key: str) -> Optional[str]
    - set(key: str, value: str, expire: int) -> bool
    - delete(key: str) -> bool
    - delete_pattern(pattern: str) -> int
    - incr(key: str, expire: int) -> int
```

### 4. Database Layer

#### Oracle Database Schema
```sql
-- Sequência para ID (Oracle não tem UUID nativo)
CREATE SEQUENCE data_records_seq
    START WITH 1
    INCREMENT BY 1
    NOCACHE
    NOCYCLE;

-- Tabela principal com JSON support (Oracle 12c+)
CREATE TABLE data_records (
    id VARCHAR2(36) PRIMARY KEY DEFAULT SYS_GUID(),
    user_id VARCHAR2(100) NOT NULL,
    payload CLOB CHECK (payload IS JSON),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Trigger para auto-update do updated_at
CREATE OR REPLACE TRIGGER data_records_updated_at
    BEFORE UPDATE ON data_records
    FOR EACH ROW
BEGIN
    :NEW.updated_at := CURRENT_TIMESTAMP;
END;
/

-- Índices para performance
CREATE INDEX idx_user_id ON data_records(user_id);
CREATE INDEX idx_created_at ON data_records(created_at);
CREATE INDEX idx_user_created ON data_records(user_id, created_at);

-- Índice JSON para busca no payload (Oracle 12c+)
CREATE INDEX idx_payload_json ON data_records(payload) 
    INDEXTYPE IS CTXSYS.CONTEXT 
    PARAMETERS ('SYNC (ON COMMIT)');

-- Particionamento por data (Range Partitioning)
CREATE TABLE data_records_partitioned (
    id VARCHAR2(36) PRIMARY KEY,
    user_id VARCHAR2(100) NOT NULL,
    payload CLOB CHECK (payload IS JSON),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
)
PARTITION BY RANGE (created_at) (
    PARTITION p_2025_q1 VALUES LESS THAN (TIMESTAMP '2025-04-01 00:00:00'),
    PARTITION p_2025_q2 VALUES LESS THAN (TIMESTAMP '2025-07-01 00:00:00'),
    PARTITION p_2025_q3 VALUES LESS THAN (TIMESTAMP '2025-10-01 00:00:00'),
    PARTITION p_2025_q4 VALUES LESS THAN (TIMESTAMP '2026-01-01 00:00:00')
);
```

#### Redis Schema
```
# Cache de dados de usuário
user_data:{user_id}:{page}:{page_size} = JSON(data + metadata)
TTL: 300 seconds (5 minutos)

# Rate limiting
rate_limit:{user_id} = counter
TTL: 60 seconds

# Session data
session:{session_id} = JSON(user_data)
TTL: 3600 seconds (1 hora)
```

## Patterns de Design Aplicados

### 1. Repository Pattern
- Abstração completa do acesso a dados
- Facilita testing com mocks
- Permite troca de database sem impacto na lógica

### 2. Dependency Injection
- FastAPI Depends() para injeção automática
- Facilita testing e configuração
- Baixo acoplamento entre componentes

### 3. Factory Pattern
- Connection factories para PostgreSQL e Redis
- Configuration factory para diferentes ambientes
- Service factory para dependency injection

### 4. Strategy Pattern
- Cache strategy configurável (write-through, write-behind)
- Validation strategy por tipo de dados
- Pagination strategy (offset vs cursor-based)

### 5. Circuit Breaker Pattern
- Proteção contra falhas em dependencies externas
- Fallback para operações críticas
- Auto-recovery após período de cool-down

## Decisões Arquiteturais

### 1. Async/Await vs Sync
**Decisão: Full Async**

**Justificativa:**
- I/O bound operations (database, cache, network)
- Melhor utilização de recursos
- Maior throughput com mesma infraestrutura

**Trade-off:**
- Complexidade maior no código
- Debugging mais complexo
- Curva de aprendizado

### 2. Microservice vs Monolith
**Decisão: Single Microservice**

**Justificativa:**
- Escopo bem definido (data management)
- Deploy independente
- Escalabilidade horizontal
- Technology choice flexibility

**Trade-off:**
- Network latency para comunicação
- Complexity em distributed systems
- Monitoring mais complexo

### 3. Database Per Service vs Shared Database
**Decisão: Database Per Service**

**Justificativa:**
- Data ownership clara
- Schema evolution independente
- Melhor fault isolation

**Trade-off:**
- Não há ACID transactions cross-services
- Data consistency eventual
- Duplicação potencial de dados

### 4. Event-Driven vs Request-Response
**Decisão: Request-Response (com preparação para events)**

**Justificativa:**
- Simplicidade inicial
- Debugging mais fácil
- Consistency stronger

**Evolução Planejada:**
- Event publishing para auditoria
- Integration events para outros services
- CQRS para read-heavy operations

## Estratégias de Escalabilidade

### 1. Horizontal Scaling

#### Application Tier
```yaml
Kubernetes HPA:
  Min Replicas: 3
  Max Replicas: 20
  Target CPU: 70%
  Target Memory: 80%
```

#### Database Tier
```yaml
PostgreSQL:
  Master: 1 instance (writes)
  Read Replicas: 2-5 instances (reads)
  Connection Pooling: PgBouncer

Redis:
  Cluster Mode: 3 master + 3 replica
  Memory: 8GB per node
  Persistence: RDB + AOF
```

### 2. Vertical Scaling

#### Resource Limits
```yaml
Production Pods:
  CPU: 500m - 2000m
  Memory: 512Mi - 2Gi
  
Database:
  CPU: 2 cores - 8 cores
  Memory: 8GB - 32GB
  Storage: SSD, 100GB - 1TB
```

### 3. Data Partitioning

#### Estratégia de Sharding
```sql
-- Shard por user_id (consistent hashing)
Shard 1: user_id hash % 4 == 0
Shard 2: user_id hash % 4 == 1
Shard 3: user_id hash % 4 == 2
Shard 4: user_id hash % 4 == 3

-- Particionamento temporal
Partition monthly: data_records_2025_01, data_records_2025_02, etc.
```

### 4. Caching Strategy

#### Multi-Level Cache
```
L1: Application Memory (Python dict, 100MB)
L2: Redis (Hot data, 5-minute TTL)
L3: Database Connection Pool
L4: Database Buffer Pool
```

#### Cache Patterns
- **Read-through**: Cache miss triggers DB read
- **Write-through**: Writes go to cache + DB simultaneously
- **Cache-aside**: Application manages cache explicitly
- **Write-behind**: Async writes to DB (for non-critical data)

## Performance Optimization

### 1. Database Optimization

#### Query Optimization
```sql
-- Oracle-specific optimizations
-- Usar ROWNUM para paginação eficiente
SELECT * FROM (
    SELECT ROWNUM rn, t.* FROM (
        SELECT * FROM data_records 
        WHERE user_id = ? 
        ORDER BY created_at DESC
    ) t WHERE ROWNUM <= ?
) WHERE rn > ?;

-- Usar hints Oracle para otimização
SELECT /*+ INDEX(data_records idx_user_date) */
    * FROM data_records 
WHERE user_id = ? 
ORDER BY created_at DESC;

-- JSON queries otimizadas
SELECT * FROM data_records 
WHERE JSON_VALUE(payload, '$.type') = 'order'
AND user_id = ?;

-- Bulk operations com FORALL
DECLARE
    TYPE user_ids_t IS TABLE OF VARCHAR2(100);
    user_ids user_ids_t := user_ids_t('user1', 'user2', 'user3');
BEGIN
    FORALL i IN user_ids.FIRST..user_ids.LAST
        UPDATE data_records 
        SET updated_at = SYSTIMESTAMP 
        WHERE user_id = user_ids(i);
END;
```

#### Index Strategy
```sql
-- Composite indexes para queries frequentes Oracle
CREATE INDEX idx_user_date ON data_records(user_id, created_at DESC);

-- Function-based indexes para JSON queries
CREATE INDEX idx_payload_type ON data_records(JSON_VALUE(payload, '$.type'));

-- Domain indexes para full-text search
CREATE INDEX idx_payload_text ON data_records(payload) 
    INDEXTYPE IS CTXSYS.CONTEXT;

-- Bitmap indexes para low-cardinality data
CREATE BITMAP INDEX idx_status ON data_records(JSON_VALUE(payload, '$.status'));

-- Partitioned indexes para performance
CREATE INDEX idx_created_local ON data_records(created_at) LOCAL;
```

### 2. Application Optimization

#### Connection Pooling
```python
# SQLAlchemy async pool para Oracle
pool_size=10,           # Menor para Oracle (licensing)
max_overflow=20,        # Conexões extras em pico
pool_timeout=30,        # Timeout para obter conexão
pool_recycle=3600,      # Reciclar conexões antigas
pool_pre_ping=True,     # Validar conexões antes do uso
oracle_thick_mode=True  # Use Oracle thick client for better performance
```

#### Response Optimization
```python
# Streaming para grandes responses
async def stream_data():
    async for record in repository.stream_records():
        yield f"data: {record.json()}\n\n"

# Compression automática
middleware.add(GZipMiddleware, minimum_size=1000)

# Response caching
@lru_cache(maxsize=1000)
def format_response(data):
    return ResponseModel.parse_obj(data)
```

### 3. Infrastructure Optimization

#### Resource Allocation
```yaml
# Oracle Database resources
Oracle Database:
  CPU: 4 cores - 16 cores
  Memory: 16GB - 64GB (SGA + PGA)
  Storage: SSD, 200GB - 2TB
  
# Application pods
API Pods:
  CPU: High priority
  Memory: Medium priority
  
Background Workers:
  CPU: Medium priority  
  Memory: High priority

# Oracle connection limits consideration
Max Connections: 100-300 (Enterprise Edition)
```

## Security Architecture

### 1. Authentication & Authorization
```
API Gateway → JWT Validation → RBAC → Service Access
```

### 2. Data Protection
- **Encryption at Rest**: Oracle TDE (Transparent Data Encryption) + Redis AUTH
- **Encryption in Transit**: TLS 1.3 for all communications
- **PII Handling**: Oracle Data Redaction + JSON field encryption

### 3. Network Security
- **VPC**: Private subnets para databases
- **Security Groups**: Least privilege access
- **WAF**: Protection against OWASP Top 10

### 4. Monitoring & Auditing
- **Access Logs**: All API calls logged
- **Audit Trail**: Data modification tracking
- **Security Events**: Failed auth attempts, suspicious patterns

## Fault Tolerance & Resilience

### 1. Circuit Breaker Pattern
```python
# Para database connections
@circuit_breaker(failure_threshold=5, recovery_timeout=30)
async def database_operation():
    # Database call with automatic fallback
    pass
```

### 2. Retry Strategies
```python
# Exponential backoff para transient failures
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=10)
)
async def external_api_call():
    pass
```

### 3. Graceful Degradation
- **Cache-only mode**: Durante database outages
- **Read-only mode**: Durante maintenance
- **Reduced functionality**: Em caso de sobrecarga

### 4. Health Checks
```python
# Liveness: Is the service running?
GET /health → 200 OK (always responds if process is alive)

# Readiness: Can the service handle requests?
GET /ready → 200 OK (only if dependencies are healthy)
```

## Observability & Monitoring

### 1. Metrics (Prometheus)
```yaml
Application Metrics:
  - request_duration_seconds{method, endpoint, status}
  - requests_total{method, endpoint, status}
  - active_connections{pool, database}
  - cache_operations_total{operation, result}

Business Metrics:
  - data_records_created_total{user_type}
  - unique_users_active{time_window}
  - payload_size_bytes{percentile}
```

### 2. Distributed Tracing (Jaeger)
```
Request ID: 123e4567-e89b-12d3-a456-426614174000
├── API Controller (2ms)
├── Data Service (15ms)
│   ├── Cache Check (1ms)
│   └── Database Query (12ms)
└── Response Serialization (3ms)
Total: 20ms
```

### 3. Logging (Structured)
```json
{
  "timestamp": "2025-06-03T10:30:00Z",
  "level": "INFO",
  "service": "data-microservice",
  "request_id": "123e4567-e89b-12d3-a456-426614174000",
  "user_id": "user123",
  "endpoint": "POST /data",
  "duration_ms": 85,
  "status": 201,
  "payload_size": 1024
}
```

## Disaster Recovery

### 1. Backup Strategy
```yaml
Oracle Database:
  RMAN Backup: Daily (3AM UTC)
  Archive Log Mode: Enabled
  Retention: 30 days
  Standby Database: For DR
  
Redis:
  RDB Snapshots: Every 6 hours
  AOF: Every second
  Retention: 7 days
```

### 2. Multi-Region Strategy
```
Primary Region: us-east-1
├── Master Database
├── Application Instances
└── Redis Primary

Disaster Recovery: us-west-2
├── Read Replica (standby)
├── Standby Application (scaled to 0)
└── Redis Replica

RTO: 15 minutes
RPO: 1 minute
```

### 3. Data Recovery Procedures
1. **Point-in-time recovery**: Oracle RMAN + Archive Logs
2. **Logical backups**: Data Pump (expdp/impdp) para dados críticos
3. **Cross-region replication**: Oracle Data Guard para DR
4. **Automated failover**: Kubernetes + Oracle health checks

Esta arquitetura foi projetada para evoluir de um MVP simples para um sistema de escala enterprise, mantendo sempre a performance, confiabilidade e manutenibilidade como prioridades principais.
