# Informe de Arquitectura - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-ARCH-001 |
| Estado | DRAFT |
| Versión | 0.5 |
| Fecha | 2026-10-20 |
| Idioma | Español; nombres de código en inglés |
| Baseline funcional | Requisitos aprobados entregados para este informe |

> Este documento registra arquitectura, decisiones aprobadas y limites de la implementacion Campaign inspeccionada. Los contratos externos, la habilitacion real de scheduling y los OPENs se identifican por separado. Esta version no aprueba formalmente los SPECS.

## 1. Objetivo

Definir cómo PEC puede administrar y ejecutar campañas automatizadas que reutilizan encuestas existentes, calculan una audiencia segmentada por compras elegibles y envían invitaciones por Emblue. La arquitectura debe conservar la separación entre encuestas, campañas y ejecuciones; impedir duplicados en un entorno con varias réplicas; permitir recuperación de fallos; y minimizar la exposición de datos personales.

## 2. Alcance

### Incluido

- Reutilización de `SurveyJS` y de las encuestas PEC existentes.
- Administración de `Campaign`, segmentación, `Post-purchase Offset` y programación única o recurrente.
- Vista previa de audiencia calculada en backend, con conteos de `Eligible Purchases` y `Eligible Customers` y una muestra limitada opcional.
- Procesamiento incremental, registro de ejecuciones y destinatarios, deduplicación por compra y atribución segura de respuestas.
- Envío mediante `Emblue Transactional Email API` y recepción de eventos por `DataLab/Webhooks`, sujeto a sus contratos.
- Pantallas de dashboard, listado, detalle, creación/edición y consulta de ejecuciones.

### Fuera de alcance

- Un nuevo editor de encuestas o cambios al flujo de autoría `SurveyJS`.
- Rangos manuales de fechas de compra.
- `Auxiliary Subclassification` en la primera versión.
- Definir las vistas corporativas, su conexión o el contrato de Emblue sin evidencia de sus propietarios.
- Definir todavía schema Prisma definitivo, rutas HTTP definitivas, migraciones o tareas de ejecución.

## 3. Contexto del repositorio

### Hallazgos verificados en código

| Área | Hallazgo | Evidencia del checkout |
|---|---|---|
| API | El backend usa NestJS y organiza funcionalidades en módulos, controllers y services. | `pec-api/src/app.module.ts`, `pec-api/src/surveys/` |
| Persistencia | El API usa Prisma y PostgreSQL; el modelo de encuesta existente se llama `sspectsurvey`. | `pec-api/prisma/schema.prisma`, `pec-api/package.json` |
| Survey | Existe un módulo de encuestas PEC que usa SurveyJS y administra contenido, estado, acceso público y respuestas. | `pec-api/src/surveys/`; `pec-api/prisma/schema.prisma` |
| Disponibilidad | `availableFrom` y `availableTo` se manejan en el contenido/flujo de la encuesta existente. | `pec-api/src/surveys/surveys.service.ts` |
| Scheduler | `@nestjs/schedule` está instalado y `ScheduleModule.forRoot()` se registra globalmente. Hay tareas cron de mantenimiento/sincronización. | `pec-api/src/app.module.ts`, `pec-api/src/case-locks/case-locks.scheduler.ts`, `pec-api/src/sync/sync.service.ts` |
| UI | El frontend administrativo es Angular 17 con Angular Material y tiene un módulo de encuestas. | `pec-ui/package.json`, `pec-ui/src/app/modules/otros/surveys/` |
| Navegación | Hay una ruta/pantalla inicial para campañas automatizadas, pero no se verificó un servicio API persistente de campañas. | `pec-ui/src/app/modules/otros/surveys/`, rutas de `surveys` |
| Audiencia corporativa | El servicio de clientes encontrado consulta APIs de contacto individuales; no se identificó en el código una vista corporativa de compras/divisiones/departamentos que resuelva la audiencia solicitada. | `pec-api/src/client/client.service.ts` |
| Emblue | En el baseline inspeccionado antes del Sprint 3.0 no se identificó un adaptador; ver la actualización de implementación más reciente en §17.1. | Búsqueda inicial en `pec-api/src/` y dependencias |
| Cola/claim compartido | No se verificó una infraestructura actual de cola o mecanismo de claim para ejecuciones de campañas. | Inspección de módulos y dependencias del API |

Los hallazgos describen el checkout inspeccionado, no certifican configuración desplegada ni contratos externos. Los archivos de UI de campañas pueden ser un shell visual; no se consideran prueba de persistencia ni de envío real.

### Arquitectura actual relevante

```text
pec-ui (Angular 17 / Angular Material)
  └─ módulo administrativo de encuestas y SurveyJS
       │ HTTP (contratos existentes por revisar al implementar)
       ▼
pec-api (NestJS)
  ├─ SurveysModule / controller / service
  ├─ otros módulos NestJS
  ├─ @nestjs/schedule registrado en AppModule
  └─ Prisma ── PostgreSQL

pec-public-surveys-web
  └─ experiencia pública de respuesta de encuestas
```

El esquema y los nombres del diagrama representan componentes inspeccionados. La integración exacta entre el sitio público y la atribución de campañas se debe validar antes de implementarla.

## Decisiones del incremento 2.5

| ID | Estado | Decisión |
|---|---|---|
| DEC-SC-01 | **APPROVED** | Campaign es multiempresa; solo `ADMIN` y `SUPERADMIN` consultan y gestionan campañas de todas las empresas. La empresa activa PEC no limita el módulo. Autenticación y CASL backend siguen obligatorios; no cambian grants de otros módulos. |
| DEC-SC-02 | **APPROVED** | El selector usa un endpoint de Campaign para Surveys y no amplía permisos generales de Surveys. Survey debe existir y ser accesible bajo la política aprobada. |
| DEC-SC-03 | **APPROVED** | `Enterprise` PEC y `Company` CF/Analytics son conceptos distintos; no asumir equivalencias ni relaciones entre identificadores. |
| DEC-SC-04 | **OPEN** | Los contratos reales de catálogos CF/Analytics no están disponibles; no fijar endpoints, autenticación, campos ni jerarquías reales. |
| DEC-SC-05 | **MOCK** | Proveedor temporal con datos sintéticos marcados MOCK; sus relaciones solo sirven para probar controles y no representan jerarquías confirmadas de CF. |
| DEC-SC-06 | **APPROVED** | El proveedor Analytics futuro debe sustituir al MOCK sin cambiar el contrato Angular. |
| DEC-SC-07 | **APPROVED** | MOCK se deshabilita por defecto en producción y nunca es fallback automático ante fallo de Analytics. Sin proveedor, el backend falla explícitamente. |
| DEC-SC-08 | **APPROVED** | Division, Department y Company son obligatorios para programar, pero pueden faltar en DRAFT. |
| DEC-SC-09 | **APPROVED** | Auxiliary Classification, Format y Store son opcionales; Auxiliary Subclassification se excluye de V1. |
| DEC-SC-10 | **APPROVED** | Se admite multiselección, OR dentro del filtro y AND entre filtros. Backend valida IDs y relaciones contra el proveedor configurado; jerarquías MOCK no se atribuyen a CF. |
| DEC-SC-11 | **DEFERRED** | Scheduler, evaluación de audiencia real y envíos no pueden usar catálogos MOCK. |
| DEC-SC-12 | **APPROVED para Campaign** | Survey elegible si status=1, isPublic=true y archived=false; esta regla no redefine un estado global PUBLISHED. |
| DEC-SC-13 | **OPEN** | La validación PostgreSQL del Sprint 2.4 continúa pendiente. No se hicieron operaciones remotas en esta actualización. |

Este registro no cambia el estado formal DRAFT ni equivale a aprobación documental.

## 4. Baseline funcional

### Regla de idioma de interfaz

Toda la UI Campaign se presenta en español. Angular centraliza etiquetas por enums, frecuencias y modalidades; el backend, API, Prisma y PostgreSQL conservan nombres y valores técnicos en inglés. Traducción visible nunca modifica filtros ni payloads. Formatos de fecha, hora y paginador deben usar español. El glosario funcional y las equivalencias obligatorias están en FOUND-001; los detalles de pruebas están en TEST-001.

Los siguientes requisitos se toman del baseline entregado para este documento. No son decisiones derivadas del código.

### 4.1 Survey y Campaign

- `Survey` y `Campaign` son conceptos separados. `Campaign` referencia una encuesta existente y no reemplaza SurveyJS.
- Una campa puede guardarse en `DRAFT` aunque la Survey no cumpla el criterio Campaign. Para programarla, requiere status=1, isPublic=true y archived=false, ademas de disponibilidad durante calendario.
- `Available From` y `Available Until` continúan perteneciendo al módulo Survey.
- **APPROVED para Campaign:** La regla Campaign aprobada es status=1, isPublic=true y archived=false; la semantica global de PUBLISHED no se define aqui.

### 4.2 Segmentación y elegibilidad

| Jerarquía | Filtro | Selección |
|---|---|---|
| Producto | `Division` | Requerido, multiselect |
| Producto | `Department` | Requerido, multiselect |
| Producto | `Auxiliary Classification` | Opcional, multiselect |
| Producto | `Auxiliary Subclassification` | Fuera de alcance v1 |
| Comercial | `Company` externo | Requerido para programar, multiselect |
| Comercial | `Format` | Opcional, multiselect |
| Comercial | `Store` | Opcional, multiselect |

- Dentro de cada filtro, los valores seleccionados se combinan con OR; entre filtros distintos se combinan con AND. Enterprise PEC no equivale a Company CF/Analytics. Division, Department y Company pueden faltar en DRAFT y se exigen al programar.
- Un filtro opcional vacío no restringe resultados. Al cambiar un padre se limpian selecciones hijas incompatibles.
- `Post-purchase Offset` es obligatorio, entero no negativo, en días. `0` significa sin espera adicional.
- Una compra se vuelve elegible cuando cumple la segmentación y `PurchaseDate + Offset <= ExecutionDate`, y no ha sido procesada antes por esa campaña.
- La deduplicación es por `campaignId + purchaseId/transactionId`. Un mismo cliente puede volver a ser elegible por otra compra.
- No se define un rango manual de fechas de compra.

