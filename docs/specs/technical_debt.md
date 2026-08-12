# Technical Debt — CriptoPass Workspace (Consolidado)

| Campo | Valor |
|---|---|
| **Status** | `active` |
| **Proyecto** | CriptoPass (greenfield) |
| **Creado por** | enterprise-architect |
| **Fecha creación** | 2026-08-11 |

---

## Registro Global de Deuda Técnica

> Consolidado de las deudas técnicas de todos los proyectos del workspace. Las deudas locales de cada proyecto se registran en `projects/<project>/docs/specs/technical_debt.md`.

| ID | Proyecto | Descripción | Impacto | Plan de mitigación | Estado | Fecha registro |
|---|---|---|---|---|---|---|

*No hay deuda técnica registrada al inicio del proyecto greenfield. Este archivo se poblará conforme los proyectos avancen en desarrollo.*

---

## Reglas de Registro

1. Cualquier agente que introduzca un bypass de diseño, baja cobertura temporal o parche rápido debe añadir una entrada aquí.
2. Campos requeridos: ID (formato: `TD-{project}-{seq}`), descripción, impacto (bajo/medio/alto), plan de mitigación, plazo estimado, estado (`active`/`resolved`).
3. El `enterprise-architect` consolida periódicamente las deudas locales en este archivo global.
4. Al planificar incrementos, usar `graphify query` para identificar deudas activas que bloqueen o afecten la tarea.
