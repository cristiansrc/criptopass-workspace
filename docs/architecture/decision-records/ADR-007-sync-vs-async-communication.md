# ADR-007: Comunicación Sync vs Async — REST vía ALB + EventBridge/SQS

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-007 |
| **Título** | Estrategia de comunicación: sync (OpenAPI/REST vía ALB) para user-facing, async (EventBridge/SQS) para cross-service events. Regla de idempotencia/retries/timeout/compensation obligatoria |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass tiene múltiples patrones de comunicación entre componentes:
- Frontends → Backend: requests iniciados por usuario, respuesta inmediata esperada.
- Microservicios → Microservicios: consultas de datos (evento, orden) y operaciones (emisión de boletas).
- Microservicios → Lambdas: notificaciones, trazabilidad.
- Externo → CriptoPass: webhooks de Mercado Pago.
- CriptoPass → Externo: checkout Mercado Pago, anclaje Polygon, emails SES.

Cada patrón requiere garantías diferentes de latencia, consistencia, resiliencia y acoplamiento.

## Decisión

### Regla de Decisión: ¿Sync o Async?

```
¿El usuario (persona) está esperando una respuesta inmediata?
  → SÍ: Sync REST API
  → NO: Continuar

¿La operación es parte de una transacción ACID local?
  → SÍ: Sync (dentro del mismo bounded context / microservicio)
  → NO: Continuar

¿La operación puede ser eventual sin afectar la experiencia del usuario?
  → SÍ: Async (eventos, colas, lambda)
  → NO: Continuar

¿La operación necesita orquestación con compensación?
  → SÍ: Sync dentro de Step Functions (orquestador llama APIs sync)
  → NO: Async event-driven
```

### Matriz de Comunicación por Escenario

| Escenario | Modo | Mecanismo | Justificación |
|---|---|---|---|
| **Frontend → ms-catalog (catálogo público)** | Sync | REST vía ALB | Usuario espera ver eventos en < 200ms |
| **Frontend → ms-orders (crear orden)** | Sync | REST vía ALB | Usuario espera confirmación de reserva antes de ir a pagar |
| **Frontend → ms-tickets (mis boletas)** | Sync | REST vía ALB | Usuario espera ver sus boletas |
| **Frontend → ms-tickets (QR dinámico)** | Sync | REST vía ALB | QR debe generarse en < 400ms para rotación |
| **Admin → ms-tickets (validación QR)** | **Sync crítico** | REST vía ALB | Validador en puerta espera respuesta inmediata (p95 < 500ms) |
| **ms-orders → Mercado Pago (checkout)** | Sync | REST externo | Redirección del usuario; necesita respuesta inmediata |
| **Mercado Pago → ms-orders (webhook)** | Async inbound | Webhook HTTP | Mercado Pago notifica cambio de estado; CriptoPass no controla el timing |
| **ms-orders → EventBridge (PaymentConfirmed)** | Async | EventBridge | La emisión de boletas no necesita ser instantánea; la saga (Step Functions) se dispara async |
| **Step Functions → ms-tickets (emisión)** | Sync en saga | HTTP Task | La saga necesita confirmación de emisión para decidir si compensar |
| **ms-tickets → EventBridge (TicketIssued)** | Async | EventBridge | El anclaje blockchain puede esperar hasta 5 min |
| **EventBridge → fn-notifications (email)** | Async via SQS | SQS → Lambda | El email no es crítico para el flujo de compra; eventual acceptable |
| **EventBridge Scheduler → fn-blockchain-anchor** | Async programado | Scheduled Lambda | El anclaje es periódico, no en tiempo real |
| **fn-blockchain-anchor → Polygon** | Sync | JSON-RPC | La lambda necesita confirmación de tx para guardar el hash |

### Reglas Transversales Obligatorias

**Regla de Oro:** Ningún flujo cross-service carece de idempotencia, reintentos, timeout y compensación definidos.