### 4.3 Audience Preview

- El backend calcula `Eligible Purchases` y `Eligible Customers` aplicando segmentación y offset.
- Para campañas existentes, se excluyen compras ya procesadas por esa campaña.
- Puede devolverse una muestra acotada, si es necesaria para revisión.
- Angular nunca carga la audiencia completa.

### 4.4 Programación y frecuencias

`SendType` admite `ONE_TIME` y `FREQUENCY`.

| Tipo | Campos requeridos |
|---|---|
| `ONE_TIME` | `Send Date`, `Send Time` |
| `FREQUENCY` | `Start Date`, `End Date`, `Send Time`, `Frequency` |

Frecuencias: `DAILY`, `WEEKLY`, `BIWEEKLY`, `MONTHLY`, `QUARTERLY`.

El catálogo permitido depende de la duración entre fechas:

| Duración | Frecuencias disponibles |
|---|---|
| 1-6 días | `DAILY` |
| 7-13 días | `DAILY`, `WEEKLY` |
| 14-29 días | `DAILY`, `WEEKLY`, `BIWEEKLY` |
| 30-89 días | `DAILY`, `WEEKLY`, `BIWEEKLY`, `MONTHLY` |
| 90 días o más | Todas, incluyendo `QUARTERLY` |

`nextExecutionAt` se calcula sobre fechas locales: `DAILY` +1 día, `WEEKLY` +7 días, `BIWEEKLY` +14 días, `MONTHLY` +1 mes calendario y `QUARTERLY` +3 meses calendario. En `MONTHLY`/`QUARTERLY`, cada fecha se calcula desde `StartDate` y el índice, conservando el día ancla o usando el último día disponible del mes sin cambiar el ancla siguiente; incluye años bisiestos y sucede antes de convertir a UTC. `ONE_TIME` termina tras su ejecución única. `FREQUENCY` termina cuando la siguiente ejecución supera `End Date`.

**OPEN:** zona horaria, tratamiento de horario de verano, normalización de fechas y ejecución si la fecha/hora cae en un instante inexistente o repetido.

### 4.5 Estados y acciones

`CampaignStatus`: `DRAFT`, `SCHEDULED`, `ACTIVE`, `SUSPENDED`, `COMPLETED`, `CANCELLED`.

- `SCHEDULED` aplica antes de la primera ejecución; `ACTIVE`, después de al menos una ejecución si quedan ejecuciones futuras.
- `DRAFT`: `Edit`, `Schedule`, `Cancel`, `Delete`.
- `SCHEDULED`: `Edit`, `Suspend`, `Cancel`.
- `ACTIVE`: `Suspend`, `Cancel`; para editar primero se suspende.
- `SUSPENDED`: `Edit`, `Resume`, `Cancel`.
- `COMPLETED` y `CANCELLED`: solo lectura. Con historial, la campaña no se elimina físicamente.

`CampaignExecutionStatus`: `PENDING`, `PROCESSING`, `COMPLETED`, `COMPLETED_WITH_ERRORS`, `FAILED`. Transición: `PENDING` → `PROCESSING` → estado final; los estados finales son históricos e inmutables.

`CampaignRecipientStatus`: `PENDING`, `SENT`, `FAILED`, `SKIPPED_DUPLICATE`. No se introduce un estado genérico `SKIPPED`.

### 4.6 Emblue y atribución

- Camino funcional aprobado: PEC → `Emblue Transactional Email API`; Emblue → PEC mediante `DataLab/Webhooks` para eventos.
- PEC es responsable de campaña, segmentación, offset, frecuencia, scheduler, deduplicación, ejecución y destinatario. Emblue es responsable de plantilla, entrega, deliverability, aperturas, clics y rebotes.
- Variable requerida de plantilla: `surveyUrl`. Opcionales: `customerName`, `companyName`, `formatName`, `storeName`, `expirationDate`, `surveyName`, `purchaseDate`.
- `purchaseId/transactionId` se conserva internamente en PEC y no se expone en la plantilla salvo necesidad futura.
- Cada invitación debe poder atribuir una respuesta a Campaign, Execution y Recipient sin PII en la URL.
- **OPEN:** mecanismo del token, callbacks disponibles, firma/autenticación, contrato final, idempotencia de proveedor y política de retries.

### 4.7 Dashboard y detalle

Dashboard:

- Cards: `Total`, `Active`, `Scheduled`, `Draft`, `Suspended`, `Completed`. `CANCELLED` se consulta por filtro, sin card propia.
- Search: `Campaign Name`, `Survey Name`.
- Filter: `Status`.
- Tabla: `Campaign`, `Survey`, `Frequency`, `Next Execution`, `Status`, `Actions`.

Detalle, secciones y campos funcionales:

- `Summary`: `Campaign Name`, `Description`, `Survey`, `Email Template`, `Status`, `Send Type`, `Frequency`, `Next Execution`, `Created By`, `Created At`, `Updated At`.
- `Segmentation`: `Divisions`, `Departments`, `Auxiliary Classifications`, `Companies`, `Formats`, `Stores`, `Post-purchase Offset`, `Eligible Purchases`, `Eligible Customers`.
- `Scheduling`: `Send Type`, `Send Date` para `ONE_TIME`, `Start Date`, `End Date`, `Send Time`, `Frequency` para `FREQUENCY`, `Last Execution`, `Next Execution`.
- `Executions`: `Scheduled At`, `Started At`, `Finished At`, `Eligible Purchases`, `Eligible Customers`, `Sent`, `Failed`, `Duplicates`, `Status`.

## 5. Arquitectura propuesta

La propuesta separa la administración de campañas, la resolución de audiencia, la ejecución y las integraciones. La API sigue siendo la autoridad para validación, elegibilidad y persistencia. El scheduler global existente puede iniciar el sondeo; por sí mismo no garantiza exclusión entre réplicas.

```text
Angular pec-ui
  Campaigns UI ── REST/HTTP propuesto ──► SurveyCampaignsModule (NestJS)
                                          ├─ Campaign administration
                                          ├─ Audience preview / resolver
                                          ├─ Scheduling and execution coordinator
                                          ├─ Emblue adapter
                                          ├─ DataLab/Webhook event adapter
                                          └─ Prisma ── PostgreSQL
                                                    ├─ Campaign configuration
                                                    ├─ CampaignExecution history
                                                    └─ CampaignRecipient ledger and individual outcomes

Global @nestjs/schedule tick ──► atomic claim ──► execution worker
Corporate views/Analytics ──read-only audience data──► resolver
Emblue Transactional Email API ◄── transactional email requests
Emblue DataLab/Webhooks ──delivery events──► PEC
Public survey flow ◄── opaque attribution reference in surveyUrl
```

Todos los componentes nuevos del diagrama son propuestas, no hallazgos de implementación. **OPEN:** confirmar si Audience se consulta directamente en un motor corporativo, por una API de Analytics, o mediante otro adaptador autorizado.

### 5.1 Componentes y responsabilidades

| Componente propuesto | Responsabilidad |
|---|---|
| `SurveyCampaignsModule` | Agrupar administración, validación, consulta, ejecución y adaptadores propios de campañas en NestJS. |
| `CampaignService` | Aplicar transiciones de estado, validar configuración y coordinar persistencia. No enviar correos directamente desde el controller. |
| `CampaignCatalogProvider` | Proveedor de catálogos exclusivo de Campaign, detrás de contrato estable para Angular; permite MOCK sintético y futuro Analytics sin inventar su protocolo. |
| `AudienceResolver` | Traducir filtros a consulta segura sobre la fuente corporativa; aplicar jerarquías, offset, deduplicación y conteos. |
| `AudiencePreviewService` | Ejecutar preview del lado servidor con límites y sin devolver el universo completo. |
| `CampaignScheduler` | Usar un tick global de `@nestjs/schedule` para encontrar campañas vencidas y reclamar ejecuciones. No crear un cron por campaña. |
| `CampaignExecutionService` | Procesar una ejecución reclamada, persistir avances/resultados y decidir el próximo estado/instante. |
| `EmblueClient` | Encapsular autenticación, formato de solicitud, límites, respuestas, errores e idempotencia de Emblue. Si la llamada externa es individual o batch depende del contrato, OPEN. |
| `CampaignEventHandler` | Validar y correlacionar eventos recibidos por DataLab/Webhooks; aplicar deduplicación de eventos. |
| `AttributionService` | Crear y validar la referencia opaca entre invitación y respuesta. No exponer identificadores personales en la URL. |
| Angular Campaigns UI | Dashboard, búsqueda/filtros, formulario, preview, detalle e historial mediante API; no determina la elegibilidad final. |
| PostgreSQL/Prisma | Persistir configuración, ejecuciones y `CampaignRecipient` obligatorio a nivel de dominio, con sus resultados individuales e idempotencia. Estructura física final OPEN para DATA-001. |

Los nombres anteriores son vocabulario arquitectónico sugerido. No implican que existan ya clases con esos nombres ni fijan la estructura final del código.

## 6. Modelo conceptual y límites de persistencia

Entidades conceptuales (no schema definitivo):

- `Campaign`: nombre/descripción, referencia a Survey, plantilla, filtros de audiencia, offset, tipo y datos de programación, estado, `nextExecutionAt`, `lastExecutionAt`, auditoría y zona horaria acordada.
- `CampaignExecution`: Campaign, `scheduledAt`/instante programado, estado, inicio/fin, conteos, errores sanitizados, referencia de proveedor y datos de recuperación.
- `CampaignRecipient` (obligatorio a nivel de dominio): representa individualmente el recipient/compra procesada para una Campaign y su Execution; conserva el resultado individual (`PENDING`, `SENT`, `FAILED` o `SKIPPED_DUPLICATE`) y la correlación necesaria. Cada Execution permite consultar sus recipients y resultados individuales. La estructura física (tabla, relaciones y columnas) queda para DATA-001.
- Registro de eventos externos: identificador de evento/proveedor y estado de procesamiento para evitar aplicar callbacks duplicados, si el contrato ofrece identificadores estables.

