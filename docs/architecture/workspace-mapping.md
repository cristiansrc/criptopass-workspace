# Workspace Mapping — CriptoPass

| Campo | Valor |
|---|---|
| **Status** | `active` |
| **Proyecto** | CriptoPass (greenfield) |
| **Versión** | V1 |
| **Creado por** | enterprise-architect |
| **Fecha** | 2026-08-11 |

---

## 1. Estructura del Workspace

```
criptopass-workspace/                  # Raíz del Solution Workspace (repositorio de arquitectura)
├── docs/
│   ├── architecture/                  # Artefactos macro de arquitectura
│   │   ├── system-landscape.md
│   │   ├── context-map.md
│   │   ├── integration-map.md
│   │   ├── workspace-mapping.md       # Este archivo
│   │   └── decision-records/
│   │       ├── ADR-001-hybrid-microservices-lambda.md
│   │       ├── ADR-002-blockchain-traceability.md
│   │       ├── ADR-003-cqrs-lite.md
│   │       ├── ADR-004-amazon-cognito-idp.md
│   │       ├── ADR-005-payment-strategy-saga.md
│   │       ├── ADR-006-multi-repo-iac-ambientes.md
│   │       ├── ADR-007-sync-vs-async-communication.md
│   │       ├── ADR-008-data-strategy.md
│   │       └── ADR-009-waiting-room.md
│   ├── specs/
│   │   ├── requirements/
│   │   │   └── criptopass-v1-requirements-brief.md
│   │   ├── technical_debt.md          # Deuda técnica consolidada del workspace
│   │   └── workspace_changes.md       # Registro de cambios arquitectónicos globales
│   └── graphify-out/                  # Grafo de conocimiento (Graphify)
├── projects/                          # Subrepositorios (cada uno es un repo Git independiente)
│   ├── criptopass-ms-catalog/         # Catalog bounded context
│   ├── criptopass-ms-orders/          # Orders & Payments bounded context
│   ├── criptopass-ms-tickets/         # Tickets bounded context
│   ├── criptopass-ms-users/           # Identity/Users bounded context
│   ├── criptopass-fr-portal/          # Portal Compradores (Next.js, Turborepo)
│   ├── criptopass-fr-admin/           # Panel Admin (Next.js, Turborepo)
│   ├── criptopass-fr-shared/          # Paquetes UI/API compartidos entre fronts
│   ├── criptopass-fn-traceability/    # Lambda blockchain anchor (Go)
│   ├── criptopass-fn-notifications/   # Lambda notificaciones email (Go)
│   └── criptopass-infra/              # Infrastructure as Code
├── .gitignore                         # Ignora projects/* (repos independientes)
└── README.md                          # Visión general del workspace
```

---

## 2. Registro de Repositorios

### 2.1 Microservicios Backend (Kotlin + Spring Boot, ECS Fargate)

| Repositorio | `projects/criptopass-ms-catalog` |
|---|---|
| **Bounded Context** | Catalog (CTL) |
| **Owner funcional** | Product/Content |
| **Owner técnico** | Backend Team |
| **Stack** | Kotlin 2.x + Spring Boot 3.x + JPA + Flyway + PostgreSQL + OpenAPI first |
| **Base de datos** | `catalog_db` (RDS PostgreSQL 16, schema único) |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-ms-catalog` |

| Repositorio | `projects/criptopass-ms-orders` |
|---|---|
| **Bounded Context** | Orders & Payments (ORD) |
| **Owner funcional** | Commerce |
| **Owner técnico** | Backend Team |
| **Stack** | Kotlin 2.x + Spring Boot 3.x + JPA + Flyway + PostgreSQL + OpenAPI first |
| **Base de datos** | `orders_db` (RDS PostgreSQL 16, schema único) |
| **Dependencias infra** | Redis (ElastiCache) para Waiting Room y reservas TTL; Secrets Manager para credenciales Mercado Pago |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-ms-orders` |

| Repositorio | `projects/criptopass-ms-tickets` |
|---|---|
| **Bounded Context** | Tickets (TKT) |
| **Owner funcional** | Ticketing |
| **Owner técnico** | Backend Team |
| **Stack** | Kotlin 2.x + Spring Boot 3.x + JPA + Flyway + PostgreSQL + OpenAPI first |
| **Base de datos** | `tickets_db` (RDS PostgreSQL 16, schema único) |
| **Dependencias infra** | Secrets Manager para clave de firma HMAC de QR |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-ms-tickets` |

| Repositorio | `projects/criptopass-ms-users` |
|---|---|
| **Bounded Context** | Identity/Users (IDY) |
| **Owner funcional** | Identity/Compliance |
| **Owner técnico** | Backend Team |
| **Stack** | Kotlin 2.x + Spring Boot 3.x + JPA + Flyway + PostgreSQL + OpenAPI first |
| **Base de datos** | `users_db` (RDS PostgreSQL 16, schema único) |
| **Dependencias infra** | Cognito Admin API para sincronización |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-ms-users` |

### 2.2 Frontends (Next.js + React, Vercel)

