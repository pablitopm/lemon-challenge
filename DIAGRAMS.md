# Diagramas de Arquitectura

## Diagrama de Secuencia: Flujo Completo de Autorización

```mermaid
sequenceDiagram
    participant Proveedor as Proveedor de Pagos
    participant API as API Gateway
    participant Auth as Servicio de Autorización
    participant Cache as Cache Redis
    participant DB as DynamoDB
    participant Cola as Cola NATS
    participant Trabajador as Trabajador de Trading
    participant External as API de Trading Externa
    participant User as Usuario (Push)

    %% Fase Síncrona: Autorización
    Proveedor->>API: POST /v1/authorizations<br/>{amount_ars, card_id, account_id}
    Note over API: Rate Limiting<br/>Timeout: 2s
    
    API->>Auth: Reenviar Solicitud
    
    Auth->>Cache: GET card:{card_id}
    Cache-->>Auth: Información de Tarjeta (TTL: 5min)
    
    Auth->>Cache: GET balance:{account_id}
    alt Cache Hit
        Cache-->>Auth: Información de Balance
    else Cache Miss
        Auth->>DB: GetItem accounts<br/>WHERE account_id = ?
        DB-->>Auth: Balance de Cuenta
        Auth->>Cache: SET balance:{account_id}
    end
    
    Auth->>Cache: GET rate:USDT:ARS
    Cache-->>Auth: Tasa de Cambio
    
    Note over Auth: Calcular:<br/>amount_usdt = amount_ars / rate
    
    alt Balance Insuficiente
        Auth-->>API: HTTP 402 Payment Required
        API-->>Proveedor: 402 Denegado
    else Balance Suficiente
        Auth->>DB: UpdateItem accounts<br/>ConditionExpression: version = ? AND balance >= amount<br/>UpdateExpression: SET reserved_usdt = reserved_usdt + amount
        DB-->>Auth: Item actualizado
        
        Auth->>DB: PutItem transactions<br/>(status = 'AUTHORIZED', ...)
        DB-->>Auth: ID de Transacción
        
        
        Auth->>Cache: SET idempotent:{trx_id}<br/>TTL: 24h
        
        Auth->>Cola: PUBLISH Evento TxnAuthorized<br/>{transaction_id, account_id, ...}
        Note over Cola: No bloqueante
        
        Auth-->>API: HTTP 200 Autorizado
        API-->>Proveedor: 200 Aprobado
    end
    
    Note over Proveedor: Usuario ve<br/>"APROBADO"<br/>en POS
    
    %% Fase Asíncrona: Trading
    Cola->>Trabajador: CONSUMIR Evento TxnAuthorized
    
     Trabajador->>DB: GetItem transactions<br/>Key: {transaction_id}
     DB-->>Trabajador: Detalles de Transacción
    
    alt Moneda Usuario = USDT
         Trabajador->>DB: UpdateItem accounts<br/>balance_usdt -= amount<br/>reserved_usdt -= amount
         Trabajador->>DB: UpdateItem transactions<br/>status = 'COMPLETED'
        Note over Trabajador: No se necesita trading
    else Moneda Usuario = BTC/ETH
         Trabajador->>DB: UpdateItem transactions<br/>status = 'TRADING'
        
        Trabajador->>Cache: GET rate:BTC:USDT
        Cache-->>Trabajador: Tasa Actual
        
        Note over Trabajador: Calcular monto BTC
        
        Trabajador->>External: POST /trade<br/>{from: BTC, to: USDT, amount}
        Note over External: p95 latency: 3s
        
        alt Trade Exitoso
            External-->>Trabajador: 200 OK<br/>{trade_id, status}
            
             Trabajador->>DB: UpdateItem accounts<br/>balance_btc -= amount<br/>reserved_usdt -= amount
             Trabajador->>DB: UpdateItem transactions<br/>status = 'COMPLETED'<br/>trade_id = ?
            
            Note over Trabajador: ¡Éxito!
        else Trade Timeout/Error
            External-->>Trabajador: Timeout o 500
            
            alt Reintento < 3
                Note over Trabajador: Backoff Exponencial<br/>Reintento en 1s, 2s, 4s
                Trabajador->>Cola: REENCOLAR Evento
            else Reintento >= 3
                 Trabajador->>DB: UpdateItem accounts<br/>reserved_usdt -= amount<br/>(liberar fondos)
                 Trabajador->>DB: UpdateItem transactions<br/>status = 'FAILED'
                
                Trabajador->>User: Notificación Push<br/>"Transacción Revertida"
                
                Note over Trabajador: Alertar Equipo de Ops
            end
        end
    end
```

