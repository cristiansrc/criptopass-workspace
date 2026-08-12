# Requirements Brief — CriptoPass V1

| Campo | Valor |
|---|---|
| **Status** | `ready-for-planner` |
| **Proyecto** | CriptoPass (greenfield) |
| **Incremento** | criptopass-v1 |
| **Mercado inicial** | Colombia (visión LATAM futura) |
| **Autor** | requirements-analyst |
| **Fecha** | 2026-08-11 |
| **Handoff a** | planner (SDD) |

---

## 1. Objetivo

Construir la primera versión (V1) de **CriptoPass**, un portal de venta de boletas para eventos en línea cuyo diferenciador es la **trazabilidad criptográfica de cada boleta anclada periódicamente en blockchain pública Polygon (L2)** mediante raíces Merkle por lote. El sistema debe permitir a organizadores publicar eventos, a compradores adquirir boletas con reserva temporal de inventario y pago en Mercado Pago, y a validadores redimir boletas en puerta mediante un QR dinámico rotativo anti-duplicación.

El objetivo funcional es entregar un MVP end-to-end operable en Colombia, con cumplimiento de habeas data (Ley 1581 de 2012) y delegación del scope PCI a la pasarela de pago.

---

## 2. Contexto

- **Proyecto greenfield**: el workspace está vacío; no existe sistema previo ni datos a migrar.
- **Dos aplicaciones frontend**:
  - Portal de compradores (público, SEO-indexable).
  - Panel administrativo (privado, multi-rol).
- **Diferenciador de producto**: trazabilidad blockchain pública verificable (Polygon L2, costo mínimo, verificación tipo PolygonScan).
- **Marco técnico ya decidido por el usuario** (registrado como supuestos, no rediseñado):
  - Cloud AWS; fronts en Vercel.
  - Backend: microservicios Kotlin + Spring Boot + JPA (Flyway gestiona esquema, JPA solo valida) con arquitectura hexagonal y OpenAPI first; Lambdas en Go para flujos asíncronos/event-driven; Step Functions para la saga de compra.
  - BD PostgreSQL; imágenes de eventos en S3.
  - CQRS-lite: modelos de lectura (catálogo público cacheable) y escritura (órdenes/pagos transaccional) sobre la misma BD, sin event sourcing.
  - AuthN/AuthZ: Amazon Cognito V1 (OAuth2/OIDC); roles como grupos/atributos.
  - Pagos con patrón Strategy (Mercado Pago primer adaptador).
  - Multi-repo: `criptopass-ms-<funcionalidad>`, `criptopass-fr-portal`, `criptopass-fr-admin`, `criptopass-fn-*`; workspace raíz `criptopass-workspace`.
  - Fronts en monorepo Turborepo (sin microfrontends), partiendo de una plantilla HTML existente del usuario migrada a React/Next.js.
- **No existe Master Spec previa**; este brief es el primer artefacto SDD del proyecto.

---

## 3. Actores y Permisos

### 3.1 Actores del Portal de Compradores

| Actor | Descripción | Permisos |
|---|---|---|
| **Visitante anónimo** | Cualquier persona en internet | Navegar catálogo, ver detalle de evento, buscar/filtrar. No puede comprar. |
| **Comprador registrado** | Usuario autenticado vía Cognito | Todo lo del visitante + iniciar compra, pagar, ver "Mis boletas", abrir QR dinámico, gestionar perfil. |
| **Comprador en sala de espera** | Visitante/registrado en cola de venta pico | Esperar turno; al salir de la fila puede iniciar compra si hay inventario. |

### 3.2 Actores del Panel Administrativo

| Rol | Permisos |
|---|---|
| **SUPER_ADMIN** | Acceso total: gestión de usuarios, asignación de roles, gestión de organizadores, todos los eventos, todas las órdenes, configuración global. |
| **ORGANIZER** | Crear y gestionar **solo SUS eventos** (incluye carga de imágenes a S3), ver ventas de sus eventos, definir capacidad/precios/fechas de sus eventos. No ve eventos de otros organizadores. |
| **VALIDATOR** | Únicamente validación/escaneo de QR en puerta (cámara web desde el admin web). No accede a ventas ni configuración. |
| **SUPPORT** | Consulta de órdenes, reenvío de boletas, soporte a compradores. No edita eventos ni usuarios. |

### 3.3 Reglas de permisos funcionales

- Un ORGANIZER **no puede ver ni editar** eventos ajenos.
- VALIDATOR **no puede emitir ni anular** boletas; solo marca uso.
- SUPPORT **no puede reembolsar** (el reembolso es flujo de pago, no de soporte directo — ver pregunta abierta crítica).
- SUPER_ADMIN es el único rol que puede crear/eliminar ORGANIZER y asignar roles.
- La redención de una boleta solo puede ejecutarla VALIDATOR o SUPER_ADMIN.

