# Solución Técnica: Sistema de Autorización de Pagos

## Tabla de Contenidos

1. [Resumen](#resumen-ejecutivo)
2. [Arquitectura de la Solución](#arquitectura-de-la-solución)
3. [Flujo de Autorización](#flujo-de-autorización)
4. [Decisiones Técnicas y Trade-offs](#decisiones-técnicas-y-trade-offs)
5. [Esquema de Datos](#esquema-de-datos)
6. [Manejo de Casos Límite](#manejo-de-casos-límite)
7. [Observabilidad y Monitoreo](#observabilidad-y-monitoreo)
8. [Suposiciones](#suposiciones)

---

## Resumen

La solución propuesta utiliza un **patrón asíncrono híbrido** que permite cumplir con la restricción de 2 segundos de LEMON, mientras se integra con un servicio de trading que tiene latencias de hasta 3 segundos.

### Estrategia Principal: Pre-autorización + Procesamiento Asíncrono

1. **Autorización inmediata** (< 500ms): Validar fondos y responder a LEMON
2. **Trading asíncrono** (segundo plano): Ejecutar conversión de criptomonedas si es necesario
3. **Reconciliación** (post-autorización): Confirmar o revertir transacciones fallidas

---

## Arquitectura de la Solución

### Componentes Principales

```
┌─────────────────────────────────────────────────────────────┐
│                    LEMON Payment Network                    │
└────────────────────────┬────────────────────────────────────┘
                         │ POST /v1/authorizations (2s timeout)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  API Gateway / Load Balancer                │
│                    (Rate Limiting + Auth)                   │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│               Authorization Servicio (Go)                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  1. Validación de Tarjeta (Redis Cache)               │  │ 
│  │  2. Consulta de Balance (DynamoDB)                    │  │
│  │  3. Conversión ARS → USDT (Redis Cache)               │  │
│  │  4. Bloqueo de Fondos (DynamoDB Conditional Write)    │  │
│  │  5. Respuesta Inmediata HTTP 200/402                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                         │                                   │
│                         ▼                                   │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Async: Publicar Evento a Queue                       │  │
│  └───────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Mensaje Cola (NATS / Kafka)                    │
└────────────┬──────────────────────────────┬─────────────────┘
             │                              │
             ▼                              ▼
┌────────────────────────┐    ┌────────────────────────────────┐
│  Trading Worker        │    │  Reconciliation Worker         │
│                        │    │                                │
│  - Consume eventos     │    │  - Verificar estados           │
│  - Ejecutar trades     │    │  - Liberar fondos bloqueados   │
│  - Actualizar estado   │    │  - Manejar reversiones         │
│  - Retry con backoff   │    │  - Alertas de inconsistencias  │
└────────────┬───────────┘    └────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────────┐
│              External Trading Proveedor API                 │
│                    POST /trade (p95: 3s)                    │
└─────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                    Data Layer                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ DynamoDB     │  │ Redis Cache  │  │  TimescaleDB │      │
│  │ (Balances,   │  │ (Tarjetas,   │  │  (Métricas)  │      │
│  │  Transacc.)  │  │  Tasas)      │  │              │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────────────────────────────────────────┘
```

---

## Flujo de Autorización

### Fase 1: Autorización Síncrona (< 2 segundos)

```
┌─────────┐
│ Request │
└────┬────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 1. Validar Request                  │ < 5ms
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 2. Validar Tarjeta y Cuenta         │ < 50ms
│    - check card exista              │
│    - Check account activa           │
│    - Check card linkeada a account  │
│    - Cache: 5 min TTL               │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 3. Obtener Balance del Usuario      │ < 10ms
│    - Dynamo                         │
│    - Balance en USDT, BTC, ETH      │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 4. Conversión ARS → USDT            │ < 20ms
│    - Tasa desde Redis/Cache         │
│    - Calcular: amount_usdt =        │
│      amount_ars / tasa_usdt_ars     │
└────┬────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│ 5. Validar Balance Suficiente       │ < 10ms
│    - Si user.currency == USDT:      │
│      ✓ balance_usdt >= amount_usdt  │
│    - Si user.currency == BTC/ETH:   │
│      ✓ Convertir balance a USDT     │
│      ✓ balance_usdt_equiv >= amt    │
└────┬────────────────────────────────┘
     │
     ├─── NO ──────────────────────────┐
     │                                 ▼
     │                        ┌──────────────────┐
     │                        │ HTTP 402         │
     │                        │ Payment Required │
     │                        │ (Fondos Insuf.)  │
     │                        └──────────────────┘
     │
     │─── SI ───────────────────────────┐
                                        ▼
                           ┌──────────────────────────────┐
                           │ 6. Bloqueo de Fondos         │ < 100ms
                           │    - DynamoDB UpdateItem     │
                           │      reserved_balance = +amt │
                           │      WHERE account_id = X    │
                           │    - Conditional Write       │
                           │    - Atomicidad garantizada  │
                           └──────┬───────────────────────┘
                                  │
                                  ▼
                           ┌──────────────────────────────┐
                           │ 7. Crear Registro            │ < 50ms
                           │    - DynamoDB PutItem        │
                           │    - Status: AUTHORIZED      │
                           │    - Estado inicial          │
                           └──────┬───────────────────────┘
                                  │
                                  ▼
                           ┌──────────────────────────────┐
                           │ 8. Publicar Evento           │ < 50ms
                           │    - Mensaje Cola           │
                           │    - Event: TxnAuthorized    │
                           │    - Non-blocking            │
                           └──────┬───────────────────────┘
                                  │
                                  ▼
                           ┌──────────────────────────────┐
                           │ 9. Responder HTTP 200        │
                           │    - Authorization approved  │
                           │    - Total: ~400ms           │
                           └──────────────────────────────┘
```

### Fase 2: Procesamiento Asíncrono (Background)

```
┌──────────────────────────────┐
│ Mensaje Cola Event          │
│ {                            │
│   transaction_id,            │
│   account_id,                │
│   amount_usdt,               │
│   user_currency              │
│ }                            │
└──────┬───────────────────────┘
       │
       ▼
┌──────────────────────────────┐
│ Trading Worker Consumer      │
└──────┬───────────────────────┘
       │
       ▼
   ┌───────────────────┐
   │ ¿User currency    │
   │ es USDT?          │
   └───┬───────────────┘
       │
       ├─── SÍ ────────────────────────┐
       │                               ▼
       │                    ┌──────────────────────┐
       │                    │ Finalizar            │
       │                    │ Status: COMPLETED    │
       │                    │ Deducir de balance   │
       │                    │ Liberar reserved     │
       │                    └──────────────────────┘
       │
       ├─── NO (BTC/ETH) ──────────────┐
                                       ▼
                          ┌─────────────────────────────┐
                          │ Ejecutar Trade              │
                          │ POST /trade                 │
                          │ {                           │
                          │   from: BTC/ETH,            │
                          │   to: USDT,                 │
                          │   amount: X                 │
                          │ }                           │
                          │ Timeout: 5s                 │
                          │ Retry: 3 intentos           │
                          └──────┬──────────────────────┘
                                 │
                                 ├─── Success ──────────┐
                                 │                      ▼
                                 │          ┌────────────────────┐
                                 │          │ Status: COMPLETED  │
                                 │          │ Deducir balance    │
                                 │          │ Liberar reserved   │
                                 │          └────────────────────┘
                                 │
                                 ├─── Timeout/Error ────┐
                                 │                      ▼
                                 │          ┌────────────────────┐
                                 │          │ Status: FAILED     │
                                 │          │ Revertir txn       │
                                 │          │ Liberar fondos     │
                                 │          │ Notificar usuario  │
                                 │          │ Alerta ops         │
                                 │          └────────────────────┘
                                 │
                                 └─── Retry ────────────┐
                                                       ▼
                                          ┌────────────────────┐
                                          │ Recola con        │
                                          │ Backoff            │
                                          │ (1s, 2s, 4s)       │
                                          └────────────────────┘
```

---

## Decisiones Técnicas y Trade-offs

### Patrón Asíncrono para Trading

**Decisión:** Separar la autorización (síncrona) del trading (asíncrono).

**Justificación:**
- El proveedor de trading tiene p95 de 3s, excediendo el límite de 2s
- Mantener la UX: usuario ve "aprobado" inmediatamente en el POS
- Bloqueo de fondos garantiza que el usuario no pueda gastar el mismo dinero

**Trade-offs:**
- **Pro:** Cumple con timeout de 2 segundos
- **Pro:** Mejor experiencia de usuario
- **Pro:** Permite reintentos de trading sin afectar autorización
- **Con:** Complejidad adicional en reconciliación
- **Con:** Posibles reversiones si el trade falla
- **Con:** Necesita sistema de notificaciones para reversiones

**Mitigación de Riesgos:**
- Bloqueo inmediato de fondos (reserved_balance)
- Sistema robusto de reconciliación
- Notificaciones push al usuario en caso de reversión
- Monitoreo de tasa de reversiones (debe ser < 0.1%)

---

### Bloqueo de Fondos con DynamoDB Conditional Writes

**Decisión:** Usar DynamoDB conditional writes para bloqueo atómico de fondos.

**Justificación:**
- **Latencia 10x menor:** 2-5ms vs 20-50ms de PostgreSQL
- **Conditional writes nativos:** Atomicidad garantizada sin optimistic locking manual
- **Durabilidad multi-AZ:** Replicación automática sin configuración
- **Escalabilidad automática:** Sin sharding manual
- **Transacciones ACID:** Disponibles desde 2019

**Operación Atómica:**
- UpdateItem con ConditionExpression
- Verificación de version y balance disponible en una operación
- Falla atómica si condiciones no se cumplen

**Trade-offs:**
- **Pro:** Latencia ultra-baja (2-5ms)
- **Pro:** Atomicidad nativa sin race conditions
- **Pro:** Durabilidad garantizada multi-AZ
- **Pro:** Sin ops overhead (managed servicio)
- **Con:** Costo mayor que PostgreSQL (~$1,075/mes vs $200/mes)
- **Con:** Vendor lock-in a AWS

**Mitigación:**
- Costo justificado por 10x mejora en latencia
- Backup automático y point-in-time recovery
- CloudTrail integration para auditoría

---

### Cache Multi-Nivel para Tasas de Conversión

**Decisión:** Cache diferenciado por volatilidad de moneda.

**Configuración de Cache Diferenciado:**
- **USDT/ARS**: Cache de 30 segundos (estable, baja volatilidad)
- **BTC/USDT**: Cache de 5 segundos (volátil, alta volatilidad) 
- **ETH/USDT**: Cache de 5 segundos (muy volátil, muy alta volatilidad)
- **Validación de drift**: Verificar desviación antes de usar cache (0.5% USDT, 2% BTC, 2.5% ETH)

**Justificación:**
- **USDT/ARS**: Cache de 30s (estable, baja volatilidad)
- **BTC/ETH**: Cache de 5s (volátil, alta volatilidad)
- **Validación de drift**: Verificar desviación antes de usar cache
- **Fallback en tiempo real**: Para pares muy volátiles

**Trade-offs:**
- **Pro:** Latencia optimizada por tipo de moneda
- **Pro:** Reduce riesgo de arbitraje en crypto volátiles
- **Pro:** Mantiene performance para pares estables
- **Con:** Complejidad adicional en lógica de cache
- **Con:** Más llamadas a API para crypto volátiles

**Mitigación:**
- **TTL diferenciado**: 30s para estables, 5s para volátiles
- **Validación de drift**: Rechazar cache si desviación > umbral
- **Fallback en tiempo real**: API directa para crypto volátiles
- **Circuit breaker**: Si tasas divergen > umbral por moneda

---

### Mensaje Cola: NATS Streaming vs Kafka

**Decisión:** NATS Streaming (o NATS JetStream).

**Justificación:**
- Latencia más baja que Kafka (< 1ms pub)
- Más simple de operar
- At-least-once delivery suficiente
- Throughput adecuado para caso de uso

**Comparación:**

| Criterio | NATS | Kafka |
|----------|------|-------|
| Latencia pub | < 1ms | 5-10ms |
| Throughput | 1M+ msg/s | 10M+ msg/s |
| Operación | Simple | Compleja |
| Retención | Limitada | Ilimitada |
| Use case | Real-time | Event sourcing |

**Para nuestro caso:** NATS es suficiente. Kafka sería over-engineering a menos que necesitemos event sourcing completo.

---

### Base de Datos: Arquitectura Híbrida Optimizada

**Decisión:** DynamoDB para operaciones críticas + Redis para cache + TimescaleDB para métricas.

**Justificación:**
- **DynamoDB:** Latencia 10x menor (2-5ms vs 20-50ms) para bloqueo de fondos
- **Conditional writes nativos:** Mejor que optimistic locking manual
- **Durabilidad garantizada:** Multi-AZ automático sin ops overhead
- **Redis:** Cache ultra-rápido (< 1ms) para datos frecuentemente accedidos
- **TimescaleDB:** Optimizado para métricas de tiempo y analytics

**Arquitectura de Datos:**
- **DynamoDB:** Balances, transacciones, bloqueo de fondos
- **Redis:** Cache de tarjetas, tasas, rate limiting, idempotencia
- **TimescaleDB:** Métricas, logs de auditoría, analytics

**Alternativas Consideradas:**
- **PostgreSQL puro:** Latencia 10x mayor para operaciones críticas
- **MongoDB:** No garantiza transacciones ACID consistentes

### Observabilidad: TimescaleDB vs Datadog/CloudWatch

**Decisión:** Estrategia híbrida con TimescaleDB + Datadog.

**Justificación:**

**TimescaleDB para métricas de negocio:**
- **Costo predecible**: aprox 200/mes vs 2,000/mes con Datadog
- **Queries SQL complejas**: Análisis ad-hoc y JOINs con datos transaccionales
- **Retención ilimitada**: 2+ años sin costo adicional
- **Control total**: Queries personalizadas y análisis profundo

**Datadog para observabilidad operacional:**
- **Setup cero**: 5 minutos
- **Observabilidad completa**: Métricas + logs + APM + alertas
- **Mantenimiento automático**: Sin ops overhead
- **Alertas inteligentes**: Machine learning y detección de anomalías
- **Integración con 500+ servicios**: Ecosistema completo

**Estrategia híbrida implementada:**
- **TimescaleDB**: Métricas de negocio (transacciones, balances, performance)
- **Datadog**: Infraestructura, logs, APM, alertas operacionales
- **Costo total**: ~$500/mes vs 2,000/mes solo con Datadog
- **Beneficio**: Control total + observabilidad completa + costo optimizado

**Trade-offs:**
- **Pro:** Análisis profundo con TimescaleDB + observabilidad completa con Datadog
- **Pro:** Costo optimizado para métricas de alto volumen
- **Pro:** Queries complejas para auditoría y análisis de negocio
- **Con:** Dos sistemas de observabilidad para mantener
- **Con:** Curva de aprendizaje para TimescaleDB

---

### Manejo de Idempotencia

**Decisión:** Usar `transaction_id` del proveedor como clave de idempotencia con caché en Redis.

**Justificación:**
- La red puede causar reintentos del proveedor
- Evita doble cobro al usuario
- Requerimiento regulatorio en sistemas financieros
- Caché en Redis para latencia ultra-baja (< 1ms)

**Implementación:**
- **DynamoDB**: `transaction_id` como clave primaria (único)
- **Redis**: Caché de respuesta por 24 horas
- **Flujo**: Verificar Redis → Verificar DynamoDB → Procesar → Guardar respuesta
- **Resultado**: Misma respuesta para solicitudes duplicadas

**Ejemplo Práctico:**
1. **Primera solicitud**: `transaction_id: "trx_123"` → Procesar y guardar respuesta
2. **Segunda solicitud**: `transaction_id: "trx_123"` → Devolver respuesta guardada (sin procesar)
3. **Usuario**: Solo cobrado una vez, aunque el proveedor haya enviado dos solicitudes

---

### Circuit Breaker para Trading Proveedor

**Decisión:** Implementar circuit breaker con estados abierto/semi-abierto/cerrado.

**Configuración:**
- **Threshold:** 50% de errores en ventana de 10s
- **Timeout:** 30 segundos en estado abierto
- **Half-open:** Permitir 3 solicituds de prueba

**Justificación:**
- Trading proveedor puede tener downtime
- Evitar cascada de failures
- Degradación elegante del servicio

**Comportamiento cuando abierto:**
- No intentar trades
- Marcar transacciones como PENDING_TRADE
- Procesamiento diferido cuando se recupere

---

### Rate Limiting por Usuario**

**Decisión:** Token bucket algorithm con límites:
- 10 autorizaciones / minuto por usuario
- 50 autorizaciones / minuto por IP

**Justificación:**
- Prevenir abuso / fraude
- Proteger sistema de spike inesperado
- Requerimiento de seguridad

---

## Esquema de Datos

### Arquitectura de Datos

El sistema utiliza una arquitectura híbrida optimizada para diferentes tipos de datos:

#### **DynamoDB - Datos Críticos**
- **Cuentas**: Balances en USDT, BTC, ETH con fondos reservados
- **Tarjetas**: Información de tarjetas con límites y estado
- **Transacciones**: Estado completo del ciclo de vida de cada transacción
- **Características**: Conditional writes para bloqueo atómico de fondos

#### **Redis - Cache de Alto Rendimiento**
- **Tasas de conversión**: Cache con TTL diferenciado por volatilidad
- **Idempotencia**: Respuestas de transacciones por 24 horas
- **Rate limiting**: Contadores temporales por usuario
- **Información de tarjetas**: Cache de 5 minutos para consultas frecuentes

#### **TimescaleDB - Métricas de Negocio**
- **Tasas históricas**: Series temporales para análisis de volatilidad
- **Auditoría de balances**: Event sourcing para trazabilidad completa
- **Métricas de negocio**: Volúmenes de transacciones, conversiones por moneda
- **Características**: Optimizado para queries de series temporales y análisis complejos

#### **Datadog - Observabilidad Operacional**
- **Métricas de performance**: Latencias, throughput, errores
- **Logs de aplicación**: Trazabilidad de requests y errores
- **APM**: Trazas distribuidas para debugging
- **Alertas**: Monitoreo proactivo de SLOs

### Principios de Diseño

1. **Separación por Caso de Uso**: Cada base de datos optimizada para su función específica
2. **Consistencia Eventual**: Para datos no críticos (métricas, cache)
3. **Consistencia Fuerte**: Para datos financieros (balances, transacciones)
4. **Escalabilidad Independiente**: Cada componente escala según su demanda
5. **Auditoría Completa**: Trazabilidad de todos los cambios financieros

---

## Manejo de Montos Grandes

### Límite Operacional del Sistema

**Alcance del Sistema:**
- **Monto máximo por transacción**: 10K USDT
- **Justificación**: Evitar problemas de liquidez y ejecución en el mercado
- **Comportamiento**: Rechazo automático para montos > 10K USDT con mensaje informativo

### Extensión Futura para Montos Mayores

**Para montos > 10K USDT**, el sistema podría implementar automáticamente:
- División automática de órdenes en chunks de 10K USDT
- Estrategias TWAP (Time-Weighted Average Price) para distribuir el volumen
- Manejo automático de ejecuciones parciales

**Nota**: Estas funcionalidades están fuera del scope del problema original pero podrían implementarse automáticamente sin intervención manual.

---

## Manejo de Casos Límite

### Doble Solicitud del Proveedor

**Escenario:** Red inestable causa retry del proveedor.

**Solución:**
**Manejo de Doble Solicitud:**
- **Verificación de idempotencia**: Consultar cache Redis primero
- **Constraint único**: transaction_id como clave primaria en DynamoDB
- **Respuesta consistente**: Devolver misma respuesta para solicitudes duplicadas
- **Sin procesamiento duplicado**: Evitar doble cobro al usuario

### Race Condition en Balance

**Escenario:** Usuario intenta dos compras simultáneas que en suma exceden su balance.

**Solución:** DynamoDB conditional writes atómicos.

**Prevención de Race Conditions:**
- **Conditional writes**: Verificar versión y balance disponible en operación atómica
- **Actualización de versión**: Incrementar versión en cada modificación
- **Falla atómica**: Si condiciones no se cumplen, operación falla completamente
- **Respuesta HTTP 402**: Fondos insuficientes cuando race condition detectada

**Ventaja:** Atomicidad garantizada sin optimistic locking manual.

### Trading Proveedor Timeout

**Escenario:** Trade tarda más de 5 segundos (nuestro timeout).

**Problema:** El usuario ya se fue con el producto, no podemos revertir la transacción.

**Solución:**
- Trabajador marca transacción como `TRADING`
- Timeout ocurre, trabajador reintenta con backoff exponencial
- Después de 3 intentos, marca como `FAILED`
- **Procede con trade a pérdida** - usuario ya tiene el producto
- **Acepta el costo adicional** como parte del riesgo operacional
- Notificación push al usuario: "Transacción completada con delay"

### Trading Proveedor Retorna 200 pero No Ejecuta Trade

**Escenario:** Bug en proveedor, responde éxito pero no ejecuta.

**Problema:** El usuario ya se fue con el producto, no podemos revertir la transacción.

**Solución:**
- Reconciliation trabajador verifica estado del trade cada 5 minutos
- Consulta API del proveedor: `GET /trade/{trade_id}`
- Si no existe o está fallido, **procede con trade a pérdida**
- **Acepta el costo adicional** como parte del riesgo operacional
- Alerta al equipo de operaciones para investigación

### Cache de Tasa Desactualizado Causa Pérdida

**Escenario:** Tasa cacheada 1050 ARS/USDT, real es 1100. Usuario compra con BTC.

**Problema:** El usuario ya se fue con el producto, no podemos cancelar la transacción.

**Estrategias para Equilibrar Riesgos:**

**Opción 1: Empresa asume costo (actual)**
- Trading trabajador usa tasa en tiempo real del exchange
- Valida que diferencia con tasa cacheada sea < 5%
- Si > 5%, **procede con trade a pérdida** (usuario ya tiene el producto)
- **Acepta el costo adicional** como parte del riesgo operacional

**Opción 2: Usuario asume costo (alternativa)**
- Trading trabajador usa tasa en tiempo real del exchange
- Valida que diferencia con tasa cacheada sea < 5%
- Si > 5%, **cobra diferencia al usuario** (ajuste de balance)
- **Usuario paga tasa real** - transparencia total

**Opción 3: Compartir riesgo (híbrida)**
- Trading trabajador usa tasa en tiempo real del exchange
- Valida que diferencia con tasa cacheada sea < 5%
- Si > 5%, **compartir costo** (50% empresa, 50% usuario)
- **Balance entre riesgo y transparencia**

**Recomendación:** Opción 1 para MVP, evaluar Opción 3 para producción.

### Casos donde el Usuario NO va a Pérdida

**Escenarios sin riesgo para el usuario:**

1. **Usuario con moneda de pago = USDT**
   - **Resultado**: **Sin trading** - transacción se completa inmediatamente
   - **Sin riesgo**: No hay conversión de moneda, no hay trading externo
   - **Porcentaje**: ~70% de usuarios

2. **Trading exitoso**
   - **Resultado**: **Usuario paga tasa correcta** - trade se ejecuta según lo planeado
   - **Sin pérdida**: El trade se completa exitosamente
   - **Porcentaje**: ~95% de trades exitosos

3. **Cache preciso (spread < 5%)**
   - **Resultado**: **Usuario paga tasa correcta** - no hay desviación significativa
   - **Sin pérdida**: El cache funciona correctamente
   - **Porcentaje**: ~90% de casos

**Resumen de riesgo:**
- **Usuarios sin riesgo**: ~70% (USDT) + ~30% × 95% (trades exitosos) = **~98.5%**
- **Usuarios con riesgo**: ~1.5% (trades fallidos o cache desactualizado)

### Partición de Red: DB y Trabajador Desconectados

**Escenario:** Trabajador no puede acceder a DB para actualizar estado.

**Solución:**
- Mensaje cola retiene mensajes (NATS persistence)
- Trabajador reintenta con exponential backoff
- Dead letter cola después de 10 intentos
- Alerta al equipo, reconciliación automática

### Usuario Deposita Durante Transacción Pendiente

**Escenario:** Balance crece mientras hay fondos reservados.

**Solución:**
- Depósitos incrementan `balance_usdt` en DynamoDB
- `reserved_usdt` se mantiene independiente
- Available = balance - reserved (siempre correcto)
- DynamoDB conditional writes garantizan atomicidad

---

## Observabilidad y Monitoreo

### Métricas Clave (Prometheus)

**Métricas Clave (Prometheus):**
- **Latencias**: Autorización, trading, consultas de base de datos
- **Throughput**: Solicitudes aprobadas/denegadas, operaciones de trading
- **Métricas de negocio**: Tasa de denegación, reversiones, montos promedio
- **Errores**: Por tipo (DB, timeout, validación) y proveedor
- **Cache**: Hit rate y misses por tipo de cache
- **Cola**: Profundidad y duración de procesamiento por worker

### Logs Estructurados (JSON)

**Logs Estructurados (JSON):**
- **Timestamp**: ISO 8601 con precisión de milisegundos
- **Contexto**: Servicio, transaction_id, account_id
- **Datos de transacción**: Montos en ARS y USDT
- **Performance**: Duración en milisegundos
- **Trazabilidad**: Trace ID para correlación entre servicios

### Traces (OpenTelemetry)

**Trazas Distribuidas (OpenTelemetry):**
- **Span principal**: Endpoint de autorización con duración total
- **Sub-spans**: Cada operación (validación, cache, DB, cola)
- **Cache hits/misses**: Indicadores de performance de cache
- **Latencias detalladas**: Por componente para identificar cuellos de botella
- **Correlación**: Trace ID único para seguir request completo

### Alertas

**Alertas Configuradas:**
- **Latencia alta**: p95 > 1.8s por 5 minutos (warning)
- **Tasa de reversión**: > 0.1% por 10 minutos (critical)
- **Errores de trading**: > 10% por 3 minutos (critical)
- **Cola saturada**: > 1000 mensajes por 5 minutos (warning)
- **Severidades**: Warning para degradación, Critical para fallos

### Dashboards (Grafana)

**Dashboard: Authorization Overview**
- RPS de autorizaciones (aprobadas vs denegadas)
- Latencia p50, p95, p99
- Tasa de error por tipo
- Distribución de montos

**Dashboard: Trading Operations**
- Latencia del proveedor de trading
- Tasa de éxito/fallo/timeout
- Volumen de trades por currency pair
- Reversiones por hora

**Dashboard: Balance & Reserves**
- Total reserved balance por moneda
- Ratio reserved/available
- Top cuentas con más fondos reservados
- Trend de depósitos vs withdrawals

---

## Asunciones

### Escala y Volumen

| Métrica | Valor Asumido | Notas |
|---------|---------------|-------|
| **RPS peak** | 1,000 req/s | ~86M transacciones/día |
| **RPS promedio** | 200 req/s | ~17M transacciones/día |
| **Usuarios activos** | 500K | Con tarjeta activa |
| **Latencia DB** | p95 < 5ms | DynamoDB conditional writes |
| **Latencia Redis** | p95 < 5ms | Mismo datacenter |
| **Tasa aprobación** | 95% | 5% denegado por fondos |
| **% usuarios USDT** | 70% | No requieren trading |
| **% usuarios BTC/ETH** | 30% | Requieren trading async |
| **Tasa reversión** | < 0.1% | Trade failures |

### SLOs y SLAs

| Métrica | SLO | SLA |
|---------|-----|-----|
| Disponibilidad | 99.95% | 99.9% |
| Latencia p95 | < 800ms | < 2000ms |
| Latencia p99 | < 1500ms | < 2000ms |
| Tasa de error | < 0.1% | < 1% |

### Regulatorio y Compliance

- **PCI-DSS:** No almacenamos datos completos de tarjeta (solo hash y last 4)
- **Auditoría:** Todas las transacciones logged por 7 años
- **KYC/AML:** Asumido que se hace en otro servicio
- **Limites regulatorios:** Implementados a nivel de tarjeta/cuenta

### Consideraciones de Seguridad

- **Autenticación:** JWT tokens en Authorization header
- **Rate limiting:** Por usuario, IP, y global
- **Encriptación:** TLS 1.3 en tránsito, AES-256 en reposo
- **Secrets:** Vault para API keys del trading proveedor
- **Network:** VPC privada, solo API Gateway expuesto

---

## Preguntas Frecuentes (FAQ)

### **¿Cuál es el objetivo de la reconciliación?**

La reconciliación tiene 4 objetivos principales:
1. **Verificar completitud**: Asegurar que todas las transacciones lleguen a estado final (COMPLETED/FAILED)
2. **Detectar trades fallidos**: Verificar que el proveedor externo realmente ejecutó el trade
3. **Proceder con trades a pérdida**: Cuando el usuario ya tiene el producto, aceptar el costo adicional como riesgo operacional
4. **Auditoría y compliance**: Mantener trazabilidad completa para regulaciones financieras

**Importante**: No se revierten transacciones cuando el usuario ya se fue con el producto, para evitar inconsistencias de negocio.

### **¿Pueden ocurrir race conditions en la reconciliación?**

Sí, pueden ocurrir varios tipos de race conditions:
- **Trading Worker vs Reconciliation Worker**: Ambos intentando actualizar la misma transacción
- **Múltiples Reconciliation Workers**: Procesando la misma transacción simultáneamente

**Soluciones implementadas**:
- **Conditional writes en DynamoDB**: Solo actualizar si el estado actual es el esperado
- **Distributed locking con Redis**: Un solo worker puede procesar cada transacción
- **Idempotencia**: Verificar si ya fue procesado antes de actuar

### **¿Qué tipo de seguridad tiene el sistema?**

El sistema implementa múltiples capas de seguridad:

**Validación de entrada**: Formato, rangos de montos, estructura de datos
**Rate limiting**: Por cuenta (10/min), tarjeta (5/min) y global
**Detección de fraude**: ML + reglas de negocio para identificar transacciones sospechosas
**Validación de integridad**: Verificar que tarjeta pertenece a cuenta, cuenta activa, límites
**Auditoría completa**: Logs de todos los eventos de seguridad con alertas automáticas
**Encriptación**: Datos sensibles encriptados, secrets en Vault, TLS 1.3

### **¿Por qué DynamoDB en lugar de PostgreSQL para bloqueo de fondos?**

DynamoDB se eligió por:
- **Latencia 10x menor**: 2-5ms vs 20-50ms de PostgreSQL
- **Atomicidad nativa**: Conditional writes sin optimistic locking manual
- **Durabilidad garantizada**: Multi-AZ automático
- **Sin ops overhead**: Servicio completamente gestionado
- **Escalabilidad automática**: Sin configuración de particionamiento

**Trade-off**: Mayor costo (~$1,075/mes vs $200/mes) pero justificado por la mejora de rendimiento.

### **¿Cómo se maneja el timeout de 2 segundos con trading de 3 segundos?**

Se implementa un **patrón asíncrono híbrido**:
1. **Fase síncrona (< 500ms)**: Validar fondos, bloquear saldo, responder a LEMON
2. **Fase asíncrona (background)**: Ejecutar trade con proveedor externo
3. **Reconciliación**: Confirmar o revertir transacciones fallidas

**Resultado**: Usuario ve "aprobado" inmediatamente en POS, trade se ejecuta en segundo plano.

### **¿Qué pasa si el trading falla después de la autorización?**

El sistema tiene un proceso robusto de manejo de fallos:
1. **Reintentos automáticos**: 3 intentos con backoff exponencial (1s, 2s, 4s)
2. **Si falla definitivamente**: **Proceder con trade a pérdida** - el usuario ya tiene el producto
3. **Aceptar costo adicional**: Como parte del riesgo operacional del negocio
4. **Notificación al usuario**: "Transacción completada con delay"
5. **Alerta al equipo**: Para investigación y optimización del sistema

**Importante**: No se revierten transacciones cuando el usuario ya se fue con el producto, para mantener consistencia de negocio.

### **¿Cómo se manejan montos excesivamente grandes?**

Para montos grandes (ej: $10M ARS), el sistema implementa varias estrategias:

**Límite técnico del sistema**:
- **Monto máximo**: 10K USDT por transacción
- **Comportamiento**: Rechazo automático para montos mayores
- **Extensión futura**: División automática de órdenes (fuera del scope actual)

**Beneficios**:
- **Control de riesgo**: Evita impacto excesivo en el mercado
- **Mejor ejecución**: Órdenes más pequeñas se ejecutan mejor
- **Transparencia**: Usuario informado sobre el proceso

### **¿Por qué diferentes TTLs de cache para diferentes monedas?**

El cache de tasas de conversión se adapta a la volatilidad de cada moneda:

**USDT/ARS (Estable)**:
- **TTL**: 30 segundos
- **Justificación**: Baja volatilidad (0.1-0.5% por hora)
- **Riesgo**: Mínimo de arbitraje

**BTC/USDT (Volátil)**:
- **TTL**: 5 segundos
- **Justificación**: Alta volatilidad (2-10% por hora)
- **Riesgo**: Cache de 30s podría causar pérdidas significativas

**ETH/USDT (Muy Volátil)**:
- **TTL**: 5 segundos
- **Justificación**: Muy alta volatilidad (3-15% por hora)
- **Riesgo**: Cache largo podría causar arbitraje

**Validación de Drift**:
- **Verificación**: Antes de usar cache, validar desviación vs tasa actual
- **Umbrales**: 0.5% para USDT, 2% para BTC, 2.5% para ETH
- **Fallback**: Si drift > umbral, usar API en tiempo real

**Resultado**: Balance óptimo entre performance y precisión según volatilidad.

### **¿Por qué TimescaleDB en lugar de solo Datadog para métricas?**

La estrategia híbrida TimescaleDB + Datadog optimiza costo y funcionalidad:

**TimescaleDB para métricas de negocio:**
- **Costo**: $200/mes vs $2,000/mes con Datadog
- **Queries complejas**: Análisis ad-hoc y JOINs con datos transaccionales
- **Retención**: 2+ años sin costo adicional
- **Control**: Queries personalizadas para auditoría

**Datadog para observabilidad:**
- **Setup**: 5 minutos vs 2-4 horas de configuración
- **Completo**: Métricas + logs + APM + alertas
- **Mantenimiento**: Cero ops overhead
- **Alertas**: Machine learning y detección de anomalías

**Resultado**: $500/mes total vs $2,000/mes solo con Datadog, con análisis más profundo.

---

## Conclusión

Esta arquitectura balancea los requerimientos de:
1.   **Baja latencia:** < 2s respuesta al proveedor
2.   **Alta disponibilidad:** 99.95% uptime
3.   **Integridad financiera:** Sin double-spending
4.   **Escalabilidad:** 1000+ RPS
5.   **Observabilidad:** Métricas, logs, traces completos

El trade-off principal es la complejidad de manejo asíncrono vs la imposibilidad de cumplir el timeout con trading síncrono. La solución mitiga riesgos con bloqueo de fondos, reconciliación robusta y notificaciones al usuario.

