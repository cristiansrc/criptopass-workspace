# ADR-001: Arquitectura Híbrida — Microservicios ECS + Lambdas Go + Step Functions

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-001 |
| **Título** | Arquitectura híbrida: microservicios (ECS Fargate + ALB) + Lambdas Go event-driven + Step Functions para saga de compra |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass es un sistema greenfield de venta de boletas con los siguientes requerimientos:
- **APIs sync de latencia crítica:** validación de QR en puerta (p95 < 500ms), checkout de compra, catálogo público con alta concurrencia
- **Flujos async:** emisión de boletas post-pago, notificaciones email, anclaje blockchain programado
- **Saga de compra:** flujo multi-paso (pago → emisión → notificación) que requiere compensación ante fallos parciales (EC-01, EC-03)
- **Escalabilidad:** ventas pico con miles de usuarios concurrentes sobre un mismo evento (supuesto S-16)

El usuario estableció (S-01): microservicios Kotlin + Spring Boot, Lambdas Go para flujos async, Step Functions para la saga.

## Decisión

**Arquitectura híbrida con tres capas de cómputo diferenciadas por responsabilidad:**

### Capa 1: Microservicios en ECS Fargate + ALB (Kotlin + Spring Boot)
Para lógica de negocio que **posee aggregates**, maneja **transacciones ACID** y expone **APIs sync**.

Microservicios V1:
| Servicio | Justificación para ir en ECS |
|---|---|
| **ms-catalog** | API pública de lectura masiva con cache. Conexión persistente a PostgreSQL (connection pooling crítico). Múltiples endpoints REST. |
| **ms-orders** | Lógica transaccional de órdenes, pagos y reservas. Conexión a PostgreSQL + Redis. Webhook handler de Mercado Pago (endpoint público). Necesita transacciones ACID. |
| **ms-tickets** | API de validación QR sync de latencia crítica (p95 < 500ms). Conexión persistente a PostgreSQL evita cold starts de Lambda. Endpoint de emisión de boletas con idempotencia. |
| **ms-users** | CRUD de perfil, consentimiento, derechos ARSO. API de administración de usuarios. Conexión a PostgreSQL + Cognito Admin API. |

### Capa 2: Lambdas Go (Event-Driven + Scheduled)
Para procesamiento **asíncrono, stateless, event-driven o programado** que **no posee aggregates propios** (o posee datos simples en DynamoDB).

Lambdas V1:
| Lambda | Justificación para ir en Lambda |
|---|---|
| **fn-notifications** | Stateless. Consume eventos de SQS, envía email vía SES. Sin base de datos. Escala a cero cuando no hay eventos. |
| **fn-blockchain-anchor** | Procesamiento programado (cada 5 min) + event-driven (SQS). Construye Merkle tree (CPU-bound, sin estado entre invocaciones). DynamoDB como almacenamiento key-value simple. Sin necesidad de connection pooling. |

### Capa 3: Step Functions para Orquestación de Saga
Para el flujo de compra que involucra **múltiples pasos con compensaciones**.

| Flujo | Justificación |
|---|---|
| **Saga de compra** | Paso 1: esperar confirmación de pago (webhook). Paso 2: invocar ms-tickets para emitir boletas. Paso 3 (compensación): si paso 2 falla, invocar ms-orders para reembolso. Step Functions maneja timeouts, reintentos y estado de la saga de forma nativa, sin que ningún microservicio tenga que implementar lógica de orquestación. |

### Regla de decisión: ¿Microservicio o Lambda?

```
¿El componente posee un aggregate de negocio con su propia base de datos?
  → SÍ: Microservicio en ECS
  → NO: Continuar

¿Necesita conexiones persistentes a base de datos (connection pooling)?
  → SÍ: Microservicio en ECS
  → NO: Continuar

¿Expone múltiples endpoints REST sync con latencia predecible?
  → SÍ: Microservicio en ECS
  → NO: Continuar

¿Es stateless, event-driven o scheduled?
  → SÍ: Lambda Go
  → NO: Re-evaluar con arquitecto
```

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **Todo en microservicios ECS** | Sobrecarga operativa para funciones simples (notificaciones, anclaje). Costos de idle. Mayor complejidad de despliegue para cambios pequeños. |
| **Todo en Lambdas** | Cold start afecta latencia de validación QR. Connection pooling a PostgreSQL es problemático en Lambda (RDS Proxy necesario, costo adicional). Lógica transaccional compleja en funciones sin estado. |
| **Kubernetes (EKS) en lugar de ECS Fargate** | Mayor complejidad operativa innecesaria para 4 microservicios. ECS Fargate es suficiente y reduce superficie de gestión. |
| **Choreography (eventos) en lugar de Step Functions** | Mayor desacoplamiento pero más difícil de razonar sobre compensaciones y timeouts. Step Functions da visibilidad centralizada del estado de cada compra. |

## Consecuencias

**Positivas:**
- Cada tipo de carga de trabajo usa el cómputo óptimo: ECS para APIs transaccionales, Lambda para async/eventos, Step Functions para orquestación.
- Cold starts no afectan las APIs críticas (validación QR, checkout).
- Lambdas escalan a cero reduciendo costos en horas de baja actividad.
- Step Functions da visibilidad y trazabilidad del estado de cada compra (útil para SUPPORT).
- Separación clara de responsabilidades: cada servicio/lambda tiene un bounded context y una razón de existir.

**Negativas:**
- Tres modelos de despliegue distintos (ECS, Lambda, Step Functions) aumentan la superficie de conocimiento del equipo.
- Depuración de flujos cross-model (sync → async → saga) requiere trazabilidad distribuida robusta (X-Ray).
- Step Functions tiene un límite de 1 año de ejecución (no aplica; las sagas de compra duran minutos).

**Riesgos mitigados:**
- **R-02 (Saga distribuida):** Step Functions con reintentos y compensaciones. Timeouts explícitos por paso.
- **R-03 (Idempotencia webhooks):** ms-orders maneja idempotencia local; Step Functions reintenta pasos idempotentes.

---

## Notas

- Los servicios en ECS Fargate usan **Spring Boot 3.x con Virtual Threads (Project Loom)** para manejar alta concurrencia con bajo consumo de threads del SO.
- El ALB hace health checks a cada microservicio vía `/actuator/health` y enruta tráfico solo a instancias healthy.
- Las lambdas Go usan `provided.al2` runtime con compilación nativa para mínimo cold start.