**OPEN:** fuente autoritativa y estabilidad de `purchaseId/transactionId`, relación de compra con cliente/email, retención, índices y restricciones únicas. No persistir copias completas de vistas ni PII que no se requiera operacionalmente.

La lectura de audiencia debe permanecer en el servidor. El cliente Angular recibe agregados, muestras acotadas y datos de campaña autorizados. La selección incremental requiere una estrategia consistente frente a cambios tardíos en las vistas y compras que pasan el offset entre ejecuciones.

## 7. Flujo end-to-end propuesto

1. Un usuario autorizado crea `Campaign` en `DRAFT`, selecciona una encuesta existente y configura filtros, offset, plantilla y programación.
2. Angular solicita preview; backend valida filtros y consulta el resolver corporativo. Devuelve conteos y, si se autoriza, una muestra limitada.
3. Al programar, el API valida campos, ventana/frecuencia permitida, permisos y elegibilidad de Survey con status=1, isPublic=true y archived=false, ademas de calendario dentro de su ventana de disponibilidad.
4. Se persiste la programación y `nextExecutionAt`; la campaña queda `SCHEDULED` antes de su primera ejecución.
5. El tick global busca campañas vencidas. Una operación atómica de claim crea o reclama una `CampaignExecution` para una ejecución programada (`campaignId + scheduledAt / executionKey`), evitando que dos réplicas procesen simultáneamente la misma ejecución.
6. `AudienceResolver` obtiene compras que cumplen segmentación y `PurchaseDate + Offset <= ExecutionDate`; excluye la clave campaign + purchase ya procesada.
7. PEC registra cada `CampaignRecipient` y su estado con idempotencia, y envía a Emblue solicitudes de email transaccional con `surveyUrl` y las variables aprobadas. PEC puede procesar internamente en lotes; si cada llamada externa es individual o batch depende del contrato definitivo de Emblue (**OPEN**). El email se almacena/usa según la política de minimización y no se incluye en logs.
8. Se persiste cada resultado operacional como `SENT`, `FAILED` o `SKIPPED_DUPLICATE`, según respuesta del contrato. El cierre resume conteos y finaliza `COMPLETED`, `COMPLETED_WITH_ERRORS` o `FAILED`.
9. Se actualizan `lastExecutionAt`, `nextExecutionAt` y el estado de campaña solo de acuerdo con la política de éxito/recuperación aprobada. Una falla total no debe marcar como procesada audiencia pendiente ni provocar que ejecuciones futuras omitan compras que todavía deben ser consideradas.
10. Eventos de entrega recibidos por DataLab/Webhooks se validan, deduplican y correlacionan. La respuesta pública valida una referencia de atribución opaca para ligar Survey Response con Campaign/Execution/Recipient.

**OPEN:** semántica exacta de `Sent` (aceptado por API frente a entregado), ventana de captura, callback tardío y reconciliación del estado con Emblue.

## 8. Scheduler, recuperación y concurrencia

### 8.1 Scheduler central

Usar un proceso/tick global del `@nestjs/schedule` ya registrado es compatible con el patrón presente en API. El tick debe ser corto: localizar trabajo vencido y ejecutar un claim; el envío y procesamiento pueden ser delegados al coordinador/worker interno. No registrar tareas cron por cada Campaign.

### 8.2 Claim multi-instancia

El scheduler de NestJS se ejecuta en cada réplica que carga la aplicación. Por tanto, la propuesta requiere un claim compartido respaldado por PostgreSQL antes de empezar trabajo. Evaluar una transacción con locking de fila/`SKIP LOCKED`, advisory lock o lease con vencimiento; elegir mediante ADR tras revisar versión/configuración PostgreSQL, volumen y política de operación. El mecanismo debe:

- garantizar un único claim activo por Campaign + ejecución programada (`campaignId + scheduledAt / executionKey`); la clave física definitiva queda pendiente para DATA/SCHED;
- permitir recuperar claims abandonados tras caída;
- registrar `claimedAt`, propietario/lease y vencimiento sin PII;
- proteger también la creación única de `CampaignExecution` mediante restricción idempotente;
- mantener transacciones de claim breves y no mantener locks durante llamadas HTTP a Emblue.

No se verificó un mecanismo reutilizable de claim de campañas en el checkout. El lock de otros procesos no debe considerarse automáticamente apto para estas ejecuciones.

### 8.3 Recuperación

- Al volver de una caída, detectar campañas vencidas y procesar las compras elegibles pendientes en una ejecución de recuperación, evitando una ráfaga de envíos atrasados por cada intervalo omitido.
- Una falla total no debe marcar como procesada audiencia pendiente ni provocar que ejecuciones futuras omitan compras que todavía deben ser consideradas.
- Para fallo parcial, registrar outcomes por destinatario y cerrar `COMPLETED_WITH_ERRORS` de acuerdo con la política aprobada.
- Al reintentar, reutilizar claves idempotentes campaign + purchase y, si está disponible, la clave idempotente exigida por Emblue.
- Persistir progreso interno por recipient/request para que una caída no reenvíe destinatarios ya aceptados por el proveedor. La granularidad de la llamada externa (individual o batch) depende del contrato Emblue (**OPEN**).

**OPEN:** backoff, máximo de intentos, expiración de lease, tratamiento de resultados ambiguos/timeout de Emblue, ventana máxima de recuperación y cuándo se considera seguro avanzar el próximo ciclo.

## 9. Integraciones

| Integración | Dirección | Responsabilidad y validación pendiente |
|---|---|---|
| Fuente corporativa CF/Analytics | PEC leerá catálogos y, en otra fase, audiencia | **OPEN:** no hay contrato real ni evidencia de endpoint, autenticación, campos o jerarquías. |
| `MockCampaignCatalogProvider` | Catálogos sintéticos aislados del frontend mediante el contrato Campaign | **MOCK:** seis catálogos; IDs y labels sintéticos identificados como MOCK; relaciones para probar dependencias, no confirmadas por CF. Se activa solo con `CAMPAIGN_CATALOG_PROVIDER=mock` fuera de producción; no es fallback. |
| Emblue Transactional Email API | PEC → Emblue | **Parcialmente confirmado:** envío individual v2.3 por `POST /emails/send`, Bearer, body confirmado y response `{ activityId }`. **OPEN:** plantilla, límites, procesamiento/entrega, idempotencia, consulta de actividad y reconciliación. |
| Emblue DataLab/Webhooks | Emblue → PEC | **OPEN:** eventos y campos disponibles, firma, reintentos del proveedor, orden, identificador único, política de replay y exposición de aperturas/clics/bounces. |
| PEC Surveys | Campaign → Survey existente | Reutilizar metadatos y URL/flujo existentes. Validar disponibilidad y estado de publicación sin duplicar SurveyJS. |
| PEC Public Survey | Invitación → respuesta | Incorporar correlación segura sin PII en URL; validar cambios mínimos requeridos al flujo público y modelo de respuesta. |

No registrar secretos de integración en código ni en documentos versionados. La configuración por ambiente debe alinearse con el mecanismo de secretos ya usado en despliegue, que debe confirmarse antes del diseño de configuración final.

## 10. Seguridad y protección de datos

- Aplicar autenticación PEC y CASL backend a todas las rutas Campaign, incluido selector de Surveys y catálogos. Solo `ADMIN` y `SUPERADMIN` tienen acceso multiempresa; los demás roles no acceden. No se modifican permisos de otros módulos.
- La empresa activa PEC no limita Campaign. El endpoint específico de Campaign valida Surveys conforme a esta política sin ampliar `/surveys`.
- Requerir autorización del backend; ocultar acciones en Angular no constituye control de acceso.
- Usar solo identificadores y correo indispensables para resolver y enviar invitaciones. No copiar una audiencia completa a la base PEC sin justificarlo.
- Restringir acceso a destinatarios y proteger datos persistidos con las medidas de plataforma aprobadas.
- No escribir emails completos, tokens, secretos ni payloads completos que contengan PII en logs/errores. Usar correlation IDs y datos agregados o enmascarados.
- La URL de encuesta no contiene email, `purchaseId`, `transactionId` ni tokens de proveedor. El token de atribución debe ser opaco, impredecible, limitado al propósito y validado en servidor; almacenamiento hash/expiración/usos quedan para ADR y revisión de seguridad.
- Validar autenticidad de webhooks y limitar replay una vez que el contrato esté disponible.
- QA y pruebas integradas usan destinatarios sintéticos y credenciales no productivas.
- Definir retención/eliminación de correos, registros de destinatarios, respuestas correlacionadas y datos de eventos antes de producción. **OPEN.**

## 11. Trazabilidad y observabilidad

Cada ejecución debe permitir reconstruir sin PII innecesaria: Campaign, `scheduledAt`, cutoff de elegibilidad de esa ejecución, conteos, progreso de procesamiento interno, referencias de solicitudes transaccionales y resultados individuales de `CampaignRecipient`. La relación interna con cada compra se conserva con identificador mínimo y restricción de unicidad campaign + purchase.

Propuesta de observabilidad:

- Logs estructurados con `campaignId`, `executionId`, etapa, resultado, tiempo y correlation ID; excluir email, token y payload sensible.
- Métricas agregadas: campañas vencidas/reclamadas, duración, ejecuciones por estado, elegibles, enviados/aceptados, fallidos, duplicados, retries, antigüedad del trabajo pendiente y errores por integración.
- Alertas operativas por ejecuciones atascadas, fallas repetidas de fuente/Emblue, crecimiento de backlog y expiración de leases.
- Historial de estados/eventos administrativos: quién programó, pausó, reanudó o canceló y cuándo, sujeto a los campos de auditoría disponibles en PEC.
- No agregar al dashboard de negocio indicadores de apertura/clic como si estuvieran disponibles hasta verificar contrato y eventos.

