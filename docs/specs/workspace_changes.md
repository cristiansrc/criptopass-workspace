# Workspace Changes — CriptoPass

| Campo | Valor |
|---|---|
| **Status** | `active` |
| **Proyecto** | CriptoPass (greenfield) |
| **Creado por** | enterprise-architect |
| **Fecha creación** | 2026-08-11 |

---

## Registro de Cambios Arquitectónicos Globales

> Cada entrada documenta un cambio que afecta a los proyectos del workspace. Los agentes locales (`planner`, `requirements-analyst`) deben revisar este archivo al iniciar cada incremento.

| Fecha | ID | Descripción | Proyectos afectados | Acción requerida |
|---|---|---|---|---|
| 2026-08-11 | WSC-001 | Creación inicial del Solution Workspace: landscape, context map, integration map, workspace mapping y 9 ADRs aceptados | Todos (greenfield) | Leer artefactos de `docs/architecture/` antes de iniciar desarrollo |
| 2026-08-11 | WSC-002 | Decisión: Waiting Room como módulo interno de ms-orders (ADR-009). No es bounded context separado | `criptopass-ms-orders` | Implementar `WaitingRoomModule` en ms-orders con Redis |
| 2026-08-11 | WSC-003 | Decisión: Ticket Issuance como API sync en ms-tickets, invocada por Step Functions. No es lambda separada | `criptopass-ms-tickets`, Step Functions | Implementar endpoint `POST /api/v1/tickets/issuance` con idempotencia por `order_id` |
| 2026-08-11 | WSC-004 | Decisión: Terraform como IaC para `criptopass-infra` | `criptopass-infra` (nuevo repo) | Crear módulos Terraform según estructura definida en ADR-006 |
| 2026-08-11 | WSC-005 | Convención de repos: 11 repos totales (4 ms + 2 fr + 1 fr-shared + 2 fn + 1 infra + 1 workspace) | Todos | Crear repos en GitHub con naming `criptopass-{tipo}-{nombre}` |

---

## Preguntas Críticas Abiertas (sin resolver a la fecha)

> Estas preguntas del Requirements Brief (Q-CR-01 a Q-CR-05) están pendientes de respuesta por el usuario. La arquitectura macro actual NO depende de su resolución (se usan los supuestos provisionales S-03 a S-07).

| ID | Pregunta | Supuesto provisional aplicado | Impacto si cambia |
|---|---|---|---|
| Q-CR-01 | ¿Localidades/asientos numerados o entrada general? | S-03: Entrada general con tipos de boleta (general/VIP) | Ampliar modelo de Catalog y Tickets si se requieren asientos numerados |
| Q-CR-02 | ¿Límite máximo de boletas por orden/usuario? | S-04: 6 por orden, 10 por usuario por evento | Ajustar constante de configuración |
| Q-CR-03 | ¿Quién asume el fee por servicio? | S-05: Comprador como fee visible | Ajustar cálculo de monto en checkout |
| Q-CR-04 | ¿Reembolsos self-service, SUPPORT o no aplican? | S-06: Solo vía SUPPORT | Agregar endpoint self-service si cambia |
| Q-CR-05 | ¿Facturación electrónica DIAN en V1? | S-07: Tiquete POS electrónico; factura completa en V2 | Agregar lambda `criptopass-fn-dian-invoicing` y paso en saga si se requiere en V1 |