---

## 4. Alcance (Scope)

### 4.1 Portal de Compradores

| ID | Funcionalidad |
|---|---|
| F-PC-01 | Catálogo público de eventos (listado, detalle, búsqueda, filtros) accesible sin login. |
| F-PC-02 | SEO: páginas de evento indexables por buscadores (contenido renderizado server-side). |
| F-PC-03 | Registro/login (Cognito) requerido **solo al iniciar compra**. |
| F-PC-04 | Sala de espera virtual para ventas pico (cola FIFO con control de admisión). |
| F-PC-05 | Reserva de inventario con TTL (expira si no se completa el pago). |
| F-PC-06 | Flujo de compra con pago vía Mercado Pago (sin almacenamiento de datos de tarjeta). |
| F-PC-07 | Sección "Mis boletas": boletas futuras y pasadas. |
| F-PC-08 | QR dinámico rotativo (15-30s) con firma criptográfica anti-duplicación. |
| F-PC-09 | Boleta redimida: el QR no se vuelve a mostrar; la boleta queda marcada como usada. |
| F-PC-10 | Módulo de perfil de usuario (datos personales). |
| F-PC-11 | Compra de múltiples boletas en una sola orden. |

### 4.2 Panel Administrativo

| ID | Funcionalidad |
|---|---|
| F-PA-01 | Autenticación del admin vía Cognito con roles. |
| F-PA-02 | Gestión de usuarios y roles (SUPER_ADMIN). |
| F-PA-03 | CRUD de eventos por ORGANIZER (solo los propios), con carga de imágenes a S3. |
| F-PA-04 | Definición de capacidad, precios y fechas por evento. |
| F-PA-05 | Visualización de ventas por evento (ORGANIZER ve solo las suyas). |
| F-PA-06 | Módulo de validación de QR con cámara web (VALIDATOR). |
| F-PA-07 | Consulta de órdenes y reenvío de boletas (SUPPORT). |

### 4.3 Trazabilidad Blockchain

| ID | Funcionalidad |
|---|---|
| F-TB-01 | Generación de hash criptográfico por boleta emitida. |
| F-TB-02 | Anclaje periódico en Polygon (L2) de la raíz Merkle del lote de boletas emitidas. |
| F-TB-03 | Verificación pública de emisión legítima de una boleta (tipo PolygonScan / explorador). |
| F-TB-04 | Diseño funcional **no bloqueante** para futura tokenización NFT / reventa (V1 no la implementa). |

### 4.4 Cumplimiento

| ID | Funcionalidad |
|---|---|
| F-CO-01 | Consentimiento de tratamiento de datos personales (Ley 1581 de 2012) al registrarse. |
| F-CO-02 | Política de privacidad y derechos del titular (acceso, rectificación, supresión). |
| F-CO-03 | No almacenamiento de PAN/CVV (scope PCI delegado a Mercado Pago). |
| F-CO-04 | Facturación electrónica DIAN — integración esperada (mecanismo exacto en pregunta abierta). |

---

## 5. No Objetivos (Out of Scope, V1)

- **Reventa ni transferencia** de boletas entre usuarios.
- **Tokenización NFT** de boletas (solo diseño no bloqueante para el futuro).
- **App móvil nativa** de escaneo (la validación se hace desde el admin web con cámara).
- **Multi-moneda** (V1 opera en COP — ver pregunta abierta no crítica).
- **Microfrontends** (se usa monorepo Turrerepo de fronts).
- **Event sourcing** (CQRS-lite sin ES).
- **Pasarelas de pago distintas a Mercado Pago** (el patrón Strategy queda listo para extender, pero V1 solo implementa Mercado Pago).
- **Notificaciones SMS/push** si la decisión final es solo email (ver pregunta abierta no crítica).
- **Dashboard analítico avanzado** de organizadores (V1 solo muestra ventas básicas).
- **Soporte offline** de validación en puerta.
- **Personalización de layout de asientos** (ver pregunta crítica sobre localidades).

---

## 6. Flujos de Usuario

### F-UF-01 — Navegación anónima del catálogo
- **Actor**: Visitante anónimo.
- **Disparador**: El visitante entra al portal.
- **Flujo principal**: Ve listado de eventos → busca/filtra → entra al detalle de un evento.
- **Resultado esperado**: El visitante puede consultar eventos sin autenticarse; las páginas son indexables (SEO).
- **Flujos alternos**: Evento sin fechas futuras → se muestra como "agotado" o "pasado".

### F-UF-02 — Compra estándar (sin sala de espera)
- **Actor**: Comprador registrado.
- **Disparador**: El comprador selecciona "Comprar" en un evento con inventario disponible.
- **Flujo principal**:
  1. Si no está autenticado, se le pide login/registro (Cognito).
  2. Selecciona cantidad de boletas (dentro del límite — ver pregunta crítica).
  3. El sistema **reserva inventario con TTL**.
  4. Ingresa datos de pago en Mercado Pago (tokenización del lado del proveedor).
  5. Mercado Pago confirma pago → se emiten boletas con hash criptográfico.
  6. Las boletas quedan visibles en "Mis boletas".