**OPEN:** stack de métricas/alertas efectivamente desplegado, retención de logs/auditoría, umbrales, y disponibilidad del identificador estable de evento Emblue.

## 12. Impacto por capa

| Capa | Impacto arquitectónico propuesto | Situación actual / límite |
|---|---|---|
| Frontend | Continuar administracion, segmentacion y QA; contrato de catalogos independiente del proveedor. | Dashboard/listado/detalle y formulario conectados; catalogos MOCK visibles. Preview real y scheduler quedan fuera/deferred. |
| Backend | Evolucionar `SurveyCampaignsModule`, CASL, CRUD, selector Campaign de Surveys y provider de catalogos. | CRUD/dashboard, selector de Surveys y catalogos MOCK existen en checkout. Analytics real, Emblue, preview real y scheduler no implementados. |
| Database | Mantener configuracion y asociaciones de selecciones en modelos Prisma Campaign existentes. | Modelos y migracion Campaign estan en checkout; validacion PostgreSQL de Sprint 2.4 pendiente. Este cambio documental no aplica migraciones. |
| Fuente de datos | Catalogos sinteticos durante desarrollo y fuente real cuando haya contrato. | Analytics real sigue OPEN; MOCK no resuelve audiencia ni autoriza envios. |
| Emblue | Cliente de envío y receptor/correlacionador de eventos. | Contratos de API/DataLab/Webhooks y retries OPEN. |
| Encuesta pública | Atribución Campaign/Execution/Recipient sin exponer PII. | Flujo existente requiere revisión específica antes de decidir integración. |
| Operación | Scheduler replicado con claim, recuperación, métricas y límites de carga. | Existe scheduler global; no se verificó claim de campañas ni configuración operativa de workers. |

## 13. Decisiones arquitectónicas

| ID | Decisión / propuesta | Estado | Consecuencia / validación |
|---|---|---|---|
| ADR-SC-01 | Mantener `Campaign` separada de Survey y respuestas. | Baseline aprobado | Referenciar encuesta existente; no duplicar autoría SurveyJS. |
| ADR-SC-02 | Ejecutar una consulta central del scheduler, no un cron por campaña. | Baseline aprobado; encaje verificado | Usar `@nestjs/schedule` global existente como disparador; el claim sigue pendiente de diseño. |
| ADR-SC-03 | Resolver preview y audiencia incremental en backend. | Baseline aprobado | Angular recibe conteos y muestra acotada; se requiere contrato corporativo. |
| ADR-SC-04 | Deduplicar por Campaign + Purchase. | Baseline aprobado | No deduplicar solo por cliente; confirmar clave de compra en fuente. |
| ADR-SC-05 | Reclamar trabajo con mecanismo transaccional compartido. | Propuesta técnica | Seleccionar locking, lease o alternativa con ADR y validar en despliegue multi-réplica. |
| ADR-SC-06 | Integrar envío transaccional y eventos con Emblue. | Baseline aprobado | Adaptador desacoplado; detalles de contrato, retries e idempotencia OPEN. |
| ADR-SC-07 | Correlacionar respuestas con token opaco, sin PII en URL. | Objetivo aprobado; mecanismo OPEN | Revisar seguridad, expiración, almacenamiento y enlace al flujo público existente. |
| ADR-SC-08 | Persistir historial final de ejecuciones como inmutable y conservar campañas con historial. | Baseline aprobado | Auditoría y consulta; definir política de datos personales/retención. |

## 14. Riesgos y mitigaciones

| Riesgo | Impacto | Mitigación propuesta |
|---|---|---|
| Esquema/frescura de CF/Analytics no disponible | No se puede calcular audiencia ni asegurar incrementalidad | Obtener contrato, propietario, semántica, permisos, claves y SLA antes de implementar el adapter. |
| `purchaseId/transactionId` no estable o no único | Duplicados o pérdida de elegibilidad | Acordar clave de negocio y constraint de idempotencia con el propietario de datos. |
| Emblue no ofrece idempotencia o estado por elemento | Reenvíos o conteos inconsistentes ante timeout | Confirmar contrato; diseñar tabla/outcome y reconciliación antes de habilitar producción. |
| Scheduler corre en cada réplica | Dos envíos de una misma ejecución | Claim compartido atómico y unicidad de `campaignId + scheduledAt / executionKey`; probar con réplicas concurrentes. |
| Caída durante el procesamiento de solicitudes transaccionales | Estado parcial o destinatarios ambiguos | Persistir progreso por recipient, idempotencia y recuperación sin marcar audiencia pendiente como procesada ante falla total. |
| Cambios tardíos en fuente o compras atrasadas | Elegibles omitidos o procesados dos veces | Evaluar watermark, lookback u optimizaciones equivalentes en DATA/SCHED con el contrato real de Analytics; no son requisito funcional confirmado. |
| Atribución insegura | Exposición de clientes o respuestas asociadas incorrectamente | Token opaco, validación de servidor, controles anti-replay y revisión de seguridad. |
| PII en observabilidad/retención excesiva | Riesgo de privacidad y cumplimiento | Minimización, redacción de logs, acceso controlado y política de retención acordada. |
| `PUBLISHED` no coincide con un estado general de Survey | Campaign debe usar su regla aprobada status=1, isPublic=true, archived=false; no redefinir status global. | Elegibilidad Campaign |

## 15. OPENs y validaciones necesarias

| OPEN | Información requerida | Responsable sugerido |
|---|---|---|
| OPEN-01 Fuente CF/Analytics | Contrato/API o vistas, propietario, conexión, permisos, columnas, frecuencia de actualización y límites. | Equipo de datos/Analytics |
| OPEN-02 Identificador de compra | Nombre, tipo, estabilidad, unicidad y relación entre compra, cliente, email y jerarquías. | Equipo de datos/negocio |
| OPEN-03 Elegibilidad temporal | Zona horaria, corte, lookback, datos tardíos y significado de `ExecutionDate`. | Negocio/datos |
| OPEN-04 Emblue | API transaccional, autenticación, plantillas, tamaños, rate limits, respuesta por destinatario e idempotency key. | Equipo PEC/Emblue |
| OPEN-05 Eventos Emblue | DataLab/Webhooks, firma, eventos, claves, orden, retries y deduplicación. | Equipo PEC/Emblue |
| OPEN-06 Recovery/retries | Semántica funcional cerrada en 2.9.1: hasta 3 intentos automáticos de retry además del inicial, para transitorios confirmados, backoff exponencial+jitter, no retry ambiguo/permanente y terminales inmutables. OPEN: clasificación real de errores, valores base/tope, lease y reconciliación provider. | PEC/operación/Emblue |
| OPEN-07 Atribución | Diseño del token, expiración, almacenamiento, consumo/reuso y cambios en public survey. | Seguridad/equipo PEC |
| OPEN-08 Roles | El acceso Campaign esta aprobado solo para ADMIN/SUPERADMIN efectivos; siguen pendientes los permisos especificos para recipients y operaciones no cubiertas. | Producto/seguridad |
| OPEN-09 Timezone/calendario | Cerrado parcialmente: Ecuador usa `America/Guayaquil` como zona predeterminada y se persiste explícitamente sin inferir el navegador; End Date es inclusiva local, el slot exacto a Send Time es válido y no hay posteriores; MONTHLY/QUARTERLY usan ancla StartDate y ajuste al último día sin drift. Permanecen OPEN conversión local→UTC, DST y efectos de recovery/downtime/reanudación sobre End Date. | Negocio/arquitectura |
| OPEN-10 Retención | Retención de destinatarios, compras procesadas, eventos y respuestas vinculadas. | Seguridad/privacidad/negocio |
| OPEN-11 Volumen y operación | Audiencia máxima, ventana de envío, lotes, concurrencia, límites y alertas. | Negocio/operaciones/Emblue |
| OPEN-12 Acceso Survey | Confirmar uniformidad de zona local del proceso API y si las respuestas deben seguir abiertas despues de availableTo; no reabre la regla de elegibilidad Campaign aprobada. | Equipo Surveys |

## 16. Estrategia incremental de implementación

Estas fases expresan secuencia arquitectónica, no tareas técnicas ni autorización de implementación.

1. **Cerrar contratos y decisiones:** obtener fuente de audiencia, claves de compra, contrato Emblue, roles, reglas temporales pendientes (conversión local→UTC, DST y recovery) y atribución. La zona predeterminada Ecuador, End Date inclusiva y regla MONTHLY/QUARTERLY están decididas; documentar las decisiones que cambien interfaces o persistencia.
2. **Definir modelo y API:** diseñar entidades/restricciones, transiciones, contratos de preview y administración, outcomes e idempotencia. Revisar límites de datos y migraciones antes de implementarlos.
3. **Completar administración UI/API:** crear/listar/editar campañas, estados y detalle con valores persistidos; preservar SurveyJS y disponibilidad de la encuesta.
4. **Resolver segmentación y preview:** integrar fuente corporativa de solo lectura, filtros jerárquicos, offset, conteos y muestra limitada; verificar no exponer audiencia completa.
5. **Implementar scheduler y ejecución:** claim multi-réplica, idempotencia, procesamiento interno por lotes si se requiere, persistencia de progreso, recovery y próximo calendario. El envío externo individual o batch queda sujeto al contrato Emblue. Validar fallas parciales y totales.
6. **Integrar Emblue y eventos:** enviar por ambiente no productivo, validar callbacks, correlación, límites y política de retries.
7. **Atribución, seguridad y operación:** integrar token con respuesta pública, roles, retención, logs sanitizados, métricas, alertas y procedimientos operativos.
8. **Habilitación gradual:** ejecutar con audiencia y credenciales sintéticas, reconciliar conteos, pilotear volumen controlado y habilitar producción tras aprobación de negocio/seguridad/operaciones.

## 17. Criterios arquitectónicos de aceptación