| Repositorio | `projects/criptopass-fr-portal` |
|---|---|
| **Propósito** | Portal de compradores (público + autenticado) |
| **Owner funcional** | Product/UX |
| **Owner técnico** | Frontend Team |
| **Stack** | Next.js 14/15 (React 18/19) + Turborepo monorepo. SSR para catálogo SEO, SPA para flujo autenticado |
| **Dominios** | `criptopass.com` (prod), `staging.criptopass.com`, `dev.criptopass.com` |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-fr-portal` |

| Repositorio | `projects/criptopass-fr-admin` |
|---|---|
| **Propósito** | Panel administrativo multi-rol (SUPER_ADMIN, ORGANIZER, VALIDATOR, SUPPORT) |
| **Owner funcional** | Product/UX |
| **Owner técnico** | Frontend Team |
| **Stack** | Next.js 14/15 (React 18/19) + Turborepo monorepo. SPA con cámara web para validación QR |
| **Dominios** | `admin.criptopass.com` (prod), `admin.staging.criptopass.com` |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-fr-admin` |

| Repositorio | `projects/criptopass-fr-shared` |
|---|---|
| **Propósito** | Paquetes compartidos entre los dos frontends: componentes UI comunes, cliente API tipado, utilidades de auth (Cognito PKCE), tipos TypeScript compartidos |
| **Owner funcional** | Frontend Platform |
| **Owner técnico** | Frontend Team |
| **Stack** | TypeScript, React. Publicado como paquete npm privado `@criptopass/ui` y `@criptopass/api-client` |
| **Consumo** | Ambos `criptopass-fr-portal` y `criptopass-fr-admin` lo referencian como dependencia npm (GitHub Packages o npm private registry) |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-fr-shared` |

#### Decisión sobre Frontends: Dos repos vs Monorepo único

El usuario especificó dos nombres de repositorio: `criptopass-fr-portal` y `criptopass-fr-admin`. Se respeta esta decisión. Evaluación:

| Opción | Ventaja | Desventaja |
|---|---|---|
| **Dos repos Turborepo separados** (elegida) | Despliegue independiente, CI/CD aislado, propiedad clara del equipo, cada uno escala su Turborepo | Código duplicado sin shared packages |
| **Monorepo único `criptopass-fr` con apps/portal y apps/admin** | Código compartido más simple, un solo CI/CD, consistencia de versiones | Acoplamiento de deploys, el admin afecta el CI del portal, no respeta los nombres del usuario |

**Solución para compartir código:** El repositorio `criptopass-fr-shared` publica los paquetes `@criptopass/ui` (componentes React: botones, formularios, layout, tema común) y `@criptopass/api-client` (cliente HTTP tipado generado desde OpenAPI specs de los microservicios). Ambos frontends consumen estos paquetes como dependencia npm.

### 2.3 Lambdas Go (Event-Driven / Scheduled)

| Repositorio | `projects/criptopass-fn-traceability` |
|---|---|
| **Bounded Context** | Traceability (TRC) |
| **Owner funcional** | Blockchain/Traceability |
| **Owner técnico** | Backend Team |
| **Stack** | Go 1.22+ + AWS Lambda + DynamoDB SDK |
| **Trigger** | EventBridge Scheduler (cada 5 min) + SQS (eventos TicketIssued) |
| **Base de datos** | DynamoDB `criptopass-traceability` (batch_id → Merkle metadata) |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-fn-traceability` |