- **Resultado esperado**: Comprador recibe N boletas válidas y trazables.
- **Flujos alternos**:
  - Pago rechazado → la reserva se libera al expirar el TTL o al confirmarse el rechazo.
  - TTL expira en pasarela → ver Edge Case EC-03.
  - Pago aprobado pero emisión falla → compensación/reembolso (EC-01).

### F-UF-03 — Compra en venta pico (con sala de espera)
- **Actor**: Visitante/Comprador.
- **Disparador**: Evento marcado como venta pico o con concurrencia sobre el inventario.
- **Flujo principal**:
  1. El usuario entra a la sala de espera virtual (cola FIFO).
  2. Espera su turno (posición visible estimada).
  3. Al salir de la fila, si hay inventario, continúa al flujo F-UF-02.
- **Resultado esperado**: Admisión controlada para evitar saturación.
- **Flujos alternos**:
  - Se acaba el inventario mientras está en fila → se le notifica y se le ofrece salir o esperar por liberaciones (EC-08).

### F-UF-04 — Visualización y uso de boleta
- **Actor**: Comprador registrado.
- **Disparador**: El comprador abre "Mis boletas" → boleta futura.
- **Flujo principal**:
  1. Se muestra el QR dinámico rotativo (15-30s) con firma criptográfica.
  2. El validador escanea en puerta → el sistema verifica firma + vigencia temporal + no-uso previo.
  3. La boleta se marca como usada → el QR deja de mostrarse para siempre.
- **Resultado esperado**: Boleta redimida una sola vez, anti-duplicación garantizada.
- **Flujos alternos**:
  - QR presentado fuera de su ventana de rotación → rechazo (EC-05).
  - Misma boleta presentada dos veces → segundo intento rechazado (EC-04).
  - Boleta ya redimida → no se muestra QR (EC-04).

### F-UF-05 — Creación de evento (ORGANIZER)
- **Actor**: ORGANIZER.
- **Disparador**: Quiere publicar un nuevo evento.
- **Flujo principal**:
  1. Crea evento con datos (nombre, fechas, lugar, descripción, capacidad, precios).
  2. Sube imágenes del evento (almacenadas en S3).
  3. Publica el evento → aparece en el catálogo público.
- **Resultado esperado**: Evento visible y comprable.
- **Flujos alternos**: SUPER_ADMIN puede auditar/desactivar cualquier evento.

### F-UF-06 — Validación en puerta (VALIDATOR)
- **Actor**: VALIDATOR.
- **Disparador**: Asistente llega al acceso del evento.
- **Flujo principal**:
  1. VALIDATOR abre el módulo de validación en el admin web.
  2. Activa la cámara del dispositivo (PC/celular).
  3. Escanea el QR del asistente.
  4. El sistema valida firma criptográfica, ventana temporal y estado de uso.
  5. Muestra resultado (válido/inválido/ya usado) y marca la boleta como usada si corresponde.
- **Resultado esperado**: Acceso autorizado o rechazado con motivo claro.
- **Flujos alternos**: Sin cámara disponible → entrada manual de código (si aplica — ver supuestos).

### F-UF-07 — Soporte a comprador (SUPPORT)
- **Actor**: SUPPORT.
- **Disparador**: Un comprador reporta no recibir sus boletas.
- **Flujo principal**:
  1. SUPPORT consulta la orden del comprador.
  2. Verifica estado de pago y emisión.
  3. Reenvía las boletas al correo del comprador.
- **Resultado esperado**: Comprador recibe boletas sin exponer datos sensibles.
- **Flujos alternos**: Orden con pago incompleto → no se reenvían boletas.

### F-UF-08 — Anclaje blockchain periódico
- **Actor**: Sistema (proceso asíncrono).
- **Disparador**: Cron/lote periódico (frecuencia a definir por Planner).
- **Flujo principal**:
  1. Se agrupan las boletas emitidas en el período.
  2. Se construye el árbol Merkle y se calcula la raíz.
  3. Se ancla la raíz en Polygon (L2).
  4. Se registra el hash de transacción para verificación pública.
- **Resultado esperado**: Trazabilidad pública verificable de cada boleta del lote.

---

## 7. Entidades Funcionales

> Datos de negocio, **no esquema de BD**. Planner definirá persistencia.