- Campaign y Survey permanecen separadas; se referencia la encuesta existente y no se crea editor alternativo.
- La persistencia y los contratos propuestos cubren estados, acciones, recurrencias y transiciones del baseline sin cambiar su significado.
- Preview y ejecución aplican la misma segmentación, offset, clave de deduplicación y exclusión de compras ya procesadas; la audiencia completa no llega a Angular.
- Una misma ejecución programada (`campaignId + scheduledAt / executionKey`) no puede ser reclamada/procesada simultáneamente por varias réplicas; un claim abandonado puede recuperarse sin duplicar envíos confirmados.
- La recuperación conserva compras pendientes tras falla total y soporta resultados parciales trazables sin marcar audiencia pendiente como procesada.
- Cada envío puede correlacionarse con Campaign, Execution y Recipient mediante referencia segura, sin PII ni claves de compra expuestas en URL.
- El adaptador Emblue y la integración de eventos tienen contrato, autenticación, manejo de límites, idempotencia, errores y retries definidos antes de producción.
- La información personal está minimizada, autorizada, protegida y ausente de logs/payloads diagnósticos no sanitizados.
- El historial de ejecución y los estados finales son auditables e inmutables; una Campaign con historial no se elimina físicamente.
- Los estados de Survey y roles PEC se mapean al comportamiento real del repositorio y no quedan como supuestos en producción.
- Métricas, alertas y métricas externas se limitan a señales soportadas y acordadas; no se presentan aperturas/clics no verificados.

## 17.1 Decisiones de scheduler/recovery v1 y brecha actual

Políticas APPROVED: consolidar slots vencidos elegibles en una sola recovery por Campaign; backlog automático inicial de 30 días configurable, con alerta/intervención al excederlo y sin descartar datos; Suspend bloquea claims/lotes nuevos y deja drenar el lote iniciado; Resume antes de EndDate consolida recovery y calcula el siguiente slot futuro, después de EndDate solo recupera slots originalmente válidos. Campaign solo es COMPLETED cuando terminó el calendario y no hay trabajo pendiente, en curso o recuperable. Execution terminales son inmutables; retries selectivos se modelan con nuevas Executions relacionadas, hasta 3 intentos automáticos de retry además del inicial para errores transitorios confirmados y backoff exponencial+jitter. Ambiguo/permanente no se reintenta. `lastExecutionAt` refleja `finishedAt` de la terminal más reciente, incluido FAILED.

Diseño recomendado: clave determinista/versionada para slot, rango consolidado y retry; claim atómico en transacción corta; lease con generación de fencing validada antes de cada dispatch/escritura; snapshot inmutable con slots cubiertos y configuración temporal. No mantener transacción abierta en llamadas externas. La solución concreta row-lock/CAS y parámetros operativos requieren validación de versión Postgres/Prisma y carga.

**Brecha del checkout inspeccionado previo al Sprint 3.0:** Prisma ya contiene `executionKey` único por Campaign, `recovery`, FK singular `recoveryOfExecutionId`, lease (`claimOwner/claimedAt/claimExpiresAt`), `nextExecutionAt`, `lastExecutionAt` y ledger único Campaign + purchase. En esa inspección no se encontró scheduler ni adapter Emblue; no hay fencing generation, programación retry persistida ni estado Attempt inequívoco para resultado ambiguo/aceptación/procesamiento/entrega. El servicio actual carece de comandos Schedule/Suspend/Resume y no se observó uso optimista de `version`. Estos campos/mecanismos requieren diseño de cambio Prisma/API en sprint posterior; no se aplican aquí.

**Evidencia Emblue reportada por el usuario:** campaña de integración y acción Email activa; `actionId=6388219`; prueba manual Postman con `POST https://api.embluemail.com/v2.3/emails/send` recibió correo. Esto valida únicamente una prueba individual externa reportada. No prueba integración NestJS, asíncrono, idempotencia, reconciliación ni recuperación. API acceptance, processing y entrega son hechos separados. No se documentan secretos ni se asume una garantía de Emblue.

**Actualización Sprint 3.0:** el checkout incluye `EmblueModule` y `EmblueService.sendEmail()` para el contrato de envío v2.3 confirmado: `POST /emails/send`, body `{ sendRequest: { to, html, actionId, subject } }` y respuesta `{ activityId: string }`. Pasaron las 21 pruebas unitarias del servicio y el build NestJS. Además, el usuario ejecutó el script manual NestJS: una solicitud devolvió HTTP 200 y un `activityId`. Esto demuestra aceptación de esa solicitud por la API; no verifica procesamiento posterior ni entrega final. No están implementados `findActivity()`, idempotencia persistida, scheduler ni conexión del adapter al flujo de campañas. No se registra el destinatario ni el identificador de actividad.

**Actualización Sprint 3.1A:** el checkout incorpora `CampaignExecutionService`, registrado/exportado únicamente en `SurveyCampaignsModule`; no añade rutas públicas y no invoca `EmblueService`. Se agregaron localmente `claimGeneration` y campos de outcome/attempt/activityId; la lista de slots originales se conserva en `configurationSnapshot`. La migración es aditiva y está sin aplicar. Build NestJS y 13 pruebas unitarias mock pasan; `prisma validate` pasa. Prisma Client no pudo regenerarse por `EPERM` al reemplazar el engine Windows. No hay variable `CAMPAIGN_TEST_DATABASE_URL`, por lo que no se verificó PostgreSQL real ni concurrencia interproceso. No se inicia scheduler ni envíos.

## 18. Estado del documento

- **Status:** DRAFT
- **Version:** 0.5
- **Validación técnica:** Se inspeccionó la estructura actual del checkout para identificar módulos de encuestas, Prisma/PostgreSQL, Angular y `@nestjs/schedule`.
- **Pendiente:** Validar contratos externos, decisiones OPEN y encaje final con los responsables de PEC, Analytics, Emblue, seguridad y operación.
- **Aprobación formal:** pendiente; esta actualización no cambia DRAFT.
- **Validación PostgreSQL:** Sprint 2.4 pendiente.

## 19. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Se establece `CampaignRecipient` como obligatorio a nivel de dominio con consulta de resultados por Execution; se neutraliza la forma de solicitud transaccional a Emblue y se deja OPEN si es individual o batch; se expresa el claim por Campaign + ejecución programada (`campaignId + scheduledAt / executionKey`); se aclara que watermark/lookback son optimizaciones evaluables y no requisitos funcionales, y que una falla total no marca audiencia pendiente como procesada; se agregan Search por `Campaign Name`/`Survey Name` y Filter por `Status` al Dashboard, conservando cards y columnas. |
| 0.3 | 2026-10-20 | Se incorporan decisiones multiempresa y roles, separación Enterprise/Company, catálogos MOCK sintéticos reemplazables y sus límites, estado OPEN de publicación y validación PostgreSQL pendiente. El documento permanece DRAFT. |
| 0.4 | 2026-10-20 | Se establece español obligatorio en UI, etiquetas centralizadas, preservación de valores técnicos, fechas y paginación localizadas. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Formaliza zona predeterminada `America/Guayaquil` y End Date inclusiva local; delimita los OPEN temporales restantes sin cambiar el estado DRAFT. |
| 0.4 | 2026-10-21 | Registra la política aprobada de ancla StartDate y ajuste MONTHLY/QUARTERLY al último día disponible sin drift acumulativo, también en año bisiesto. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Formaliza políticas de scheduler/recovery, diseño de claim/fencing y brechas Prisma/API; registra prueba manual Emblue reportada con alcance limitado. No implementa ni activa ejecución. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra el contrato de envío emBlue API v2.3 confirmado, el adapter NestJS y 21 pruebas unitarias; mantiene consulta de actividad, idempotencia persistida, ejecución de campañas y entrega final como pendientes. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra una ejecución manual NestJS reportada por el usuario con HTTP 200 y `activityId`; solo acredita aceptación API, sin inferir entrega ni registrar destinatario/ID. Estado DRAFT conservado. |

### Validación Sprint 3.1A.1 — base de desarrollo (2026-10-23)

La consulta READ ONLY confirmo que el SQL 20261023000000_add_campaign_execution_controls coincide con el esquema y no figura en _prisma_migrations. El DDL ya fue aplicado manualmente segun confirmacion del usuario; bajo Database-First no requiere reconciliar el historial Prisma.

La fila incompleta 20250907231907_add_user_establishment_table es un dato historico de _prisma_migrations; no se modifico. Bajo Database-First no bloquea Prisma Client ni las operaciones DDL manuales; el proceso acordado no ejecuta migrate deploy.


### Convención Database-First y despliegue (2026-10-23)

Esta sección es la referencia vigente y supersede notas anteriores que trataran la ausencia de filas en _prisma_migrations como bloqueo o recomendaran prisma migrate resolve. PostgreSQL es la fuente de verdad. El DDL se prepara como SQL revisable y se ejecuta manualmente; schema.prisma se sincroniza con el esquema aplicado y prisma generate actualiza el cliente. No se ejecutan migrate deploy/dev/resolve, db push ni migrate reset como mecanismo de despliegue de esquema.

Inspección del checkout: Dockerfile genera Prisma Client durante build. docker-entrypoint.sh también intenta generarlo al iniciar. Ese entrypoint solo corre migrate deploy si RUN_MIGRATIONS=true. El template Helm revisado no define esa variable ni reemplaza CMD start:prod; con esos valores omite DDL. Overrides externos deben verificarse aparte. El DDL de los controles de ejecución se mantiene en prisma/manual-sql/campaign_execution_controls.sql; la migración inicial `20261018000000_add_automated_survey_campaigns` continúa en prisma/migrations y podría ser considerada si se habilitara migrate deploy.

La migración no registrada en _prisma_migrations y el registro incompleto preexistente de 20250907231907_add_user_establishment_table son datos históricos; no bloquean Prisma Client bajo el modelo aprobado. Prisma generate finalizó correctamente en la verificación posterior; build y 84 pruebas unitarias Campaign pasaron con el cliente generado. Pruebas PostgreSQL con escrituras siguen pendientes de entorno aislado. Estado DRAFT conservado.

