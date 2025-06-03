# Estratégia de Testes - Microsserviço

## Visão Geral

Estratégia de testes abrangente seguindo a pirâmide de testes, com foco em qualidade, performance e confiabilidade para o microsserviço.

## Pirâmide de Testes

```
                    ┌─────────────────┐
                    │   Manual Tests  │  ← 5%
                    │                 │
                  ┌─┴─────────────────┴─┐
                  │    E2E Tests        │  ← 15%
                  │                     │
                ┌─┴─────────────────────┴─┐
                │  Integration Tests      │  ← 30%
                │                         │
              ┌─┴─────────────────────────┴─┐
              │     Unit Tests              │  ← 50%
              │                             │
              └─────────────────────────────┘

Target: 80%+ coverage para código crítico
```

## 1. Unit Tests (50% - Base da Pirâmide)

### Estrutura de Testes
```python
tests/unit/
├── test_models/           # SQLAlchemy models + Oracle types
├── test_schemas/          # Pydantic validation
├── test_services/         # Business logic
├── test_repositories/     # Data access layer
├── test_controllers/      # API endpoints
└── test_utils/           # Helper functions
```

### Exemplo - Teste de Model Oracle
```python
# tests/unit/test_models/test_data_model.py
import pytest
import json
from app.models.data_model import DataRecord

class TestDataModel:
    def test_data_record_creation_valid(self):
        """Teste criação válida de DataRecord"""
        payload = {"user": "john", "action": "login"}
        record = DataRecord(
            user_id="user123",
            payload=json.dumps(payload)
        )
        
        assert record.user_id == "user123"
        assert json.loads(record.payload) == payload
        assert record.id is not None
    
    def test_oracle_json_constraint(self, db_session):
        """Teste constraint JSON do Oracle"""
        record = DataRecord(
            user_id="user123",
            payload="invalid json"
        )
        db_session.add(record)
        
        with pytest.raises(IntegrityError, match="payload_json_check"):
            db_session.commit()
    
    @pytest.mark.parametrize("payload_size", [1024, 1048576, 1048577])
    def test_payload_size_limits(self, payload_size):
        """Teste limites de tamanho do payload"""
        large_payload = {"data": "x" * payload_size}
        
        if payload_size > 1048576:  # 1MB limit
            with pytest.raises(ValueError, match="Payload too large"):
                DataRecord(user_id="user123", payload=json.dumps(large_payload))
        else:
            record = DataRecord(user_id="user123", payload=json.dumps(large_payload))
            assert record.payload is not None
```

### Exemplo - Teste de Service
```python
# tests/unit/test_services/test_data_service.py
import pytest
from unittest.mock import AsyncMock
from app.services.data_service import DataService

class TestDataService:
    @pytest.fixture
    def data_service(self):
        mock_repository = AsyncMock()
        mock_cache = AsyncMock()
        return DataService(mock_repository, mock_cache)
    
    @pytest.mark.asyncio
    async def test_create_data_success(self, data_service):
        """Teste criação bem-sucedida de dados"""
        request = DataCreateRequest(
            user_id="user123",
            payload={"action": "login"}
        )
        
        # Mock repository response
        data_service.repository.create.return_value = Mock(
            id="test_id", user_id="user123"
        )
        
        result = await data_service.create_data(request)
        
        assert result.user_id == "user123"
        data_service.repository.create.assert_called_once()
        data_service.cache.delete_pattern.assert_called_once()
    
    @pytest.mark.asyncio
    async def test_rate_limit_exceeded(self, data_service):
        """Teste rate limiting"""
        data_service.cache.get.return_value = "101"  # Above limit
        
        with pytest.raises(ValueError, match="Rate limit exceeded"):
            await data_service.create_data(request)
```

### Configuração pytest.ini
```ini
[tool:pytest]
testpaths = tests
addopts = 
    --cov=app
    --cov-report=term-missing
    --cov-report=html
    --cov-fail-under=80
    -v
markers =
    unit: Unit tests
    integration: Integration tests
    e2e: End-to-end tests
    oracle: Oracle database specific tests
    performance: Performance tests
```

