# Informe de Arquitectura - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-ARCH-001 |
| Estado | DRAFT |
| Versión | 0.2 |
| Fecha | 2026-10-18 |
| Idioma | Español; nombres de código en inglés |
| Baseline funcional | Requisitos aprobados entregados para este informe |

> Este documento describe una arquitectura propuesta y su encaje con el repositorio inspeccionado. No define todavía el schema Prisma definitivo, endpoints, tareas de implementación, código ni migraciones. Los puntos no comprobables se marcan como **OPEN**.

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
| Emblue | No se identificó un adaptador o contrato Emblue implementado en el checkout inspeccionado. | Búsqueda en `pec-api/src/` y dependencias |
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

## 4. Baseline funcional

Los siguientes requisitos se toman del baseline entregado para este documento. No son decisiones derivadas del código.

### 4.1 Survey y Campaign

- `Survey` y `Campaign` son conceptos separados. `Campaign` referencia una encuesta existente y no reemplaza SurveyJS.
- Una campaña puede guardarse en `DRAFT` aunque su `Survey` no esté publicada. Para programarla, la encuesta debe satisfacer el estado funcional `PUBLISHED`.
- `Available From` y `Available Until` continúan perteneciendo al módulo Survey.
- **OPEN:** confirmar cómo se representa y valida el estado `PUBLISHED` en el modelo/flujo vigente de encuestas.

### 4.2 Segmentación y elegibilidad

| Jerarquía | Filtro | Selección |
|---|---|---|
| Producto | `Division` | Requerido, multiselect |
| Producto | `Department` | Requerido, multiselect |
| Producto | `Auxiliary Classification` | Opcional, multiselect |
| Producto | `Auxiliary Subclassification` | Fuera de alcance v1 |
| Comercial | `Company` | Requerido, multiselect |
| Comercial | `Format` | Opcional, multiselect |
| Comercial | `Store` | Opcional, multiselect |

- Dentro de cada filtro, los valores seleccionados se combinan con OR; entre filtros distintos se combinan con AND.
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

`nextExecutionAt` se calcula con aritmética de calendario: `DAILY` +1 día, `WEEKLY` +7 días, `BIWEEKLY` +14 días, `MONTHLY` +1 mes calendario y `QUARTERLY` +3 meses calendario. `ONE_TIME` termina tras su ejecución única. `FREQUENCY` termina cuando la siguiente ejecución supera `End Date`.

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
3. Al programar, el API valida campos, ventana/frecuencia permitida, permisos y elegibilidad de Survey (`PUBLISHED`, sujeto a mapear el estado real).
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
| Fuente corporativa CF/Analytics | PEC lee divisiones, departamentos, compras y clientes/contactos | **OPEN:** motor/conexión autorizada, vistas/API, columnas, filtros, permisos, frescura, volumen, claves y significado de cada jerarquía. Implementar adaptador de lectura sin asumir nombres de vistas. |
| Emblue Transactional Email API | PEC → Emblue | **OPEN:** autenticación, endpoint, plantilla, tamaño/rate limit, respuesta por elemento, idempotencia, límites, estados y manejo de errores. |
| Emblue DataLab/Webhooks | Emblue → PEC | **OPEN:** eventos y campos disponibles, firma, reintentos del proveedor, orden, identificador único, política de replay y exposición de aperturas/clics/bounces. |
| PEC Surveys | Campaign → Survey existente | Reutilizar metadatos y URL/flujo existentes. Validar disponibilidad y estado de publicación sin duplicar SurveyJS. |
| PEC Public Survey | Invitación → respuesta | Incorporar correlación segura sin PII en URL; validar cambios mínimos requeridos al flujo público y modelo de respuesta. |

No registrar secretos de integración en código ni en documentos versionados. La configuración por ambiente debe alinearse con el mecanismo de secretos ya usado en despliegue, que debe confirmarse antes del diseño de configuración final.

## 10. Seguridad y protección de datos

