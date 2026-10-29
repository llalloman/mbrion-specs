# Especificación Funcional - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-FOUND-001 |
| Estado | DRAFT |
| Versión | 0.5 |
| Fecha | 2026-10-20 |
| Idioma | Español; nombres técnicos en inglés |
| Documento relacionado | [PEC-SC-ARCH-001.md](PEC-SC-ARCH-001.md) |

> Esta especificación describe **qué** debe hacer Automated Survey Campaigns. La arquitectura e implementación se rigen por `PEC-SC-ARCH-001`. Los comportamientos sin definición aprobada están marcados como **OPEN** y no se convierten en reglas implícitas.

## 1. Problema

PEC permite crear encuestas y recibir respuestas, pero el proceso de seleccionar compras/clientes elegibles y distribuir invitaciones de forma programada no está automatizado. El negocio necesita configurar campañas que reutilicen encuestas PEC, segmenten una audiencia y envíen invitaciones por correo transaccional mediante Emblue.

## 2. Objetivo

Permitir a usuarios autorizados crear, revisar, programar, suspender, reanudar, cancelar y consultar campañas de encuestas. Cada campaña debe aplicar los criterios de audiencia aprobados, procesar compras elegibles sin repetir una compra ya procesada por esa campaña, y ofrecer trazabilidad de ejecuciones y resultados.

## 3. Alcance

- Reutilizar encuestas existentes creadas con SurveyJS.
- Configurar nombre, descripción, encuesta, plantilla, filtros, `Post-purchase Offset` y programación de una campaña.
- Calcular `Audience Preview` con `Eligible Purchases` y `Eligible Customers`.
- Ejecutar campañas `ONE_TIME` o `FREQUENCY` con las frecuencias aprobadas.
- Distribuir invitaciones por Emblue Transactional Email y complementar el ciclo con eventos recibidos mediante DataLab/Webhooks.
- Consultar dashboard, detalle, estados, conteos e historial de ejecuciones.
- Relacionar de forma segura una respuesta con la campaña, ejecución y destinatario correspondientes.

## 4. Fuera de alcance

- Crear o editar encuestas desde Automated Survey Campaigns. SurveyJS y el módulo actual de encuestas siguen siendo responsables de la autoría.
- Cambiar la semántica de `Available From` / `Available Until`, que pertenecen al módulo Survey.
- Incluir `Auxiliary Subclassification` en la primera versión.
- Permitir rangos manuales de fechas de compra.
- Enviar invitaciones por canales distintos al correo transaccional Emblue.
- Definir detalles técnicos de tablas, schema Prisma, NestJS, endpoints o tareas de implementación.
- Presentar como disponibles aperturas, clics, rebotes u otros eventos hasta confirmar qué información entrega Emblue.

## 5. Actores

| Actor | Necesidad funcional |
|---|---|
| Administrador de campañas | Crear y mantener campañas, revisar audiencia, programar, suspender, reanudar, cancelar y consultar resultados según sus permisos. |
| Usuario de consulta | Consultar campañas y ejecuciones a las que tenga acceso, sin modificar ni disparar envíos. |
| Scheduler centralizado | Detectar campañas cuya ejecución corresponde y coordinar el procesamiento conforme a su calendario y estado. No es un usuario interactivo. |
| Emblue | Recibir solicitudes de correo transaccional y entregar eventos disponibles mediante DataLab/Webhooks. |
| Fuente corporativa CF/Analytics | Proveer la información acordada de jerarquías, compras, clientes y datos requeridos para formar la audiencia. |
| Respondiente | Recibir invitación y responder la encuesta asociada. |

**APPROVED:** solo `ADMIN` y `SUPERADMIN` como roles efectivos pueden consultar y gestionar Campaigns en alcance multiempresa. Autenticación y CASL backend son obligatorios. Los permisos de consulta de recipients y reintentos operativos mantienen las limitaciones ya definidas/OPEN; esta decisión no amplía grants generales.

## 6. Conceptos de dominio

| Concepto | Significado funcional |
|---|---|
| `Survey` | Encuesta existente de PEC creada con SurveyJS. La campaña la referencia; no la crea ni modifica. |
| `Campaign` | Configuración de distribución que enlaza una encuesta con audiencia, programación y plantilla. |
| `Audience Segmentation` | Conjunto de filtros de producto y comerciales usados para determinar compras elegibles. |
| `Post-purchase Offset` | Cantidad de días que deben transcurrir desde la compra para que esta sea elegible. |
| `Audience Preview` | Consulta previa que muestra conteos elegibles y, opcionalmente, una muestra acotada. |
| `Campaign Execution` | Instancia de procesamiento de una campaña para una ejecución programada o de recuperación. |
| `Campaign Recipient` | Destinatario relacionado con una compra y con el resultado de envío dentro de una campaña/ejecución. |
| `Purchase` | Compra identificada por `purchaseId/transactionId`, base de la elegibilidad y deduplicación. |
| `Response Attribution` | Relación segura entre una respuesta y Campaign, Execution y Recipient. |
| `Emblue Transactional Email` | Canal aprobado para solicitar el envío de invitaciones por correo. |

## 7. User stories

### 6.1 Idioma y etiquetas de la interfaz

**Regla obligatoria:** toda la interfaz de Automated Survey Campaigns se presenta en español. Enums, nombres de propiedades, DTO, entidades, tablas, rutas y valores enviados a API/Prisma/PostgreSQL permanecen en inglés. Angular obtiene las etiquetas desde un mapeo centralizado por tipo/valor; no traduce el valor técnico ni dispersa condicionales en plantillas.

#### Glosario funcional

| Término técnico | Etiqueta visible |
|---|---|
| Dashboard / Summary | Resumen de campañas |
| Campaign / Campaigns | Campaña / Campañas |
| Survey / Surveys | Encuesta / Encuestas |
| Schedule / Scheduling | Programación |
| Resume / Suspend / Cancel | Reanudar / Suspender / Cancelar |
| Frequency | Frecuencia |
| Division / Department | División / Departamento |
| Company / Format / Store | Empresa / Formato / Local |
| Auxiliary Classification | Clasificación auxiliar |
| Execution / Recipient / Attempt | Ejecución / Destinatario / Intento |
| Status / Actions | Estado / Acciones |
| Next Execution | Próxima ejecución |
| Post-purchase offset | Días después de la compra |
| ONE_TIME / FREQUENCY | Una sola vez / Recurrente |

#### Estados

| Entidad | Valor técnico | Etiqueta visible |
|---|---|---|
| Campaign | `DRAFT` | Borrador |
| Campaign | `SCHEDULED` | Programada |
| Campaign | `ACTIVE` | Activa |
| Campaign | `SUSPENDED` | Suspendida |
| Campaign | `COMPLETED` | Finalizada |
| Campaign | `CANCELLED` | Cancelada |
| Execution | `PENDING` | Pendiente |
| Execution | `PROCESSING` | En proceso |
| Execution | `COMPLETED` | Finalizada |
| Execution | `COMPLETED_WITH_ERRORS` | Finalizada con errores |
| Execution | `FAILED` | Fallida |
| Recipient / Attempt | `PENDING` | Pendiente |
| Recipient / Attempt | `SENT` | Enviada |
| Recipient / Attempt | `FAILED` | Fallida |
| Recipient / Attempt | `SKIPPED_DUPLICATE` | Omitida por duplicado |

#### Frecuencias, formularios y listados

| Valor técnico | Etiqueta visible |
|---|---|
| `DAILY` | Diaria |
| `WEEKLY` | Semanal |
| `BIWEEKLY` | Cada dos semanas |
| `MONTHLY` | Mensual |
| `QUARTERLY` | Trimestral |

| Área | Etiquetas visibles aprobadas |
|---|---|
| Dashboard | Total, Activas, Programadas, Borradores, Suspendidas, Finalizadas. Canceladas se consultan con filtro; no agregan tarjeta. |
| Filtros | Nombre de campaña, Nombre de encuesta, Estado, Todos, Limpiar filtros. |
| Tabla | Campaña, Encuesta, Frecuencia, Próxima ejecución, Estado, Acciones. |
| Formulario | Información general, Segmentación, Configuración de envío, Nombre de campaña, Encuesta existente, Descripción, Días después de la compra, Fecha de envío, Fecha de inicio, Fecha de fin, Hora de envío, Frecuencia. |
| Segmentación | División, Departamento, Empresa, Clasificación auxiliar, Formato, Local; obligatorio para programar / opcional. |
| Acciones | Nueva campaña, Crear campaña, Editar, Ver detalle, Cancelar borrador, Guardar borrador, Volver a campañas. |
| Detalle | Resumen, Programación, Segmentación guardada, Ejecuciones, Fecha programada, Inicio, Fin, Compras elegibles, Clientes elegibles, Enviadas, Fallidas, Duplicadas. |
| Vista previa | Vista previa de audiencia; Vista previa no disponible. |

Fechas y horas visibles usan formato español (`dd/MM/yyyy`, hora local configurada y nombres de meses/días en español); no se altera su representación técnica en solicitudes. El paginador presenta textos en español. Cargas, vacíos, validaciones, confirmaciones y errores se redactan en español, sin exponer excepciones, SQL ni detalles internos innecesarios. Las opciones MOCK usan etiquetas españolas y una marca visible de demostración; sus IDs no cambian.

