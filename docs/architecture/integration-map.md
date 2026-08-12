# Integration Map — CriptoPass

| Campo | Valor |
|---|---|
| **Status** | `active` |
| **Proyecto** | CriptoPass (greenfield) |
| **Versión** | V1 |
| **Creado por** | enterprise-architect |
| **Fecha** | 2026-08-11 |

---

## 1. Catálogo de Integraciones

### 1.1 Frontends → Backend (Vercel → ALB → Microservicios ECS)

| Integración | I-001: Portal Compradores → ms-catalog |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `GET /api/v1/catalog/events`, `GET /api/v1/catalog/events/{id}`, búsqueda y filtros |
| **Owner del contrato** | ms-catalog (OpenAPI spec) |
| **Auth** | Sin auth para endpoints públicos (catálogo). JWT (Cognito) para endpoints de compra |
| **Timeout** | Cliente (Next.js): 5s. Servidor (ms-catalog): 2s |
| **Retry** | Cliente: 2 reintentos con backoff exponencial (500ms, 1s) en errores 5xx |
| **Idempotencia** | GET es naturalmente idempotente |
| **CORS** | Orígenes: `https://criptopass.com`, `https://*.vercel.app` (dev/staging). Headers: `Authorization`, `X-Trace-Id` |
| **Rate Limiting** | ALB + WAF: 60 req/min por IP en endpoints públicos; 120 req/min para autenticados |
| **SLA** | p95 < 200ms (con cache Redis) |

| Integración | I-002: Portal Compradores → ms-orders |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `POST /api/v1/orders` (crear orden), `GET /api/v1/orders/{id}` (consultar), `GET /api/v1/orders/mine` (Mis órdenes) |
| **Owner del contrato** | ms-orders (OpenAPI spec) |
| **Auth** | JWT (Cognito), rol BUYER |
| **Timeout** | Cliente: 10s. Servidor: 5s |
| **Retry** | Cliente: NO reintentar POST (puede duplicar órdenes). GET: 2 reintentos |
| **Idempotencia** | `POST /api/v1/orders`: idempotency key enviada por el frontend (`Idempotency-Key` header) |
| **Consistencia** | Fuerte (transaccional en PostgreSQL) |
| **SLA** | p95 < 500ms para creación de orden |

| Integración | I-003: Portal Compradores → ms-tickets |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `GET /api/v1/tickets/mine` (Mis boletas), `GET /api/v1/tickets/{id}/qr` (QR dinámico) |
| **Owner del contrato** | ms-tickets (OpenAPI spec) |
| **Auth** | JWT (Cognito), rol BUYER |
| **Timeout** | Cliente: 5s. Servidor: 3s |
| **Retry** | GET: 2 reintentos |
| **Idempotencia** | GET naturalmente idempotente |
| **SLA** | p95 < 200ms para listado de boletas; < 400ms para generación de QR |

| Integración | I-004: Portal Compradores → ms-users |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `GET /api/v1/users/profile`, `PUT /api/v1/users/profile` |
| **Owner del contrato** | ms-users (OpenAPI spec) |
| **Auth** | JWT (Cognito), rol BUYER |
| **Timeout** | Cliente: 5s. Servidor: 2s |
| **Retry** | GET: 2 reintentos. PUT: 1 reintento (con idempotency key) |
| **Idempotencia** | `PUT`: idempotente por sub del JWT (mismo perfil) |

| Integración | I-005: Panel Admin → ms-catalog |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `POST/PUT/DELETE /api/v1/admin/catalog/events` (CRUD eventos), `POST /api/v1/admin/catalog/events/{id}/images` (upload presigned URL) |
| **Owner del contrato** | ms-catalog |
| **Auth** | JWT (Cognito), roles ORGANIZER (solo sus eventos) o SUPER_ADMIN |
| **Timeout** | Cliente: 15s (upload puede ser lento). Servidor: 10s |
| **Retry** | POST/PUT: no reintentar sin idempotency key. DELETE: idempotente |
| **Idempotencia** | `Idempotency-Key` header para operaciones de creación/edición |

