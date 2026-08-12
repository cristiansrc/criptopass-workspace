# ADR-005: Patrón Strategy para Pasarelas de Pago + Saga de Compra con Step Functions

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-005 |
| **Título** | Patrón Strategy para pasarelas de pago (PaymentProvider) + Saga de compra orquestada con Step Functions y compensaciones |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

El flujo de compra en CriptoPass involucra múltiples pasos con estado y posibles fallos parciales que requieren compensación:

- **Edge Case EC-01:** Pago aprobado en Mercado Pago pero emisión de boleta falla → reembolso automático.
- **Edge Case EC-03:** TTL de reserva expira mientras el usuario está en la pasarela → si el pago luego se aprueba, reembolso automático.
- **R-02:** Saga de compra distribuida — la compensación ante fallos parciales es crítica.
- El usuario estableció (S-01): patrón Strategy para pagos (Mercado Pago primer adaptador) y Step Functions para la saga.

## Decisión

### Parte A: Patrón Strategy para Proveedores de Pago

**Puerto `PaymentProvider` (interfaz en capa de dominio de ms-orders):**

```kotlin
interface PaymentProvider {
    fun createCheckout(order: Order): CheckoutResult
    fun processWebhook(payload: WebhookPayload): PaymentStatus
    fun refund(payment: Payment, amount: Money): RefundResult
    fun queryStatus(paymentId: String): PaymentStatus
}
```

**MercadoPagoAdapter (infrastructure layer):** Implementación concreta usando Mercado Pago SDK/API. Traduce modelos internos ↔ externos (ACL).

**V1:** Solo se implementa `MercadoPagoAdapter`. El puerto queda definido para que en V2+ se puedan agregar `StripeAdapter`, `PayUAdapter`, etc., sin cambiar la lógica de dominio de Orders.

### Parte B: Saga de Compra Orquestada con Step Functions

```
State Machine: PurchaseSaga
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│  ┌──────────┐     ┌───────────────┐     ┌──────────────────┐   │
│  │  Order   │────▶│ Wait for      │────▶│  Issue Tickets   │   │
│  │ Created  │     │ Payment       │     │  (ms-tickets)    │   │
│  │ (sync)   │     │ (webhook)     │     │                  │   │
│  └──────────┘     └───────┬───────┘     └────────┬─────────┘   │
│                           │                       │             │
│                           │ Timeout (15 min)      │ Success     │
│                           ▼                       ▼             │
│                    ┌──────────────┐     ┌──────────────────┐   │
│                    │  Release     │     │  Notify + Done   │   │
│                    │  Reservation │     │  (EventBridge)   │   │
│                    │  + Notify    │     └──────────────────┘   │
│                    └──────────────┘                             │
│                                                                  │
│  Compensation Path:                                              │
│  ┌──────────────────┐                                           │
│  │  Issue Tickets   │─── Failure ──▶ ┌──────────────────┐      │
│  │  (falló)         │                │  Refund Payment   │      │
│  └──────────────────┘                │  + Notify Buyer   │      │
│                                      │  + Alert SUPPORT  │      │
│                                      └──────────────────┘      │
└──────────────────────────────────────────────────────────────────┘
```

### Pasos de la Saga

| Paso | Responsable | Timeout | Retry | Compensación |
|---|---|---|---|---|
| 1. **Order Created** | ms-orders (sync desde frontend) | N/A | Frontend con idempotency key | N/A (no hay pago aún) |
| 2. **Wait for Payment** | Step Functions espera webhook de Mercado Pago o timeout | 15 min (TTL de reserva) | N/A (espera pasiva) | Timeout → liberar reserva + notificar |
| 3. **Issue Tickets** | Step Functions → ms-tickets API | 30s | 3 reintentos (1s, 2s, 5s) | **Compensar:** paso 4 |
| 4. **Refund Payment (compensación)** | Step Functions → ms-orders API | 30s | 3 reintentos | Alerta manual a SUPPORT |
| 5. **Notify Buyer** | EventBridge → SQS → Lambda SES | Async | SQS retry + DLQ | N/A |
| 6. **Release Reservation** | ms-orders (timeout interno o al cancelar) | Inmediato | N/A | N/A |

### Lógica de Compensación

**Caso EC-01 (Pago OK, emisión falla):**
1. Step Functions recibe `PaymentConfirmed`.
2. Invoca `POST /api/v1/tickets/issuance` (idempotente por `order_id`).
3. Si falla tras 3 reintentos → ejecuta paso de compensación: `POST /api/v1/orders/{orderId}/refund` en ms-orders.
4. ms-orders llama a `paymentProvider.refund(payment, amount)` → Mercado Pago API de reembolso.
5. Orden queda en estado `failed`. Evento `OrderFailed` → notificación email + alerta SUPPORT.

**Caso EC-03 (TTL expirado, pago tardío):**
1. ms-orders recibe webhook de pago aprobado.
2. Verifica estado de la orden: si la reserva ya expiró → orden en estado `expired`.
3. ms-orders llama a `paymentProvider.refund()` directamente (sin pasar por Step Functions, porque no hay emisión que fallar).
4. Notifica al comprador que su pago será reembolsado porque la reserva expiró.

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **Choreography (sin orquestador central)** | Cada servicio reacciona a eventos. Más desacoplado pero difícil razonar sobre compensaciones, timeouts y estado global de la compra. Debugging más complejo. |
| **Saga dentro de ms-orders (sin Step Functions)** | ms-orders se vuelve el orquestador. Acopla lógica de emisión y pago en un solo servicio. Viola bounded context de Tickets. |
| **Two-Phase Commit (transacciones distribuidas)** | No viable en microservicios. Bloquea recursos. Step Functions con compensaciones es el patrón estándar para este escenario. |
| **Reembolso manual (sin compensación automática)** | Riesgo de negocio: comprador paga y no recibe boleta. SUPPORT tendría que intervenir manualmente. Mala UX. |

## Consecuencias

**Positivas:**
- Saga con visibilidad centralizada: Step Functions muestra el estado exacto de cada compra (útil para SUPPORT).
- Compensación automática para EC-01 y EC-03 sin intervención manual (excepto si el reembolso también falla).
- Puerto PaymentProvider permite agregar pasarelas en V2 sin cambiar la saga ni ms-orders.
- Step Functions maneja reintentos, backoff y timeouts nativamente.

**Negativas:**
- Step Functions tiene costo por transición de estado (miles de compras/día = costo bajo pero no cero).
- La máquina de estados se define en JSON/ASL (Amazon States Language), que puede ser verboso. Se recomienda usar CDK o Terraform para generarlo.
- Debugging de una saga fallida requiere correlacionar logs de Step Functions + ms-tickets + ms-orders.

**Riesgos mitigados:**
- **R-02 (saga distribuida):** Compensación automática para fallos de emisión. Timeout de 15 min evita sagas eternas.
- **R-03 (idempotencia webhooks):** ms-orders deduplica por `payment_id`. Step Functions es idempotente por `order_id` en el paso de emisión.

---

## Notas

- La reserva de inventario se crea en el paso 1 (sync) y se libera automáticamente al expirar el TTL en Redis (sin intervención de Step Functions).
- El fee por servicio (supuesto S-05) se calcula en ms-orders al crear la orden y se incluye en el `amount_total` enviado a Mercado Pago.
- Si en V2 se agrega DIAN (facturación electrónica), se inserta un paso adicional en la saga post-emisión.
- La máquina de estados de Step Functions debe tener logging detallado en cada transición para auditoría.