| Garantía | Implementación |
|---|---|
| **Idempotencia** | Cada operación mutante cross-service debe ser idempotente. Usar `Idempotency-Key` (REST) o `event_id` (Async). Los servicios mantienen registro de operaciones ya procesadas (ej. `processed_webhooks` en ms-orders). |
| **Retries** | Sync: 2-3 reintentos con backoff exponencial. Async: SQS redrive policy (3 intentos → DLQ). Step Functions: reintentos por paso. |
| **Timeout** | Sync: definido por endpoint en OpenAPI spec (1s-30s según criticidad). Async: Visibility Timeout en SQS > tiempo máximo de procesamiento de la Lambda. |
| **Compensación** | Flujos con efectos secundarios (pago, emisión) deben tener paso de compensación explícito. Step Functions implementa la saga con compensating actions. |
| **Circuit Breaker** | Sync cross-service: circuit breaker en el caller (Resilience4j). 5 fallos en 30s → circuito abierto 60s. |
| **Dead Letter Queue** | Toda cola SQS tiene DLQ asociada. CloudWatch alarm si DLQ no vacía > 5 min. |

### Trazabilidad Distribuida

| Modo | Propagación |
|---|---|
| **Sync (REST)** | `X-Trace-Id` header generado por ALB (si no existe) y propagado por todos los servicios. Logs incluyen trace_id. X-Ray tracing automático con AWS Distro for OpenTelemetry (ADOT) o X-Ray SDK |
| **Async (EventBridge, SQS)** | `trace_id` como atributo de mensaje en EventBridge. SQS message attribute. Lambda lo extrae y lo usa en sus logs |

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **Todo async (incluso user-facing)** | Inviable: el usuario no puede esperar segundos por una respuesta de catálogo. UX inaceptable. |
| **Todo sync (incluso notificaciones)** | Acoplamiento innecesario: si SES falla, la compra no debería fallar. Bloquea al usuario. |
| **gRPC en lugar de REST** | Mejor performance binaria, pero requiere load balancing L7 (gRPC sobre HTTP/2). ALB soporta gRPC pero agrega complejidad de tooling (protobuf, generación de código). REST + JSON es más simple para V1 y suficiente para la escala esperada. |
| **GraphQL como API unificada** | Un solo endpoint flexible, pero complejidad de implementación, caching difícil, y riesgo de consultas costosas. No justificado para 4 microservicios con esquemas simples. |
| **Kafka en lugar de EventBridge + SQS** | Mayor throughput y retención, pero requiere gestión de clúster (MSK) o Confluent Cloud (costo). Para la escala de CriptoPass V1, EventBridge + SQS es suficiente y serverless. |

## Consecuencias

**Positivas:**
- Separación clara: user-facing = sync, background = async.
- EventBridge + SQS es serverless: sin gestión de brokers.
- Step Functions da visibilidad de cada paso de la saga.
- Regla de oro previene flujos sin resiliencia (anti-patrón común en microservicios).

**Negativas:**
- EventBridge tiene límites (throughput, tamaño de evento 256KB). Para CriptoPass V1, los eventos son pequeños y el throughput bajo.
- Depuración de flujos async requiere correlacionar logs de múltiples servicios (X-Ray mitiga esto).
- La idempotencia requiere implementación cuidadosa en cada servicio (no es automática).

**Riesgos mitigados:**
- **R-02 (saga distribuida):** Step Functions con timeouts y compensación.
- **R-03 (idempotencia webhooks):** ms-orders con tabla `processed_webhooks`.
- **R-04 (desync reloj QR):** Tolerancia de ±5s en validación. La API es sync, minimizando latencia de red.

---

## Notas

- **Service-to-service auth interna (VPC):** V1 simplifica usando API keys estáticas rotadas periódicamente (vía Secrets Manager) o confianza por VPC (security groups). En V2 se puede migrar a OAuth2 Client Credentials.
- **Ordenamiento de eventos:** No se requiere ordenamiento estricto en ningún flujo V1. Si en V2 se necesita (ej. secuencia de estados de orden), se usará SQS FIFO o Kafka con partitioning key.
- **EventBridge schema registry:** Se recomienda usar EventBridge Schema Registry para versionar y validar esquemas de eventos automáticamente.