## 2. Integration Tests (30% - Componentes Integrados)

### Estrutura de Testes
```python
tests/integration/
├── test_api_integration/      # API + Service + Repository
├── test_database_integration/ # Oracle CRUD operations
├── test_cache_integration/    # Redis cache operations
└── test_external_services/    # Monitoring, logging
```

### Exemplo - Teste API + Oracle Integration
```python
# tests/integration/test_api_integration.py
import pytest
import httpx
from tests.helpers.oracle_test_helper import OracleTestHelper

class TestDataEndpointsIntegration:
    @pytest.fixture(scope="class")
    async def oracle_helper(self):
        helper = OracleTestHelper()
        await helper.setup_test_schema()
        yield helper
        await helper.cleanup_test_schema()
    
    @pytest.fixture
    async def test_client(self):
        from app.main import app
        async with httpx.AsyncClient(app=app, base_url="http://test") as client:
            yield client
    
    @pytest.mark.asyncio
    @pytest.mark.integration
    async def test_create_data_end_to_end(self, test_client, oracle_helper):
        """Teste completo: API -> Service -> Repository -> Oracle"""
        payload = {
            "user_id": "integration_user_123",
            "payload": {
                "action": "integration_test",
                "timestamp": "2025-06-03T10:30:00Z"
            }
        }
        
        # Create via API
        response = await test_client.post("/data", json=payload)
        
        # Assert API response
        assert response.status_code == 201
        response_data = response.json()
        assert response_data["user_id"] == "integration_user_123"
        created_id = response_data["id"]
        
        # Assert data in Oracle Database
        db_record = await oracle_helper.get_record_by_id(created_id)
        assert db_record is not None
        assert db_record["user_id"] == "integration_user_123"
    
    @pytest.mark.asyncio
    async def test_oracle_json_query_integration(self, test_client, oracle_helper):
        """Teste queries JSON complexas no Oracle"""
        user_id = "json_query_user"
        
        # Insert test data
        await oracle_helper.insert_test_record(
            user_id=user_id,
            payload={"type": "order", "amount": 100.50}
        )
        
        # Query by JSON path
        response = await test_client.get(
            f"/data/search?user_id={user_id}&json_path=$.type&value=order"
        )
        
        assert response.status_code == 200
        data = response.json()
        assert len(data["data"]) == 1
        assert data["data"][0]["payload"]["type"] == "order"
```

### Docker Compose para Testes
```yaml
# docker/docker-compose.test.yml
version: '3.8'

services:
  oracle-test:
    image: gvenzl/oracle-xe:21-slim
    environment:
      ORACLE_PASSWORD: TestPassword123
    ports:
      - "1521:1521"
    healthcheck:
      test: ["CMD", "sqlplus", "-L", "system/TestPassword123@//localhost:1521/XE"]
      interval: 30s
      timeout: 10s
      retries: 10
  
  redis-test:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
  
  app-test:
    build:
      context: ..
      dockerfile: docker/Dockerfile
      target: development
    environment:
      - DATABASE_URL=oracle+cx_oracle://system:TestPassword123@oracle-test:1521/XE
      - REDIS_URL=redis://redis-test:6379/0
      - TESTING=true
    depends_on:
      oracle-test:
        condition: service_healthy
      redis-test:
        condition: service_healthy
    command: ["pytest", "tests/integration/", "-v"]
```

## 3. End-to-End Tests (15% - Fluxos Completos)

### Estrutura de Testes
```python
tests/e2e/
├── test_user_workflows/       # Fluxos completos de usuário
├── test_performance/          # Cenários de carga
├── test_security/             # Fluxos de segurança
└── test_monitoring/           # Métricas e logs
```

