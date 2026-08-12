# ADR-008: Estrategia de Datos — PostgreSQL por microservicio + Flyway + Redis/ElastiCache

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-008 |
| **Título** | Estrategia de datos: PostgreSQL con base de datos separada por microservicio, Flyway como gestor de esquema, JPA modo validate, Redis para cache/cola/TTL |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass tiene cuatro bounded contexts que requieren persistencia relacional (transacciones ACID, integridad referencial, consultas complejas):

- **Catalog:** eventos, venues, imágenes → SQL con búsquedas full-text y filtros.
- **Orders:** órdenes, pagos, reservas → transacciones estrictas para evitar doble venta.
- **Tickets:** boletas, hashes, estado de redención → integridad para anti-duplicación.
- **Users:** perfiles, consentimientos → datos con PII que requieren control de acceso.

Además, se requieren datos volátiles de alta velocidad: cola de espera, reservas TTL, cache de catálogo.

El usuario estableció (S-01): PostgreSQL como BD, Flyway para esquema, JPA en modo validate, sin event sourcing.

## Decisión

### Parte A: Base de Datos Separada por Microservicio

**Prohibido shared database.** Cada microservicio tiene su propia base de datos PostgreSQL en la misma instancia RDS o en instancias separadas.

| Microservicio | Base de datos | Schemas |
|---|---|---|
| ms-catalog | `catalog_db` | `catalog` (eventos, venues, imágenes) |
| ms-orders | `orders_db` | `orders` (órdenes, pagos, reservas, webhooks) |
| ms-tickets | `tickets_db` | `tickets` (boletas, hashes, redenciones, QRs) |
| ms-users | `users_db` | `users` (perfiles, consentimientos, auditoría) |

**Implementación en RDS:**

| Opción | Evaluación |
|---|---|
| **Opción A:** Una instancia RDS con 4 bases de datos lógicas (`CREATE DATABASE`) | Menor costo (1 instancia RDS). Separación lógica. Riesgo: contención de recursos (CPU, IOPS) entre servicios. |
| **Opción B:** 4 instancias RDS independientes | Máximo aislamiento, pero 4x costo. Overkill para V1 con baja carga de escritura. |
| **Opción C (elegida):** Dos clústeres RDS: uno para `catalog_db` (lectura intensiva) y otro para `orders_db + tickets_db + users_db` (escritura transaccional) | Balance costo/aislamiento. Catalog_db puede tener réplicas de lectura. Orders/tickets/users comparten instancia con BD separadas. |

**V1: Opción C con 2 instancias RDS.** En V2, si la carga lo justifica, se separan en instancias independientes.

### Parte B: Flyway como Único Gestor de Esquema

- **Flyway** es la única herramienta autorizada para modificar el esquema de PostgreSQL.
- Prohibido: `hibernate.ddl-auto = update`, `create`, `create-drop` en cualquier ambiente que no sea tests locales.
- `spring.jpa.hibernate.ddl-auto = validate`: Hibernate valida que las entidades JPA coincidan con el esquema, pero **no modifica** la BD.
- **Migraciones versionadas:** `V{version}__{description}.sql`. Ejemplo: `V001__create_events_table.sql`.
- Las migraciones se ejecutan al iniciar el microservicio (modo `flyway migrate` en startup).
- **Rollback:** Flyway no soporta rollback automático. Estrategia: migraciones forward-only con `UNDO` solo en dev (opcional). En staging/prod: nueva migración que revierte el cambio.

### Parte C: JPA en Modo Validate

- Entidades JPA mapean el dominio a tablas, pero **no crean ni modifican el esquema**.
- Flyway es la fuente de verdad del esquema físico. JPA es la fuente de verdad del mapeo objeto-relacional.
- Coherencia garantizada por `ddl-auto = validate`: si una entidad no coincide con la tabla, el startup falla con error claro.
- Uso de `@Column`, `@Table`, `@Enumerated`, etc., con nombres explícitos (no depender de naming strategy implícita).

### Parte D: Redis/ElastiCache para Datos Volátiles

| Uso | Estructura Redis | TTL | Dueño lógico |
|---|---|---|---|
| **Sala de espera (cola FIFO)** | `ZSET` por evento (`waiting_room:{event_id}`) con score = timestamp de llegada. Control de admisión con contador atómico | Sin TTL (se limpia al terminar venta pico o agotar inventario) | ms-orders |
| **Reservas de inventario** | `STRING` con TTL: `reservation:{order_id}` → `JSON {event_id, quantity, ttl_timestamp}` | 15 minutos (configurable por evento). Redis evicta automáticamente | ms-orders |
| **Cache de catálogo** | `STRING` con serialización JSON: `event_list:{filters_hash}`, `event_detail:{event_id}` | 60s para listados, 30s para detalle | ms-catalog |
| **Rate limiting** | `INCR` + `EXPIRE`: `rate_limit:{ip}` o `rate_limit:{user_sub}` | Ventana de 1 minuto | ALB (WAF) o ms-catalog |

