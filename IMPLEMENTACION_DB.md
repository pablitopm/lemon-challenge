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
**Estructura:**
- Clave de partición: account_id (String)
- Atributos principales: user_id, balances por moneda (USDT, BTC, ETH), fondos reservados por moneda, moneda de pago por defecto, estado de cuenta, versión para conditional writes, timestamps de creación y actualización

**Operaciones críticas:**
- Conditional Write para bloqueo de fondos: verificar versión y balance disponible, incrementar fondos reservados y versión
- GSI user-accounts: particionado por user_id, ordenado por fecha de creación

#### **Tabla: cards**
**Estructura:**
- Clave de partición: card_id (String)
- Atributos principales: account_id, hash del número de tarjeta (PCI-DSS), últimos 4 dígitos, nombre del titular, fecha de expiración, estado (ACTIVE, BLOCKED, EXPIRED, CANCELLED), límites diarios y mensuales en USDT, timestamps

**Índices:**
- GSI account-cards: particionado por account_id, ordenado por fecha de creación
- GSI status-cards: particionado por status, ordenado por card_id, incluye atributos específicos

#### **Tabla: transactions**
**Estructura:**
- Clave de partición: transaction_id (String, del proveedor)
- Atributos principales: account_id, card_id, montos en ARS/USDT, tasas de cambio, moneda del usuario, estado del ciclo de vida (AUTHORIZED, TRADING, COMPLETED, FAILED, REVERSED), información de trade (ID, estado, timestamps), contadores de reintento, metadatos del comercio, timestamps de eventos

**Índices:**
- GSI account-transactions: particionado por account_id, ordenado por fecha de creación
- GSI status-transactions: particionado por status, ordenado por fecha de creación (para reconciliación)
- GSI card-transactions: particionado por card_id, ordenado por fecha de creación

### **Redis - Cache de Alto Rendimiento**

#### **Estructura de Cache**
**Claves principales:**
- `card:{card_id}`: Información de tarjeta (TTL: 5 minutos) - account_id, estado, límites diarios, gasto diario, últimos 4 dígitos, fecha de expiración
- `rate:USDT:ARS`: Tasa de conversión estable (TTL: 30 segundos) - valor numérico
- `rate:BTC:USDT`: Tasa de conversión volátil (TTL: 5 segundos) - valor numérico
- `rate:ETH:USDT`: Tasa de conversión muy volátil (TTL: 5 segundos) - valor numérico
- `ratelimit:auth:{account_id}`: Rate limiting por cuenta (TTL: 60 segundos) - contador de solicitudes
- `idempotent:{transaction_id}`: Idempotencia (TTL: 24 horas) - estado y respuesta de transacción
- `balance_summary:{account_id}`: Resumen de balance (TTL: 10 segundos) - equivalente total en USDT, moneda de pago, timestamp de actualización

### **TimescaleDB - Métricas de Negocio**

#### **Tabla: exchange_rates**
**Estructura:**
- Campos principales: from_currency, to_currency, rate, source (BINANCE, KRAKEN, etc.), valid_from, valid_until, created_at
- Optimización: Hypertable para time-series con particionamiento por valid_from
- Índices: Por par de monedas y fecha, por fuente y fecha

#### **Tabla: balance_ledger (Event Sourcing)**
**Estructura:**
- Campos principales: account_id, created_at, transaction_id, currency, amount_delta (puede ser negativo), balance_before, balance_after, operation_type (DEPOSIT, WITHDRAWAL, PURCHASE, REFUND, RESERVE, RELEASE_RESERVE, TRADE_IN, TRADE_OUT), description, metadata
- Optimización: Hypertable para time-series con particionamiento por created_at
- Índices: Por account_id y fecha, por transaction_id, por operation_type y fecha

## Operaciones Críticas

### **Bloqueo de Fondos (DynamoDB)**

#### **Operación Atómica**
**Funcionalidad:**
- Bloquea fondos de manera atómica usando conditional writes de DynamoDB
- Verifica versión esperada y balance disponible en una sola operación
- Incrementa fondos reservados y versión si las condiciones se cumplen
- Retorna False si hay race condition o fondos insuficientes

#### **Liberación de Fondos**
**Funcionalidad:**
- Libera fondos reservados de manera atómica usando conditional writes
- Verifica versión esperada y fondos reservados disponibles
- Decrementa fondos reservados y incrementa versión si las condiciones se cumplen
- Retorna False si hay race condition

### **Cache de Tasas de Conversión (Redis)**

#### **Configuración Diferenciada por Volatilidad**
**Estrategia de cache multi-nivel:**
- **USDT-ARS**: TTL Redis 30s, TTL memoria 10s, drift máximo 0.5% (estable)
- **BTC-USDT**: TTL Redis 5s, TTL memoria 2s, drift máximo 2.0% (volátil)
- **ETH-USDT**: TTL Redis 5s, TTL memoria 2s, drift máximo 2.5% (muy volátil)

**Flujo de cache:**
1. L1: Cache en memoria (más rápido)
2. L2: Cache en Redis (persistente)
3. L3: API en tiempo real (fallback)
4. Validación de drift antes de usar cache
5. Cachear con TTL apropiado según volatilidad

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