### Exemplo - Teste de Workflow Completo
```python
# tests/e2e/test_user_workflows.py
import pytest
import asyncio
from datetime import datetime, timedelta

class TestDataCreationWorkflow:
    @pytest.mark.asyncio
    @pytest.mark.e2e
    async def test_complete_data_creation_workflow(self, e2e_helper):
        """Teste fluxo completo: criação -> validação -> busca"""
        
        # Step 1: Create data via API
        create_payload = {
            "user_id": "workflow_user_123",
            "payload": {
                "workflow_test": True,
                "timestamp": datetime.now().isoformat()
            }
        }
        
        create_response = await e2e_helper.post("/data", json=create_payload)
        assert create_response.status_code == 201
        created_id = create_response.json()["id"]
        
        # Step 2: Verify data persistence
        get_response = await e2e_helper.get(f"/data?user_id=workflow_user_123")
        assert get_response.status_code == 200
        
        retrieved_data = get_response.json()
        assert retrieved_data["total"] >= 1
        
        # Step 3: Verify metrics were updated
        metrics_response = await e2e_helper.get("/metrics")
        assert "data_records_created_total" in metrics_response.text
        
        # Step 4: Test pagination
        page_response = await e2e_helper.get(
            f"/data?user_id=workflow_user_123&page=1&page_size=1"
        )
        assert page_response.status_code == 200
        assert len(page_response.json()["data"]) == 1
        
        # Cleanup
        await e2e_helper.cleanup_user_data("workflow_user_123")
    
    @pytest.mark.asyncio
    @pytest.mark.performance
    async def test_high_volume_workflow(self, e2e_helper):
        """Teste fluxo de alto volume"""
        user_id = "high_volume_user"
        batch_size = 100
        
        # Create records concurrently
        async def create_record(index):
            payload = {
                "user_id": user_id,
                "payload": {"batch_test": True, "index": index}
            }
            return await e2e_helper.post("/data", json=payload)
        
        start_time = datetime.now()
        tasks = [create_record(i) for i in range(batch_size)]
        responses = await asyncio.gather(*tasks, return_exceptions=True)
        end_time = datetime.now()
        
        # Analyze results
        successful = [r for r in responses if not isinstance(r, Exception) and r.status_code == 201]
        success_rate = len(successful) / batch_size
        execution_time = (end_time - start_time).total_seconds()
        throughput = batch_size / execution_time
        
        # Assertions
        assert success_rate >= 0.95  # 95% success rate
        assert throughput >= 10      # 10 requests/second minimum
        
        await e2e_helper.cleanup_user_data(user_id)
```

## 4. Test Data Management

### Test Data Factory
```python
# tests/factories/data_factory.py
from faker import Faker
import json
import random

class TestDataFactory:
    def __init__(self):
        self.fake = Faker(['pt_BR', 'en_US'])
    
    def create_user_id(self, prefix="test_user"):
        return f"{prefix}_{self.fake.uuid4()[:8]}"
    
    def create_simple_payload(self):
        return {
            "action": self.fake.random_element(["login", "logout", "view"]),
            "timestamp": datetime.now().isoformat(),
            "user_agent": self.fake.user_agent()
        }
    
    def create_complex_payload(self):
        return {
            "event_type": "user_interaction",
            "user_info": {
                "id": self.fake.uuid4(),
                "name": self.fake.name(),
                "email": self.fake.email()
            },
            "session_data": {
                "session_id": self.fake.uuid4(),
                "actions": [
                    {
                        "type": self.fake.word(),
                        "timestamp": datetime.now().isoformat()
                    }
                    for _ in range(random.randint(1, 5))
                ]
            }
        }
    
    def create_large_payload(self, target_size_kb=500):
        """Payload grande para testes de performance"""
        base_payload = self.create_complex_payload()
        large_text = self.fake.text(max_nb_chars=target_size_kb * 1024)
        base_payload["large_data"] = {"description": large_text}
        return base_payload
    
    def create_edge_case_payloads(self):
        return [
            {},  # Empty object
            {"null_value": None},
            {"unicode": "测试数据 🚀 émojis"},
            {"large_array": [self.fake.word() for _ in range(100)]}
        ]
```