---

## Diagrama de Arquitectura: Componentes y Flujo de Datos

```mermaid
graph TB
    subgraph External
        PaymentProveedor[Proveedor de Red de Pagos<br/>Visa/Mastercard]
        TradingAPI[Proveedor de Trading Externo<br/>Binance/FTX]
    end
    
    subgraph "Capa de Límite"
        LB[Balanceador de Carga<br/>AWS ALB / nginx]
        APIGateway[API Gateway<br/>Rate Limiting + Auth]
    end
    
    subgraph "Capa de Aplicación"
        AuthServicio1[Servicio de Autorización<br/>Pod 1]
        AuthServicio2[Servicio de Autorización<br/>Pod 2]
        AuthServicio3[Servicio de Autorización<br/>Pod N]
        
        TradingTrabajador1[Trabajador de Trading<br/>Pod 1]
        TradingTrabajador2[Trabajador de Trading<br/>Pod 2]
        
        ReconTrabajador[Trabajador de Reconciliación<br/>Pod 1]
    end
    
    subgraph "Capa de Mensajes"
        NATS[NATS JetStream<br/>Cola de Mensajes]
    end
    
    subgraph "Capa de Cache"
        RedisCluster[Cluster Redis<br/>3 Nodos]
    end
    
        subgraph "Capa de Datos"
            DynamoDB[(DynamoDB<br/>On-demand<br/>1-5ms latencia)]
            RedisCluster[Cluster Redis<br/>3 Nodos<br/>< 1ms latencia]
            TimescaleDB[(TimescaleDB<br/>Métricas de Series de Tiempo)]
        end
    
    subgraph "Observabilidad"
        Prometheus[Prometheus<br/>Métricas]
        Grafana[Grafana<br/>Dashboards]
        Jaeger[Jaeger<br/>Trazado Distribuido]
        Loki[Loki<br/>Agregación de Logs]
    end
    
    %% Conexiones Externas
    PaymentProveedor -->|POST /v1/authorizations<br/>Timeout: 2s| LB
    
    %% Límite a App
    LB --> APIGateway
    APIGateway --> AuthServicio1
    APIGateway --> AuthServicio2
    APIGateway --> AuthServicio3
    
    %% Conexiones Servicio Auth
    AuthServicio1 --> RedisCluster
    AuthServicio2 --> RedisCluster
    AuthServicio3 --> RedisCluster
    
    AuthServicio1 --> DynamoDB
    AuthServicio2 --> DynamoDB
    AuthServicio3 --> DynamoDB
    
    AuthServicio1 --> NATS
    AuthServicio2 --> NATS
    AuthServicio3 --> NATS
    
    %% Cola de Mensajes a Trabajadores
    NATS --> TradingTrabajador1
    NATS --> TradingTrabajador2
    NATS --> ReconTrabajador
    
    %% Conexiones Trabajadores
    TradingTrabajador1 --> RedisCluster
    TradingTrabajador2 --> RedisCluster
    TradingTrabajador1 --> DynamoDB
    TradingTrabajador2 --> DynamoDB
    
    TradingTrabajador1 --> TradingAPI
    TradingTrabajador2 --> TradingAPI
    
    ReconTrabajador --> DynamoDB
    ReconTrabajador --> TradingAPI
    
    %% Database Integration
    DynamoDB -.->|Multi-AZ<br/>Replication| DynamoDB
    
    %% Observability Connections
    AuthServicio1 -.-> Prometheus
    AuthServicio1 -.-> Jaeger
    AuthServicio1 -.-> Loki
    TradingTrabajador1 -.-> Prometheus
    TradingTrabajador1 -.-> Jaeger
    
    Prometheus --> Grafana
    Jaeger --> Grafana
    Loki --> Grafana
    
    style PaymentProveedor fill:#ff6b6b
    style TradingAPI fill:#ff6b6b
    style AuthServicio1 fill:#4ecdc4
    style AuthServicio2 fill:#4ecdc4
    style AuthServicio3 fill:#4ecdc4
    style NATS fill:#ffe66d
    style RedisCluster fill:#95e1d3
    style DynamoDB fill:#6c5ce7
```