## 7. User stories

- **US-001:** Como administrador, quiero elegir una encuesta PEC existente para crear una campaña que distribuya esa encuesta.
- **US-002:** Como administrador, quiero filtrar la audiencia por jerarquías de producto y comerciales para dirigir invitaciones a las compras adecuadas.
- **US-003:** Como administrador, quiero ver compras y clientes elegibles antes de programar para validar el alcance de la audiencia.
- **US-004:** Como administrador, quiero programar una fecha/hora única o una recurrencia para automatizar las invitaciones.
- **US-005:** Como administrador, quiero suspender una campaña activa antes de editarla para controlar cambios mientras se ejecuta.
- **US-006:** Como administrador o usuario de consulta, quiero consultar el estado, conteos e historial para conocer el resultado operativo de una campaña.
- **US-007:** Como respondiente, quiero abrir la encuesta desde la invitación y responderla sin que la URL revele mis datos personales.
- **US-008:** Como responsable funcional, quiero que una compra ya procesada no vuelva a generar otra invitación de la misma campaña, conservando la posibilidad de procesar otra compra del mismo cliente.

## 8. Flujos principales

### 8.1 Crear y revisar una campaña

1. El administrador inicia una campaña y completa su información.
2. Selecciona una encuesta PEC existente. Puede guardar la campaña como `DRAFT` aunque la encuesta aún no esté publicada.
3. Define filtros de audiencia y `Post-purchase Offset`.
4. Solicita `Audience Preview`; el sistema muestra `Eligible Purchases` y `Eligible Customers`.
5. Define plantilla y tipo de envío/calendario.
6. Revisa la configuración y la guarda. La campaña continúa en `DRAFT` hasta que se programe válidamente.

### 8.2 Programar una campaña

1. El administrador solicita programar la campaña.
2. El sistema valida campos requeridos, filtros, reglas de frecuencia y elegibilidad de la encuesta.
3. La encuesta debe cumplir el requisito funcional `PUBLISHED` para programar.
4. Si la configuración es válida, la campaña queda `SCHEDULED` antes de su primera ejecución.

### 8.3 Ejecutar una campaña

1. El scheduler centralizado identifica la campaña vencida, siempre que esté habilitada para ejecutarse.
2. El sistema obtiene las compras que cumplen segmentación y offset y que no se hayan procesado previamente por esa Campaign.
3. Se preparan invitaciones con `surveyUrl` y las variables autorizadas de plantilla.
4. PEC solicita el envío a Emblue y registra los resultados por recipient y por execution.
5. El sistema actualiza el historial y determina el siguiente estado/ejecución conforme a las reglas de recurrencia y recuperación.

### 8.4 Administrar el ciclo de vida

El administrador realiza las acciones permitidas según el estado. Una campaña `ACTIVE` debe suspenderse antes de editarse. Las campañas `COMPLETED` o `CANCELLED` son de solo lectura. Una campaña con historial no se elimina físicamente.

### 8.5 Responder una invitación

1. El destinatario abre la `surveyUrl` de la invitación.
2. La referencia de la invitación permite al sistema correlacionar la respuesta con Campaign, Execution y Recipient.
3. La URL no expone PII, `purchaseId`, `transactionId` ni identificadores sensibles.

**OPEN:** mecanismo de atribución, caducidad/reuso de enlaces y comportamiento ante enlace reenviado o respuesta fuera de vigencia.

## 9. Reglas de negocio

### 9.1 Survey y selección

- **RB-001:** Automated Survey Campaigns reutiliza encuestas PEC existentes basadas en SurveyJS; no crea ni modifica encuestas.
- **RB-002:** Una Campaign debe referenciar una Survey existente.
- **RB-003:** Una Campaign puede guardarse en `DRAFT` aunque la Survey no esté publicada.
- **RB-004:** Para programar una Campaign, la Survey seleccionada debe cumplir el estado funcional `PUBLISHED`.
- **RB-005:** `Available From` y `Available Until` se mantienen bajo responsabilidad del módulo Survey.
- **RB-006 (APPROVED para Campaign):** la elegibilidad de Survey requiere `status = 1`, `isPublic = true` y `archived = false`; `availableFrom`/`availableTo`, si existen, deben cubrir el calendario Campaign. Esto no define sem?ntica global `PUBLISHED`.

### 9.2 Audience Segmentation

- **RB-007:** Todos los filtros disponibles soportan multiselect.
- **RB-008:** `Division` es obligatoria y multiselect.
- **RB-009:** `Department` es obligatorio y multiselect.
- **RB-010:** `Auxiliary Classification` es opcional y multiselect.
- **RB-011:** `Auxiliary Subclassification` queda fuera del alcance de la primera versión.
- **RB-012:** `Company` es obligatoria y multiselect.
- **RB-013:** `Format` es opcional y multiselect.
- **RB-014:** `Store` es opcional y multiselect.
- **RB-015:** Los valores seleccionados dentro de un mismo filtro/grupo se combinan con OR.
- **RB-016:** Los grupos/filtros distintos se combinan con AND.
- **RB-017:** Un filtro opcional sin selecciones no restringe la audiencia.
- **RB-018:** Al cambiar un filtro padre, se limpian las selecciones hijas que ya no sean compatibles.
- **RB-019 (OPEN):** Catálogos, relaciones de jerarquía, valores inválidos y reglas específicas para selecciones sin descendientes dependen del contrato CF/Analytics.

### 9.3 Post-purchase Offset y elegibilidad

- **RB-020:** `Post-purchase Offset` es obligatorio y debe ser un entero mayor o igual a cero, expresado en días.
- **RB-021:** Un offset `0` no agrega espera posterior a la compra.
- **RB-022:** Una compra es elegible por fecha cuando `PurchaseDate + Offset <= ExecutionDate`.
- **RB-023:** La elegibilidad también exige que la compra cumpla la segmentación configurada.
- **RB-024:** No se ofrecen rangos manuales de fechas de compra.
- **RB-025 (OPEN):** Definición de fecha de corte, zona horaria, inclusión de día/hora y tratamiento de compras tardías se confirma con negocio y CF/Analytics.

### 9.4 Audience Preview

- **RB-026:** La vista previa muestra `Eligible Purchases` y `Eligible Customers`.
- **RB-027:** La vista previa considera segmentación, `Post-purchase Offset` y las compras ya procesadas cuando se consulta una Campaign existente.
- **RB-028:** La vista previa puede ofrecer una muestra limitada, pero no debe presentar ni descargar la audiencia completa.
- **RB-029:** La audiencia completa no se entrega al frontend Angular.
- **RB-030 (OPEN):** La caducidad del conteo, la etiqueta de fecha de cálculo y los límites de muestra quedan por definir; el preview no garantiza que la audiencia permanezca idéntica hasta la ejecución.

### 9.5 Scheduling y frecuencias

- **RB-031:** `SendType` admite `ONE_TIME` y `FREQUENCY`.
- **RB-032:** `ONE_TIME` requiere `Send Date` y `Send Time`.
- **RB-033:** `FREQUENCY` requiere `Start Date`, `End Date`, `Send Time` y `Frequency`.
- **RB-034:** Las frecuencias permitidas son `DAILY`, `WEEKLY`, `BIWEEKLY`, `MONTHLY` y `QUARTERLY`.
- **RB-035:** Para una duración de 1 a 6 días, el catálogo ofrece solo `DAILY`.
- **RB-036:** Para 7 a 13 días, el catálogo ofrece `DAILY` y `WEEKLY`.
- **RB-037:** Para 14 a 29 días, el catálogo ofrece `DAILY`, `WEEKLY` y `BIWEEKLY`.
- **RB-038:** Para 30 a 89 días, el catálogo ofrece `DAILY`, `WEEKLY`, `BIWEEKLY` y `MONTHLY`.
- **RB-039:** Para 90 días o más, el catálogo ofrece todas las frecuencias, incluida `QUARTERLY`.
- Para RB-035–RB-039, `durationDays = calendarDayDifference(endDate, startDate) + 1`: el conteo incluye `Start Date` y `End Date` y usa fechas de calendario, sin diferencias horarias, timestamps ni conversiones UTC.
- **RB-040:** La siguiente ejecución se calcula sobre fechas locales de calendario antes de convertir a UTC: `DAILY` +1 día, `WEEKLY` +7 días, `BIWEEKLY` +14 días, `MONTHLY` +1 mes calendario y `QUARTERLY` +3 meses calendario. Para `MONTHLY` y `QUARTERLY`, cada ocurrencia se calcula desde `StartDate` y su índice, conservando el día original como ancla. Si el mes destino no contiene ese día, se usa el último día disponible; esta normalización no cambia el ancla de ocurrencias posteriores y aplica también a años bisiestos.
- **RB-041:** `ONE_TIME` queda `COMPLETED` después de su única ejecución, según el resultado terminal registrado.
- **RB-042:** `End Date` es inclusiva como fecha local de negocio. Una ocurrencia en `End Date` exactamente a `Send Time` es válida; no se generan ocurrencias en fechas locales posteriores. `COMPLETED` requiere calendario terminado y ausencia de trabajo pendiente, en curso o recuperable.
- **RB-043:** El scheduler es centralizado; no se define un scheduler individual por Campaign.
- **RB-044 (OPEN parcial):** Para Campaigns de Ecuador, la zona predeterminada aprobada es `America/Guayaquil` y debe persistirse explícitamente antes de programar; no se infiere del navegador. Siguen OPEN la zona para otros mercados y el tratamiento de horas locales DST ambiguas o inexistentes. El anclaje y ajuste de fin de mes de RB-040 están aprobados.

