# ADR-006: Convención Multi-Repo, Naming de Lambdas y Estrategia IaC + Ambientes

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-006 |
| **Título** | Convención multi-repo, naming de lambdas, elección de Terraform como IaC, y estrategia de ambientes |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass es un sistema con múltiples componentes desplegables independientemente:
- 4 microservicios Kotlin + Spring Boot
- 2 frontends Next.js
- 2+ lambdas Go
- Infraestructura AWS (ECS, RDS, ElastiCache, Lambda, EventBridge, SQS, DynamoDB, Cognito, S3, SES)

El usuario estableció (S-01): multi-repo, naming `criptopass-ms-<funcionalidad>`, `criptopass-fr-portal`, `criptopass-fr-admin`, `criptopass-fn-*`. Pidió propuesta para naming de lambdas y elección de IaC.

## Decisión

### Parte A: Convención Multi-Repo

Cada componente desplegable tiene su propio repositorio Git bajo `projects/`:

```
projects/
├── criptopass-ms-catalog/       # Catalog bounded context
├── criptopass-ms-orders/        # Orders & Payments bounded context
├── criptopass-ms-tickets/       # Tickets bounded context
├── criptopass-ms-users/         # Identity/Users bounded context
├── criptopass-fr-portal/        # Portal Compradores (Next.js, Turborepo)
├── criptopass-fr-admin/         # Panel Admin (Next.js, Turborepo)
├── criptopass-fr-shared/        # Paquetes UI/API compartidos (@criptopass/ui, @criptopass/api-client)
├── criptopass-fn-traceability/  # Lambda blockchain anchor (Go)
├── criptopass-fn-notifications/ # Lambda notificaciones email (Go)
└── criptopass-infra/            # IaC (Terraform)
```

**11 repositorios totales.** Cada uno con su propio ciclo de CI/CD, versionado semántico y propietario claro.

### Parte B: Naming de Lambdas

**Convención:** `criptopass-fn-<funcionalidad-cohesionada>`

| Regla | Descripción |
|---|---|
| **Por funcionalidad, no por trigger** | La lambda se nombra por lo que hace (`notifications`, `traceability`), no por cómo se activa (`sqs-consumer`, `cron-anchor`) |
| **Agrupación por bounded context** | Si múltiples lambdas comparten bounded context, lógica de dominio y datastore → mismo repositorio |
| **Despliegue independiente** | Cada repo `criptopass-fn-*` tiene su propio pipeline de despliegue. Una lambda puede tener múltiples triggers (SQS + EventBridge Scheduler) |
| **Una lambda por repositorio (V1)** | V1 tiene dos lambdas: `fn-notifications` (trigger: SQS) y `fn-traceability` (triggers: SQS + EventBridge Scheduler). Cada una en su repo |

**Ejemplos de naming para futuras lambdas:**

| Propósito | Naming | Justificación |
|---|---|---|
| Facturación electrónica DIAN | `criptopass-fn-dian-invoicing` | Funcionalidad cohesionada (facturación). Trigger: evento `OrderCompleted` |
| Tokenización NFT de boletas | `criptopass-fn-nft-mint` | Funcionalidad cohesionada (mint NFT). Trigger: evento `TicketIssued` o manual |
| Generación de reportes | `criptopass-fn-reporting` | Funcionalidad cohesionada. Trigger: EventBridge Scheduler (diario/semanal) |

### Parte C: Elección de IaC — Terraform sobre AWS CDK

| Criterio | Terraform | AWS CDK | Ganador |
|---|---|---|---|
| **Madurez y comunidad** | ✅ Muy maduro, enorme comunidad, miles de módulos en registry | ⚠️ Más nuevo, comunidad creciendo | Terraform |
| **Multi-cloud potencial** | ✅ AWS + Vercel + otros providers desde un solo tool | ❌ Solo AWS (CDKTF no es maduro) | Terraform |
| **State management** | ✅ Remote state en S3 + DynamoDB lock (estándar) | ⚠️ CloudFormation maneja el state (menos control) | Terraform |
| **Lenguaje** | HCL (declarativo) — curva de aprendizaje baja | TypeScript, Python, etc. — más familiar para devs | CDK |
| **Proveedores externos** | ✅ Vercel provider, PagerDuty, Datadog, etc. | ❌ Solo AWS nativo | Terraform |
| **Plan/Apply predecible** | ✅ `terraform plan` muestra todos los cambios antes de aplicar | ⚠️ `cdk diff` + CloudFormation Change Sets (menos granular) | Terraform |
| **Modularidad** | ✅ Módulos reutilizables entre ambientes y proyectos | ✅ Constructs reutilizables | Empate |

**Decisión final: Terraform** para `criptopass-infra`.