---

## Diagrama de Estados: Ciclo de Vida de Transacción

```mermaid
stateDiagram-v2
    [*] --> AUTHORIZED: Solicitud de Autorización<br/>Balance Reservado
    
    AUTHORIZED --> TRADING: Moneda usuario = BTC/ETH<br/>Iniciar trade asíncrono
    AUTHORIZED --> COMPLETED: Moneda usuario = USDT<br/>No se necesita trade
    
    TRADING --> COMPLETED: Trade exitoso<br/>Balance deducido
    TRADING --> TRADING: Trade timeout<br/>Reintento < 3
    TRADING --> FAILED: Trade falló<br/>Reintento >= 3
    
    FAILED --> REVERSED: Liberar fondos reservados<br/>Notificar usuario
    
    COMPLETED --> [*]: Transacción finalizada
    REVERSED --> [*]: Fondos devueltos
    
    note right of AUTHORIZED
        Fondos bloqueados en DB
        reserved_usdt += amount
        Respuesta HTTP 200 enviada
    end note
    
    note right of TRADING
        Trabajador ejecutando trade
        Latencia: 0-5 segundos
        Retry con backoff exponencial
    end note
    
    note right of COMPLETED
        Balance actualizado
        balance_usdt -= amount
        reserved_usdt -= amount
    end note
    
    note right of REVERSED
        Fondos liberados
        reserved_usdt -= amount
        Push notification enviada
    end note
```

---

## Diagrama de Infraestructura: Despliegue en Kubernetes

```mermaid
graph TB
    subgraph "VPC: 10.0.0.0/16"
        subgraph "Public Subnet: 10.0.1.0/24"
            ALB[AWS Application<br/>Load Balancer]
            NAT[NAT Gateway]
        end
        
        subgraph "Private Subnet: 10.0.2.0/24"
            subgraph "EKS Cluster: lemon-production"
                subgraph "Namespace: authorization"
                    AuthDeploy[Despliegue: auth-servicio<br/>Replicas: 4<br/>Resource: 4 vCPU, 8GB]
                    AuthHPA[HPA: 2-10 pods<br/>Objetivo: CPU 70%]
                    AuthServicio[Servicio: auth-svc<br/>Type: ClusterIP]
                end
                
                subgraph "Namespace: trabajadors"
                    TradingDeploy[Despliegue: trading-trabajador<br/>Replicas: 6<br/>Resource: 2 vCPU, 4GB]
                    ReconDeploy[Despliegue: recon-trabajador<br/>Replicas: 2<br/>Resource: 2 vCPU, 4GB]
                end
                
                subgraph "Namespace: messaging"
                    NATSStateful[StatefulSet: nats-cluster<br/>Replicas: 3<br/>Resource: 4 vCPU, 8GB]
                end
            end
        end
        
        subgraph "Private Subnet: 10.0.3.0/24"
            DynamoDB[(DynamoDB<br/>On-demand<br/>Multi-AZ<br/>1-5ms latency)]
            
            ElastiCache[ElastiCache Redis<br/>Cluster Mode<br/>3 Shards x 2 Replicas<br/>8 vCPU, 32GB per node]
            
            TimescaleDB[(TimescaleDB<br/>4 vCPU, 16GB<br/>Time-series)]
        end
    end
    
    Internet((Internet)) --> ALB
    ALB --> AuthServicio
    
    AuthDeploy --> DynamoDB
    AuthDeploy --> ElastiCache
    AuthDeploy --> NATSStateful
    
    TradingDeploy --> DynamoDB
    TradingDeploy --> ElastiCache
    TradingDeploy --> NATSStateful
    
    ReconDeploy --> DynamoDB
    
    TradingDeploy --> NAT
    ReconDeploy --> NAT
    NAT --> Internet
    
    AuthHPA -.-> AuthDeploy
    
    style ALB fill:#ff6b6b
    style DynamoDB fill:#6c5ce7
    style ElastiCache fill:#95e1d3
    style NATSStateful fill:#ffe66d
```