| Integración | I-006: Panel Admin → ms-orders |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `GET /api/v1/admin/orders` (consulta órdenes, SUPPORT), `GET /api/v1/admin/orders/events/{eventId}` (ventas por evento, ORGANIZER), `POST /api/v1/admin/orders/{id}/resend-tickets` (reenvío, SUPPORT) |
| **Owner del contrato** | ms-orders |
| **Auth** | JWT (Cognito), roles SUPPORT, ORGANIZER o SUPER_ADMIN |
| **Timeout** | 10s |
| **Retry** | GET: 2 reintentos. POST resend: 1 reintento con idempotency key |
| **Idempotencia** | `resend-tickets`: idempotente por order_id (no duplica emails) |

| Integración | I-007: Panel Admin → ms-tickets (Validación QR) |
|---|---|
| **Tipo** | **Sync REST API — Latencia Crítica** |
| **Método** | `POST /api/v1/tickets/validate` (valida QR y redime boleta) |
| **Owner del contrato** | ms-tickets |
| **Auth** | JWT (Cognito), roles VALIDATOR o SUPER_ADMIN |
| **Timeout** | Cliente (admin web): 3s. Servidor: 2s |
| **Retry** | Cliente: 1 reintento solo en timeout (no en errores de negocio). Servidor: idempotencia por ticket_id |
| **Idempotencia** | **Crítica.** La redención es atómica: segunda llamada con mismo ticket_id ya redimido → rechazo con motivo "boleta ya usada" (no reintentar) |
| **SLA** | **p95 < 500ms** (objetivo estricto). Timeout máximo del servidor: 2s. Si > 2s, el validador muestra error y reintenta una vez |
| **Fallback** | Entrada manual de código de boleta (`POST /api/v1/tickets/validate` con `mode=manual` + `ticket_code` en lugar de `qr_data`). Misma lógica de validación |
| **Tolerancia de reloj** | Ventana de rotación del QR: ±5s de tolerancia para compensar desync de reloj entre dispositivo y servidor |

| Integración | I-008: Panel Admin → ms-users |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `GET /api/v1/admin/users` (listar usuarios), `PUT /api/v1/admin/users/{id}/roles` (asignar roles, SUPER_ADMIN) |
| **Owner del contrato** | ms-users |
| **Auth** | JWT (Cognito), roles SUPER_ADMIN |
| **Timeout** | 5s |
| **Retry** | GET: 2 reintentos. PUT: idempotente (mismo rol → sin cambio) |
| **Idempotencia** | Roles: idempotente por sub + rol |

---

### 1.2 Microservicios → Microservicios (comunicación interna)

| Integración | I-009: ms-orders → ms-catalog |
|---|---|
| **Tipo** | Sync REST API (via ALB interno o VPC directa) |
| **Método** | `GET /api/v1/catalog/events/{eventId}`, `GET /api/v1/catalog/events/{eventId}/availability` |
| **Propósito** | Obtener datos del evento y disponibilidad durante el checkout |
| **Owner del contrato** | ms-catalog |
| **Auth** | API Key interna (x-api-key header) o mTLS VPC |
| **Timeout** | 3s |
| **Retry** | 2 reintentos con backoff (500ms, 1s). Circuit breaker: 5 fallos en 30s → circuito abierto 60s |
| **Consistencia** | Lectura eventual (cacheable 60s). Si el evento se canceló durante el checkout, ms-orders maneja la inconsistencia |
| **Fallback** | Si ms-catalog no responde → checkout rechazado con error "servicio no disponible" |

| Integración | I-010: Step Functions → ms-tickets (Emisión) |
|---|---|
| **Tipo** | Sync HTTP Task (via ALB) |
| **Método** | `POST /api/v1/tickets/issuance` |
| **Propósito** | Ordenar emisión de boletas tras pago confirmado |
| **Owner del contrato** | ms-tickets |
| **Auth** | API Key interna o IAM role de Step Functions |
| **Timeout** | Step Functions task: 30s. ms-tickets: 20s |
| **Retry** | Step Functions: 3 reintentos con backoff (1s, 2s, 5s). Si todos fallan → paso de compensación |
| **Idempotencia** | **Crítica.** `Idempotency-Key: order_id`. Si la saga reintenta la emisión, ms-tickets detecta order_id ya procesado y retorna las mismas boletas (o estado anterior) sin duplicar |
| **Compensación** | Si emisión falla definitivamente → Step Functions ejecuta `POST /api/v1/orders/{orderId}/refund` en ms-orders |

