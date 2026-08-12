# ADR-009: Sala de Espera Virtual — Redis-backed dentro de ms-orders con reserva de inventario con TTL

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-009 |
| **Título** | Sala de espera virtual + reserva de inventario con TTL: Redis-backed dentro del bounded context Orders, fairness ante concurrencia sobre última boleta |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass debe soportar ventas pico (eventos de alta demanda con miles de usuarios concurrentes, supuesto S-16). Los requisitos:

- **F-PC-04:** Sala de espera virtual para ventas pico (cola FIFO con control de admisión).
- **F-PC-05:** Reserva de inventario con TTL (expira si no se completa el pago).
- **EC-07:** Dos usuarios por la última boleta — solo uno la obtiene; el otro recibe "agotado".
- **EC-08:** Se acaba el inventario mientras el usuario está en fila → notificación, opción de salir o esperar.
- **R-01:** Concurrencia sobre inventario pico — requiere diseño de reserva con TTL, locks o patrón secuencial.
- **S-16:** Sistema debe soportar miles de usuarios concurrentes sobre un mismo evento.

### Pregunta: ¿Waiting Room como bounded context propio o parte de Orders?

La sala de espera está intrínsecamente acoplada al inventario:
- Un usuario sale de la fila ↔ hay inventario disponible.
- La posición en fila solo tiene significado si hay boletas que comprar.
- La admisión controlada es una decisión del flujo de compra (Orders).

## Decisión

### Parte A: Waiting Room dentro del bounded context Orders

**La sala de espera NO es un bounded context independiente.** Es un **módulo interno** de `ms-orders` implementado completamente en Redis.

**Justificación:**
1. **Acoplamiento natural:** El estado de la cola (quién está en fila, en qué posición) solo tiene sentido en relación con el estado del inventario (cuántas boletas quedan). Separarlos en dos bounded contexts crearía una dependencia circular o una coreografía compleja que añade latencia en el hot path de compra.
2. **Atomicidad:** La operación "salir de la fila + reservar inventario" debe ser atómica. Si Waiting Room fuera otro microservicio, requeriría una transacción distribuida o un protocolo de two-phase commit entre dos servicios, agregando latencia y puntos de fallo.
3. **Simplicidad operativa:** Un solo servicio (ms-orders) gestiona la cola, la reserva y el checkout. Menos infraestructura, menos contratos cross-service, menos latencia.
4. **Redis ya es requerido por Orders:** ms-orders ya usa Redis para las reservas TTL. La sala de espera usa las mismas estructuras de Redis (Sorted Sets para cola FIFO). Sin infraestructura adicional.

### Parte B: Diseño de la Sala de Espera en Redis

#### Cola FIFO por evento

```
Redis Key: waiting_room:{event_id}
Estructura: Sorted Set (ZSET)
  - Member: user_sub (UUID del usuario Cognito)
  - Score: timestamp Unix de llegada a la cola (milisegundos)
```

**Operaciones atómicas con Lua scripting (garantiza atomicidad y fairness):**

1. **Entrar a la cola (`JOIN_QUEUE`):**
   ```
   ZADD waiting_room:{event_id} {current_timestamp_ms} {user_sub}
   ZRANK waiting_room:{event_id} {user_sub}  → retorna posición (0-indexed)
   ```
   - El usuario ve "Estás en la posición N de M".
   - Si el evento no tiene sala de espera activa (flag en PostgreSQL), va directo al checkout.

2. **Estado de la cola (`QUEUE_STATUS`):**
   ```
   ZRANK waiting_room:{event_id} {user_sub}  → posición actual
   ZCARD waiting_room:{event_id}              → total en cola
   GET inventory:{event_id}:{ticket_type}      → boletas disponibles
   ```

3. **Admitir siguiente usuario (`ADMIT_NEXT`):**
   ```
   -- Lua script atómico:
   local user = ZPOPMIN waiting_room:{event_id}
   if user then
     local available = DECR inventory:{event_id}:{ticket_type}
     if available >= 0 then
       return {user, available}  -- Admitido
     else
       INCR inventory:{event_id}:{ticket_type}  -- Rollback
       return nil  -- Sin inventario
     end
   end
   ```
   - Trigger: cada vez que un usuario completa/expira su reserva, se admite al siguiente.
   - También se ejecuta periódicamente (cada 500ms-1s) para drenar la cola de forma proactiva.

