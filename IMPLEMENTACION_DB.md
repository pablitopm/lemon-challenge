# Implementación de Base de Datos

## Arquitectura de Datos

El sistema utiliza una arquitectura híbrida optimizada para diferentes tipos de datos y casos de uso:

### **Resumen de Esquemas**

#### **Tablas Principales (DynamoDB)**

**accounts**
- Balances: `balance_usdt`, `balance_btc`, `balance_eth`
- Reservados: `reserved_usdt`, `reserved_btc`, `reserved_eth`
- Version: Para conditional writes (optimistic locking)
- Validación: `balance >= reserved`

**transactions**
- Estados: `AUTHORIZED → TRADING → COMPLETED/FAILED`
- Particionado por transaction_id
- Índices en `transaction_id`, `status`, `account_id`

**cards**
- Vinculación con accounts
- Hash de número de tarjeta (PCI-DSS)
- Límites diarios/mensuales

#### **Cache (Redis)**

```
balance:{account_id}     → TTL: 60s
rate:USDT:ARS           → TTL: 30s
rate:BTC/USDT           → TTL: 5s
rate:ETH/USDT           → TTL: 5s
card:{card_id}          → TTL: 5min
idempotent:{txn_id}     → TTL: 24h
```

### **DynamoDB - Datos Críticos y Transaccionales**

#### **Tabla: accounts**
```yaml
TableName: accounts
PartitionKey: account_id (String)
Attributes:
  - account_id: String (PK)
  - user_id: String
  - balance_usdt: Number
  - balance_btc: Number
  - balance_eth: Number
  - reserved_usdt: Number
  - reserved_btc: Number
  - reserved_eth: Number
  - payment_currency: String
  - status: String
  - version: Number (para conditional writes)
  - created_at: String (ISO timestamp)
  - updated_at: String (ISO timestamp)

# Conditional Write para bloqueo de fondos:
# ConditionExpression: "version = :expected_version AND (balance_usdt - reserved_usdt) >= :amount"
# UpdateExpression: "SET reserved_usdt = reserved_usdt + :amount, version = version + :inc"

# GSI: user-accounts
PartitionKey: user_id
SortKey: created_at
ProjectionType: ALL
```

#### **Tabla: cards**
```yaml
TableName: cards
PartitionKey: card_id (String)
Attributes:
  - card_id: String (PK)
  - account_id: String
  - card_number_hash: String # Hash del número completo (PCI-DSS)
  - last_four: String
  - cardholder_name: String
  - expiry_date: String
  - status: String # ACTIVE, BLOCKED, EXPIRED, CANCELLED
  - daily_limit_usdt: Number
  - monthly_limit_usdt: Number
  - created_at: String
  - updated_at: String

# GSI: account-cards
PartitionKey: account_id
SortKey: created_at
ProjectionType: ALL

# GSI: status-cards
PartitionKey: status
SortKey: card_id
ProjectionType: INCLUDE
NonKeyAttributes: [account_id, last_four, expiry_date]
```

#### **Tabla: transactions**
```yaml
TableName: transactions
PartitionKey: transaction_id (String)
Attributes:
  - transaction_id: String (PK) # Del proveedor
  - account_id: String
  - card_id: String
  - amount_ars: Number
  - amount_usdt: Number
  - exchange_rate_usdt_ars: Number
  - user_currency: String
  - amount_in_user_currency: Number
  - exchange_rate_to_usdt: Number
  - status: String # AUTHORIZED, TRADING, COMPLETED, FAILED, REVERSED
  - trade_id: String
  - trade_status: String
  - trade_executed_at: String
  - trade_error: String
  - authorized_at: String
  - completed_at: String
  - reversed_at: String
  - retry_count: Number
  - last_retry_at: String
  - metadata: Map # Merchant info, location, etc.
  - created_at: String
  - updated_at: String

# GSI: account-transactions
PartitionKey: account_id
SortKey: created_at
ProjectionType: ALL

# GSI: status-transactions (para reconciliación)
PartitionKey: status
SortKey: created_at
ProjectionType: INCLUDE
NonKeyAttributes: [transaction_id, account_id, retry_count]

# GSI: card-transactions
PartitionKey: card_id
SortKey: created_at
ProjectionType: ALL
```