- **Evento**: nombre, descripción, fechas (inicio/fin), lugar, capacidad total, configuración de localidades (ver pregunta crítica), precios, imágenes, organizador dueño, estado (borrador/publicado/agotado/cancelado/reprogramado), flag de venta pico.
- **Localidad/Zona** (condicional a pregunta crítica): nombre, capacidad propia, precio propio, asientos numerados (sí/no).
- **Boleta**: código único, hash criptográfico, evento, localidad/tipo, comprador, orden de origen, estado (emitida/usada/anulada), timestamp de uso, timestamp de anclaje Merkle, prueba de inclusión en lote.
- **Orden**: comprador, evento, lista de boletas, estado (creada/reservada/pagada/cancelada/reembolsada/fallida), TTL de reserva, referencia de pago, monto total, fee por servicio (si aplica), timestamp de creación/pago.
- **Pago**: referencia externa Mercado Pago, estado, monto, moneda, método (tokenizado), idempotencia.
- **Usuario**: datos personales (ver pregunta no crítica sobre mínimos), rol(es), consentimiento habeas data, timestamps.
- **Organizador**: datos de organización, eventos propios.
- **Lote Merkle**: raíz, lista de hashes incluidos, tx hash Polygon, timestamp, estado (anclado/pendiente/fallido).
- **Sala de espera**: evento asociado, cola de usuarios, posición estimada, estado de admisión.
- **Auditoría**: actor, acción, recurso, timestamp (para acciones sensibles: emisión, redención, reembolso, reenvío, cambios de rol).

---

## 8. Integraciones

| Sistema | Propósito | Dirección | Criticidad |
|---|---|---|---|
| **Amazon Cognito** | AuthN/AuthZ OAuth2/OIDC, gestión de usuarios y roles | Bidireccional | Crítica — sin auth no opera el sistema. |
| **Mercado Pago** | Procesamiento de pagos y tokenización de tarjeta | Bidireccional (checkout + webhooks) | Crítica — sin pago no hay venta. |
| **Polygon (L2)** | Anclaje de raíces Merkle para trazabilidad pública | Salida (escritura on-chain) + lectura pública | Alta — diferenciador de producto. |
| **PolygonScan / explorador** | Verificación pública de trazabilidad | Lectura | Media — UX de verificación. |
| **AWS S3** | Almacenamiento de imágenes de eventos | Salida | Alta — sin imágenes no se publican eventos. |
| **DIAN (facturación electrónica)** | Emisión de factura/tiquete electrónico | Salida | **Pendiente de definición** — ver pregunta crítica. |
| **Servicio de email** | Confirmaciones de compra, reenvío de boletas, notificaciones | Salida | Alta — experiencia del comprador. |
| **SMS/Push (opcional)** | Notificaciones alternas | Salida | Baja — ver pregunta no crítica. |

---

## 9. Seguridad y Restricciones

### 9.1 Acceso
- Toda acción de compra, perfil, admin y validación requiere autenticación Cognito.
- Autorización por rol (SUPER_ADMIN, ORGANIZER, VALIDATOR, SUPPORT, comprador).
- ORGANIZER solo accede a sus eventos (aislamiento por dueño).
- Catálogo público: lectura sin auth, sin exponer datos personales de organizadores ni compradores.

### 9.2 Datos sensibles
- **No almacenar PAN/CVV ni datos de tarjeta**: scope PCI delegado a Mercado Pago (tokenización).
- Datos personales del comprador: mínimo necesario, con consentimiento Ley 1581.
- Boleta: el hash criptográfico es la prueba de emisión; no debe filtrar datos personales del comprador.
- Logs de auditoría sin datos sensibles en claro.

### 9.3 Anti-abuso / Anti-fraude
- QR dinámico rotativo (15-30s) con firma criptográfica para mitigar screenshot.
- Redención idempotente: una boleta se usa una sola vez.
- Reserva de inventario con TTL para evitar acaparamiento sin pago.
- Sala de espera para prevenir saturación y bots en ventas pico.
- Idempotencia en webhooks de Mercado Pago (EC-02).
- Rate limiting funcional en catálogo y endpoints de compra.

### 9.4 Auditoría
- Toda emisión, redención, reembolso, reenvío y cambio de rol debe quedar registrado con actor, timestamp y recurso.

### 9.5 Cumplimiento
- **Ley 1581 de 2012**: consentimiento explícito al registro, política de privacidad visible, ejercicio de derechos ARSO (acceso, rectificación, supresión, oposición).
- **DIAN**: mecanismo de facturación electrónica por definir (pregunta crítica).
- **PCI**: delegación contractual a Mercado Pago; CriptoPass no toca datos de tarjeta.

---

## 10. Edge Cases