4. **Salir de la cola (`LEAVE_QUEUE`):**
   ```
   ZREM waiting_room:{event_id} {user_sub}
   ```

5. **Limpiar cola (`CLEANUP_QUEUE`):**
   ```
   DEL waiting_room:{event_id}
   ```
   - Al terminar la venta pico (evento agotado o finalizado).

#### Estado global:
| Key | Tipo | Propósito |
|---|---|---|
| `waiting_room:{event_id}` | ZSET | Cola FIFO de usuarios |
| `inventory:{event_id}:{ticket_type}` | INT | Contador atómico de inventario disponible (decrementado al admitir, incrementado al liberar reserva) |
| `reservation:{order_id}` | STRING (JSON) + TTL | Reserva activa con TTL. Al expirar, Redis notifica a ms-orders vía keyspace notification o el TTL se verifica en el checkout |
| `waiting_room_meta:{event_id}` | HASH | Metadatos: `active` (bool), `total_admitted`, `total_waiting`, `started_at` |

### Parte C: Reserva de Inventario con TTL

1. **Al crear orden (post-admisión o compra directa):**
   - ms-orders verifica `inventory:{event_id}:{ticket_type}` >= cantidad solicitada.
   - Si hay suficiente: `DECRBY inventory {quantity}` y crea `reservation:{order_id}` con TTL de 15 minutos.
   - Si no hay suficiente: error "agotado".

2. **Al expirar TTL:**
   - Redis keyspace notification en key `reservation:{order_id}` con evento `expired`.
   - ms-orders escucha estas notificaciones (Redis Pub/Sub `__keyevent@0__:expired`) y:
     - `INCRBY inventory:{event_id}:{ticket_type} {quantity}` (libera inventario).
     - Marca la orden como `expired` en `orders_db`.
     - Si hay usuarios en `waiting_room:{event_id}`, admite al siguiente.

3. **Edge case EC-03 (pago tardío tras TTL expirado):**
   - Si llega webhook de pago aprobado para una orden expirada, ms-orders detecta `status = expired` y ejecuta reembolso automático.

### Parte D: Fairness ante Concurrencia sobre la Última Boleta (EC-07)

**Problema:** Dos compradores (o uno en checkout y otro saliendo de la fila) intentan reservar la última boleta.

**Solución: Operaciones atómicas en Redis con Lua scripting.**

El contador `inventory:{event_id}:{ticket_type}` en Redis es la fuente de verdad para disponibilidad en tiempo real. Al ser operaciones atómicas (`DECR`/`INCR` en Lua), se garantiza que:
- Si inventory = 1 y dos usuarios intentan `DECRBY 1` concurrentemente, Redis procesa uno primero (inventory → 0, éxito) y el segundo después (inventory → -1 → se detecta y se revierte con `INCR`).
- No se requiere lock distribuido porque Redis es single-threaded para operaciones atómicas.

**Flujo:**
1. Usuario A (ya en checkout) tiene `DECRBY inventory 1` → resultado = 0. Éxito. Reserva creada.
2. Usuario B (sale de la fila) intenta `DECRBY inventory 1` → resultado = -1. Se detecta en Lua script → `INCRBY inventory 1` (rollback) → se le notifica "agotado". Se admite al siguiente en fila o se cierra la sala.

### Parte E: Componentes Involucrados