| Repositorio | `projects/criptopass-fn-notifications` |
|---|---|
| **Bounded Context** | Notifications (NTF) |
| **Owner funcional** | Communications |
| **Owner técnico** | Backend Team |
| **Stack** | Go 1.22+ + AWS Lambda + SES SDK |
| **Trigger** | SQS (`criptopass-notifications-queue`) |
| **Base de datos** | Ninguna (stateless) |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-fn-notifications` |

#### Convención de Naming para Lambdas

| Decisión | `criptopass-fn-<funcionalidad>` |
|---|---|
| **Justificación** | Un repositorio por lambda o grupo cohesionado de lambdas del mismo bounded context. Despliegue independiente. Nombres descriptivos de la funcionalidad, no del trigger técnico |
| **Regla** | Si dos lambdas comparten el mismo bounded context, lógica de dominio y misma DB/servicio externo → mismo repo. Si son bounded contexts distintos o usan distintos datastores → repos separados |
| **Ejemplos V1** | `criptopass-fn-traceability` (una lambda con dos triggers: SQS + EventBridge Scheduler), `criptopass-fn-notifications` (una lambda con trigger SQS) |
| **Futuro (V2)** | Si se agrega SMS/push → `criptopass-fn-notifications` se extiende con canales adicionales. Si se agrega tokenización NFT → `criptopass-fn-nft-mint` como nuevo repo |

### 2.4 Infraestructura como Código (IaC)

| Repositorio | `projects/criptopass-infra` |
|---|---|
| **Propósito** | Definición declarativa de toda la infraestructura AWS + recursos Vercel |
| **Owner funcional** | Platform/SRE |
| **Owner técnico** | Infrastructure Team |
| **Stack** | **Terraform** (ver ADR-006) |
| **Estado** | `planned` (greenfield — no iniciado) |
| **Repositorio remoto** | `github.com/criptopass/criptopass-infra` |
| **Módulos esperados** | `vpc`, `ecs`, `rds`, `elasticache`, `lambda`, `eventbridge`, `sqs`, `dynamodb`, `cognito`, `secretsmanager`, `route53`, `waf` |

---

## 3. Matriz de Propiedad y Dependencias

| Repositorio | Bounded Context | Depende de (repos) | Depende de (infra) |
|---|---|---|---|
| `criptopass-ms-catalog` | Catalog | `criptopass-fr-shared` (OpenAPI → api-client types) | PostgreSQL `catalog_db`, S3 `catalog-images`, Cognito (JWT), Redis (cache) |
| `criptopass-ms-orders` | Orders & Payments | `criptopass-ms-catalog` (API de eventos) | PostgreSQL `orders_db`, Cognito (JWT), Redis (cola + TTL), Mercado Pago, EventBridge, Secrets Manager |
| `criptopass-ms-tickets` | Tickets | `criptopass-ms-orders` (API de orden) | PostgreSQL `tickets_db`, Cognito (JWT), EventBridge, Secrets Manager (HMAC key) |
| `criptopass-ms-users` | Identity/Users | `criptopass-fr-shared` (OpenAPI → api-client) | PostgreSQL `users_db`, Cognito (JWT + Admin API) |
| `criptopass-fn-traceability` | Traceability | `criptopass-ms-tickets` (eventos TicketIssued vía SQS) | DynamoDB `traceability`, Polygon RPC, Polygon private key (Secrets Manager) |
| `criptopass-fn-notifications` | Notifications | `criptopass-ms-orders` (eventos vía SQS) | SES, SQS |
| `criptopass-fr-portal` | N/A (Frontend) | `criptopass-fr-shared` (@criptopass/ui, @criptopass/api-client) | Vercel, Cognito Hosted UI |
| `criptopass-fr-admin` | N/A (Frontend) | `criptopass-fr-shared` (@criptopass/ui, @criptopass/api-client) | Vercel, Cognito Hosted UI |
| `criptopass-fr-shared` | N/A (Shared libs) | Ninguno | GitHub Packages (npm registry) |
| `criptopass-infra` | N/A (Infra) | Ninguno (lee specs de arquitectura) | AWS Account, Terraform Cloud |

---

## 4. Flujo de Coordinación del Workspace

### 4.1 Sincronización Ascendente (Proyecto → Workspace)

1. Al finalizar cada incremento en un proyecto (`projects/<project>/`):
   - El `planner` local consolida la spec y los contratos OpenAPI en `projects/<project>/docs/specs/`
   - El `enterprise-architect` ejecuta `graphify --update` en la raíz del workspace para actualizar el grafo de dependencias inter-servicios
   - Se actualiza `docs/specs/workspace_changes.md` si hubo cambios en interfaces públicas

### 4.2 Sincronización Descendente (Workspace → Proyecto)

1. Los agentes locales de planeación de cada proyecto deben revisar al iniciar:
   - `docs/architecture/integration-map.md` (contratos que les aplican)
   - `docs/architecture/context-map.md` (su posición en el ecosistema)
   - `docs/specs/workspace_changes.md` (cambios recientes que les afectan)
   - `graphify-out/GRAPH_REPORT.md` (dependencias de su proyecto)

2. Si un proyecto es afectado por un cambio global, se crea una Delta Spec local para adaptar el código.

### 4.3 Gestión de Deuda Técnica

- **Local:** Cada proyecto mantiene `projects/<project>/docs/specs/technical_debt.md`
- **Global:** El workspace consolida en `docs/specs/technical_debt.md`
- Cualquier agente que introduzca un bypass de diseño o baja cobertura debe registrar la deuda con ID, descripción, impacto, plan de mitigación y estado (`active`)
- Al planificar incrementos, usar `graphify query` para identificar deudas activas que bloqueen la tarea

---

## 5. Notas y Supuestos

- **Turborepo en frontends:** Cada frontend (`criptopass-fr-portal`, `criptopass-fr-admin`) es un monorepo Turborepo independiente. Ambos consumen `@criptopass/ui` y `@criptopass/api-client` desde `criptopass-fr-shared`. No se usa un monorepo único para ambos porque el usuario especificó nombres separados y el despliegue independiente es preferible para availability (el admin nunca debe tumbar el portal público).
- **Convención de ramas:** Cada repositorio sigue GitFlow o trunk-based (a decidir por el equipo). El workspace de arquitectura (`criptopass-workspace`) usa `main` como rama principal con ADRs inmutables (solo se supersede, no se borra).
- **Versionado de APIs:** Los microservicios usan versionado en URL (`/api/v1/`). Cambios breaking requieren nueva versión (`/api/v2/`) y ADR de migración. Esto se refleja en `criptopass-fr-shared` que regenera el api-client tipado desde los OpenAPI specs.
- **`projects/` en .gitignore:** El `.gitignore` raíz del workspace ignora `projects/*` para no versionar subrepositorios dentro del repositorio de arquitectura.