### 9.6 Audiencia incremental y deduplicación

- **RB-045:** Cada ejecución recurrente procesa las compras que cumplen segmentación y offset y que aún no hayan sido procesadas por esa Campaign.
- **RB-046:** La deduplicación es por `campaignId + purchaseId/transactionId`, no únicamente por cliente.
- **RB-047:** Un mismo cliente puede ser elegible nuevamente cuando exista una compra distinta que cumpla las reglas.
- **RB-048:** Una compra marcada como procesada no genera otro envío de la misma Campaign en una ejecución futura.
- **RB-049 (OPEN):** Se debe confirmar si el identificador autoritativo será `purchaseId`, `transactionId` o una clave acordada entre ellos.

### 9.7 Campaign lifecycle

- **RB-050:** Los estados de Campaign son `DRAFT`, `SCHEDULED`, `ACTIVE`, `SUSPENDED`, `COMPLETED` y `CANCELLED`.
- **RB-051:** `SCHEDULED` representa una campaña programada antes de su primera ejecución.
- **RB-052:** `ACTIVE` representa una campaña con al menos una ejecución y con futuras ejecuciones pendientes.
- **RB-053:** Desde `DRAFT` se permiten `Edit`, `Schedule`, `Cancel` y `Delete`, sujeto a permisos y a las condiciones de borrado.
- **RB-054:** Desde `SCHEDULED` se permiten `Edit`, `Suspend` y `Cancel`.
- **RB-055:** Desde `ACTIVE` se permiten `Suspend` y `Cancel`; para editar se debe suspender primero.
- **RB-056:** Desde `SUSPENDED` se permiten `Edit`, `Resume` y `Cancel`.
- **RB-057:** `COMPLETED` y `CANCELLED` son estados de solo lectura.
- **RB-058:** Una Campaign con historial de ejecución no se elimina físicamente.
- **RB-059:** Suspender o cancelar impide nuevas ejecuciones de acuerdo con la acción solicitada y conserva el historial.
- **RB-060:** `Suspend` bloquea nuevos claims y lotes; el lote iniciado puede finalizar y persistir. La semántica de Cancel con trabajo en curso permanece OPEN.

### 9.8 Execution lifecycle y recuperación

- **RB-061:** Los estados de ejecución son `PENDING`, `PROCESSING`, `COMPLETED`, `COMPLETED_WITH_ERRORS` y `FAILED`.
- **RB-062:** Una ejecución sigue la transición `PENDING` → `PROCESSING` → estado final.
- **RB-063:** Los estados finales de Execution son históricos e inmutables.
- **RB-064:** Una caída del sistema no debe hacer perder las compras elegibles pendientes.
- **RB-065:** La recuperación consolida los slots vencidos elegibles en una sola unidad por Campaign, sin reproducir un envío por slot omitido. El backlog automático inicial es 30 días, configurable; excederlo alerta y requiere intervención sin descartar datos.
- **RB-066:** Una falla total no debe marcar compras elegibles pendientes como procesadas ni provocar que futuras ejecuciones omitan compras que todavía deben considerarse.
- **RB-067:** Una ejecución parcial puede terminar en `COMPLETED_WITH_ERRORS` si conserva sus resultados por destinatario.
- **RB-068:** Reintentar significa crear una nueva Execution relacionada y limitarse a recipients elegibles que no tengan resultado externo ambiguo. Solo errores transitorios confirmados admiten hasta 3 intentos automáticos de retry, con backoff exponencial y jitter, además del envío inicial. Errores permanentes y resultados ambiguos no se reintentan automáticamente. Contrato de clasificación/reconciliación Emblue permanece OPEN.

### 9.9 Recipient lifecycle

- **RB-069:** Los estados de Recipient son `PENDING`, `SENT`, `FAILED` y `SKIPPED_DUPLICATE`.
- **RB-070:** No se utiliza un estado genérico `SKIPPED`.
- **RB-071:** El resultado debe permitir distinguir envío exitoso, fallo y duplicado sin reinterpretar el estado de entrega final de Emblue.
- **RB-072 (OPEN parcial):** La aceptación HTTP/API, el procesamiento asíncrono de Emblue y la entrega efectiva al buzón son hechos distintos y no deben presentarse como equivalentes. La semántica exacta de `SENT`, estados/eventos disponibles y reconciliación se confirma con el contrato Emblue.

### 9.10 Emblue, invitación y Response Attribution

- **RB-073:** Las invitaciones se solicitan a Emblue mediante su canal de email transaccional aprobado.
- **RB-074:** Los eventos disponibles se reciben complementariamente mediante DataLab/Webhooks, según el contrato final.
- **RB-075:** PEC mantiene la responsabilidad funcional sobre Campaign, segmentación, offset, frecuencia, scheduler, deduplicación, ejecución y recipients.
- **RB-076:** Emblue mantiene la responsabilidad funcional sobre plantilla, entrega de email, deliverability, aperturas, clics y rebotes.
- **RB-077:** `surveyUrl` es una variable obligatoria para construir la invitación.
- **RB-078:** Variables opcionales aprobadas: `customerName`, `companyName`, `formatName`, `storeName`, `expirationDate`, `surveyName` y `purchaseDate`.
- **RB-079:** `purchaseId/transactionId` se conserva internamente para procesamiento/deduplicación y no se expone en la plantilla salvo necesidad futura aprobada.
- **RB-080:** Una respuesta debe poder atribuirse a Campaign, Execution y Recipient.
- **RB-081:** Ninguna URL de invitación/respuesta debe exponer PII, `purchaseId`, `transactionId`, tokens de proveedor ni identificadores sensibles.
- **RB-082 (OPEN):** Diseño, caducidad, uso/reuso y validación del identificador seguro de atribución; autenticidad y semántica de eventos Emblue.

### 9.11 Dashboard y Campaign Detail

- **RB-083:** El dashboard presenta cards `Total`, `Active`, `Scheduled`, `Draft`, `Suspended` y `Completed`.
- **RB-084:** `CANCELLED` se consulta mediante filtro y no tiene una card propia.
- **RB-085:** La tabla del dashboard incluye `Campaign`, `Survey`, `Frequency`, `Next Execution`, `Status` y `Actions`.
- **RB-086:** El detalle contiene las secciones `Summary`, `Segmentation`, `Scheduling` y `Executions`.
- **RB-087:** `Summary` muestra Campaign Name, Description, Survey, Email Template, Status, Send Type, Frequency, Next Execution, Created By, Created At y Updated At.
- **RB-088:** `Segmentation` muestra Divisions, Departments, Auxiliary Classifications, Companies, Formats, Stores, Post-purchase Offset, Eligible Purchases y Eligible Customers.
- **RB-089:** `Scheduling` muestra Send Type, Send Date para `ONE_TIME`, Start Date, End Date, Send Time, Frequency para `FREQUENCY`, Last Execution y Next Execution.
- **RB-090:** `Executions` muestra Scheduled At, Started At, Finished At, Eligible Purchases, Eligible Customers, Sent, Failed, Duplicates y Status.
- **RB-091:** El dashboard debe permitir buscar por `Campaign Name` y por `Survey Name`, y filtrar por `Status`. La búsqueda por nombre de campaña y la búsqueda por nombre de encuesta pueden usarse junto con el filtro de estado.
- **OPEN:** La paginación, el ordenamiento, las acciones de fila y otros detalles adicionales de UX del listado quedan por definir.

### 9.12 Datos personales

- **RB-092:** Solo se usan los datos personales necesarios para seleccionar destinatario, solicitar envío y atribuir respuesta.
- **RB-093:** No se registran innecesariamente emails completos, tokens, credenciales ni payloads completos con PII.
- **RB-094:** El acceso a recipients debe estar restringido a usuarios autorizados.
- **RB-095:** QA y validaciones integradas utilizan datos sintéticos y credenciales no productivas.
- **RB-096 (OPEN):** Consentimiento/supresión, retención y eliminación de destinatarios y datos de atribución se definen con negocio, seguridad y privacidad antes de producción.

### 9.13 Alcance multiempresa y catálogos