---

## Diagrama de Flujo: Manejo de Concurrencia de Saldo

```mermaid
flowchart TD
    Start([Solicitud 1 & Solicitud 2<br/>Simultáneos])
    
     Start --> Read1[Solicitud 1:<br/>GetItem account<br/>balance=1000, version=5]
     Start --> Read2[Solicitud 2:<br/>GetItem account<br/>balance=1000, version=5]
    
    Read1 --> Check1{Balance suficiente?<br/>600 <= 1000}
    Read2 --> Check2{Balance suficiente?<br/>500 <= 1000}
    
    Check1 -->|Sí| Update1[UpdateItem accounts<br/>ConditionExpression: version = 5<br/>UpdateExpression: SET reserved_usdt = 600]
    
    Check2 -->|Sí| Update2[UpdateItem accounts<br/>ConditionExpression: version = 5<br/>UpdateExpression: SET reserved_usdt = 500]
    
    Update1 --> Result1{Rows Updated?}
    Update2 --> Result2{Rows Updated?}
    
    Result1 -->|1 row| Success1[  Solicitud 1 Aprobado<br/>version = 6<br/>reserved = 600]
    Result1 -->|0 rows| Retry1[  Version Conflict<br/>Retry once]
    
    Result2 -->|1 row| Success2[  Solicitud 2 Aprobado]
    Result2 -->|0 rows| Fail2[  Solicitud 2 Denegado<br/>HTTP 402<br/>Fondos Insuficientes]
    
     Retry1 --> ReRead1[GetItem account again<br/>balance=1000<br/>reserved=600<br/>version=6]
    
    ReRead1 --> ReCheck1{Available?<br/>1000 - 600 >= 500?}
    ReCheck1 -->|No| Fail1[  Solicitud 1 Denegado<br/>HTTP 402]
     ReCheck1 -->|Sí| ReUpdate1[UpdateItem con version=6]
    ReUpdate1 --> FinalSuccess[  Solicitud 1 Aprobado]
    
    Success1 --> End([Fin])
    Success2 --> End
    Fail1 --> End
    Fail2 --> End
    FinalSuccess --> End
    
    style Success1 fill:#90EE90
    style Success2 fill:#90EE90
    style FinalSuccess fill:#90EE90
    style Fail1 fill:#FFB6C1
    style Fail2 fill:#FFB6C1
```

---

## Diagrama de Componentes: Trabajador de Trading con Circuit Breaker