| ID | Escenario | Comportamiento esperado |
|---|---|---|
| EC-01 | Pago aprobado en Mercado Pago pero la emisión de boleta falla | La saga de compra debe compensar: marcar orden como fallida, iniciar reembolso en Mercado Pago, notificar al comprador y a SUPPORT, registrar auditoría. La boleta no se considera emitida. |
| EC-02 | Webhook de pago duplicado o fuera de orden | Idempotencia por referencia de pago: el segundo webhook no duplica emisión ni estado. Procesamiento tolerante a desorden. |
| EC-03 | TTL de reserva expira mientras el usuario está en la pasarela de pago | La reserva se libera; si el pago luego se aprueba, el sistema debe detectar la falta de reserva y reembolsar automáticamente (compensación). El usuario debe ver mensaje claro. |
| EC-04 | Intento de doble uso de la misma boleta en puerta | Primer escaneo válido → marca usada. Segundo escaneo → rechazo con motivo "boleta ya usada". El QR ya no se muestra al comprador. |
| EC-05 | Screenshot de QR presentado fuera de ventana de rotación | El validador rechaza con motivo "código expirado"; el comprador debe refrescar el QR (que sigue rotando en su app). |
| EC-06 | Compra de múltiples boletas en una orden | Una sola orden genera N boletas con hashes independientes; cada una es redimible por separado en puerta. |
| EC-07 | Venta pico masiva — dos usuarios por la última boleta | La reserva con TTL + control de concurrencia debe garantizar que solo uno la obtenga; el otro recibe "agotado" y se libera para el siguiente en fila. |
| EC-08 | Sala de espera: se acaba el inventario estando en fila | Se notifica al usuario; se le ofrece salir o permanecer por posibles liberaciones de reservas expiradas. |
| EC-09 | Evento cancelado | Las boletas vendidas deben anularse; se debe definir política de reembolso (ver pregunta crítica sobre reembolsos). Las boletas no son redimibles. |
| EC-10 | Evento reprogramado | Las boletas siguen válidas para la nueva fecha; se notifica a los compradores. No se reemiten nuevos hashes (la boleta original sigue siendo válida). |
| EC-11 | Organizador sube imagen inválida/corrupta | Se rechaza la carga con mensaje claro; el evento puede publicarse sin imagen o con imagen placeholder. |
| EC-12 | Validador sin cámara disponible | Se debe ofrecer entrada manual del código (supuesto — ver Supuestos). |
| EC-13 | Comprador pierde acceso a su cuenta Cognito | Flujo de recuperación de cuenta vía Cognito; al recuperar, sus boletas siguen asociadas a su identidad. |
| EC-14 | Anclaje Merkle falla en Polygon | Reintento con backoff; el lote queda en estado "pendiente" y se reancla en el siguiente ciclo. Las boletas siguen siendo válidas internamente; la trazabilidad pública se actualiza al anclar. |
| EC-15 | Reenvío de boleta por SUPPORT a correo equivocado | El reenvío se registra en auditoría; el comprador legítimo puede solicitar reenvío correcto. No se invalida la boleta. |

---

## 11. Criterios de Aceptación

> Cada criterio es verificable por test funcional o inspección de usuario.

### Portal de Compradores

- **CA-PC-01**: Un visitante anónimo puede ver el listado de eventos, abrir el detalle de uno y aplicar filtros sin que se le solicite login.
- **CA-PC-02**: Las páginas de detalle de evento son rastreables por buscadores (contenido server-side renderizado; meta tags presentes).
- **CA-PC-03**: Al iniciar una compra sin sesión, el sistema redirige a login/registro; tras autenticarse, el flujo de compra continúa.
- **CA-PC-04**: Al seleccionar boletas, el sistema crea una reserva con TTL visible; si el pago no se completa dentro del TTL, la reserva se libera y el inventario vuelve a estar disponible.
- **CA-PC-05**: El flujo de pago redirige a Mercado Pago; al aprobarse, las boletas se emiten y aparecen en "Mis boletas" en menos de 30 segundos (umbral funcional).
- **CA-PC-06**: En "Mis boletas" se distinguen boletas futuras y pasadas; las pasadas no muestran QR activo.
- **CA-PC-07**: El QR de una boleta futura rota cada 15-30 segundos y cada rotación incluye una firma criptográfica verificable.
- **CA-PC-08**: Tras la redención de una boleta, el QR deja de mostrarse permanentemente y la boleta aparece como "usada".
- **CA-PC-09**: El comprador puede editar los datos personales de su perfil dentro de los campos habilitados.
- **CA-PC-10**: Una orden puede contener múltiples boletas y cada una se emite con hash independiente.

### Sala de espera

- **CA-SE-01**: En un evento con sala de espera activa, el usuario entra a una cola FIFO y ve su posición estimada.
- **CA-SE-02**: Solo cuando el usuario sale de la fila y hay inventario, puede iniciar el flujo de compra.
- **CA-SE-03**: Si el inventario se agota mientras el usuario está en fila, recibe notificación y opción de salir o esperar.

### Panel Administrativo