### Sprint 3.1A.2 — aislamiento de integración y despliegue (2026-10-23)

La suite PostgreSQL de Campaign admite únicamente un servidor local loopback, base dedicada `pec_campaigns_test` y rol propietario `campaign_test`, comprobando además la identidad reportada por PostgreSQL antes de cualquier fixture. No se permite Neon ni el uso de `DATABASE_URL` como fallback. Las filas sintéticas se etiquetan con UUID y se limpian por clave primaria rastreada; no se ejecutan limpiezas globales.

El modelo de despliegue sigue siendo Database-First: el SQL se aplica manualmente y `prisma generate` genera el cliente. El chart Helm revisado no activa `RUN_MIGRATIONS`; sin embargo, `docker-entrypoint.sh` conserva la rama que ejecuta `prisma migrate deploy` si un override establece esa variable a `true`. No se alteró el comportamiento de despliegue. Recomendación pendiente de aprobación: retirar la rama de migración automática o rechazar explícitamente `RUN_MIGRATIONS=true` para impedir DDL accidental en bases administradas manualmente. Deben revisarse también overrides del release y pipelines, porque los valores actuales del chart no prueban su configuración efectiva en todos los entornos.

La concurrencia PostgreSQL aún no está comprobada: no existe URL configurada para la instancia local aislada y no se conectó a Neon. Build/unit tests no acreditan comportamiento multiworker real.

## Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.5 | 2026-10-23 | Añade política de aislamiento PostgreSQL para pruebas de Campaign y registra el riesgo pendiente de la ruta condicional `RUN_MIGRATIONS=true` bajo Database-First. No cambia despliegues. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra base PostgreSQL Docker aislada y validación real de concurrencia 3.1A; documenta IP interna Docker, carrera de `executionKey` corregida y riesgo `RUN_MIGRATIONS` pendiente. Estado DRAFT conservado. |

Aclaración del árbol actual: `prisma/migrations/20261018000000_add_automated_survey_campaigns/migration.sql` sigue existiendo en el checkout. El DDL adicional de controles está en `prisma/manual-sql/campaign_execution_controls.sql`, pero eso no elimina la migración inicial de Campaign ni el resto del historial. Si `RUN_MIGRATIONS=true`, Prisma podría intentar aplicar cualquier migración pendiente de esa cadena contra la base configurada. El riesgo refuerza la recomendación de bloquear la ruta automática bajo Database-First; no se cambió ni movió este archivo.

### Sprint 3.1A.3 — decisión operativa Database-First (2026-10-23)

El entorno actual no tiene PostgreSQL local disponible: Docker Desktop está instalado, pero `com.docker.service` está detenido y el cliente no accede al engine; PostgreSQL server/client no están instalados; 5432 está libre; la URL de pruebas no está definida. No se inició servicio, no se creó una base ni se conectó a Neon. El `docker-compose.yml` existente no es seguro como banco de prueba aislado porque lee `.env`, levanta NestJS, retiene un volumen y publica en 5435.

Para validar Campaign se requerirá autorización explícita para iniciar Docker Desktop y crear un contenedor local separado con bind `127.0.0.1:5432`, base/rol propietario `pec_campaigns_test`/`campaign_test`, volumen propio y credenciales sintéticas temporales. Después se verificará la identidad SQL del servidor antes de configurar la URL solo en el proceso de Jest. No aplicar migraciones automáticamente ni importar datos reales. Debe prepararse primero un SQL bootstrap completo y revisable compatible con el esquema requerido; el SQL manual de controles por sí solo no crea las tablas PEC base.

Corrida de regresión actual: ningún test Campaign aparece entre las 23 suites fallidas; las pruebas Campaign seleccionadas pasan 90/90. Sin una corrida comparable del commit anterior a Campaign, no se puede afirmar que los 61 fallos restantes sean preexistentes o causados por cambios de este módulo.

Bajo Database-First se recomienda retirar del entrypoint la ejecución condicional de `prisma migrate deploy` y fallar explícitamente si se intenta activar `RUN_MIGRATIONS=true`; `prisma generate` no ejecuta DDL y debe permanecer independiente. Añadir verificación CI de overrides y release values. Esta protección requiere un cambio posterior aprobado; no se realizó aquí.

Estado Sprint 3.1A: pendiente hasta validar concurrencia real PostgreSQL. SPECS v0.5 / DRAFT conservado.

### Sprint 3.1A.3 — entorno aislado comprobado (2026-10-23)

Se creó el contenedor local dedicado `pec-campaign-test-db-sprint313` desde `postgres:15.4-alpine`, sin volumen, sin política de reinicio y con puerto ligado exclusivamente a `127.0.0.1:5432`. La base `pec_campaigns_test` es propiedad de `campaign_test`. El esquema se derivó del `schema.prisma` actual y se aplicó solo a esta base vacía mediante SQL generado; no se ejecutó Prisma Migrate, no se modificó el contenedor de pruebas anterior, Neon, ni otra base.

El servidor dentro de Docker reporta una IP privada RFC1918 aunque la conexión de la aplicación use `127.0.0.1`. El guard mantiene la validación de host URL loopback, nombre de base, puerto, esquema, rol y propietario, y acepta para `inet_server_addr` loopback o una dirección privada Docker. Se añadieron pruebas de rechazo de IP pública.

La concurrencia real validó CAS/fencing, leases, unicidad de `executionKey`, terminales inmutables, progreso, retry selectivo y recovery consolidada. La carrera de creación concurrente reveló que `upsert` podía devolver `P2002`; Campaign ahora consulta la fila existente al recibir esa colisión y devuelve al ganador, con cobertura unitaria y PostgreSQL. El fixture utiliza registros sintéticos con UUID, crea Country/Enterprise/User/Survey con sus campos requeridos y los borra por IDs rastreados; la verificación posterior encontró cero registros de prueba.

Resultado de cierre Sprint 3.1A: 15/15 pruebas PostgreSQL, 108/108 pruebas Campaign, build NestJS y ESLint focalizado aprobados. Esto permite cerrar la validación de infraestructura de persistencia/concurrencia de Sprint 3.1A en ambiente local; no habilita scheduler ni envíos y no inicia Sprint 3.1B. La suite general sigue con 61 fallos distribuidos en módulos ajenos; sin baseline comparable no se clasifican como preexistentes.

La rama `RUN_MIGRATIONS=true` sigue siendo un riesgo de DDL automático para Database-First. El chart revisado no la habilita por defecto, pero no se verificaron overrides de despliegue reales. Se recomienda un cambio posterior que la rechace/elimine y un guard CI; no se modificó despliegue en esta fase.

### Sprint 3.1B — arquitectura implementada localmente (2026-10-23)

`CampaignLifecycleService` coordina transiciones bajo `SELECT ... FOR UPDATE` de Campaign y cambios CAS/versionados. Rutas autenticadas/CASL: `PATCH /survey-campaigns/:campaignId/schedule`, `/suspend`, `/resume`; `GET /survey-campaigns/:campaignId/executions`. Schedule/Resume fallan cerrados con blockers explícitos para Survey PUBLISHED no comprobado, audiencia real inexistente, catálogos MOCK y plantilla no verificada. `CampaignSchedulerService.runOnce()` no tiene Cron ni endpoint y usa `dryRun=true` por defecto. No llama a emBlue ni crea audiencia/recipients. No hay cambios Prisma/DDL ni Helm. En la sesión de cierre, build, lint focalizado y 108 pruebas unitarias Campaign pasaron; la suite de integración marcó 17 casos como skipped por falta de PostgreSQL local/Docker accesible. Neon no se utilizó. SPECS permanecen 0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra lifecycle y motor invocable manualmente, sin activación automática; validación PostgreSQL de Sprint 3.1B pendiente en el entorno de esta sesión. DRAFT conservado. |

### Sprint 3.1B.1 — entorno de validación no disponible (2026-10-23)

La inspección no pudo confirmar la existencia ni configuración del contenedor `pec-campaign-test-db-sprint313`: Docker CLI devolvió `permission denied` al conectar con `npipe:////./pipe/docker_engine`, `com.docker.service` está detenido con inicio Manual y no hay listener local en 5432. No se inició el servicio, no se inspeccionaron datos del contenedor, no se usó PostgreSQL del proyecto y no se contactó Neon. Para habilitar la corrida, el usuario puede abrir Docker Desktop manualmente; después se verificará solo lectura el contenedor y la identidad SQL antes de que el proceso Jest ejecute escrituras sintéticas y su limpieza por IDs. No ejecutar `docker start` por el agente.

| 0.5 | 2026-10-23 | Registra que el entorno PostgreSQL aislado no fue inspeccionable en esta sesión; no se iniciaron ni alteraron servicios/containers. DRAFT conservado. |

### Sprint 3.1B.1 — validación PostgreSQL completada (2026-10-23)

El contenedor existente `pec-campaign-test-db-sprint313` estaba activo y no fue iniciado ni modificado por el agente. Imagen `postgres:15.4-alpine`, política restart `no`, volumen Docker propio y publicación `127.0.0.1:5432`. Consultas de solo lectura verificaron `pec_campaigns_test`, esquema `public`, usuario y propietario `campaign_test`; diez tablas Campaign y siete tipos Campaign presentes. La suite de integración conectada desde el host pasó la validación de identidad del guard PostgreSQL, incluido el origen Docker privado. La aplicación se conectó solo a esta base aislada.

La suite PostgreSQL pasó 20/20 y la suite Campaign completa 128/128 (11 suites). El único fallo inicial fue un instante fijo de claim ya vencido en el test Suspend; se ajustó el fixture a tiempos relativos y la repetición pasó. Consultas posteriores confirmaron cero filas en Campaign, Execution, Recipient, Attempt, ResponseCorrelation y fixtures sintéticos de Survey, Enterprise, User y Country. No se usaron Neon, emBlue, scheduler automático ni despliegues. Sprint 3.1B queda técnicamente validado en PostgreSQL aislado; las barreras funcionales para habilitar Schedule/Resume permanecen vigentes. SPECS v0.5 / DRAFT.
## Registro de validación Sprint 3.1C