| Componente | Responsabilidad |
|---|---|
| **ms-orders (WaitingRoomModule)** | Lógica de negocio: inicio/parada de sala, admisión, polling de estado. API: `POST /api/v1/waiting-room/{eventId}/join`, `GET /api/v1/waiting-room/{eventId}/status` |
| **Redis (Sorted Sets + Counters)** | Estado de la cola (FIFO), contadores atómicos de inventario, reservas con TTL. Sin estado en PostgreSQL para estos datos volátiles |
| **EventBridge + Step Functions (opcional)** | La saga de compra existente maneja el flow post-admisión. Sin cambios |
| **PostgreSQL (`orders_db`)** | Estado canónico de la orden. Redis es cache/cola volátil; PostgreSQL es la verdad |

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **Waiting Room como microservicio separado** | Mayor aislamiento pero requiere coreografía con ms-orders para cada admisión (latencia, punto de fallo). La operación "admitir + reservar" deja de ser atómica. Overhead operativo para una funcionalidad que no tiene aggregate propio (no persiste datos de negocio). |
| **Cola en PostgreSQL (tabla `waiting_room`)** | Transaccional pero lento: miles de usuarios consultando posición cada pocos segundos saturarían PostgreSQL. Redis es 100-1000x más rápido para este patrón. |
| **Amazon SQS FIFO como cola** | Infraestructura manejada, pero sin visibilidad de posición ("cuántos hay delante de mí") ni capacidad de "salir de la cola sin consumir mensaje". No apto para sala de espera con UX de posición. |
| **Sin sala de espera (solo reserva TTL)** | En ventas pico, miles de usuarios golpeando el endpoint de checkout simultáneamente. Sin control de admisión, la experiencia es un "race condition" masivo. La sala de espera es necesaria para fairness y control de carga. |
| **Lock optimista en PostgreSQL para inventario** | No escala a miles de concurrentes (contention en la fila de locks). Redis con operaciones atómicas es más eficiente para el contador de inventario en tiempo real. |

## Consecuencias

**Positivas:**
- Operación atómica "admitir + reservar" en un solo Lua script → sin race conditions.
- Redis maneja miles de operaciones/segundo sin degradación (single-threaded pero extremadamente rápido).
- La cola es visible para el usuario ("posición 47 de 200") mejorando UX.
- Si Redis falla, la sala de espera se desactiva y el sistema degrada a "compra directa con reserva TTL" (graceful degradation). La información canónica de órdenes está en PostgreSQL.
- Sin infraestructura adicional: Redis ya existe para reservas TTL y cache.

**Negativas:**
- Redis keyspace notifications para TTL expirado no son 100% confiables (pueden perderse). Mitigación: ms-orders también verifica TTL al procesar el pago y periódicamente barre órdenes con TTL expirado (job cada 1 minuto).
- Si Redis se reinicia, se pierde el estado de las colas activas. Mitigación: al iniciar ms-orders, reconstruye las colas desde las órdenes activas en PostgreSQL (usuarios que no han completado compra).
- La sala de espera añade complejidad al frontend (polling de posición, vista de espera).

**Riesgos mitigados:**
- **R-01 (concurrencia sobre inventario pico):** Lua scripts atómicos en Redis + contador atómico.
- **EC-07 (última boleta):** Atomicidad de Redis garantiza que solo un usuario obtiene la última boleta.
- **EC-08 (inventario agotado en fila):** El Lua script de admisión detecta inventory ≤ 0 y notifica al usuario; se cierra la sala.
- **Fairness:** ZSET con score = timestamp de llegada garantiza orden FIFO estricto.

---

## Notas

- **Activación de sala de espera:** El organizador (o SUPER_ADMIN) marca un evento como "venta pico" (`is_high_demand = true` en catalog_db). ms-orders verifica este flag y activa la sala de espera. Alternativa automática: si `inventory` < umbral y `concurrent_users` > umbral.
- **Timeout de inactividad en fila:** Si un usuario no responde al ser admitido en N segundos (ej. 60s), se le saca de la fila y se admite al siguiente. Implementado con TTL en una key temporal `admission_pending:{user_sub}`.
- **Rate limiting adicional:** La sala de espera actúa como rate limiter natural. Aun así, los endpoints de `JOIN_QUEUE` y `QUEUE_STATUS` deben tener rate limiting en ALB/WAF.
- **Métricas de sala de espera:** CloudWatch metrics para: `waiting_room_size`, `admission_rate`, `abandonment_rate`, `avg_wait_time`. Dashboard en tiempo real para operaciones durante ventas pico.