**Topología Redis:**
- V1: una instancia ElastiCache (Redis OSS, `cache.t4g.micro` en dev, `cache.m6g.large` multi-AZ en prod).
- V2: separar en dos clústeres si la sala de espera compite con el cache de catálogo.

### Parte E: DynamoDB para Traceability

La lambda `fn-blockchain-anchor` usa **DynamoDB** en lugar de PostgreSQL:
- Sin connection pooling (lambda-friendly).
- Modelo key-value simple: `batch_id` (PK) → `merkle_root`, `ticket_hashes`, `polygon_tx_hash`, `status`, `created_at`.
- On-demand capacity (sin provisioning).
- No compite con las BD relacionales.

## Alternativas Consideradas

| Aspecto | Alternativa | Evaluación |
|---|---|---|
| **Shared database** | Una sola BD con schemas por servicio | **Bloqueado por regla de arquitectura.** Viola bounded contexts. Acopla deployments (un cambio de esquema en catalog puede romper orders). |
| **Schema por servicio en misma BD** | `catalog.*`, `orders.*`, etc. en misma BD física | Menor acoplamiento que shared tables, pero aún comparten recursos (conexiones, locks, backups). No es eliminación completa de acoplamiento. |
| **NoSQL para todo (DynamoDB)** | DynamoDB para todos los bounded contexts | Pierde transacciones ACID, integridad referencial, consultas SQL complejas. Catálogo con filtros avanzados sería muy complejo en DynamoDB. |
| **MySQL en lugar de PostgreSQL** | Alternativa de BD relacional | PostgreSQL tiene mejor soporte para JSON, full-text search, y es el estándar en el ecosistema AWS con RDS. Sin ventaja de cambio. |
| **Hibernate DDL auto-update** | `hibernate.ddl-auto = update` | **Riesgo en producción:** cambios implícitos, sin control de versión del esquema, sin rollback. Flyway es el estándar de industria. |
| **Redis para todo (incluso datos permanentes)** | Redis como BD primaria | No es durable para datos de negocio (boletas, órdenes). Configurable para persistencia pero no reemplaza ACID de PostgreSQL. |

## Consecuencias

**Positivas:**
- Cada microservicio es dueño absoluto de su BD: puede cambiar esquema, índices y queries sin afectar otros servicios.
- Flyway + JPA validate garantiza que esquema y entidades estén siempre sincronizados.
- Redis separa datos volátiles (cola, cache) de datos permanentes (eventos, órdenes, boletas), optimizando cada storage para su propósito.
- DynamoDB para traceability evita que una lambda Go tenga que manejar connection pooling a PostgreSQL.

**Negativas:**
- 2 instancias RDS + 1 ElastiCache + 1 DynamoDB = costo operativo mensual ~$150-300/mes (dependiendo del tamaño).
- Consultas cross-service no pueden usar SQL JOINs; deben componerse vía APIs (ej. catálogo + disponibilidad).
- Flyway forward-only requiere disciplina en migraciones: cada cambio de esquema debe ser compatible con la versión anterior del código (expand/contract pattern para zero-downtime).

**Riesgos mitigados:**
- **Zero-downtime deployments:** Las migraciones Flyway deben ser backward-compatible (agregar columna con default, no renombrar/droppear columnas usadas). Ver `zero-downtime-migrations` skill.
- **Rendimiento Redis:** Monitorear memoria y evicciones en ElastiCache. Alarmas si evictions > 0 o si memory usage > 80%.

---

## Notas

- Los backups de RDS son automáticos (7 días en dev, 30 días en prod). Point-in-time recovery habilitado.
- Cada microservicio tiene credenciales de BD independientes (usuario `ms_catalog`, `ms_orders`, etc.) con permisos solo sobre su BD (GRANT ALL ON DATABASE catalog_db TO ms_catalog).
- Las credenciales se inyectan vía variables de entorno desde Secrets Manager en la task definition de ECS.
- **Migraciones en CI/CD:** Flyway migrate se ejecuta como parte del startup del microservicio. Alternativa: paso separado en CI/CD antes del deploy. V1: startup (más simple).
- La traceability DynamoDB tiene TTL de 7 años en los datos (una boleta puede necesitar verificarse años después). On-demand pricing evita costo de capacidad provisionada para datos fríos.