- **RB-097:** Campaign es multiempresa para los roles autorizados; consultas y dashboard incluyen campañas de todas las empresas del módulo.
- **RB-098:** Solo los roles efectivos `ADMIN` y `SUPERADMIN` acceden a Campaign; la empresa activa seleccionada en PEC no reduce el alcance. Backend valida autenticación y CASL en cada operación.
- **RB-099:** La selección de Survey usa una ruta específica de Campaign y aplica la política multiempresa aprobada; no se amplían permisos de las rutas generales de Surveys. Survey asociada debe existir y ser accesible conforme a esa política.
- **RB-100:** `Enterprise` PEC y `Company` CF/Analytics son conceptos distintos; no se infiere relación entre ellos.
- **RB-101 (MOCK):** Mientras no haya contrato CF/Analytics, un proveedor Campaign puede entregar catálogos con IDs y etiquetas sintéticos identificables como MOCK. Sus relaciones no describen la jerarquía real de CF.
- **RB-102:** El proveedor de catálogos es exclusivo de Campaign y sustituible sin cambiar el contrato Angular. No se inventan endpoints ni autenticación Analytics.
- **RB-103:** El proveedor MOCK debe seleccionarse explícitamente, estar deshabilitado por defecto en producción y no actuar como fallback si Analytics falla.
- **RB-104:** Backend valida IDs activos y relaciones jerárquicas según proveedor configurado. IDs MOCK no son válidos para Analytics.
- **RB-105 (DEFERRED):** Datos MOCK no pueden alimentar Preview real, scheduler, destinatarios ni envíos.
- **RB-106:** Toda Campaign debe tener `timeZone` persistida explícitamente antes de Schedule/Resume efectivos. Para Ecuador, `America/Guayaquil` es el valor predeterminado aprobado; Angular no deriva ni sustituye la zona desde el navegador. La conversión a instantes usa la zona persistida.

## 10. Criterios de aceptación

### Casos positivos

- **AC-001:** Dada una Survey PEC existente no publicada, cuando un administrador guarda una Campaign, entonces se guarda en `DRAFT` sin obligar a publicar la Survey.
- **AC-002:** Dada una Campaign válida y una Survey `PUBLISHED`, cuando el administrador la programa, entonces queda `SCHEDULED` y se muestra su próxima ejecución.
- **AC-003:** Dada una configuración válida, cuando se solicita `Audience Preview`, entonces se muestran `Eligible Purchases` y `Eligible Customers` calculados con segmentación, offset y deduplicación existentes.
- **AC-004:** Dada una segmentación con varias selecciones, cuando se evalúa, entonces las selecciones de cada filtro se combinan con OR y los filtros/grupos distintos con AND.
- **AC-005:** Dada una Campaign con `Post-purchase Offset` cero, cuando se calcula elegibilidad, entonces no se añade espera posterior a la fecha de compra.
- **AC-006:** Dada una Campaign `ONE_TIME` con fecha y hora válidas, cuando llega el momento programado, entonces se registra una única ejecución y la Campaign pasa a `COMPLETED` solo cuando la ejecución/calendario terminó y no queda trabajo pendiente, en curso o recuperable.
- **AC-007:** Dada una Campaign `FREQUENCY` válida, cuando el usuario selecciona un rango, entonces solo se ofrecen las frecuencias correspondientes a la duración definida.
- **AC-008:** Dada una compra elegible ya procesada por la misma Campaign, cuando se calcula una ejecución recurrente posterior, entonces no se vuelve a contar como pendiente ni se reenvía.
- **AC-009:** Dado el mismo cliente con otra compra elegible, cuando se ejecuta la Campaign, entonces esa compra puede procesarse porque la deduplicación es por Campaign + compra.
- **AC-010:** Dada una Campaign `ACTIVE`, cuando el administrador intenta editarla directamente, entonces el cambio no se permite; tras suspenderla, puede editarse según permisos.
- **AC-011:** Dada una Campaign con historial, cuando se solicita eliminarla, entonces el sistema conserva el registro y no la elimina físicamente.
- **AC-012:** Dada una falla total de ejecución, cuando se registra el resultado, entonces queda trazable, no se marcan como procesadas las compras elegibles pendientes y futuras ejecuciones no omiten compras que todavía deben considerarse.
- **AC-013:** Dada una invitación válida, cuando se presenta al destinatario, entonces incluye `surveyUrl` y no expone PII ni identificadores de compra en la URL.
- **AC-014:** Dada una respuesta vinculada a una invitación válida, cuando el sistema la registra, entonces puede correlacionarla con Campaign, Execution y Recipient mediante el mecanismo aprobado.
- **AC-015:** Dado el dashboard, cuando se consulta y se aplican criterios de búsqueda y estado, entonces permite buscar por `Campaign Name` y `Survey Name` y filtrar por `Status`, pudiendo combinarse esos criterios; `CANCELLED` es accesible mediante el filtro de estado y no tiene card propia. También aparecen las seis cards definidas y la tabla con las columnas funcionales aprobadas.
- **AC-016:** Dado el detalle de una Campaign, cuando se consulta, entonces muestra las secciones `Summary`, `Segmentation`, `Scheduling` y `Executions` con los campos especificados.

### Casos negativos

- **AC-017:** Dada una Campaign sin `Division`, `Department` o `Company`, cuando se intenta programar, entonces la validación informa que los filtros requeridos están incompletos.
- **AC-018:** Dado un valor de `Post-purchase Offset` negativo, fraccionario o vacío, cuando se guarda/programa, entonces el sistema lo rechaza con validación comprensible.
- **AC-019:** Dada una Campaign con `Auxiliary Subclassification`, cuando se configura la audiencia v1, entonces el filtro no se ofrece como criterio disponible.
- **AC-020:** Dada una Survey inexistente o no `PUBLISHED`, cuando se intenta programar una Campaign, entonces no se programa y se informa el requisito aplicable; guardar en `DRAFT` sigue siendo posible para una Survey existente no publicada.
- **AC-021:** Dada una Campaign `SUSPENDED`, `COMPLETED` o `CANCELLED`, cuando llega su hora de ejecución, entonces no se inicia una nueva ejecución.
- **AC-022:** Dada una Campaign `COMPLETED` o `CANCELLED`, cuando se intenta cambiar su configuración, entonces se mantiene en solo lectura.
- **AC-023:** Dada una muestra de preview solicitada, cuando se responde al frontend, entonces no se devuelve la audiencia completa.
- **AC-024:** Dada una solicitud de envío con una `surveyUrl` ausente, cuando se valida la invitación, entonces no se considera una invitación válida para envío.
- **AC-025:** Dada información personal en datos de envío, cuando ocurre un error, entonces logs y mensajes operativos no exponen el email completo, token o payload personal completo.

### Edge cases

- **AC-026:** Dado un cambio en un filtro padre, cuando sus selecciones hijas dejan de ser compatibles, entonces se limpian dichas selecciones antes de aceptar la nueva configuración.
- **AC-027:** Dado un filtro opcional vacío, cuando se evalúa la audiencia, entonces ese filtro no limita los resultados.
- **AC-028:** Dada una Campaign `FREQUENCY` cuyo próximo ciclo queda después de `End Date`, cuando finaliza su última ejecución prevista, entonces la campaña termina en `COMPLETED`.
- **AC-029:** Dada una recurrencia `MONTHLY` o `QUARTERLY` cuyo mes destino no contiene el día original de `StartDate`, cuando se calcula la ocurrencia, entonces se usa el último día disponible del mes sin modificar el ancla original para ocurrencias posteriores, incluso en años bisiestos.
- **AC-030:** Dado un período de indisponibilidad con varios slots omitidos dentro de 30 días, cuando el scheduler se recupera, entonces procesa el pendiente en una sola recovery por Campaign. Si el backlog excede la ventana configurable, alerta y requiere intervención sin descartar datos.
- **AC-031:** Dado un fallo parcial de destinatarios, cuando se cierra la ejecución, entonces se conservan los resultados individuales y puede utilizarse `COMPLETED_WITH_ERRORS`.
- **AC-032:** Dado un fallo confirmado transitorio, cuando se reintenta, entonces se crea nueva Execution relacionada, solo con recipients elegibles y hasta 3 intentos de retry automáticos además del envío inicial, con backoff exponencial+jitter; un fallo permanente o ambiguo no genera retry automático.
- **AC-033:** Dada una Campaign que se suspende con batch en curso, cuando la suspensión queda persistida, entonces no se reclaman nuevos lotes y el batch ya iniciado finaliza/persiste de forma segura.
- **AC-034:** Dado un destinatario sin correo válido, duplicado en la fuente o suprimido por una regla de contacto, cuando se prepara la audiencia, entonces se aplica una regla acordada y se contabiliza de forma auditable. **OPEN:** reglas de validez, supresión y conteo.
- **AC-035:** Dado un callback repetido o fuera de orden, cuando PEC lo recibe, entonces no altera incorrectamente la correlación/resultado. **OPEN:** depende del contrato y de identificadores de evento Emblue.

- **AC-036:** ADMIN/SUPERADMIN consulta Campaigns y Surveys desde varias empresas mediante rutas Campaign, aunque cambie la empresa activa PEC.
- **AC-037:** Otros roles, incluida una membresía ADMIN que no sea el rol efectivo, reciben denegación backend/CASL; permisos generales de otros módulos no cambian.
- **AC-038:** DRAFT puede guardarse sin Division, Department o Company; Schedule rechaza los valores requeridos faltantes.
- **AC-039:** El proveedor MOCK solo devuelve valores sintéticos marcados MOCK y sus relaciones de demostración no se presentan como catálogos CF.
- **AC-040:** Ausencia o fallo de Analytics no activa MOCK como fallback.
- **AC-041:** Backend rechaza IDs inexistentes, inactivos o incompatibles con las relaciones del proveedor activo; MOCK IDs no se aceptan en Analytics.
- **AC-042 (DEFERRED):** MOCK no inicia scheduling, resolución real de audiencia ni envíos.