| Integración | I-011: Step Functions → ms-orders (Compensación/Reembolso) |
|---|---|
| **Tipo** | Sync HTTP Task (via ALB) |
| **Método** | `POST /api/v1/orders/{orderId}/refund` |
| **Propósito** | Reembolsar pago cuando la emisión falla (EC-01) o TTL expiró con pago posterior (EC-03) |
| **Owner del contrato** | ms-orders |
| **Auth** | API Key interna o IAM role |
| **Timeout** | 30s |
| **Retry** | Step Functions: 3 reintentos con backoff. Si falla → alerta manual (SUPPORT interviene) |
| **Idempotencia** | `Idempotency-Key: order_id`. ms-orders no duplica reembolsos para la misma orden |
| **Reembolso idempotente** | ms-orders llama a Mercado Pago API de refund con `external_reference = order_id` |

| Integración | I-012: ms-tickets → ms-orders (consulta de orden) |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `GET /api/v1/orders/{orderId}` |
| **Propósito** | ms-tickets verifica el estado del pago antes de emitir boletas |
| **Owner del contrato** | ms-orders |
| **Auth** | API Key interna |
| **Timeout** | 3s |
| **Retry** | 2 reintentos |
| **Fallback** | Si no puede consultar, emisión rechazada (prefiere no emitir que emitir sin pago confirmado) |

---

### 1.3 Integración con Mercado Pago (Externo)

| Integración | I-013: ms-orders → Mercado Pago (Checkout) |
|---|---|
| **Tipo** | Sync REST API |
| **Método** | `POST /checkout/preferences` (crear preferencia de pago) |
| **Propósito** | Iniciar checkout redirigiendo al comprador a Mercado Pago |
| **Owner del contrato** | Mercado Pago (API externa) |
| **Auth** | Access Token de Mercado Pago (vía Secrets Manager) |
| **Timeout** | 10s |
| **Retry** | 2 reintentos con backoff. Si falla → checkout rechazado |
| **Idempotencia** | `external_reference = order_id` en la preferencia. Si se duplica la creación, Mercado Pago retorna la preferencia existente |
| **Datos sensibles** | CriptoPass NUNCA recibe datos de tarjeta. La tokenización ocurre del lado de Mercado Pago |

| Integración | I-014: Mercado Pago → ms-orders (Webhook IPN) |
|---|---|
| **Tipo** | **Webhook (Async Inbound)** |
| **Método** | `POST /api/v1/webhooks/mercadopago` (endpoint expuesto públicamente en ALB) |
| **Propósito** | Notificar cambios de estado de pago (aprobado, rechazado, pendiente) |
| **Owner del contrato** | ms-orders (handler del webhook) |
| **Auth** | Validación de firma HMAC `x-signature` + `x-request-id` de Mercado Pago. Secreto compartido vía Secrets Manager |
| **Timeout** | ms-orders procesa en < 5s. Mercado Pago espera HTTP 200 en < 10s |
| **Retry** | Mercado Pago reintenta webhooks no confirmados con backoff propio. ms-orders es idempotente |
| **Idempotencia** | **Crítica.** `payment_id` de Mercado Pago como llave de idempotencia en ms-orders. Si el webhook ya fue procesado, se retorna 200 sin efectos secundarios. Implementado con tabla `processed_webhooks` (payment_id único) |
| **Ordenamiento** | Tolerante a desorden. Si llega notificación de pago aprobado antes que el pendiente, ms-orders procesa la aprobación y luego ignora la pendiente (estado ya superado) |
| **Dead Letter** | Webhooks que fallan validación de firma → HTTP 403 (no reintentar). Webhooks que fallan procesamiento interno → HTTP 500 (Mercado Pago reintenta) |
| **IP Allowlist** | Opcional: restringir IPs de origen a rangos de Mercado Pago (documentados en su API reference). WAF rule |

---

### 1.4 Eventos de Dominio (EventBridge)

