# ADR-003: CQRS-lite — Modelos Lectura/Escritura sobre misma BD, sin Event Sourcing

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-003 |
| **Título** | CQRS-lite: modelos de lectura y escritura separados sobre la misma base de datos PostgreSQL, sin Event Sourcing en V1 |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass tiene dos perfiles de carga radicalmente distintos:

1. **Lecturas masivas (catálogo público):** miles de visitantes anónimos navegando eventos, buscando y filtrando. La latencia debe ser baja (p95 < 200ms) y el contenido altamente cacheable. El catálogo es el lado Query.
2. **Escrituras transaccionales (compra):** flujo de checkout, reserva con TTL, pago, emisión de boletas. Consistencia fuerte requerida. Baja frecuencia relativa comparada con lecturas (cientos de compras vs miles de navegaciones). Las órdenes y boletos son el lado Command.

El usuario estableció (S-01): CQRS-lite, modelos de lectura y escritura sobre la misma BD, sin event sourcing. Catálogo público cacheable.

## Decisión

**CQRS-lite con las siguientes características:**

### Separación de modelos de lectura y escritura

| Aspecto | Lado Command (Escritura) | Lado Query (Lectura) |
|---|---|---|
| **Servicio** | ms-orders, ms-tickets | ms-catalog |
| **Base de datos** | `orders_db`, `tickets_db` (PostgreSQL) | `catalog_db` (PostgreSQL) |
| **Modelo de dominio** | Entidades JPA ricas con invariantes de negocio: Order, Payment, Ticket, Reservation | Vistas materializadas, proyecciones JPA/DTOs: EventSummary, EventDetail, EventAvailability |
| **Transacciones** | ACID, locks optimistas/pesimistas para concurrencia sobre inventario | Solo lectura. Read-only transactions o queries sin transacción |
| **Optimización** | Índices para escritura eficiente, constraints de integridad | Índices compuestos para búsquedas/filtros, desnormalización controlada en vistas |
| **Cache** | No aplica (datos en tiempo real) | Redis/ElastiCache: TTL 60s para listados, 30s para detalle de evento |

### Sin Event Sourcing en V1
- No se almacena un log de eventos como fuente de verdad.
- Las proyecciones de lectura se actualizan directamente en la misma transacción de escritura (o mediante invalidación de cache).
- No hay necesidad de reconstruir estado desde eventos.
- Se evita la complejidad operativa de event sourcing (event stores, snapshots, proyecciones asíncronas, consistencia eventual entre proyecciones).

### Mecanismo de sincronización Command → Query
Cuando ms-catalog crea o modifica un evento (CRUD desde el admin), la misma operación:
1. Escribe en `catalog_db` el estado canónico del evento.
2. Invalida la entrada correspondiente en Redis (cache aside pattern).

Cuando ms-orders cambia el inventario (reserva, liberación, emisión):
1. Actualiza `orders_db`.
2. Opcional: emite evento `InventoryChanged` a EventBridge. ms-catalog puede consumirlo para actualizar disponibilidad en cache.
3. En V1 simplificado: ms-catalog consulta disponibilidad bajo demanda (`GET /api/v1/catalog/events/{id}/availability` → ms-catalog calcula desde `catalog_db` + consulta opcional a ms-orders).

### Por qué NO Event Sourcing en V1
| Razón | Detalle |
|---|---|
| **Complejidad operativa** | Event sourcing requiere event store, snapshots, proyecciones, manejo de eventos duplicados, evolución de esquemas de eventos. Inviable para un MVP V1 con 4 microservicios. |
| **Curva de aprendizaje** | El equipo necesitaría dominar patrones como CQRS+ES, que son no triviales de implementar correctamente. |
| **Beneficio insuficiente en V1** | El principal beneficio de ES (auditabilidad completa y reconstrucción de estado) está cubierto por la tabla de auditoría y la trazabilidad blockchain. |
| **Out of scope explícito** | El brief V1 lista event sourcing como "out of scope". |

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **CQRS completo con proyecciones en BD separada (lectura)** | Mayor desacoplamiento pero más infraestructura (dos BD por bounded context, sincronización async). Complejidad innecesaria para V1. |
| **CQRS + Event Sourcing** | Máxima auditabilidad pero complejidad operativa y de desarrollo incompatible con MVP. Evaluado para V2+. |
| **Modelo único sin CQRS (mismas entidades para lectura y escritura)** | Las consultas del catálogo se volverían lentas al unirse con datos de escritura. La cache sería menos efectiva. No escala para el tráfico de lectura masiva. |
| **Read Replicas de PostgreSQL** | Complementa CQRS-lite (las réplicas de lectura pueden usarse para el lado Query en V2). V1: no necesario; el cache Redis absorbe la carga de lectura. |

## Consecuencias

**Positivas:**
- Simplicidad: una BD por bounded context, sin sincronización async de proyecciones.
- Cache Redis para catálogo reduce carga en PostgreSQL en ~90% de las requests de lectura.
- Las escrituras transaccionales no compiten con las lecturas masivas (distintos servicios, distintas BD).
- Fácil de evolucionar a CQRS completo en V2 si es necesario (agregar réplicas de lectura, separar proyecciones).

**Negativas:**
- La disponibilidad de inventario en el catálogo no es "en tiempo real" (cache 60s). Aceptable para catálogo; el checkout siempre consulta disponibilidad real en ms-orders.
- Si se necesita una vista consolidada "evento + ventas", se requiere una llamada cross-service (ms-catalog + ms-orders).
- Sin event sourcing, no hay historial completo de cambios de estado de órdenes (mitigado con tabla de auditoría y logs).

**Riesgos mitigados:**
- **R-10 (SEO vs CQRS-lite):** El catálogo cacheado garantiza que eventos recién publicados están disponibles en < 60s. SSR de Next.js usa los mismos endpoints cacheados.

---

## Notas

- El lado Query (ms-catalog) puede usar **JPA projections** o **native queries** para optimizar lecturas sin cargar entidades completas. Spring Data JPA soporta interface-based projections y `@Query` nativo.
- La cache Redis usa **cache aside**: la aplicación checkea Redis primero; si miss, consulta PostgreSQL y guarda en Redis con TTL.
- La invalidación de cache en ms-catalog se hace en los endpoints de escritura (POST/PUT/DELETE de eventos). Patrón: write-through invalidation.