- **CA-PA-01**: SUPER_ADMIN puede listar usuarios, asignar roles y crear/eliminar ORGANIZER.
- **CA-PA-02**: ORGANIZER puede crear un evento, subir imágenes y publicarlo; el evento aparece en el catálogo público.
- **CA-PA-03**: ORGANIZER no puede ver ni editar eventos creados por otro ORGANIZER (aislamiento verificado).
- **CA-PA-04**: ORGANIZER puede ver las ventas de sus propios eventos, no las de otros.
- **CA-PA-05**: VALIDATOR puede abrir el módulo de validación, activar la cámara y escanear un QR; el sistema responde válido/inválido/ya usado.
- **CA-PA-06**: SUPPORT puede consultar una orden por referencia o comprador y reenviar boletas; la acción queda en auditoría.
- **CA-PA-07**: VALIDATOR no puede acceder a ventas, eventos ni gestión de usuarios.
- **CA-PA-08**: SUPPORT no puede editar eventos ni asignar roles.

### Trazabilidad Blockchain

- **CA-TB-01**: Cada boleta emitida tiene un hash criptográfico único y verificable.
- **CA-TB-02**: Periódicamente se ancla en Polygon (L2) la raíz Merkle del lote de boletas emitidas, con tx hash registrado.
- **CA-TB-03**: Es posible verificar públicamente (sin login) que una boleta pertenece a un lote anclado en Polygon.
- **CA-TB-04**: El diseño funcional no impide una futura evolución a tokenización NFT/reventa (validado por revisión de modelo).

### Cumplimiento

- **CA-CO-01**: Al registrarse, el usuario debe aceptar explícitamente el consentimiento de tratamiento de datos (Ley 1581).
- **CA-CO-02**: Existe una política de privacidad accesible públicamente con los derechos del titular.
- **CA-CO-03**: No se almacenan PAN/CVV en ningún punto del sistema (verificado por revisión de datos persistidos).
- **CA-CO-04**: La integración de facturación electrónica DIAN está documentada como integración esperada con su criticidad y mecanismo (resuelto por pregunta crítica).

### Edge Cases

- **CA-EC-01**: Ante un pago aprobado con emisión fallida, la orden queda "fallida", se inicia reembolso y se notifica a comprador y SUPPORT.
- **CA-EC-02**: Un webhook duplicado de Mercado Pago no genera doble emisión de boletas.
- **CA-EC-03**: Si el TTL expira en pasarela y luego se aprueba el pago, el sistema reembolsa automáticamente y notifica al comprador.
- **CA-EC-04**: Una boleta redimida no puede ser redimida nuevamente; el segundo intento se rechaza con motivo explícito.
- **CA-EC-05**: Un QR presentado fuera de su ventana de rotación se rechaza como "expirado".
- **CA-EC-07**: Ante concurrencia sobre la última boleta, solo un comprador la obtiene; el otro recibe "agotado".
- **CA-EC-09**: Ante cancelación de evento, las boletas se anulan y se aplica la política de reembolso definida.
- **CA-EC-14**: Ante fallo de anclaje en Polygon, el lote se reintenta en el siguiente ciclo sin invalidar las boletas.

---

## 12. Preguntas Abiertas

### 12.1 Críticas (bloquean decisiones de Planner si no se resuelven; el Master Orchestrator debe elevarlas al usuario)

| ID | Pregunta | Impacto |
|---|---|---|
| Q-CR-01 | ¿Los eventos tienen entradas por **zonas/localidades con precios distintos**, **asientos numerados**, o solo **entrada general**? | Afecta todo el modelo de compra, la entidad Boleta, el flujo de selección y la UI. |
| Q-CR-02 | ¿Cuál es el **límite máximo de boletas por orden y por usuario por evento**? | Afecta control de inventario, anti-acaparamiento y UX de compra. |
| Q-CR-03 | ¿Quién asume el **cargo por servicio (fee)**: el comprador (fee visible) o el organizador (deducido del pago)? | Afecta cálculo de monto, modelo de orden y liquidación al organizador. |
| Q-CR-04 | ¿Los **reembolsos** en V1 son self-service, solo vía SUPPORT, o no aplican? | Afecta el flujo de compensación, permisos y los edge cases EC-01/EC-03/EC-09. |
| Q-CR-05 | ¿**Facturación electrónica DIAN** en V1 (factura electrónica vs tiquete POS electrónico) o se pospone a V2? | Afecta integraciones, datos del comprador (¿CC obligatoria?) y cumplimiento. |

### 12.2 No críticas (no bloquean Planner; se registran para resolución posterior)