| Integración | I-015: ms-orders → EventBridge |
|---|---|
| **Tipo** | Async Event (EventBridge PutEvents) |
| **Eventos** | `PaymentConfirmed`, `PaymentFailed`, `OrderCancelled`, `OrderRefunded` |
| **Owner del contrato** | ms-orders (schema del evento) |
| **Schema** | JSON con versionado (`"version": "1"`). Campos: `order_id`, `user_sub`, `event_id`, `amount_total`, `currency`, `payment_id`, `timestamp` |
| **Event Bus** | `criptopass-events` (custom event bus) |
| **Retry** | EventBridge built-in: 3 reintentos con backoff si el target falla. Luego → SQS DLQ |
| **Idempotencia** | Cada evento lleva `event_id` único. Consumers deduplican por event_id |
| **Ordering** | Eventos de una misma orden pueden llegar desordenados. Cada consumer debe tolerar y manejar estado actual |

| Integración | I-016: ms-tickets → EventBridge |
|---|---|
| **Tipo** | Async Event (EventBridge PutEvents) |
| **Eventos** | `TicketIssued` |
| **Owner del contrato** | ms-tickets (schema del evento) |
| **Schema** | `ticket_id`, `ticket_hash` (SHA-256), `order_id`, `event_id`, `user_sub`, `timestamp_issuance` |
| **Event Bus** | `criptopass-events` |
| **Idempotencia** | `event_id` único. Deduplicación en consumer (fn-blockchain-anchor) |

---

### 1.5 Colas SQS (EventBridge Targets)

| Integración | I-017: EventBridge → SQS notifications → fn-notifications |
|---|---|
| **Tipo** | Async Queue (SQS Standard) |
| **Cola** | `criptopass-notifications-queue` |
| **Eventos routeados** | `PaymentConfirmed` → email confirmación. `OrderRefunded` → email reembolso. `OrderCancelled` → email cancelación |
| **Consumer** | Lambda `fn-notifications` (Go) |
| **Batch size** | 10 mensajes por invocación |
| **Visibility Timeout** | 60s (mayor que el tiempo máximo de envío SES, ~5s) |
| **Retry** | SQS: máximo 3 intentos. Luego → DLQ `criptopass-notifications-dlq` |
| **Dead Letter** | DLQ con retención 14 días. Alarma CloudWatch si DLQ no está vacía por > 5 min |
| **Idempotencia** | Lambda es idempotente: si un mismo evento se entrega dos veces, envía el email dos veces (aceptable). En V2: tabla de emails enviados para deduplicación |
| **Ordering** | No requerido para notificaciones (cada email es independiente) |
| **Retention** | 4 días en cola principal |

| Integración | I-018: EventBridge → SQS ticket-events → fn-blockchain-anchor |
|---|---|
| **Tipo** | Async Queue (SQS Standard) |
| **Cola** | `criptopass-ticket-events-queue` |
| **Eventos routeados** | `TicketIssued` |
| **Consumer** | Lambda `fn-blockchain-anchor` (Go). También se ejecuta programada (EventBridge Scheduler) para procesar lotes |
| **Batch size** | 100 mensajes por invocación |
| **Visibility Timeout** | 120s (construcción de Merkle tree + anclaje puede tomar ~30-60s) |
| **Retry** | SQS: 3 intentos. Luego → DLQ `criptopass-ticket-events-dlq` |
| **Dead Letter** | DLQ con retención 7 días. Alarma si DLQ no vacía |
| **Idempotencia** | Lambda usa `ticket_hash` como llave de deduplicación. Si el hash ya fue incluido en un lote, se ignora |
| **Ordering** | No requerido (los hashes son independientes; el orden en el Merkle tree no importa mientras sea determinístico) |
| **Retention** | 1 día en cola principal (si no se procesa en 1 día, el anclaje programado lo captura) |

---

### 1.6 Jobs Programados

| Integración | I-019: EventBridge Scheduler → fn-blockchain-anchor |
|---|---|
| **Tipo** | Scheduled Job (cron) |
| **Schedule** | `rate(5 minutes)` o cada 500 boletas acumuladas (el que ocurra primero) |
| **Propósito** | Barrido periódico para construir lote Merkle con boletas no procesadas aún y anclar raíz en Polygon |
| **Consumer** | Lambda `fn-blockchain-anchor` (misma lambda que procesa eventos, activada por scheduler) |
| **Timeout** | Lambda: 180s máximo |
| **Retry** | Lambda: 2 reintentos si falla el anclaje Polygon. Backoff: 30s, 60s. Si falla definitivamente → lote queda en estado "fallido" en DynamoDB y se reintenta en el siguiente ciclo |
| **Idempotencia** | La lambda consulta DynamoDB para no re-procesar lotes ya anclados. Usa `ticket_hash` como deduplicación |
| **Alerta** | Si un lote falla 3 ciclos consecutivos → alarma CloudWatch → SNS → equipo |