### **Redis - Cache de Alto Rendimiento**

#### **Estructura de Cache**
```redis
# Información de tarjeta (TTL: 5 minutos)
Key: "card:{card_id}"
Value: {
    "account_id": "acc_xyz",
    "status": "ACTIVE",
    "daily_limit": "10000.00",
    "daily_spent": "2500.00",
    "last_four": "1234",
    "expiry_date": "2026-12-31"
}

# Tasas de conversión (TTL diferenciado por volatilidad)
Key: "rate:USDT:ARS"
Value: "1050.50"
TTL: 30 segundos

Key: "rate:BTC:USDT"
Value: "67500.00"
TTL: 5 segundos

Key: "rate:ETH:USDT"
Value: "3200.00"
TTL: 5 segundos

# Rate limiting (TTL: 60 segundos)
Key: "ratelimit:auth:{account_id}"
Value: "5" # Contador de solicitudes
Expire: 60 segundos

# Idempotencia (TTL: 24 horas)
Key: "idempotent:{transaction_id}"
Value: {
    "status": "AUTHORIZED",
    "response": {...}
}

# Cache de balances (TTL: 10 segundos) - Solo para consultas frecuentes
Key: "balance_summary:{account_id}"
Value: {
    "total_usdt_equivalent": "1500.25",
    "payment_currency": "USDT",
    "last_updated": 1729505000
}
```

### **TimescaleDB - Métricas de Negocio**

#### **Tabla: exchange_rates**
```sql
CREATE TABLE exchange_rates (
    from_currency VARCHAR(10) NOT NULL,
    to_currency VARCHAR(10) NOT NULL,
    rate DECIMAL(20,8) NOT NULL,
    source VARCHAR(50) NOT NULL, -- BINANCE, KRAKEN, etc.
    valid_from TIMESTAMPTZ NOT NULL,
    valid_until TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Convertir a hypertable para optimización de time-series
SELECT create_hypertable('exchange_rates', 'valid_from');

-- Índices para consultas eficientes
CREATE INDEX idx_exchange_rates_currency_pair ON exchange_rates (from_currency, to_currency, valid_from DESC);
CREATE INDEX idx_exchange_rates_source ON exchange_rates (source, valid_from DESC);
```

#### **Tabla: balance_ledger (Event Sourcing)**
```sql
CREATE TABLE balance_ledger (
    account_id VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL,
    transaction_id VARCHAR(100) NOT NULL,
    currency VARCHAR(10) NOT NULL,
    amount_delta DECIMAL(20,8) NOT NULL, -- Puede ser negativo
    balance_before DECIMAL(20,8) NOT NULL,
    balance_after DECIMAL(20,8) NOT NULL,
    operation_type VARCHAR(50) NOT NULL, -- DEPOSIT, WITHDRAWAL, PURCHASE, REFUND, RESERVE, RELEASE_RESERVE, TRADE_IN, TRADE_OUT
    description TEXT,
    metadata JSONB
);

-- Convertir a hypertable
SELECT create_hypertable('balance_ledger', 'created_at');

-- Índices para consultas eficientes
CREATE INDEX idx_balance_ledger_account ON balance_ledger (account_id, created_at DESC);
CREATE INDEX idx_balance_ledger_transaction ON balance_ledger (transaction_id);
CREATE INDEX idx_balance_ledger_operation ON balance_ledger (operation_type, created_at DESC);
```

## Operaciones Críticas

### **Bloqueo de Fondos (DynamoDB)**

#### **Operación Atómica**
```python
def reserve_funds(account_id: str, amount: Decimal, expected_version: int) -> bool:
    """
    Bloquea fondos de manera atómica usando conditional writes
    """
    try:
        response = dynamodb.update_item(
            TableName='accounts',
            Key={'account_id': {'S': account_id}},
            ConditionExpression='version = :expected_version AND (balance_usdt - reserved_usdt) >= :amount',
            UpdateExpression='SET reserved_usdt = reserved_usdt + :amount, version = version + :inc',
            ExpressionAttributeValues={
                ':expected_version': {'N': str(expected_version)},
                ':amount': {'N': str(amount)},
                ':inc': {'N': '1'}
            },
            ReturnValues='UPDATED_NEW'
        )
        return True
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            return False  # Race condition o fondos insuficientes
        raise
```

