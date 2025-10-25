# Implementación de Base de Datos

## DynamoDB - Estructura de Tablas

### **Tabla: accounts**
**Clave de partición:** `account_id` (String)

**Atributos principales:**
- `user_id` (String)
- `balance_usdt`, `balance_btc`, `balance_eth` (Number)
- `reserved_usdt`, `reserved_btc`, `reserved_eth` (Number)
- `payment_currency` (String) - "USDT", "BTC", "ETH"
- `status` (String) - "ACTIVE", "SUSPENDED", "CLOSED"
- `version` (Number) - Para conditional writes
- `created_at`, `updated_at` (String)

**GSI:**
- `user-accounts`: PK=`user_id`, SK=`created_at`

### **Tabla: transactions**
**Clave de partición:** `transaction_id` (String - del proveedor LEMON)

**Atributos principales:**
- `account_id`, `card_id` (String)
- `amount_ars`, `amount_usdt` (Number)
- `exchange_rate` (Number)
- `user_currency` (String)
- `status` (String) - "AUTHORIZED", "TRADING", "COMPLETED", "FAILED"
- `trade_id`, `trade_status` (String)
- `retry_count` (Number)
- `authorized_at`, `completed_at` (String)

**GSI:**
- `account-transactions`: PK=`account_id`, SK=`authorized_at`
- `status-transactions`: PK=`status`, SK=`authorized_at`

### **Tabla: cards**
**Clave de partición:** `card_id` (String)

**Atributos principales:**
- `account_id` (String)
- `card_number_hash` (String) - Hash SHA-256
- `last_four` (String)
- `holder_name` (String)
- `expiry_date` (String)
- `status` (String) - "ACTIVE", "BLOCKED", "EXPIRED"
- `daily_limit_usdt` (Number)
- `created_at` (String)

**GSI:**
- `account-cards`: PK=`account_id`, SK=`created_at`

## Redis - Estructura de Claves

### **Cache de Tasas de Conversión**
```
rate:USDT:ARS     → TTL: 30s (estable)
rate:BTC:USDT     → TTL: 5s  (volátil)
rate:ETH:USDT     → TTL: 5s  (muy volátil)
```

### **Cache de Información de Usuario**
```
card:{card_id}           → TTL: 5min (account_id, status, limits)
balance:{account_id}     → TTL: 60s  (resumen de balances)
idempotent:{txn_id}      → TTL: 24h  (respuesta de transacción)
```

### **Rate Limiting**
```
ratelimit:auth:{account_id}  → TTL: 60s (contador de requests)
```

## Operaciones Críticas

### **Bloqueo de Fondos (DynamoDB Conditional Write)**
1. Leer `account_id` con versión actual
2. Verificar `balance_currency >= amount + reserved_currency`
3. Conditional Write: incrementar `reserved_currency` y `version`
4. Si falla: retry o rechazar transacción

### **Liberación de Fondos**
1. Conditional Write: decrementar `reserved_currency` y `version`
2. Si falla: log para reconciliación

### **Cache de Tasas**
1. Verificar TTL y drift máximo permitido
2. Si expirado o drift alto: fetch de API externa
3. Cachear con TTL según volatilidad de la moneda