## 11. Estados y transiciones funcionales

### 11.1 Campaign

| Estado | Evento funcional | Estado siguiente | Restricciones |
|---|---|---|---|
| `DRAFT` | Programar con datos válidos y Survey `PUBLISHED` | `SCHEDULED` | Requiere permisos y campos completos. |
| `DRAFT` | Cancelar | `CANCELLED` | Conserva registro. |
| `DRAFT` | Eliminar | Sin transición de estado definida | Solo si no tiene historial; detalle de confirmación/retención OPEN. |
| `SCHEDULED` | Llega la primera ejecución programada | `ACTIVE` al existir al menos una ejecución y futuras ejecuciones | Una única ejecución `ONE_TIME` puede avanzar a `COMPLETED`. |
| `SCHEDULED` | Suspender | `SUSPENDED` | Impide nuevas ejecuciones. |
| `SCHEDULED` | Cancelar | `CANCELLED` | Conserva historial asociado. |
| `ACTIVE` | Suspender | `SUSPENDED` | Para editar se requiere suspender primero. |
| `ACTIVE` | Cancelar | `CANCELLED` | No inicia nuevas ejecuciones. |
| `ACTIVE` | Termina la ventana recurrente | `COMPLETED` | Si no quedan futuras ejecuciones. |
| `SUSPENDED` | Reanudar | `SCHEDULED` o `ACTIVE` | Estado resultante depende de si ya hubo ejecuciones; se conserva la programación acordada. |
| `SUSPENDED` | Cancelar | `CANCELLED` | Conserva historial. |
| `COMPLETED` / `CANCELLED` | Consulta | Mismo estado | Solo lectura. |

`Resume` antes de EndDate evalúa recovery consolidado y calcula el siguiente slot futuro. Después de EndDate solo recupera trabajo de slots originalmente válidos y no genera slots ordinarios nuevos.

### 11.2 Campaign Execution

| Estado actual | Evento | Estado siguiente |
|---|---|---|
| `PENDING` | Comienza procesamiento | `PROCESSING` |
| `PROCESSING` | Todos los resultados completan según el contrato | `COMPLETED` |
| `PROCESSING` | Hay resultados parciales con errores | `COMPLETED_WITH_ERRORS` |
| `PROCESSING` | Falla total | `FAILED` |

Los estados finales son históricos e inmutables. La recuperación de fallo terminal crea una nueva Execution relacionada y no cambia retrospectivamente el estado final anterior.

### 11.3 Campaign Recipient

| Estado | Significado funcional |
|---|---|
| `PENDING` | Recipient identificado como elegible, aún sin resultado de envío registrado. |
| `SENT` | Resultado de envío positivo según la semántica que confirme Emblue. |
| `FAILED` | Fallo de envío registrado. |
| `SKIPPED_DUPLICATE` | Compra duplicada para la misma Campaign; no se envía otra invitación. |

La transición desde `FAILED` se representa mediante una nueva Execution relacionada, no mediante mutación de estado terminal. La aceptación de solicitud, procesamiento asíncrono y entrega definitiva son estados/hechos distintos; la semántica de `SENT` y su reconciliación siguen OPEN con Emblue.

## 12. Integración funcional con Emblue

- PEC solicita a Emblue el envío transaccional de invitaciones asociadas a campañas.
- Cada mensaje debe incluir `surveyUrl`; variables opcionales disponibles según datos entregados y autorizados: `customerName`, `companyName`, `formatName`, `storeName`, `expirationDate`, `surveyName`, `purchaseDate`.
- `purchaseId/transactionId` no se ofrece a la plantilla como variable en la primera versión.
- PEC conserva el control funcional de campañas, audiencias, calendario, ejecución, deduplicación y trazabilidad interna.
- Emblue administra plantilla y aspectos de entrega/deliverability, aperturas, clics y rebotes.
- DataLab/Webhooks complementan el envío para comunicar eventos disponibles; no reemplazan el registro operacional de PEC.
- **OPEN:** contratos, autenticación, tamaños de lote, límites, estados, idempotencia, eventos disponibles, retries, rebotes/supresiones y significado de aceptación/entrega.

## 13. Privacidad y restricciones de PII

- El correo y los identificadores de cliente/compra se utilizan únicamente para selección, envío, deduplicación y atribución autorizados.
- No incluir email, token, `purchaseId`, `transactionId` ni identificadores sensibles en la URL.
- No registrar innecesariamente correos completos, tokens, credenciales ni payloads completos con PII.
- El acceso a recipients queda restringido según permisos autorizados.
- Las pruebas de aceptación/integración utilizan información sintética y credenciales no productivas.
- **OPEN:** consentimiento/supresión, retención, eliminación, acceso a destinatarios y contenido exacto conservado para auditoría.

## 14. Supuestos

- Se entregará una fuente corporativa autorizada con jerarquías y datos necesarios de compras/clientes. Sus vistas, columnas, conexión y actualización siguen pendientes.
- Se obtendrá el contrato técnico definitivo de Emblue para API transaccional y DataLab/Webhooks.
- Para Campaign esta aprobada la conjuncion status=1, isPublic=true y archived=false, y vigencia que cubra el calendario si esta configurada. No se modifica el significado global de status.
- Para Campaigns de Ecuador se aprobó `America/Guayaquil` como zona predeterminada; cada Campaign persiste su `timeZone` antes de la programación efectiva. No inferir la zona del navegador. Reglas de DST, otros mercados y recuperación siguen OPEN. El fin de mes está resuelto en RB-040.
- El scheduler centralizado utilizará el scheduler global existente del proyecto, con comportamiento seguro ante múltiples réplicas según la arquitectura aprobada.

## 15. Dependencias

| Dependencia | Resultado necesario |
|---|---|
| Equipo CF/Analytics/datos | Contrato de catálogos, compras, clientes, email, jerarquías, claves, permisos, frescura y volumen. |
| Equipo PEC/Emblue | Contrato, autenticación por ambiente, plantillas, límites, resultados por destinatario, idempotencia, eventos y política de errores/retries. |
| Equipo Surveys | Confirmar encuesta elegible, publicación, URL pública y disponibilidad de encuesta. |
| Equipo de producto/negocio | Validar calendario, frecuencia, reglas de recuperación, audiencia, supresión y semántica de conteos/estados. |
| Seguridad/privacidad | Aprobar roles, atribución, acceso a PII, retención y manejo de eventos. |
| Operación PEC | Confirmar despliegue multi-réplica, observabilidad, límites y procedimiento ante ejecución fallida. |

## 16. OPENs

| ID | Punto pendiente |
|---|---|
| OPEN-01 | Contrato real de CF/Analytics y fuente autorizada de datos. |
| OPEN-02 | Nombres/columnas, conexión, permisos, actualización y significado de jerarquías/catálogos. |
| OPEN-03 | Identificador estable y único de compra (`purchaseId`/`transactionId`) y relación con cliente/email. |
| OPEN-04 | Zona horaria fuera de Ecuador, corte de ejecución y tratamiento de horarios DST ambiguos o inexistentes. La duración inclusiva está resuelta en RB-035–RB-039; la zona Ecuador y regla de fin de mes/anclaje están resueltas en RB-040 y RB-044. |
| OPEN-05 | CLOSED para reglas funcionales de consolidación, ventana inicial de 30 días, Suspend/Resume y EndDate; quedan OPEN parámetros de operación y retención de fuente. |
| OPEN-06 (parcial) | Cerrado para envío individual API v2.3: endpoint `POST /emails/send`, autenticación Bearer, body `{ sendRequest: { to, html, actionId, subject } }` y response `{ activityId: string }`, con `activityId` como aceptación API únicamente. Siguen OPEN gestión/rotación de credenciales por ambiente, template, límites, procesamiento/entrega, consulta `/activities/find`, idempotencia y reconciliación. |
| OPEN-07 | Eventos de DataLab/Webhooks, firma, orden, deduplicación, retries y semántica de entrega. |
| OPEN-08 | CLOSED: máximo 3 retries automáticos además del envío inicial, solo transitorios confirmados, backoff exponencial+jitter, nueva Execution relacionada y no retry ambiguo/permanente. Quedan OPEN el clasificador/códigos y parámetros de backoff dependientes del contrato/operación. |
| OPEN-09 | Mecanismo de Response Attribution, expiración y respuesta a enlaces reenviados. |
| OPEN-10 | Permisos específicos de consulta de recipients y reintentos operativos; no bloquea el acceso Campaign ya aprobado para roles efectivos ADMIN/SUPERADMIN. |
| OPEN-11 | Reglas de correo inválido, destinatario duplicado, supresión/consentimiento y sus conteos. |
| OPEN-12 | Retención, acceso y eliminación de PII, recipients, compras procesadas y eventos. |
| OPEN-13 | Volumen máximo, ventana de envío, muestras de preview y límites operativos. |
| OPEN-14 | Alineaci?n de zona local del endpoint p?blico entre r?plicas y acceso de respuestas despu?s de `availableTo`; la regla de elegibilidad Campaign ya est? aprobada. |