| ID | Pregunta |
|---|---|
| Q-NC-01 | ¿Los ORGANIZER son auto-registro o los crea exclusivamente SUPER_ADMIN? |
| Q-NC-02 | ¿V1 opera solo en COP o se contempla multi-moneda futura? |
| Q-NC-03 | ¿Notificaciones en V1: solo email, o también SMS/push? |
| Q-NC-04 | ¿Datos mínimos del perfil: nombre, documento de identidad (¿CC obligatoria para facturación?), teléfono, fecha de nacimiento? |
| Q-NC-05 | ¿Frecuencia esperada del anclaje Merkle en Polygon (tiempo real, cada N minutos, cada N boletas)? |
| Q-NC-06 | ¿Política de reembolso por evento cancelado: total, parcial, decisión del organizador? |

---

## 13. Supuestos

> Supuestos explícitos que Planner debe validar o descartar. Si alguno resulta falso, el brief debe revisarse.

- **S-01**: Las decisiones de marco técnico (AWS, Kotlin+Spring Boot, Lambdas Go, Step Functions, PostgreSQL, S3, Cognito, Mercado Pago, Polygon L2, multi-repo, Turrerepo) son **fijas para V1** y no se rediseñan.
- **S-02**: La plantilla HTML existente del usuario se migrará a React/Next.js; el diseño visual no se especifica en este brief (es supuesto de UI).
- **S-03**: **Localidades/asientos numerados**: supuesto provisional — V1 soporta **entrada general con tipos de boleta de precio diferenciado** (ej. general/VIP) sin asientos numerados. Si Q-CR-01 resuelve lo contrario, el modelo debe ampliarse.
- **S-04**: **Límite de boletas por orden**: supuesto provisional — máximo **6 boletas por orden, 10 por usuario por evento**. Sujeto a Q-CR-02.
- **S-05**: **Fee por servicio**: supuesto provisional — lo asume el **comprador como fee visible** en el checkout. Sujeto a Q-CR-03.
- **S-06**: **Reembolsos**: supuesto provisional — solo vía SUPPORT, no self-service en V1. Sujeto a Q-CR-04.
- **S-07**: **Facturación DIAN**: supuesto provisional — V1 emite **tiquete POS electrónico** (más simple) y la factura electrónica completa se pospone a V2. Sujeto a Q-CR-05.
- **S-08**: **Organizadores**: supuesto provisional — los crea SUPER_ADMIN (no auto-registro). Sujeto a Q-NC-01.
- **S-09**: **Moneda**: V1 opera **solo en COP**.
- **S-10**: **Notificaciones**: V1 usa **solo email**.
- **S-11**: **Datos mínimos del perfil**: nombre, correo, teléfono. Documento de identidad y fecha de nacimiento se agregan **solo si la facturación DIAN lo exige** (depende de Q-CR-05/Q-NC-04).
- **S-12**: **Validación sin cámara**: se ofrece **entrada manual del código** de la boleta como fallback en el módulo de validación.
- **S-13**: **Anclaje Merkle**: frecuencia por definir por Planner; supuesto provisional — cada **5 minutos o cada 500 boletas**, lo que ocurra primero.
- **S-14**: **Reventa/transferencia**: no se implementa en V1, pero el modelo de boleta debe permitir evolución futura a NFT sin rediseño disruptivo.
- **S-15**: **Idioma**: V1 en español (Colombia); i18n preparado pero no prioritario.
- **S-16**: **Concurrencia objetivo**: el sistema debe soportar ventas pico del orden de **miles de usuarios concurrentes** sobre un mismo evento (cifra exacta a refinar por Planner con datos del usuario).

---

## 14. Handoff para Planner

### Objetivo funcional resumido
CriptoPass V1 es un portal de venta de boletas con trazabilidad blockchain pública (Polygon L2), sala de espera para ventas pico, reserva de inventario con TTL, pago en Mercado Pago, QR dinámico anti-duplicación, panel administrativo multi-rol y cumplimiento colombiano (Ley 1581, DIAN pendiente).

### Scope y out of scope
- **In**: catálogo público SEO, compra con sala de espera y TTL, pago Mercado Pago, "Mis boletas", QR dinámico, admin multi-rol, validación en puerta, trazabilidad Merkle/Polygon, habeas data.
- **Out**: reventa/transferencia, NFT, app móvil, multi-moneda, microfrontends, event sourcing, otras pasarelas, dashboard analítico avanzado, validación offline.

### Actores, permisos y restricciones
- Comprador registrado, Visitante anónimo, SUPER_ADMIN, ORGANIZER (aislamiento por dueño), VALIDATOR (solo escaneo), SUPPORT (consulta + reenvío).
- Restricciones: no almacenar PAN/CVV, consentimiento Ley 1581, idempotencia en webhooks, redención única, TTL de reserva.

### Flujos principales y alternos
- Navegación anónima, compra estándar, compra con sala de espera, visualización/usos de boleta, creación de evento, validación en puerta, soporte, anclaje Merkle.
- Alternos cubiertos en Edge Cases EC-01 a EC-15.