**Estructura de módulos Terraform:**

```
criptopass-infra/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   ├── elasticache/
│   ├── lambda/
│   ├── eventbridge/
│   ├── sqs/
│   ├── dynamodb/
│   ├── cognito/
│   ├── secretsmanager/
│   ├── route53/
│   ├── waf/
│   └── vercel/          # Vercel project + domains via Vercel Terraform Provider
└── README.md
```

**Remote state:** S3 bucket `criptopass-terraform-state` con DynamoDB para locking.

### Parte D: Estrategia de Ambientes

| Ambiente | Cuenta AWS | Propósito | Recursos | Datos |
|---|---|---|---|---|
| **dev** | `criptopass-dev` (o misma cuenta con prefix) | Desarrollo e integración continua | Fargate Spot (mín 1 task), RDS db.t4g.small single-AZ, Redis cache.t4g.micro | Datos sintéticos, Cognito test users |
| **staging** | `criptopass-staging` | Smoke tests, performance, UAT | Fargate Spot/On-demand (2 tasks min), RDS db.t4g.medium multi-AZ, Redis cache.t4g.small | Datos sintéticos realistas |
| **prod** | `criptopass-prod` | Producción | Fargate On-demand (auto-scale 2-10 tasks), RDS db.t4g.large multi-AZ, Redis cache.m6g.large multi-AZ con failover | Datos reales, Cognito prod users |

**Promotion flow:** `dev` → `staging` → `prod`. Cada ambiente tiene su propio `terraform.tfvars` con dimensionamiento y config específica.

**Feature flags:** No se usan en V1 (alcance simple). Si se requieren en V2, se usará AWS AppConfig.

## Alternativas Consideradas

| Aspecto | Alternativa | Evaluación |
|---|---|---|
| **IaC** | AWS CDK | Más familiar para devs (TypeScript), pero limita multi-cloud, Vercel provider es menos maduro. Se sacrifica flexibilidad futura. |
| **IaC** | Pulumi | Similar a CDK pero multi-cloud. Comunidad más pequeña, menos módulos. |
| **IaC** | CloudFormation nativo | Verboso, difícil de mantener para 50+ recursos. No recomendado para proyectos de esta escala. |
| **Lambdas** | Un solo repo `criptopass-fn` con todas las lambdas | Acoplamiento de deploys: cambiar una notificación no debería desplegar el anclaje blockchain. |
| **Lambdas** | Naming por trigger: `criptopass-fn-sqs-notifications` | Si el trigger cambia (SQS → EventBridge directo), el nombre queda obsoleto. Nombrar por funcionalidad es estable. |
| **Ambientes** | Una sola cuenta AWS para todo | Riesgo: un error en dev puede afectar prod. Recomendación AWS Well-Architected: cuentas separadas. |
| **Fronts** | Monorepo único `criptopass-fr` | El usuario pidió dos nombres de repo. Respetado. `criptopass-fr-shared` resuelve la compartición de código. |

## Consecuencias

**Positivas:**
- 11 repos con propiedad y despliegue independiente: un cambio en notificaciones no afecta el catálogo.
- Terraform permite gestionar Vercel (dominios, proyectos) y AWS desde un solo tool.
- Ambientes aislados por cuenta minimizan blast radius.
- Naming de lambdas por funcionalidad es estable ante cambios de trigger.

**Negativas:**
- 11 repositorios requieren disciplina en versionado y coordinación.
- Terraform state en S3 requiere acceso y políticas IAM correctas.
- 3 cuentas AWS (dev/staging/prod) añaden overhead de gestión (mitigado con AWS Organizations).
- `criptopass-fr-shared` introduce un paso extra en el ciclo de desarrollo de frontends (publicar paquete → consumir en portal/admin). Mitigado con npm link en desarrollo local.

**Riesgos mitigados:**
- **R-08 (Facturación DIAN no definida):** Si se agrega en V1, se crea `criptopass-fn-dian-invoicing` sin afectar otros repositorios.
- **Acoplamiento de deploys:** Repositorios independientes con CI/CD independientes (GitHub Actions) evitan deploy acoplados.

---

## Notas

- **Proveedores Terraform utilizados:** `hashicorp/aws` (~5.x), `vercel/vercel` (~2.x), `hashicorp/random`.
- **CI/CD:** GitHub Actions con `terraform plan` en PR y `terraform apply` en merge a main. OIDC para autenticación AWS sin access keys.
- **Secretos:** AWS Secrets Manager para credenciales. Terraform solo referencia los ARN de los secrets (no los valores).
- **Estimación de recursos AWS:** ~50-70 recursos por ambiente (sin contar tasks de ECS que auto-escalan).