```mermaid
graph TB
    subgraph "Trabajador de Trading"
        Consumer[Consumidor NATS<br/>Concurrencia: 50]
        
        Consumer --> Processor[Procesador de Eventos]
        
        Processor --> CB{Estado Circuit Breaker?}
        
        CB -->|CERRADO| Execute[Ejecutar Trade]
        CB -->|ABIERTO| Skip[Omitir Trade<br/>Marcar PENDING_TRADE<br/>Reintentar después]
        CB -->|SEMI_ABIERTO| Test[Solicitud de Prueba<br/>1 de 3]
        
        Execute --> API[API de Trading Externa<br/>POST /trade]
        Test --> API
        
        API -->|Éxito| Success[Actualizar Estado:<br/>COMPLETED]
        API -->|Error| Error[Manejar Error]
        API -->|Timeout| Timeout[Manejar Timeout]
        
        Success --> UpdateDB[(Actualizar DynamoDB)]
        
        Error --> ErrorCount{Tasa de Error<br/>> 50%?}
        ErrorCount -->|Sí| OpenCB[Abrir Circuito<br/>Por 30s]
        ErrorCount -->|No| Retry[Reintentar con<br/>Backoff]
        
        Timeout --> RetryTimeout[Reintentar con<br/>Backoff]
        
        RetryTimeout --> RetryCount{Reintento<br/>< 3?}
        RetryCount -->|Sí| Recola[Reencolar Evento<br/>+1s, +2s, +4s]
        RetryCount -->|No| Failed[Marcar FAILED<br/>Revertir Transacción]
        
        Retry --> RetryCount
        
        OpenCB --> Alert[Alertar Equipo de Ops]
        Failed --> Notification[Notificación Push<br/>al Usuario]
        
        Recola --> Consumer
    end
    
    subgraph "Circuit Breaker States"
        StateClosed[CLOSED<br/>Normal Operation<br/>All solicituds pass]
        StateOpen[OPEN<br/>All solicituds fail fast<br/>Duration: 30s]
        StateHalf[HALF_OPEN<br/>Test with limited solicituds<br/>3 attempts]
        
        StateClosed -->|Error rate > 50%| StateOpen
        StateOpen -->|After 30s| StateHalf
        StateHalf -->|3 success| StateClosed
        StateHalf -->|Any failure| StateOpen
    end
    
    style Success fill:#90EE90
    style Failed fill:#FFB6C1
    style OpenCB fill:#FFA500
```

---

## Diagrama de Datos: Relaciones entre Entidades