## 17. Definition of Done funcional

- La especificación se revisó junto con `PEC-SC-ARCH-001` y mantiene su separación funcional entre Survey, Campaign, Execution y Recipient.
- Están definidos los estados, acciones y criterios de elegibilidad en términos verificables, sin cambiar el baseline aprobado.
- Se cubren selección de Survey, filtros requeridos/opcionales, combinación OR/AND, offset, preview y frecuencia dinámica.
- Se especifican la audiencia incremental, deduplicación por Campaign + compra, scheduler centralizado, estados de ejecución y recuperación.
- El flujo funcional de invitación contempla Emblue transaccional, DataLab/Webhooks complementarios, `surveyUrl` obligatoria y atribución sin PII en URL.
- Dashboard y Campaign Detail incluyen los campos acordados en la arquitectura; el dashboard permite buscar por `Campaign Name` y `Survey Name` y filtrar por `Status`, sin una card propia para `CANCELLED`.
- Casos positivos, negativos y edge cases tienen identificadores `AC-xxx` y están vinculados a reglas funcionales.
- Cada decisión aún no resuelta está identificada como **OPEN** y asignada a la validación correspondiente, sin inventar comportamiento.
- La especificación no contiene schema Prisma, diseño detallado de tablas, endpoints finales, implementación NestJS ni tasks.
- Roles efectivos autorizados y alcance multiempresa se fijan conforme a decisiones aprobadas; catálogo MOCK se limita a desarrollo/pruebas.
- El documento queda en estado **DRAFT**, versión **0.4**; aprobación formal pendiente.

## 17.1 Recuperación, suspensión y finalización (decisiones APPROVED)

Estas reglas cierran las propuestas de downtime/recovery para v1 y prevalecen sobre OPENs anteriores de este documento:

- Los slots vencidos elegibles se consolidan en una sola unidad de recovery por Campaign; no se reproduce un envío por cada slot omitido.
- El backlog automático inicial es de 30 días, configurable. Si la ocurrencia elegible más antigua excede la ventana, se alerta y requiere intervención; no se descartan ni avanzan como procesados los datos pendientes.
- `Suspend` impide claims y lotes nuevos. El lote ya iniciado puede finalizar de forma segura y persistir sus resultados.
- `Resume` antes de `EndDate` evalúa el recovery consolidado y calcula el siguiente slot ordinario futuro. Después de `EndDate` solo puede recuperar trabajo de slots originalmente válidos; nunca crea slots ordinarios posteriores.
- `COMPLETED` exige calendario concluido y cero trabajo pendiente, en curso o recuperable. Executions terminales son inmutables; una recuperación de fallo terminal usa una Execution nueva relacionada.
- `lastExecutionAt` representa `finishedAt` de la Execution terminal más reciente, incluyendo `FAILED`.
- Las claves de Execution son deterministas y versionadas; claims atómicos, lease y fencing o protección equivalente impiden que un worker rezagado escriba o envíe luego de perder propiedad.

**OPEN técnico/externo:** canales y umbrales de alerta, ventana configurable por ambiente, parámetros base/tope del backoff, consulta de estado/idempotencia de Emblue y clasificación documentada de errores. No se asume garantía de idempotencia del proveedor.

## 18. Estado del documento

- **Status:** DRAFT
- **Version:** 0.5
- **Base funcional:** baseline descrita en `PEC-SC-ARCH-001.md`.
- **Aprobación:** pendiente de revisión funcional y resolución de los OPENs aplicables.

## 19. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Se formalizan búsqueda por `Campaign Name` y `Survey Name`, filtro por `Status` y su cobertura en AC-015; se aclara el acceso de `CANCELLED` mediante filtro sin card propia; se reemplaza la regla de recuperación por una garantía neutral que evita marcar compras elegibles pendientes como procesadas u omitirlas en ejecuciones futuras. |
| 0.3 | 2026-10-20 | Se fijan roles ADMIN/SUPERADMIN y alcance multiempresa; se define selector de Survey específico, distinción Enterprise/Company, catálogo MOCK sintético reemplazable y límites de seguridad. Se difiere scheduler/envíos con MOCK; publicación y validación PostgreSQL permanecen OPEN. |
| 0.4 | 2026-10-20 | Se aprueba glosario de UI, equivalencias de estados/frecuencias, modalidades y etiquetas; se exige español para interfaz, MOCK, mensajes, fechas y paginación con payload técnico intacto. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se cierra la duración de recurrencias como conteo inclusivo de fechas de calendario para RB-035–RB-039; zonas horarias, DST y ejecución en End Date continúan OPEN. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se aprueba `America/Guayaquil` como zona predeterminada de Campaign para Ecuador, la persistencia explícita de `timeZone` sin inferencia del navegador y la ocurrencia inclusiva en `End Date`; publicación y recuperación/reanudación permanecen OPEN. Se corrige la referencia interna de versión 0.3. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se aprueba para `MONTHLY`/`QUARTERLY` el anclaje al día original de `StartDate`, con ajuste al último día disponible sin drift acumulativo; se aplica a años bisiestos y antes de convertir las fechas locales a UTC. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Cierra políticas v1 de backlog consolidado de 30 días, Suspend/Resume, completitud, Execution terminal inmutable, retry selectivo máximo 3 para errores transitorios confirmados y `lastExecutionAt`; contratos de clasificación/reconciliación Emblue siguen OPEN. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Cierra solo el contrato de request/response del envío individual emBlue API v2.3 con evidencia aportada; deja consulta de actividad, idempotencia y entrega OPEN. Estado DRAFT conservado. |


### Convención Database-First (2026-10-23)

PostgreSQL es la fuente de verdad. Los cambios estructurales se entregan como SQL para ejecución manual y se reflejan después en schema.prisma; Prisma Client se genera desde ese modelo. Esta convención no cambia reglas funcionales ni el estado DRAFT.

### Evidencia Sprint 3.1B (2026-10-23)

El checkout implementa contratos internos y rutas para Schedule, Suspend, Resume y consulta de Executions; Schedule/Resume responden bloqueados mientras publicación de Survey, audiencia real, catálogos no MOCK y contrato de plantilla no estén verificados. No se equipara `Survey.status = 1` con PUBLISHED. El worker `runOnce` es invocable solo como servicio, opera en dry-run por defecto y no está conectado a cron ni a `EmblueService`. Materialización y recovery de slots vencidos usan el límite configurable (30 días por defecto), transacción, lock por Campaign y Execution key determinista y versionada. No se añadió DDL. Build, ESLint focalizado y suite Campaign unitaria pasan; las pruebas PostgreSQL no pudieron ejecutarse en esta sesión porque Docker Engine denegó acceso y no había listener local en 5432. SPECS conservan v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra implementación local fail-closed del lifecycle y worker manual dry-run; no se habilita programación efectiva con precondiciones OPEN. Pruebas PostgreSQL de Sprint 3.1B pendientes por falta de acceso local a Docker/PostgreSQL. DRAFT conservado. |

### Sprint 3.1B.1 — criterios verificados

Los criterios de ciclo de vida, concurrencia del scheduler, avance del cursor, recovery tras EndDate, límite de backlog, fencing y no despacho se verificaron con 20 pruebas PostgreSQL; la regresión Campaign completa fue 128/128. Esto documenta evidencia de aceptación técnica de Sprint 3.1B, sin cambiar estado formal del SPEC ni habilitar el uso productivo de Schedule/Resume mientras continúen OPEN las dependencias reales de Survey/audiencia/plantilla. DRAFT conservado.
## Registro de validación Sprint 3.1C

La regla específica de Campaign confirma elegibilidad solo cuando `status === 1`, `isPublic === true` y `archived === false`; no define ni cambia una semántica global `PUBLISHED`. La ruta pública conserva su comportamiento para usuarios anónimos, pero no demuestra por sí sola disponibilidad durante todo el calendario ni el enlace individual. Las campañas siguen bloqueadas por audiencia real CF/Analytics, plantilla aprobada y enlace individual no verificado. Los catálogos MOCK nunca habilitan programación.

| 0.5 | 2026-10-23 | Registra el preflight fail-closed y mantiene OPEN publicación, enlace Campaign, audiencia CF/Analytics y plantilla. No cambia reglas funcionales ni estado DRAFT. |
## Sprint 3.1C.1 — decisiones funcionales OPEN

`status` se usa como estado operativo (el flujo público consulta status 1); `isPublic` habilita la búsqueda pública con fallback existente para encuestas genéricas; `archived` excluye encuestas archivadas. `availableFrom`/`availableTo` viven dentro de `content` y el servicio público los compara al momento de la solicitud. Para Campaign, la decisión aprobada define publicación con la conjunción de los tres campos; no se infiere un estado global `PUBLISHED` ni se requiere otro proceso.