### Oracle Test Helper
```python
# tests/helpers/oracle_test_helper.py
import asyncio
import json
import uuid
from sqlalchemy.ext.asyncio import create_async_engine

class OracleTestHelper:
    def __init__(self):
        self.test_db_url = "oracle+cx_oracle://system:TestPassword123@localhost:1521/XE"
        self.engine = create_async_engine(self.test_db_url)
    
    async def setup_test_schema(self):
        """Criar schema de teste"""
        async with self.engine.begin() as conn:
            await conn.execute(text("""
                CREATE TABLE test_data_records (
                    id VARCHAR2(36) PRIMARY KEY,
                    user_id VARCHAR2(100) NOT NULL,
                    payload CLOB CHECK (payload IS JSON),
                    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
                )
            """))
    
    async def cleanup_test_schema(self):
        """Limpar schema de teste"""
        async with self.engine.begin() as conn:
            await conn.execute(text("DROP TABLE test_data_records"))
    
    async def insert_test_record(self, user_id, payload):
        """Inserir registro de teste"""
        record_id = str(uuid.uuid4())
        payload_json = json.dumps(payload)
        
        async with self.engine.begin() as conn:
            await conn.execute(text("""
                INSERT INTO test_data_records (id, user_id, payload)
                VALUES (:id, :user_id, :payload)
            """), {"id": record_id, "user_id": user_id, "payload": payload_json})
        
        return record_id
    
    async def get_record_by_id(self, record_id):
        """Buscar registro por ID"""
        async with self.engine.begin() as conn:
            result = await conn.execute(text("""
                SELECT id, user_id, payload, created_at
                FROM test_data_records WHERE id = :id
            """), {"id": record_id})
            
            row = result.fetchone()
            if row:
                return {
                    "id": row[0],
                    "user_id": row[1], 
                    "payload": row[2],
                    "created_at": row[3]
                }
            return None
```

## 5. Test Execution Strategy

### Pipeline de Execução
```yaml
Phase 1 - Fast Feedback (< 5 min):
  - Unit tests
  - Linting and formatting
  - Security scanning (basic)

Phase 2 - Integration (< 15 min):
  - Integration tests with Oracle/Redis
  - API contract tests
  - Component integration

Phase 3 - E2E Validation (< 30 min):
  - End-to-end workflows
  - Performance baseline
  - Security penetration tests
```

### Quality Gates
```yaml
Code Quality:
  - Test Coverage: ≥ 80%
  - Security: Zero HIGH/CRITICAL vulnerabilities
  - Performance: Response time ≤ 500ms

Oracle Specific:
  - JSON constraint validation
  - Connection pool efficiency
  - Query performance checks
```

### CI/CD Integration
```yaml
Pull Request:
  - Unit tests (mandatory)
  - Integration tests (mandatory)
  - Security tests (mandatory)

Merge to Develop:
  - Full test suite
  - Performance baseline
  - E2E critical paths

Production Deploy:
  - Smoke tests
  - Health checks
  - Performance validation
```

### Oracle Performance Testing
```python
# tests/performance/oracle_performance.py
import pytest
import asyncio
import time
from tests.helpers.oracle_test_helper import OracleTestHelper

class TestOraclePerformance:
    @pytest.mark.asyncio
    @pytest.mark.performance
    async def test_bulk_insert_performance(self, oracle_helper):
        """Teste performance de inserção em lote"""
        records_count = 1000
        
        start_time = time.time()
        
        # Bulk insert
        tasks = []
        for i in range(records_count):
            tasks.append(oracle_helper.insert_test_record(
                f"perf_user_{i}",
                {"batch_id": 1, "sequence": i}
            ))
        
        await asyncio.gather(*tasks)
        
        end_time = time.time()
        execution_time = end_time - start_time
        throughput = records_count / execution_time
        
        # Performance assertions
        assert execution_time < 30  # Should complete in 30 seconds
        assert throughput > 50      # At least 50 inserts/second
        
        print(f"Bulk insert performance: {throughput:.1f} records/second")
```