#### **Liberación de Fondos**
```python
def release_funds(account_id: str, amount: Decimal, expected_version: int) -> bool:
    """
    Libera fondos reservados de manera atómica
    """
    try:
        response = dynamodb.update_item(
            TableName='accounts',
            Key={'account_id': {'S': account_id}},
            ConditionExpression='version = :expected_version AND reserved_usdt >= :amount',
            UpdateExpression='SET reserved_usdt = reserved_usdt - :amount, version = version + :inc',
            ExpressionAttributeValues={
                ':expected_version': {'N': str(expected_version)},
                ':amount': {'N': str(amount)},
                ':inc': {'N': '1'}
            }
        )
        return True
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            return False
        raise
```

### **Cache de Tasas de Conversión (Redis)**

#### **Configuración Diferenciada por Volatilidad**
```python
class ExchangeRateCache:
    def __init__(self):
        self.cache_configs = {
            "USDT-ARS": {
                "redis_ttl": 30,  # segundos
                "memory_ttl": 10,  # segundos
                "max_drift": 0.5,  # 0.5%
            },
            "BTC-USDT": {
                "redis_ttl": 5,   # segundos
                "memory_ttl": 2,   # segundos
                "max_drift": 2.0,  # 2%
            },
            "ETH-USDT": {
                "redis_ttl": 5,   # segundos
                "memory_ttl": 2,   # segundos
                "max_drift": 2.5,  # 2.5%
            },
        }
    
    def get_rate(self, from_currency: str, to_currency: str) -> Decimal:
        pair = f"{from_currency}-{to_currency}"
        config = self.cache_configs.get(pair)
        
        if not config:
            return self._get_rate_from_api(from_currency, to_currency)
        
        # L1: In-memory cache
        cached = self.mem_cache.get(pair)
        if cached and self._validate_drift(cached, config["max_drift"]):
            return cached.rate
        
        # L2: Redis cache
        cached = self.redis.get(f"rate:{pair}")
        if cached and self._validate_drift(cached, config["max_drift"]):
            self.mem_cache.set(pair, cached, config["memory_ttl"])
            return cached.rate
        
        # L3: API en tiempo real
        rate = self._get_rate_from_api(from_currency, to_currency)
        
        # Cachear con TTL apropiado
        self.redis.setex(f"rate:{pair}", config["redis_ttl"], rate)
        self.mem_cache.set(pair, rate, config["memory_ttl"])
        
        return rate
```

### **Event Sourcing (TimescaleDB)**

#### **Registro de Cambios de Balance**
```python
def record_balance_change(
    account_id: str,
    transaction_id: str,
    currency: str,
    amount_delta: Decimal,
    balance_before: Decimal,
    balance_after: Decimal,
    operation_type: str,
    description: str = None,
    metadata: dict = None
):
    """
    Registra un cambio de balance en el ledger para auditoría completa
    """
    query = """
    INSERT INTO balance_ledger (
        account_id, created_at, transaction_id, currency,
        amount_delta, balance_before, balance_after,
        operation_type, description, metadata
    ) VALUES (
        %s, NOW(), %s, %s, %s, %s, %s, %s, %s, %s
    )
    """
    
    conn.execute(query, (
        account_id, transaction_id, currency,
        amount_delta, balance_before, balance_after,
        operation_type, description, json.dumps(metadata) if metadata else None
    ))
```

## Consultas de Negocio

### **Análisis de Volatilidad de Tasas**
```sql
-- Análisis de volatilidad por par de monedas en las últimas 24 horas
SELECT 
    from_currency,
    to_currency,
    MIN(rate) as min_rate,
    MAX(rate) as max_rate,
    AVG(rate) as avg_rate,
    STDDEV(rate) as volatility,
    COUNT(*) as sample_count
FROM exchange_rates 
WHERE valid_from >= NOW() - INTERVAL '24 hours'
GROUP BY from_currency, to_currency
ORDER BY volatility DESC;
```