- Aplicar permisos de PEC al listado, edición, acciones de scheduling/suspend/resume/cancel/delete, preview y consulta de destinatarios. **OPEN:** roles y permisos concretos.
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
| Frontend | Completar el módulo de campañas con dashboard y cards aprobadas, listado, filtros por estado, detalle, formulario en secciones, preview acotado e historial. | Existe navegación/shell de campañas, pero no se verificó integración persistente. Mantener patrones PEC y usar API como fuente de verdad. |
| Backend | Añadir módulo NestJS para Campaign, reglas, preview, scheduler/coordinación, eventos y adaptadores. | `SurveysModule` es existente; no se encontró módulo de campañas ni adaptador Emblue implementado. Rutas definitivas fuera de este documento. |
| Database | Persistir configuración, ejecuciones y estado idempotente de las compras/destinatarios procesados; restricciones e índices adecuados. | Prisma/PostgreSQL existentes; no se define schema ni migración aquí. Modelo exacto depende del contrato fuente y volumen/retención. |
| Fuente de datos | Lectura segura de catálogos y compras elegibles. | Contrato CF/Analytics no está disponible en el checkout. |
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
| `PUBLISHED` no coincide con representación actual | Programación inválida o campaña bloqueada | Mapear el estado aprobado al modelo real de Survey antes de diseñar validación final. |

## 15. OPENs y validaciones necesarias

| OPEN | Información requerida | Responsable sugerido |
|---|---|---|
| OPEN-01 Fuente CF/Analytics | Contrato/API o vistas, propietario, conexión, permisos, columnas, frecuencia de actualización y límites. | Equipo de datos/Analytics |
| OPEN-02 Identificador de compra | Nombre, tipo, estabilidad, unicidad y relación entre compra, cliente, email y jerarquías. | Equipo de datos/negocio |
| OPEN-03 Elegibilidad temporal | Zona horaria, corte, lookback, datos tardíos y significado de `ExecutionDate`. | Negocio/datos |
| OPEN-04 Emblue | API transaccional, autenticación, plantillas, tamaños, rate limits, respuesta por destinatario e idempotency key. | Equipo PEC/Emblue |
| OPEN-05 Eventos Emblue | DataLab/Webhooks, firma, eventos, claves, orden, retries y deduplicación. | Equipo PEC/Emblue |
| OPEN-06 Recovery/retries | Número de intentos, backoff, expiración/lease, resultados ambiguos y cuándo reabrir ejecución. | Negocio/operaciones |
| OPEN-07 Atribución | Diseño del token, expiración, almacenamiento, consumo/reuso y cambios en public survey. | Seguridad/equipo PEC |
| OPEN-08 Roles | Permisos existentes para administración, acciones operativas y destinatarios. | Producto/seguridad |
| OPEN-09 Timezone/calendario | Zona horaria definitiva, DST, fin de mes, Send Time y límites de End Date. | Negocio |
| OPEN-10 Retención | Retención de destinatarios, compras procesadas, eventos y respuestas vinculadas. | Seguridad/privacidad/negocio |
| OPEN-11 Volumen y operación | Audiencia máxima, ventana de envío, lotes, concurrencia, límites y alertas. | Negocio/operaciones/Emblue |
| OPEN-12 Publicación de Survey | Campo y transición reales que equivalen a `PUBLISHED`; validación de `availableFrom/availableTo`. | Equipo Surveys |

## 16. Estrategia incremental de implementación

Estas fases expresan secuencia arquitectónica, no tareas técnicas ni autorización de implementación.

1. **Cerrar contratos y decisiones:** obtener fuente de audiencia, claves de compra, contrato Emblue, roles, timezone, recuperación y atribución. Documentar las decisiones que cambien interfaces o persistencia.
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

## 18. Estado del documento

- **Status:** DRAFT
- **Version:** 0.2
- **Validación técnica:** Se inspeccionó la estructura actual del checkout para identificar módulos de encuestas, Prisma/PostgreSQL, Angular y `@nestjs/schedule`.
- **Pendiente:** Validar contratos externos, decisiones OPEN y encaje final con los responsables de PEC, Analytics, Emblue, seguridad y operación.
- **Artefactos deliberadamente no generados:** schema Prisma definitivo, endpoints definitivos, tasks, código y migraciones.

## 19. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Se establece `CampaignRecipient` como obligatorio a nivel de dominio con consulta de resultados por Execution; se neutraliza la forma de solicitud transaccional a Emblue y se deja OPEN si es individual o batch; se expresa el claim por Campaign + ejecución programada (`campaignId + scheduledAt / executionKey`); se aclara que watermark/lookback son optimizaciones evaluables y no requisitos funcionales, y que una falla total no marca audiencia pendiente como procesada; se agregan Search por `Campaign Name`/`Survey Name` y Filter por `Status` al Dashboard, conservando cards y columnas. |