---

### 1.7 Integraciones con Infraestructura AWS

| Integración | I-020: ms-catalog → AWS S3 (Presigned URLs) |
|---|---|
| **Tipo** | AWS SDK (sync, generación de URL) |
| **Método** | `S3Client.generatePresignedUrl(PutObject)` |
| **Propósito** | El frontend (admin) obtiene URL prefirmada para subir imagen directamente a S3 sin pasar por el backend |
| **Owner del contrato** | ms-catalog (dueño del bucket) |
| **Auth** | IAM role del microservicio (EC2 task role) |
| **Timeout** | 3s para generar la URL. La URL expira en 5 minutos |
| **Retry** | 1 reintento si falla la generación |
| **Bucket** | `criptopass-catalog-images-{env}`. Estructura: `events/{event_id}/{image_uuid}.{ext}` |

| Integración | I-021: fn-notifications → AWS SES |
|---|---|
| **Tipo** | AWS SDK (SendEmail) |
| **Propósito** | Envío de emails transaccionales vía SES |
| **Auth** | IAM role de la Lambda |
| **Timeout** | 10s (SES SendEmail) |
| **Retry** | Lambda: 2 reintentos. Si SES falla → mensaje vuelve a SQS (visibility timeout) |
| **Dominio verificado** | `mail.criptopass.com` (SES verified domain). DKIM + SPF configurados |

| Integración | I-022: fn-blockchain-anchor → Polygon PoS L2 |
|---|---|
| **Tipo** | JSON-RPC (escritura on-chain) |
| **Método** | `eth_sendRawTransaction` a un nodo RPC de Polygon (Infura/Alchemy o nodo propio) |
| **Propósito** | Anclar la raíz Merkle en un smart contract desplegado en Polygon |
| **Owner del contrato** | fn-blockchain-anchor (dueño del smart contract) |
| **Auth** | Private key de la wallet de CriptoPass (vía Secrets Manager). La tx se firma offline |
| **Timeout** | 60s para confirmación de transacción (Poli p95 < 5s en L2) |
| **Retry** | 3 reintentos con backoff (10s, 30s, 60s). Nonce manejado correctamente para no duplicar tx |
| **Gas** | Polygon L2: costo mínimo (~$0.001-$0.01 por anclaje). Gas price dinámico |
| **Smart Contract** | Contrato simple: `function anchorMerkleRoot(bytes32 merkleRoot, uint256 batchTimestamp)`. Emite evento `MerkleAnchored(bytes32 indexed root, uint256 timestamp)` |
| **Red** | Polygon Mainnet (prod), Polygon Amoy testnet (dev/staging) |

---

### 1.8 Validación JWT (Cross-Cutting, Todos los Servicios)

| Integración | I-023: Todos los microservicios → Amazon Cognito (JWKS) |
|---|---|
| **Tipo** | Sync HTTP (validación de firma JWT) |
| **Método** | `GET /.well-known/jwks.json` del User Pool |
| **Propósito** | Validar firma de JWT en cada request autenticado |
| **Owner del contrato** | Amazon Cognito (OIDC standard) |
| **Auth** | Ninguna (endpoint público de Cognito) |
| **Timeout** | 2s para fetch de JWKS (cacheado localmente 5 min) |
| **Retry** | 2 reintentos si JWKS no disponible. Circuit breaker |
| **Claims validados** | `iss` (issuer del User Pool), `aud` (client ID esperado), `exp` (no expirado), `token_use` (access), `cognito:groups` (roles) |
| **Cache** | JWKS cacheado en memoria con TTL 5 min por microservicio (Spring Security OAuth2 Resource Server lo hace nativamente) |

---

### 1.9 Integraciones Futuras / Pendientes

| Integración | I-024: ms-orders → DIAN (Facturación Electrónica) |
|---|---|
| **Tipo** | Por definir (REST API o SOAP probablemente) |
| **Estado V1** | **Pendiente.** Supuesto S-07: V1 emite tiquete POS electrónico (integración simple). Si Q-CR-05 resuelve factura electrónica completa, se agrega adaptador |
| **Owner del contrato** | DIAN (regulatorio) |
| **Impacto** | No bloquea el landscape V1. El bounded context Orders ya tiene el punto de extensión para emitir comprobante fiscal post-pago |