Inspección del checkout del 2026-10-23: Campaign conserva un preflight de solo lectura en `GET /survey-campaigns/:campaignId/preflight`, protegido por CASL `Read Campaign`. El preflight informa como verificables la configuración temporal, los filtros obligatorios y que Survey esté activa en PEC y no archivada; mantiene en `OPEN` la publicación y la construcción del enlace individual. Audiencia real y plantilla permanecen bloqueadas. El proveedor Analytics continúa siendo un stub sin fallback automático. No se activó Cron, scheduler ni envío emBlue.

| 0.5 | 2026-10-23 | Registra diagnóstico e implementación fail-closed de preflight Campaign; publicación, URL individual, audiencia y plantilla permanecen sin contrato verificable. DRAFT conservado. |
## Sprint 3.1C.1 — contratos propuestos, no integrados

La inspección del checkout no encontró un flujo de publicación dedicado para Survey. `CreateSurveyDto` permite recibir `status` e `isPublic`; `UpdateSurveyDto` permite cambiar `status` entre 0/1 y conserva `isPublic`. No se identificó transición, auditoría o permiso de publicación con semántica propia. La ruta anónima existente no equivale por sí sola a publicación aprobada de Campaign.

Se propone mantener puertos internos `CampaignAudienceProvider` y `CampaignTemplateProvider` separados de sus adaptadores; sus interfaces en `pec-api/src/survey-campaigns/contracts/campaign-integration.contracts.ts` no representan contratos externos. `SurveyPublicationValidator` ya es un proveedor Campaign registrado y aplica la decisión aprobada sobre campos existentes. El `CampaignCatalogProvider` existente se conserva como puerto de catálogos; Analytics sigue siendo stub.

| 0.5 | 2026-10-23 | Documenta publicación configurable mediante campos existentes sin flujo dedicado y registra puertos de integración solo propuestos. Dependencias y responsables funcionales siguen OPEN; DRAFT conservado. |


## Sprint 3.1C.1 - publicacion de Survey para Campaign

Regla funcional aprobada para Automated Survey Campaigns: una Survey es elegible solo si `status = 1`, `isPublic = true` y `archived = false`. `status=1` aislado no demuestra publicacion ni cambia el significado global de Survey. La ruta publica existente conserva su comportamiento, incluido el fallback para Surveys genericas; Campaign exige `isPublic=true` y no utiliza ese fallback.

El gate lee `content.availableFrom` y `content.availableTo` si estan configurados. Valida formato e intervalo y verifica que el calendario Campaign completo, desde inicio local hasta fin local inclusivo, quede cubierto. Usa la zona IANA de Campaign para el calendario y el parser server-local ya usado por la ruta publica para interpretar los limites Survey. Mantener TZ consistente entre replicas del API es un requisito operativo; acceso a respuestas despues de `availableTo` y enlace individual Campaign siguen pendientes.

Preflight reporta `SURVEY_PUBLICATION=READY` unicamente con la conjuncion aprobada; `SURVEY_CALENDAR_AVAILABILITY` distingue READY, BLOCKED y OPEN. Schedule y Resume vuelven a validar antes de mutar estado. No se modifica Prisma, no se cambia la ruta Surveys, y audiencia real, catalogos reales, plantilla y enlace individual siguen bloqueando la activacion. Se conserva v0.5 / DRAFT.

## Sprint 3.1C.2 - servicio de elegibilidad Survey

El módulo Campaign registra `SurveyPublicationValidator`, que centraliza la regla aprobada y se reutiliza desde preflight y validaciones operativas de Schedule/Resume. No altera las rutas ni el comportamiento global de Surveys. La publicación puede estar CONFIRMED aunque el check de disponibilidad resulte BLOCKED u OPEN; el enlace individual se conserva como OPEN y las integraciones reales pendientes siguen cerrando el gate operativo. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra regla Campaign de Survey, gate temporal y cobertura de pruebas; DRAFT conservado. |

## Sprint 3.1C.3 - fuentes de catálogo y audiencia encontradas

La única integración cercana identificada es la sincronización existente desde `BASE_API_URL` para áreas de trabajo y locales, que copia campos a catálogos PEC. No es resolver de audiencia ni confirma la jerarquía Division→Department→Company→Format→Store requerida por Campaign. `AnalyticsCampaignCatalogProvider` sigue siendo una clase stub sin transporte. No existe fuente real de compras/destinatarios localizada en `pec-api/src` o `prisma/schema.prisma`; `CampaignAudienceProvider` permanece solo como port propuesto. No conectar estos servicios ni usar consultas PEC de clientes individuales como sustituto de una audiencia. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Actualiza arquitectura con la integración PEC cercana y los límites confirmados frente a Analytics/Data-Lab. DRAFT conservado. |
## Sprint 3.1C.4 - propuesta de integracion BI PostgreSQL

Propuesta aun no confirmada por BI CF: dos tablas snapshot en `bi_integration`, descritas en `pec-api/prisma/manual-sql/bi_campaign_staging_proposal.sql`. No se crea infraestructura ni adaptador productivo en este sprint. El contrato de proveedor debe seguir aislado tras `CampaignCatalogProvider` y `CampaignAudienceProvider`; el origen BI no debe presentarse como `ANALYTICS` hasta confirmar la taxonomia de `source` actual (`MOCK | ANALYTICS`). No depender de la empresa activa PEC ni equiparar Enterprise PEC con Company CF.

OPEN: esquema/propietario real, grano de compra, IDs, catálogos y jerarquías, control de lote completo (dos tablas no señalan de forma robusta una carga vacía; se requiere protocolo atomico y posiblemente manifiesto), acceso de solo lectura, PII, frescura y elegibilidad. Schedule/Resume/dispatch siguen bloqueados. El DDL es propuesta revisable, no ejecutada; v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra arquitectura BI staging propuesta y dependencias abiertas; sin cambio de codigo/esquema ni aprobacion formal. DRAFT conservado. |
## Sprint 3.1C.4.1 - grano de factura

La regla funcional aprobada fija una invitacion por Campaign e identidad de factura. El adaptador futuro agrupara las filas de factura-linea-producto-division del staging y emitira un unico Recipient si una o mas asignaciones califican. La constraint unica existente en `CampaignRecipient` ya expresa el alcance por Campaign; no requiere cambio de modelo. Se conserva `load_id` como evidencia de fuente en el snapshot de Execution. Contactabilidad y reuso de factura en diferentes Campaigns permanecen OPEN. v0.5 / DRAFT.
## Sprint 3.1D - proveedor provisional PostgreSQL

Se implemento `CampaignPostgresStagingProvider` detras de los ports de catalogos y audiencia Campaign. Solo se selecciona explicitamente con `CAMPAIGN_CATALOG_PROVIDER=postgres-provisional` y `CAMPAIGN_AUDIENCE_PROVIDER=postgres-provisional`, junto con `CAMPAIGN_PROVISIONAL_BI_ENABLED=true`; no hay fallback automatico. Se rechaza en produccion y el proveedor valida en runtime que la conexion corresponda a `pec_campaigns_test`, esquema `public`, rol/propietario `campaign_test`, puerto 5432 y direccion local.

El DDL y seed se ejecutaron exclusivamente en el PostgreSQL local aislado mediante `pec-api/prisma/manual-sql/bi_campaign_staging_provisional_local_test.sql` y `bi_campaign_staging_provisional_seed.sql`. No se agregaron modelos Prisma ni se modifico el ledger. La fuente se identifica `PROVISIONAL_POSTGRES`, no BI/Analytics confirmada. No se activo scheduler, Cron ni envio; Schedule/Resume siguen bloqueados. Estado DRAFT conservado.

| 0.5 | 2026-10-23 | Registra proveedor provisional y guardas locales de Sprint 3.1D; DRAFT conservado. |

## Sprint 3.1E - invitación y correlación aislada (2026-10-23)

Se incorpora `CampaignInvitationCorrelationService` para emitir tokens opacos de 256 bits, persistir solo SHA-256 en `ResponseCorrelation`, validar expiración/consumo y asociar una respuesta existente una sola vez a su Survey Campaign. Se registra como servicio interno y no se conecta a rutas públicas, scheduler ni Emblue. La plantilla `PEC-CAMPAIGN-SYNTHETIC-INVITATION` es un fixture HTML en español para pruebas; no es plantilla aprobada ni se envía. DDL/seed de Neon: ejecución reportada por el usuario, no verificada independientemente en esta entrega. Persisten OPEN el enlace público individual y el submit correlacionado. No hay cambios Prisma/SQL. v0.5 / DRAFT.
## Sprint 3.1F - Flujo de invitación Campaign hacia Survey (2026-10-23)

Se añade una frontera pública exclusiva CampaignInvitationsController con GET /campaign-invitations/:token y POST /campaign-invitations/:token/responses. Angular reutiliza el mismo FormComponent/SurveyJS mediante la ruta pública /campaign-invitation/:token; el formulario normal conserva sus rutas y guardado actuales. La correlación no se añade a SurveysController ni usa local o idD. El guardado de la respuesta y el consumo del token se ejecutan en una transacción. No hay invocación de emBlue ni scheduler. v0.5 / DRAFT.

## Sprint 3.1F.1 - atomicidad y alcance de Survey v2 (2026-10-24)

La respuesta de invitación Campaign para Survey versión 1 guarda SurveyResult y consume ResponseCorrelation en una transacción; la carrera se protege con compare-and-set. La prueba de unidad cubre el callback transaccional y el rechazo de quien pierde el consumo, pero la prueba PostgreSQL no se ejecutó en esta validación, por lo que rollback real y concurrencia en base permanecen sin evidencia actual.