```mermaid
erDiagram
    ACCOUNTS ||--o{ CARDS : tiene
    ACCOUNTS ||--o{ TRANSACTIONS : realiza
    CARDS ||--o{ TRANSACTIONS : usa
    ACCOUNTS ||--o{ BALANCE_LEDGER : registra
    TRANSACTIONS ||--o{ BALANCE_LEDGER : genera
    
    ACCOUNTS {
        uuid id PK
        string account_id UK
        decimal balance_usdt
        decimal balance_btc
        decimal balance_eth
        decimal reserved_usdt
        decimal reserved_btc
        decimal reserved_eth
        string payment_currency
        string status
        int version
        timestamp created_at
        timestamp updated_at
    }
    
    CARDS {
        uuid id PK
        string card_id UK
        uuid account_id FK
        string card_number_hash
        string last_four
        string status
        decimal daily_limit_usdt
        timestamp created_at
    }
    
    TRANSACTIONS {
        uuid id PK
        string transaction_id UK
        uuid account_id FK
        uuid card_id FK
        decimal amount_ars
        decimal amount_usdt
        decimal exchange_rate
        string user_currency
        string status
        string trade_id
        timestamp authorized_at
        timestamp completed_at
        int retry_count
    }
    
    BALANCE_LEDGER {
        bigint id PK
        uuid account_id FK
        uuid transaction_id FK
        string currency
        decimal amount_delta
        decimal balance_before
        decimal balance_after
        string operation_type
        timestamp created_at
    }
    
    EXCHANGE_RATES {
        uuid id PK
        string from_currency
        string to_currency
        decimal rate
        timestamp valid_from
        timestamp valid_until
    }
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

---

## Diagrama de Flujo: Manejo de Montos Grandes

```mermaid
flowchart TD
    Start([Transacción Grande<br/>ej: 10M ARS])
    
    Start --> Validate{Validar Límites}
    Validate -->|Excede Límite| Reject[Rechazar<br/>HTTP 402<br/>AMOUNT_EXCEEDS_LIMIT]
    Validate -->|Dentro Límite| CheckSize{¿Monto > 5M ARS?}
    
    CheckSize -->|Sí| ManualApproval[Requerir Aprobación Manual<br/>HTTP 202<br/>MANUAL_APPROVAL_REQUIRED]
    CheckSize -->|No| CheckOrderSize{¿Monto > 10K USDT?}
    
    CheckOrderSize -->|No| NormalFlow[Flujo Normal<br/>Una sola orden]
    CheckOrderSize -->|Sí| SplitOrder[División Automática<br/>TWAP Strategy]
    
    SplitOrder --> Calculate[Calcular Número de Órdenes<br/>ej: 10M ARS = 100 órdenes de 100K ARS]
    Calculate --> Schedule[Programar Órdenes<br/>30s entre cada una]
    
    Schedule --> Execute1[Ejecutar Orden 1<br/>100K ARS]
    Execute1 --> Wait1[Esperar 30s]
    Wait1 --> Execute2[Ejecutar Orden 2<br/>100K ARS]
    Execute2 --> Wait2[Esperar 30s]
    Wait2 --> ExecuteN[Ejecutar Orden N<br/>100K ARS]
    
    ExecuteN --> Reconcile[Reconciliar Ejecuciones<br/>Verificar % ejecutado]
    Reconcile --> CheckExecution{¿>95% ejecutado?}
    
    CheckExecution -->|Sí| Success[Éxito<br/>Notificar Usuario<br/>Actualizar Balance]
    CheckExecution -->|No| Revert[Revertir Todo<br/>Liberar Fondos<br/>Notificar Falla]
    
    NormalFlow --> End([Fin])
    Success --> End
    Reject --> End
    ManualApproval --> End
    Revert --> End
    
    style Start fill:#e1f5fe
    style Reject fill:#ffebee
    style ManualApproval fill:#fff3e0
    style SplitOrder fill:#f3e5f5
    style Success fill:#e8f5e8
    style Revert fill:#ffebee
```

---

## Notas de Implementación

### Consideraciones de Rendimiento

1. **Connection Pooling:**
   - DynamoDB: On-demand billing con auto-scaling
   - Redis: Pool de 50 conexiones por instancia
   - NATS: 10 conexiones por trabajador

2. **GSI Críticos:**
   - accounts: user-accounts (user_id, created_at)
   - transactions: status-transactions (status, created_at)
   - cards: account-cards (account_id, created_at)

3. **Conditional Writes:**
   - Todas las operaciones críticas usan conditional writes
   - Garantiza atomicidad sin transacciones

4. **Batch Operations:**
   - Reconciliation trabajador procesa en batches de 100 transacciones
   - Reduce round-trips a DynamoDB

### Despliegue Strategy

1. **Blue-Green Despliegue:**
   - Zero downtime despliegues
   - Rollback inmediato si hay issues

2. **Canary Releases:**
   - 10% → 50% → 100% tráfico
   - Monitoreo automático de error rates

3. **Database Migrations:**
   - Online schema changes (gh-ost / pt-online-schema-change)
   - No locking de tablas

### Disaster Recovery

1. **Backup Strategy:**
   - DynamoDB: Point-in-time recovery + cross-region backup
   - Redis: RDB snapshots cada 5 minutos
   - Retention: 30 días

2. **RTO/RPO:**
   - RTO: < 1 hora (Recovery Time Objective)
   - RPO: < 5 minutos (Recovery Point Objective)

3. **Failover:**
   - Automático para Redis y DynamoDB
   - Manual para NATS (requiere validación)