---

## 2. Matriz de Idempotencia

| Integración | Key de Idempotencia | Quién garantiza | Modo de fallo si no es idempotente |
|---|---|---|---|
| I-002 (POST orden) | `Idempotency-Key` header (UUID v4) | ms-orders | Órdenes duplicadas |
| I-007 (Validación QR) | `ticket_id` interno | ms-tickets | Doble redención (fallo de negocio grave) |
| I-010 (Emisión boletas) | `order_id` | ms-tickets | Boletas duplicadas (fallo de negocio grave) |
| I-011 (Reembolso) | `order_id` | ms-orders | Doble reembolso (pérdida financiera) |
| I-014 (Webhook MP) | `payment_id` de Mercado Pago | ms-orders (tabla `processed_webhooks`) | Doble emisión de boletas |
| I-015 (Eventos dominio) | `event_id` único | Producer (UUID v4 en cada evento) | Consumidores procesan dos veces (mitigado: consumers idempotentes) |
| I-022 (Anclaje Polygon) | `batch_id` (lote) | fn-blockchain-anchor | Doble anclaje (costo duplicado, no fallo de negocio) |

---

## 3. Matriz de Consistencia por Flujo

| Flujo | Consistencia | Mecanismo | Lag máximo |
|---|---|---|---|
| Órdenes → Pago → Emisión boletas | **Fuerte** | Saga orquestada (Step Functions) con compensaciones. Transacciones locales en cada paso | 30s post-pago (emisión) |
| Catálogo → Cache Redis | **Eventual** | Cache aside: ms-catalog invalida/actualiza cache on write. TTL 60s en Redis para lecturas | 60s máximo |
| Emisión → Anclaje Merkle | **Eventual** | Evento `TicketIssued` → SQS → Lambda → Polygon | 5 min (lote programado) |
| Perfil → Cognito sync | **Eventual** | ms-users escribe en Cognito Admin API. Sincronización en background | ~5s |
| Webhook MP → Estado de pago | **Fuerte** | Idempotencia + transacción local en ms-orders | < 5s desde webhook recibido |

---

## 4. Observabilidad de Integraciones

| Integración | Trace Propagation | Métricas Clave | Alarmas |
|---|---|---|---|
| Todas las sync REST | `X-Trace-Id` header propagado por ALB y servicios | Latencia p95, tasa de error 5xx, throughput | Latencia > SLA, error rate > 1% |
| Todas las async (EventBridge, SQS) | `trace_id` en atributo del mensaje | Mensajes en DLQ, age of oldest message | DLQ no vacía > 5 min, mensajes > 1h sin procesar |
| Webhook Mercado Pago | `X-Request-Id` de Mercado Pago + `X-Trace-Id` generado por ms-orders | Tasa de webhooks con firma inválida, latencia de procesamiento | > 10 webhooks fallando en 5 min |
| Anclaje Polygon | `batch_id` como correlation ID | Anclajes exitosos/hora, tiempo de confirmación de tx | Fallo 3 ciclos consecutivos |

---

## 5. Notas y Supuestos

- **Convención de URLs:** El ALB expone `api.criptopass.com` (prod) y `api.staging.criptopass.com`. Vercel dirige todo el tráfico de API a estas URLs con CORS configurado.
- **Presigned URLs:** La carga de imágenes usa presigned URLs de S3 generadas por ms-catalog. El frontend admin sube directamente a S3 sin que la imagen pase por el backend (eficiencia de ancho de banda).
- **QR dinámico:** La generación del QR (`GET /api/v1/tickets/{id}/qr`) usa HMAC con clave rotada periódicamente (Secrets Manager). La validación (`POST /api/v1/tickets/validate`) verifica firma HMAC + ventana temporal ±5s.
- **DIAN:** La integración I-024 está documentada como pendiente. No afecta las demás integraciones. Si se activa en V1, se agregará como una invocación adicional en la saga de Step Functions (post-emisión, pre-notificación) o como un evento async que dispare una lambda de facturación.
- **Polygon:** El smart contract de anclaje es mínimo (solo almacena raíz Merkle + timestamp). No almacena datos de negocio on-chain. El costo por anclaje es fracción de centavo de dólar en MATIC.