Survey versión 2 sigue excluida del flujo Campaign. El guardado general crea primero SurveyResult y después llama `ComplaintsService.createQuejaEncuesta`; esa operación consulta contactos mediante ClientService, crea la queja con un acceso independiente a DatabaseService, puede subir adjuntos y actualiza SurveyResult en pasos separados. El servicio general captura errores de creación de queja y aun así devuelve el SurveyResult. No se puede incluirlo en la transacción Campaign sin refactorizar sus límites y efectos. Recomendación: mantener v2 bloqueada hasta diseñar escritura atómica de DB y efectos externos posteriores al commit, sin quitar ni omitir la creación de la queja. SPECS v0.5 / DRAFT.
## Sprint 3.1F.1 - frontera de compatibilidad de Survey (2026-10-24)

Se centraliza la compatibilidad Campaign en `CampaignSurveyCompatibilityValidator`: solo acepta `version === 1` con `content.isQueja !== true`; rechaza versión 2, valores desconocidos e inconsistencias. El guard se integra en selección, preflight/lifecycle y manejo de invitaciones. El guardado de respuestas Campaign continúa limitado a Survey v1 y a su transacción existente. No se modifica Surveys/Complaints ni se habilitan scheduler, Cron o emBlue. Las pruebas unitarias Campaign pasaron 157/157; no hubo validación PostgreSQL de concurrencia en esta ejecución. v0.5 / DRAFT.
## Sprint 3.1G-A - orquestación de simulación (2026-10-24)

Se agrega `CampaignExecutionSimulationService` y `GET /survey-campaigns/:campaignId/execution-simulation`, protegido con lectura Campaign. Solo opera fuera de producción, en campañas DRAFT y con catálogos/audiencia PostgreSQL provisionales disponibles. Consume páginas del port estable, verifica que `snapshotId` no cambie, deduplica en memoria por origen + factura y prepara HTML/asunto para cada correo en memoria usando la plantilla sintética. Las URLs por candidato usan tokens efímeros bajo `example.invalid`; no se persisten ni se devuelven. El endpoint no inyecta ni llama `EmblueService`, no crea Execution/Recipient/Attempt/ResponseCorrelation y devuelve únicamente conteos, plantilla sanitizada y bloqueos productivos. Schedule/Resume y despacho permanecen bloqueados. No se ejecutaron pruebas PostgreSQL ni se declara atomicidad. SPECS v0.5 / DRAFT.


## Sprint 3.1G-B - simplificación de conexión Campaign (2026-10-25)

La arquitectura Campaign reutiliza el `DatabaseService`/PrismaClient existente y la única `DATABASE_URL` de PEC. Las tablas provisionales se consultan con nombres cualificados `bi_integration.bi_segmentation_catalog` y `bi_integration.bi_customer_purchases`. En `ENV=local|development|test` y con `NODE_ENV` no productivo se seleccionan automáticamente los proveedores PostgreSQL provisionales; en producción o entornos no reconocidos se selecciona el proveedor no configurado. No hay fallback a MOCK. Los antiguos selectores de proveedor, bandera de habilitación y selector de destino quedan retirados. La verificación Project/Branch/Endpoint de Neon permanece obligatoria antes de la primera consulta funcional de la instancia; sus credenciales/identificadores son controles de identidad, no conexiones ni selectores de proveedor.

Para la validación independiente de rama se requieren solo Project ID, Branch ID, propietario esperado de la base y credencial de API Neon (`CAMPAIGN_PROVISIONAL_NEON_PROJECT_ID`, `CAMPAIGN_PROVISIONAL_NEON_BRANCH_ID`, `CAMPAIGN_PROVISIONAL_NEON_DATABASE_OWNER`, `CAMPAIGN_PROVISIONAL_NEON_API_KEY`). El endpoint se extrae del host de `DATABASE_URL`, y la base y el rol de esa misma URL; estos datos de identidad no seleccionan proveedor ni crean otra conexión. **Registro de revisión:** v0.5, 2026-10-25. Se registra la decisión de conexión única/esquemas separados y selección automática protegida. Estado formal `DRAFT` conservado.


## Sprint 3.1G-B.1 - auditoria de configuracion Campaign (2026-10-25)

Inspeccion local de nombres de variables, sin leer ni publicar valores: `pec-api/.env` y `.env.local` contienen `DATABASE_URL` y `ENV`. `AppModule` carga globalmente `.env.local`, `.env.preprod`, `.env.prod` y `.env`; el guard lee `process.env`. No se encontro `DIRECT_URL` ni una identidad Neon reutilizable en esos archivos. `PROJECT_ID_GCS` identifica GCP y `API_KEY_STAR` pertenece a otra integracion; no son sustitutos de identidad Neon. El inventario local no prueba variables inyectadas externamente al proceso desplegado.

En el checkout, `CampaignNeonTargetGuard` aun exige Project ID, Branch ID, propietario esperado y credencial API Neon. Ninguno aparece en los archivos locales. El error 503 se origina al faltar esa configuracion: el guard falla antes de llamar a la API de Neon y antes de leer el lote en `bi_integration`. Endpoint ID, base y rol se derivan del `DATABASE_URL`; no se requiere otra conexion ni un `DIRECT_URL`. No se modifico `DATABASE_URL`.

No se retiro la verificacion de rama: `ENV` distingue el entorno PEC, pero no identifica una rama Neon; el hostname solo permite derivar el endpoint. La verificacion API Project/Branch/Endpoint y la comprobacion SQL posterior (base, esquema, rol, propietario y primary) siguen siendo controles distintos. El mecanismo equivalente no puede configurarse con los datos que hoy existen en los archivos inspeccionados. Dato pendiente para una lectura segura: una fuente confiable, ya disponible para NestJS, que relacione la conexion actual con el Project ID y Branch ID de desarrollo y permita verificar esa relacion en Neon. Hasta resolverlo, la lectura funcional falla cerrada. Los SPECS permanecen v0.5 / DRAFT.
## Consolidación de arquitectura Campaign / Neon (2026-10-25)

Esta decisión consolida las reglas de conexión de Campaign y supersede las reglas anteriores de selección mediante `CAMPAIGN_CATALOG_PROVIDER`, `CAMPAIGN_AUDIENCE_PROVIDER`, `CAMPAIGN_PROVISIONAL_BI_ENABLED` y `CAMPAIGN_PROVISIONAL_DATABASE_TARGET`. Campaign reutiliza el `DatabaseService` de PEC y su `DATABASE_URL`; las consultas BI cualifican `bi_integration`. No se crea una conexión normal adicional ni hay fallback a MOCK. Los selectores históricos quedan como registro, no como configuración vigente.

La protección de producción y entornos desconocidos se conserva. `ENV`, `NODE_ENV`, el nombre de base, el rol y el hostname no prueban por sí solos la rama Neon. La arquitectura objetivo no requiere una llamada a la API administrativa de Neon en cada lectura funcional; el guard actual sí la exige antes de la primera consulta. No se simplifica hasta disponer de un control de destino confiable y comprobable. La configuración local inspeccionada no aporta Project ID, Branch ID ni credencial Neon API; los manifiestos Copilot inspeccionados declaran `ENV=dev` y `POSTGRES_*`, pero no documentan la vinculación de `DATABASE_URL` con una rama Neon autorizada.

Las lecturas de catálogos, preview y simulación son consultas funcionales de solo lectura, protegidas por autenticación/CASL y por privilegios efectivos de PostgreSQL. El proveedor no debe recibir permisos de escritura sobre las tablas BI. El permiso `SELECT` limita operaciones, pero no acredita que el destino sea una rama de desarrollo. Las campañas reales siguen bloqueadas por sus precondiciones funcionales; habilitar lecturas no las habilita.

Las pruebas que escriben requieren PostgreSQL aislado local o una rama Neon exclusiva de pruebas, nunca la rama Neon de desarrollo compartida. La URL de pruebas no es fallback de `DATABASE_URL`.

**Pendiente para autorizar lecturas Neon:** confirmar Project ID y Branch ID en Neon Console y la fuente segura ya disponible para que NestJS obtenga la credencial Neon API; no compartir el valor del secreto. Hasta entonces se conserva el cierre seguro del guard. Estado formal `DRAFT`; versión v0.5.

## Actualización funcional 2026-10-25 — Fuentes y correspondencia de segmentación

Fuentes implementadas: División, Departamento y Clasificación auxiliar se consultan desde `bi_integration.bi_segmentation_catalog`; Empresa, Formato y Local se consultan desde los modelos activos PEC `sspectenterprise`, `sspectformat` y `sspectestablishment`. Las relaciones PEC verificadas son Formato → Empresa mediante `enterpriseId` y Local → Formato mediante `formatId`. Las compras siguen en `bi_integration.bi_customer_purchases` y se consolidan por `(invoice_source, invoice_external_id)` antes de paginar.

El esquema PEC no contiene una columna de mapeo explícita a `company_external_id`, `format_external_id` o `store_external_id` de BI. La implementación solo acepta igualdad textual exacta entre IDs; no compara por nombre/código ni similitud. Si falta igualdad exacta, preview y operaciones dependientes deben bloquearse e informar la correspondencia faltante. Este estado no declara validado el mapeo BI.

`GET /survey-campaigns/:campaignId/audience-preview?page=N` devuelve conteos de clientes con identificador, facturas elegibles/contactables, registros excluidos por correo ausente/inválido y filas consolidadas. La página contiene etiquetas generadas para cliente/factura y fecha, sin correo, ID de cliente ni identificador real de factura. Preview es de solo lectura. Mantener el estado formal DRAFT y versión 0.5.

Los datos BI persistidos y el seed ya ejecutado no se modificaron. El reemplazo por ejemplos sintéticos representativos (taxonomía general de abarrotes/productos y clientes ficticios) requiere una operación de carga autorizada; no ejecutar ni modificar datos en esta entrega. OPEN: confirmar que los IDs externos BI coinciden con las PK PEC o entregar un mapeo autorizado, además de reglas de contactabilidad/exclusión.