### Entidades funcionales y datos sensibles
- Evento, Localidad (condicional), Boleta, Orden, Pago, Usuario, Organizador, Lote Merkle, Sala de espera, Auditoría.
- Datos sensibles: datos personales del comprador (mínimo necesario), hashes de boleta (sin filtrar PII), logs de auditoría.

### Integraciones esperadas y criticidad
- Cognito (crítica), Mercado Pago (crítica), Polygon L2 (alta), S3 (alta), Email (alta), DIAN (pendiente de Q-CR-05), PolygonScan (media).

### Criterios de aceptación
- 40+ criterios verificables mapeados a funcionalidades (CA-PC, CA-SE, CA-PA, CA-TB, CA-CO, CA-EC).

### Preguntas abiertas
- **Críticas**: Q-CR-01 a Q-CR-05 (modelo de localidades, límites, fee, reembolsos, facturación DIAN).
- **No críticas**: Q-NC-01 a Q-NC-06.

### Supuestos explícitos
- S-01 a S-16 (marco técnico fijo, supuestos provisionales para las preguntas críticas, moneda COP, email-only, etc.).

### Riesgos funcionales que Planner debe resolver técnicamente
- **R-01**: Concurrencia sobre inventario pico — requiere diseño de reserva con TTL, locks optimistas/pesimistas o patrón de inventario secuencial; la sala de espera alone no garantiza fairness sobre la última boleta.
- **R-02**: Saga de compra distribuida (pago + emisión + anclaje) — la compensación ante fallos parciales (EC-01, EC-03) es crítica para no cobrar sin entregar boleta ni emitir sin cobrar.
- **R-03**: Idempotencia de webhooks de Mercado Pago — sin ella se pueden emitir boletas duplicadas.
- **R-04**: Sincronización de la ventana de rotación del QR entre el dispositivo del comprador y el validador — desync de reloj puede causar rechazos falsos (EC-05).
- **R-05**: Anclaje Merkle en Polygon — si la frecuencia es muy alta, el costo on-chain puede crecer; si es muy baja, la trazabilidad "pública" pierde valor inmediato.
- **R-06**: Aislamiento de datos entre ORGANIZER — una fuga de eventos ajenos es un riesgo de privacidad y de negocio.
- **R-07**: Cumplimiento Ley 1581 — los derechos ARSO deben tener un flujo funcional real (no solo textual), lo que impacta el modelo de datos y la retención.
- **R-08**: Facturación DIAN sin definir — si se requiere en V1, puede agregar semanas al alcance por integración y datos del comprador.
- **R-09**: Evolución futura a NFT/reventa — el modelo de boleta debe ser compatible con tokenización sin migración destructiva.
- **R-10**: SEO del catálogo vs. CQRS-lite — el modelo de lectura cacheable debe garantizar consistencia eventual aceptable para que un evento recién publicado sea rastreable.

---

## 15. Trazabilidad Requerimientos → Criterios de Aceptación

| Funcionalidad | Criterios |
|---|---|
| F-PC-01 Catálogo público | CA-PC-01 |
| F-PC-02 SEO | CA-PC-02 |
| F-PC-03 Login al comprar | CA-PC-03 |
| F-PC-04 Sala de espera | CA-SE-01, CA-SE-02, CA-SE-03 |
| F-PC-05 Reserva con TTL | CA-PC-04 |
| F-PC-06 Pago Mercado Pago | CA-PC-05 |
| F-PC-07 Mis boletas | CA-PC-06 |
| F-PC-08 QR dinámico | CA-PC-07 |
| F-PC-09 Boleta redimida | CA-PC-08 |
| F-PC-10 Perfil | CA-PC-09 |
| F-PC-11 Múltiples boletas | CA-PC-10 |
| F-PA-01 Auth admin | CA-PA-05, CA-PA-07, CA-PA-08 |
| F-PA-02 Gestión usuarios | CA-PA-01 |
| F-PA-03 CRUD eventos | CA-PA-02, CA-PA-03 |
| F-PA-04 Capacidad/precios | CA-PA-02 |
| F-PA-05 Ventas por evento | CA-PA-04 |
| F-PA-06 Validación QR | CA-PA-05 |
| F-PA-07 Soporte | CA-PA-06 |
| F-TB-01 Hash por boleta | CA-TB-01 |
| F-TB-02 Anclaje Merkle | CA-TB-02, CA-EC-14 |
| F-TB-03 Verificación pública | CA-TB-03 |
| F-TB-04 Diseño no bloqueante NFT | CA-TB-04 |
| F-CO-01 Consentimiento | CA-CO-01 |
| F-CO-02 Política privacidad | CA-CO-02 |
| F-CO-03 No PAN/CVV | CA-CO-03 |
| F-CO-04 DIAN | CA-CO-04 |
| Edge cases | CA-EC-01 a CA-EC-14 |

---

**Fin del Requirements Brief — CriptoPass V1.**