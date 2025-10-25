# Diagramas de Arquitectura

## Diagrama de Secuencia: Flujo Completo de Autorización

```mermaid
sequenceDiagram
    participant LEMON as LEMON Payment Network
    participant API as API Gateway
    participant Auth as Authorization Service
    participant Redis as Redis Cache
    participant DynamoDB as DynamoDB
    participant NATS as NATS Queue
    participant Worker as Trading Worker
    participant Trading as External Trading API

    LEMON->>API: POST /v1/authorizations<br/>{amount, card_id, account_id, transaction_id}
    API->>Auth: Forward request
    
    Auth->>Redis: GET card:{card_id}
    Redis-->>Auth: Card info (account_id, status, limits)
    
    Auth->>Redis: GET balance:{account_id}
    Redis-->>Auth: Balance summary
    
    Auth->>Redis: GET rate:USDT:ARS
    Redis-->>Auth: Exchange rate
    
    Auth->>Auth: Calculate amount in USDT<br/>Validate balance & limits
    
    Auth->>DynamoDB: UpdateItem accounts<br/>Conditional write: reserve funds
    DynamoDB-->>Auth: Success/Failure
    
    alt Funds Reserved Successfully
        Auth->>DynamoDB: PutItem transactions<br/>Status: AUTHORIZED
        Auth->>NATS: Publish trading event
        Auth->>Redis: SET idempotent:{txn_id}
        Auth-->>API: HTTP 200 {authorized: true}
        API-->>LEMON: HTTP 200 {authorized: true}
        
        Note over Worker: Asynchronous Processing
        Worker->>NATS: Consume trading event
        Worker->>Trading: POST /trade<br/>Convert user currency to USDT
        Trading-->>Worker: Trade result
        
        alt Trade Successful
            Worker->>DynamoDB: UpdateItem transactions<br/>Status: COMPLETED
            Worker->>DynamoDB: UpdateItem accounts<br/>Release reserved funds
        else Trade Failed
            Worker->>DynamoDB: UpdateItem transactions<br/>Status: FAILED
            Worker->>DynamoDB: UpdateItem accounts<br/>Release reserved funds
        end
        
    else Insufficient Funds or Error
        Auth-->>API: HTTP 402 {authorized: false, reason: "INSUFFICIENT_FUNDS"}
        API-->>LEMON: HTTP 402 {authorized: false}
    end
```

---

## Diagrama de Arquitectura: Componentes y Flujo de Datos

![Arquitectura de Componentes](./02-arquitectura-componentes.png)

---

## Diagrama de Estados: Ciclo de Vida de Transacción

![Estados de Transacción](./03-estados-transaccion.png)

---

## Diagrama de Datos: Estructura NoSQL (DynamoDB)

```mermaid
graph TB
    subgraph "DynamoDB Tables"
        A[ACCOUNTS Table<br/>PK: account_id<br/>Attributes: balances, payment_currency, version]
        T[TRANSACTIONS Table<br/>PK: transaction_id<br/>GSI: account_id-index<br/>Attributes: amount, status, timestamps]
        B[BALANCE_LEDGER Table<br/>PK: account_id + timestamp<br/>Attributes: operation_type, amounts, balances]
    end
    
    subgraph "Redis Cache"
        R1[Exchange Rates<br/>Key: currency_pair<br/>TTL: 30s-5s]
        R2[User Sessions<br/>Key: session_id<br/>TTL: 1h]
        R3[Rate Limits<br/>Key: user_id<br/>TTL: 1h]
    end
    
    A --> |"Query by account_id"| T
    A --> |"Query by account_id"| B
    
    R1 -.-> |"Cache miss fallback"| A
    R2 -.-> |"Session validation"| A
    R3 -.-> |"Rate limit check"| T
    
    style A fill:#ff9999
    style T fill:#ff9999
    style B fill:#ff9999
    style R1 fill:#99ccff
    style R2 fill:#99ccff
    style R3 fill:#99ccff
```

---

## Diagrama de Latencias: Presupuesto de Tiempo (2 segundos)

```mermaid
gantt
    title Presupuesto de Tiempo de Solicitud de Autorización (Objetivo: < 500ms)
    dateFormat SSS
    axisFormat %L ms
    
    section Procesamiento de Solicitud
    API Gateway (Rate Limit + Auth) :done, gw, 000, 20ms
    
    section Servicio de Autorización
    Validar Esquema de Solicitud :done, val, after gw, 5ms
    Obtener Info de Tarjeta (Redis) :done, card, after val, 15ms
    Obtener Balance de Cuenta (Redis) :done, bal, after card, 20ms
    Obtener Tasa de Cambio (Redis) :done, rate, after bal, 10ms
    Calcular y Validar Balance :done, calc, after rate, 5ms
    
    section Operaciones de Base de Datos
    Reservar Balance (UpdateItem) :crit, reserve, after calc, 80ms
    Crear Transacción (PutItem) :crit, insert, after reserve, 60ms
    
    section Operaciones Asíncronas
    Publicar a Cola NATS :done, pub, after insert, 15ms
    Cachear Clave de Idempotencia :done, cache, after pub, 10ms
    
    section Respuesta
    Construir y Enviar HTTP 200 :done, resp, after cache, 10ms
    
    section Total
    TIEMPO TOTAL SÍNCRONO :milestone, after resp, 0ms
```

**Desglose:**
- **Objetivo:** 250-500ms (p95)
- **Buffer:** 1500ms disponible para picos
- **Ruta Crítica:** Operaciones de base de datos (140ms)
- **Optimización:** Aciertos en cache reducen latencia a ~50ms