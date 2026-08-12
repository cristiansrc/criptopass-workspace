# ADR-004: Amazon Cognito como Identity Provider (OAuth2/OIDC)

| Campo | Valor |
|---|---|
| **ADR ID** | ADR-004 |
| **Título** | Amazon Cognito User Pools como IdP para AuthN/AuthZ — OAuth2/OIDC con roles como grupos, lock-in aceptado |
| **Estado** | `accepted` |
| **Fecha** | 2026-08-11 |
| **Owner** | enterprise-architect |
| **Reemplaza** | N/A (primera decisión) |

---

## Contexto

CriptoPass requiere autenticación y autorización para:
- **Catálogo público:** sin auth (lectura anónima).
- **Portal de compradores:** auth requerida solo al iniciar compra (F-PC-03).
- **Panel admin multi-rol:** 4 roles con permisos diferenciados (SUPER_ADMIN, ORGANIZER, VALIDATOR, SUPPORT).
- **Aislamiento de datos:** ORGANIZER solo accede a sus eventos (R-06).
- **Service-to-service:** microservicios validando identidad en cada request.

El usuario estableció (S-01): Amazon Cognito V1 (OAuth2/OIDC), roles como grupos/atributos.

## Decisión

**Amazon Cognito User Pools como único Identity Provider.**

### Configuración

| Aspecto | Decisión |
|---|---|
| **User Pool** | Un User Pool por ambiente (dev, staging, prod). Nombres: `criptopass-users-{env}` |
| **Flujo OAuth2/OIDC** | Authorization Code Grant + PKCE para frontends (Next.js). Client credentials (o API key interna) para service-to-service |
| **Hosted UI** | Cognito Hosted UI para login/registro. Customizable con branding CriptoPass |
| **Roles** | Grupos de Cognito: `SUPER_ADMIN`, `ORGANIZER`, `VALIDATOR`, `SUPPORT`, `BUYER`. Un usuario puede pertenecer a un solo grupo en V1 (simplificación) |
| **JWT Claims** | `sub` (UUID del usuario), `email`, `cognito:groups` (array de grupos), `iss`, `aud`, `exp`, `token_use` |
| **Atributos custom** | `custom:document_id` (documento de identidad, condicional a Q-NC-04), `custom:phone`, `custom:consent_1581` (timestamp de aceptación) |
| **Client IDs** | `criptopass-portal` (público, PKCE), `criptopass-admin` (público, PKCE), `criptopass-backend` (confidencial, para service-to-service si se requiere) |

### Validación JWT en Resource Servers

Cada microservicio (ms-catalog, ms-orders, ms-tickets, ms-users) actúa como **OAuth2 Resource Server**:
- Configuración Spring Security: `spring.security.oauth2.resourceserver.jwt.issuer-uri = https://cognito-idp.{region}.amazonaws.com/{userPoolId}`
- Validación automática de firma JWT contra JWKS endpoint (`/.well-known/jwks.json`).
- Claims validados: `iss`, `aud`, `exp`, `token_use=access`.
- Roles extraídos de `cognito:groups` y mapeados a `GrantedAuthority` (Spring Security).
- `@PreAuthorize("hasRole('ORGANIZER')")` para endpoints de admin.

### Sincronización con ms-users
- Cognito es fuente de verdad para: credenciales, email verificado, `sub`, grupos/roles.
- ms-users es fuente de verdad para: perfil extendido (documento de identidad, teléfono, consentimiento), metadatos de cuenta.
- ms-users consulta Cognito Admin API (`AdminGetUser`, `AdminUpdateUserAttributes`) para sincronizar atributos custom.
- Post-registro (Cognito trigger o evento), ms-users crea el registro de perfil interno.

### Lock-in aceptado
Amazon Cognito es un servicio managed de AWS. El lock-in se acepta por:
- Integración nativa con ALB (puede validar JWT en el edge).
- Sin servidores que mantener (a diferencia de Keycloak self-hosted).
- Pricing por MAU (monthly active users) que escala con el negocio.
- Si en el futuro se migra a otro IdP (Auth0, Keycloak), los microservicios solo cambian el `issuer-uri`. El modelo de roles/grupos es portable.

## Alternativas Consideradas

| Alternativa | Evaluación |
|---|---|
| **Keycloak self-hosted en ECS** | Mayor control, open source, sin lock-in. Pero requiere gestión de infraestructura, backups, alta disponibilidad. Complejidad adicional para un MVP. El usuario decidió Cognito (S-01). |
| **Auth0** | Excelente DX, pero más caro que Cognito a escala. Fuera de AWS introduce latencia adicional. |
| **Firebase Auth** | Fuera de AWS. No alineado con el stack. |
| **Cognito + API Gateway authorizer** | El ALB V1 ya hace routing; si en V2 se usa API Gateway, se puede agregar un Cognito Authorizer para validar JWT en el edge sin llegar al microservicio. |

## Consecuencias

**Positivas:**
- Fully managed: sin gestión de infraestructura de auth.
- Integración nativa con ALB (puede validar JWT en el edge si se desea).
- Pricing predecible: primeros 50,000 MAU gratis.
- Roles como grupos es simple y portable.
- Cumplimiento Ley 1581: atributos custom para consentimiento; ms-users gestiona los derechos ARSO.

**Negativas:**
- Lock-in con AWS (mitigado: migrar es cambiar issuer-uri y atributos).
- Cognito Hosted UI tiene opciones limitadas de personalización visual comparado con soluciones custom.
- La API de administración de Cognito tiene rate limits que pueden afectar operaciones batch de usuarios.
- Si se requieren flujos complejos (ej. MFA por SMS, social login avanzado), Cognito lo soporta pero con configuración adicional.

**Riesgos mitigados:**
- **R-06 (Aislamiento ORGANIZER):** Los microservicios extraen el `sub` del JWT y el grupo `ORGANIZER`. ms-catalog filtra eventos por `organizer_id = sub`. Un ORGANIZER no puede modificar el JWT para escalar privilegios (firmado por Cognito).
- **EC-13 (Pérdida de cuenta):** Cognito maneja recuperación de cuenta nativamente (email verification, forgot password).

---

## Notas

- Cognito User Pools genera dos tokens: Access Token (para autorización) e ID Token (para identidad). Los resource servers deben validar el **Access Token**, no el ID Token. Spring Security OAuth2 Resource Server valida el access token por defecto si se configura `issuer-uri`.
- Las lambdas Go **no validan JWT** (no reciben requests HTTP directos). Consumen eventos de SQS que ya fueron autenticados en el microservicio que emitió el evento.
- La rotación de refresh tokens ocurre en el frontend vía Cognito SDK (amplify-js o similar).
- **Plan de migración futura** (si se decide salir de Cognito): exportar usuarios vía CSV/API, recrear en nuevo IdP, cambiar `issuer-uri` en resource servers. Tiempo estimado: 1-2 sprints.