El API construye enlaces de pantalla `/survey/:id?idD=1` con `PUBLIC_FORM_URL` (fallback local); también existe `GET /surveys/public/:id/:local`. No se encontró en el checkout pec-ui una ruta/consumidor que pruebe el enlace completo desde correo. El preflight verifica publicación para Campaign por la regla aprobada; la URL Campaign sigue `OPEN` y `READY` nunca determina por sí solo que la campaña pueda programarse.

Las reglas de elegibilidad, semántica de fecha de compra + offset, depuración de correo y supresión requieren definición del dueño funcional y contrato de Data-Lab. Permanecen pendientes las decisiones de validez del correo, duplicados, consentimiento/supresión y conteo. No se codifica una semántica de destinatario elegible a partir de nombres de campos.

| 0.5 | 2026-10-23 | Precisa el flujo observable de Survey, el alcance del preflight y decisiones de publicación/audiencia pendientes; DRAFT conservado. |


## Sprint 3.1C.1 - publicacion de Survey para Campaign

Regla funcional aprobada para Automated Survey Campaigns: una Survey es elegible solo si `status = 1`, `isPublic = true` y `archived = false`. `status=1` aislado no demuestra publicacion ni cambia el significado global de Survey. La ruta publica existente conserva su comportamiento, incluido el fallback para Surveys genericas; Campaign exige `isPublic=true` y no utiliza ese fallback.

El gate lee `content.availableFrom` y `content.availableTo` si estan configurados. Valida formato e intervalo y verifica que el calendario Campaign completo, desde inicio local hasta fin local inclusivo, quede cubierto. Usa la zona IANA de Campaign para el calendario y el parser server-local ya usado por la ruta publica para interpretar los limites Survey. Mantener TZ consistente entre replicas del API es un requisito operativo; acceso a respuestas despues de `availableTo` y enlace individual Campaign siguen pendientes.

Preflight reporta `SURVEY_PUBLICATION=READY` unicamente con la conjuncion aprobada; `SURVEY_CALENDAR_AVAILABILITY` distingue READY, BLOCKED y OPEN. Schedule y Resume vuelven a validar antes de mutar estado. No se modifica Prisma, no se cambia la ruta Surveys, y audiencia real, catalogos reales, plantilla y enlace individual siguen bloqueando la activacion. Se conserva v0.5 / DRAFT.

## Sprint 3.1C.2 - implementacion definitiva de elegibilidad Survey

La regla aprobada es definitiva para Campaign: `status === 1 && isPublic === true && archived === false`. No existe ni se requiere otro flujo de publicación/aprobación. El componente concreto `SurveyPublicationValidator` implementa esta condición; encuestas genéricas no evaden el requisito de `isPublic`. `content.availableFrom/availableTo` se valida aparte y debe cubrir el calendario programado. Resume vuelve a evaluar ambos resultados. El check de URL individual permanece OPEN y los bloqueos de audiencia, plantilla y catálogos MOCK siguen vigentes. Version 0.5 y estado DRAFT se conservan.

| 0.5 | 2026-10-23 | Registra regla Campaign de Survey, gate temporal y cobertura de pruebas; DRAFT conservado. |

## Sprint 3.1C.3 - fuentes y decisiones abiertas

No hay evidencia en el checkout de una fuente que resuelva compras elegibles y destinatarios Campaign desde Data-Lab/Analytics. Los endpoints corporativos ya usados por la sincronización PEC solo aportan áreas de trabajo/locales con alcance y mapeo incompletos. Se mantienen OPEN la fuente/contrato real y la semántica de elegibilidad. No interpretar la disponibilidad de datos PEC o las consultas individuales `ClientService`/`CollaboratorService` como permiso o mecanismo para campañas.

Decisiones pendientes: CF/Data-Lab debe confirmar propietario, endpoint/vista autorizada, autenticación/permisos, filtros e IDs estables; el dueño funcional define ventana/cutoff, fecha de compra + offset, deduplicación y conteos; Privacidad/Seguridad define consentimiento, supresión, minimización y retención. La campaña no se puede habilitar hasta validar esos contratos y catálogos reales. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra dependencias externas OPEN para fuente, elegibilidad y tratamiento de destinatarios; no aprueba reglas nuevas. DRAFT conservado. |
## Sprint 3.1C.4 - decision de arquitectura staging BI (propuesta)

Se documenta como propuesta pendiente de confirmacion por BI CF, no como contrato disponible: dos snapshots PostgreSQL `bi_integration.bi_segmentation_catalog` y `bi_integration.bi_customer_purchases`; SQL revisable en `pec-api/prisma/manual-sql/bi_campaign_staging_proposal.sql`. El diseno conserva Division, Department y Company como filtros obligatorios; clasificacion auxiliar, Format y Store opcionales; no incorpora Auxiliary Subclassification. Las tablas no prueban por si solas relacion jerarquica, elegibilidad de compra ni permiso para usar email.

Decisiones OPEN para BI: propietario/esquema/entorno y rol; grano de fila y claves estables; catalogos, labels, relaciones y Company multiempresa; tipo/zona de fecha; snapshots completos o incrementales; frecuencia, frescura y retencion; transaccion de publicacion y caso de fuente vacia; volumen/cursor; elegibilidad y compras tardias; unidad de envio y deduplicacion; disponibilidad de identificador/email permitido. Dueño funcional debe definir cutoff y destinatario unico; Privacidad/Seguridad debe aprobar consentimiento, supresion, acceso y retencion. No usar los datos para Schedule/Resume hasta cierre. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra staging BI como alternativa de diseno pendiente; no cierra OPENs ni modifica estado formal. DRAFT conservado. |

Riesgo de modelo: el ledger Campaign identifica recipient por compra, no por linea de producto. Si el staging BI se publica a grano compra-linea, la implementacion debe agregar de forma determinista por compra o presentar un cambio de unidad de envio; BI y dueño funcional deben confirmarlo. No asumir que un customer equivale a una compra ni deduplicar unicamente por correo.
## Sprint 3.1C.4.1 - decision funcional de factura

APPROVED: una invitacion por factura elegible dentro de una Campaign. La identidad es `invoice_source + invoice_external_id`; el origen debe identificar el sistema y el ID ser estable. Una factura con multiples productos/divisiones califica cuando al menos una asignacion producto/division satisface la segmentacion. Se agrupan sus filas y se crea una sola invitacion. Deduplicacion solo por Campaign + identidad de factura; no por cedula/correo global. Un cliente puede recibir invitaciones por facturas distintas.

El ledger existente ya tiene unicidad `(campaignId, purchaseSource, purchaseExternalId)` y no requiere DDL de PEC para cumplir el alcance aprobado. Las relaciones Campaign/Execution/Recipient/Attempt existentes, junto a los campos de origen/ID en Recipient, permiten seguir la invitacion hasta la identidad de compra externa. El staging conserva lineas/productos/divisiones para explicar elegibilidad; el `load_id` se debe guardar en snapshot de Execution al integrar la fuente. Participacion de una factura en distintas Campaigns y reglas de contacto/exclusion siguen OPEN. v0.5 / DRAFT.
## Sprint 3.1D - comportamiento probado con datos sinteticos

El proveedor PostgreSQL provisional aplica OR dentro de cada dimension y AND entre dimensiones; agrupa lineas por `(invoice_source, invoice_external_id)` antes de paginar. Una factura con varias lineas elegibles produce un candidato; facturas distintas del mismo cliente permanecen separadas. Correos ausentes/invalidos se cuentan en preview agregado y se excluyen de la pagina elegible. Conflictos de identidad/contacto dentro de una factura invalidan el lote. Esto prueba el adaptador con fixtures, no reglas de contactabilidad BI aprobadas.

El preview devuelve conteos, fuente, entorno, snapshot y fecha, sin datos personales. El proveedor provisional no valida audiencia real ni desbloquea Schedule/Resume. Participacion entre campañas, consentimiento, supresion y elegibilidad real siguen OPEN. DRAFT conservado.

| 0.5 | 2026-10-23 | Registra comportamiento del proveedor provisional con fixtures sinteticos; conserva OPEN y DRAFT. |

## Sprint 3.1E - invitación individual (2026-10-23)

La unidad continúa siendo una invitación por factura elegible y Campaign. La correlación propuesta usa token aleatorio opaco asociado al Attempt; la base conserva únicamente su hash y la respuesta se puede enlazar una vez con `surveyResultId`. No se define una vigencia predeterminada. El token todavía no se incluye en un enlace funcional ni se transporta por el flujo público actual. La plantilla sintética de prueba no contiene PII y no representa aprobación funcional de contenido. No se envía correo. v0.5 / DRAFT.
## Sprint 3.1F - Regla de invitación individual (2026-10-23)

Una invitación Campaign usa un token aleatorio opaco de un solo uso; se guarda únicamente SHA-256 del token y se exige una expiración futura explícita. El token debe apuntar a un Attempt cuya Campaign determina la Survey. La lectura y el guardado vuelven a validar Survey activa, pública, no archivada y vigente en fecha de negocio America/Guayaquil. Un segundo guardado, token vencido o enlace no relacionado con la Survey se rechaza. Las encuestas version 2 quedan bloqueadas en este flujo porque el guardado general también crea una queja y ese efecto no forma parte de la transacción Campaign. v0.5 / DRAFT.