### **Auditoría de Transacciones por Usuario**
```sql
-- Historial completo de cambios de balance para un usuario
SELECT 
    bl.created_at,
    bl.transaction_id,
    bl.currency,
    bl.amount_delta,
    bl.balance_before,
    bl.balance_after,
    bl.operation_type,
    bl.description
FROM balance_ledger bl
WHERE bl.account_id = 'acc_12345'
ORDER BY bl.created_at DESC
LIMIT 100;
```

### **Métricas de Performance del Sistema**
```sql
-- Análisis de latencias por operación en las últimas 24 horas
SELECT 
    operation_type,
    COUNT(*) as total_operations,
    AVG(EXTRACT(EPOCH FROM (updated_at - created_at))) as avg_duration_seconds,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (updated_at - created_at))) as p95_duration_seconds
FROM balance_ledger 
WHERE created_at >= NOW() - INTERVAL '24 hours'
GROUP BY operation_type
ORDER BY avg_duration_seconds DESC;
```

## Configuración de Infraestructura

### **DynamoDB**
```yaml
# Configuración de tabla accounts
TableName: accounts
BillingMode: ON_DEMAND
PointInTimeRecoverySpecification:
  PointInTimeRecoveryEnabled: true
SSESpecification:
  SSEEnabled: true
  SSEType: KMS
GlobalSecondaryIndexes:
  - IndexName: user-accounts
    KeySchema:
      - AttributeName: user_id
        KeyType: HASH
      - AttributeName: created_at
        KeyType: RANGE
    Projection:
      ProjectionType: ALL
```

### **Redis Cluster**
```yaml
# Configuración de ElastiCache Redis
NodeType: cache.r6g.large
NumCacheNodes: 3
Engine: redis
EngineVersion: 7.0
ParameterGroupName: default.redis7
SnapshotRetentionLimit: 7
AutomaticFailoverEnabled: true
MultiAZEnabled: true
```

### **TimescaleDB**
```yaml
# Configuración de instancia RDS
Engine: postgres
EngineVersion: 15.4
DBInstanceClass: db.r6g.xlarge
AllocatedStorage: 1000
StorageType: gp3
StorageEncrypted: true
BackupRetentionPeriod: 30
MultiAZ: true
```

## Monitoreo y Alertas

### **Métricas de DynamoDB**
- `ConsumedReadCapacityUnits`
- `ConsumedWriteCapacityUnits`
- `ThrottledRequests`
- `ConditionalCheckFailedRequests`
- `SuccessfulRequestLatency`

### **Métricas de Redis**
- `CacheHits`
- `CacheMisses`
- `Evictions`
- `MemoryUsage`
- `ConnectedClients`

### **Métricas de TimescaleDB**
- `DatabaseConnections`
- `QueriesPerSecond`
- `AverageQueryDuration`
- `DiskUsage`
- `ReplicationLag`

## Backup y Recuperación

### **DynamoDB**
- Point-in-time recovery habilitado
- Backup automático diario
- Cross-region backup para disaster recovery

### **Redis**
- Snapshot automático cada 6 horas
- Backup manual antes de cambios importantes
- Replicación en múltiples AZs

### **TimescaleDB**
- Backup automático diario con retención de 30 días
- WAL archiving para recovery point-in-time
- Replicación síncrona en standby

## Consideraciones de Seguridad

### **Encriptación**
- **En tránsito**: TLS 1.3 para todas las conexiones
- **En reposo**: AES-256 para DynamoDB, Redis y TimescaleDB
- **Claves**: AWS KMS para gestión de claves

### **Acceso**
- **DynamoDB**: IAM roles con permisos mínimos
- **Redis**: AUTH token y VPC security groups
- **TimescaleDB**: Usuarios con permisos granulares

### **Auditoría**
- **CloudTrail**: Para operaciones de DynamoDB
- **Logs de aplicación**: Para operaciones de Redis
- **PostgreSQL logs**: Para consultas de TimescaleDB

---

**Nota**: Esta implementación está optimizada para el caso de uso específico del sistema de autorización de pagos, balanceando performance, costo y mantenibilidad.