## Sprint 3.1F.1 - compatibilidad Survey v2 (2026-10-24)

Las invitaciones Campaign siguen habilitadas solo para versiones que no generan el flujo de queja Survey v2. No se permite eludir la creación de queja requerida por Surveys. La compatibilidad v2 queda pendiente de un diseño que preserve ese efecto junto con la correlación de respuesta. v0.5 / DRAFT.
## Sprint 3.1F.1 - compatibilidad de Survey para Campaign (2026-10-24)

**APPROVED:** Automated Survey Campaigns admite únicamente encuestas de satisfacción que registran respuestas. El criterio verificable en el modelo actual es `version === 1` y `content.isQueja !== true`. Se bloquean `version === 2`, cualquier versión desconocida y registros inconsistentes que declaren `isQueja: true`; no se cambia el comportamiento general de Surveys ni de Complaints. La compatibilidad se valida centralmente en selección, preflight, Schedule/Resume y apertura/guardado de invitación. La validación unitaria focalizada aprobó 157 pruebas Campaign. La concurrencia, expiración y rollback contra PostgreSQL local no se certifican en esta entrega porque no se verificó una instancia aislada disponible. v0.5 / DRAFT.
## Sprint 3.1G-A - preparación sintética sin envío (2026-10-24)

La simulación admite solo Campaign DRAFT en entorno no productivo con fuente PostgreSQL provisional publicada. OR dentro de filtros y AND entre filtros permanecen responsabilidad del proveedor; la orquestación deduplica por origen + factura, no por email/cliente. Para cada candidata contactable genera en memoria una invitación y un correo de prueba con URL efímera `example.invalid`; la respuesta HTTP es agregada y no presenta datos personales. `productiveExecution.ready` permanece falso con blockers reales. No persiste tokens, no consume correlaciones y no llama al proveedor de correo. La simulación no constituye elegibilidad/contactabilidad BI aprobada ni prueba transaccional. v0.5 / DRAFT.


## Sprint 3.1G-B - simplificación de conexión Campaign (2026-10-25)

La forma de obtener catálogos/audiencia no cambia las reglas funcionales: Campaign consulta el proveedor provisional mediante la conexión PostgreSQL existente de PEC y el esquema `bi_integration`. La selección es automática solo en entornos local, development o test permitidos; producción/entorno desconocido queda bloqueado. El origen provisional sigue sin ser BI validado y no desbloquea Schedule/Resume.

Para la validación independiente de rama se requieren solo Project ID, Branch ID, propietario esperado de la base y credencial de API Neon (`CAMPAIGN_PROVISIONAL_NEON_PROJECT_ID`, `CAMPAIGN_PROVISIONAL_NEON_BRANCH_ID`, `CAMPAIGN_PROVISIONAL_NEON_DATABASE_OWNER`, `CAMPAIGN_PROVISIONAL_NEON_API_KEY`). El endpoint se extrae del host de `DATABASE_URL`, y la base y el rol de esa misma URL; estos datos de identidad no seleccionan proveedor ni crean otra conexión. **Registro de revisión:** v0.5, 2026-10-25. Se registra la decisión de conexión única/esquemas separados y selección automática protegida. Estado formal `DRAFT` conservado.


## Sprint 3.1G-B.1 - auditoria de configuracion Campaign (2026-10-25)

Inspeccion local de nombres de variables, sin leer ni publicar valores: `pec-api/.env` y `.env.local` contienen `DATABASE_URL` y `ENV`. `AppModule` carga globalmente `.env.local`, `.env.preprod`, `.env.prod` y `.env`; el guard lee `process.env`. No se encontro `DIRECT_URL` ni una identidad Neon reutilizable en esos archivos. `PROJECT_ID_GCS` identifica GCP y `API_KEY_STAR` pertenece a otra integracion; no son sustitutos de identidad Neon. El inventario local no prueba variables inyectadas externamente al proceso desplegado.

En el checkout, `CampaignNeonTargetGuard` aun exige Project ID, Branch ID, propietario esperado y credencial API Neon. Ninguno aparece en los archivos locales. El error 503 se origina al faltar esa configuracion: el guard falla antes de llamar a la API de Neon y antes de leer el lote en `bi_integration`. Endpoint ID, base y rol se derivan del `DATABASE_URL`; no se requiere otra conexion ni un `DIRECT_URL`. No se modifico `DATABASE_URL`.

No se retiro la verificacion de rama: `ENV` distingue el entorno PEC, pero no identifica una rama Neon; el hostname solo permite derivar el endpoint. La verificacion API Project/Branch/Endpoint y la comprobacion SQL posterior (base, esquema, rol, propietario y primary) siguen siendo controles distintos. El mecanismo equivalente no puede configurarse con los datos que hoy existen en los archivos inspeccionados. Dato pendiente para una lectura segura: una fuente confiable, ya disponible para NestJS, que relacione la conexion actual con el Project ID y Branch ID de desarrollo y permita verificar esa relacion en Neon. Hasta resolverlo, la lectura funcional falla cerrada. Los SPECS permanecen v0.5 / DRAFT.
## Consolidación de conexión Campaign (2026-10-25)

La selección de audiencia provisional desde `bi_integration` es una decisión de implementación y no convierte los datos sintéticos en una fuente BI funcionalmente aprobada. La regla vigente de arquitectura es reutilizar `DatabaseService`/`DATABASE_URL` de PEC y no habilitar Schedule, Resume ni envíos por el solo hecho de poder leer catálogos. Los selectores históricos `CAMPAIGN_*_PROVIDER` y la bandera provisional quedan supersedidos.

Las reglas de identidad de Neon, permisos PostgreSQL y protección del destino se mantienen en ARCH/INTEG; no son criterios de elegibilidad funcional de Survey o audiencia. La API administrativa de Neon no es requisito funcional ni parte del contrato de lectura objetivo. Estado formal `DRAFT`; versión v0.5.

## Actualización funcional 2026-10-25 — Fuentes y correspondencia de segmentación

Fuentes implementadas: División, Departamento y Clasificación auxiliar se consultan desde `bi_integration.bi_segmentation_catalog`; Empresa, Formato y Local se consultan desde los modelos activos PEC `sspectenterprise`, `sspectformat` y `sspectestablishment`. Las relaciones PEC verificadas son Formato → Empresa mediante `enterpriseId` y Local → Formato mediante `formatId`. Las compras siguen en `bi_integration.bi_customer_purchases` y se consolidan por `(invoice_source, invoice_external_id)` antes de paginar.

El esquema PEC no contiene una columna de mapeo explícita a `company_external_id`, `format_external_id` o `store_external_id` de BI. La implementación solo acepta igualdad textual exacta entre IDs; no compara por nombre/código ni similitud. Si falta igualdad exacta, preview y operaciones dependientes deben bloquearse e informar la correspondencia faltante. Este estado no declara validado el mapeo BI.

`GET /survey-campaigns/:campaignId/audience-preview?page=N` devuelve conteos de clientes con identificador, facturas elegibles/contactables, registros excluidos por correo ausente/inválido y filas consolidadas. La página contiene etiquetas generadas para cliente/factura y fecha, sin correo, ID de cliente ni identificador real de factura. Preview es de solo lectura. Mantener el estado formal DRAFT y versión 0.5.

Los datos BI persistidos y el seed ya ejecutado no se modificaron. El reemplazo por ejemplos sintéticos representativos (taxonomía general de abarrotes/productos y clientes ficticios) requiere una operación de carga autorizada; no ejecutar ni modificar datos en esta entrega. OPEN: confirmar que los IDs externos BI coinciden con las PK PEC o entregar un mapeo autorizado, además de reglas de contactabilidad/exclusión.


### Ajuste de seleccion y experiencia Campaign (2026-10-27, DRAFT)

El selector lista encuestas de satisfaccion activas, publicas, no archivadas y vigentes hoy en America/Guayaquil. Excluye vigencias invalidas, futuras y vencidas; los extremos son inclusivos y los limites ausentes no restringen el periodo. Esta seleccion no depende de completar primero la programacion.

Crear y editar comparten el orden: informacion general, audiencia y compras, programacion, encuesta y guardar. Tras completar el calendario, el selector limita las encuestas vigentes hoy a las que cubren todo el periodo elegido. Si la segmentacion contiene una o varias empresas, el selector muestra solo encuestas cuyo `enterpriseId` coincide textualmente con alguna empresa elegida; sin empresas seleccionadas conserva disponibles las encuestas de todas las empresas. El calendario se contrasta con la vigencia de la encuesta elegida; una asociacion antigua no disponible se informa sin volver a ofrecerla como opcion.

Observar presenta resumen, programacion, filtros plegables, audiencia, incidencias corregibles e historial. Una unica accion de envio manual usa el transporte bulk existente, con confirmacion. El transporte individual permanece en el API. La revision automatica conserva las validaciones del servidor; los diagnosticos de integracion no se exponen como tareas del usuario. No se habilitan cron jobs con este ajuste.
