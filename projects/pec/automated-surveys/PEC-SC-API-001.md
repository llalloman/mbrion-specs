# Contrato API Interno - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-API-001 |
| Estado | DRAFT |
| Versión | 0.2 |
| Fecha | 2026-10-18 |
| Idioma | Explicaciones en español; rutas, DTOs, enums y nombres técnicos en inglés |
| Documentos base | `PEC-SC-ARCH-001.md`, `PEC-SC-FOUND-001.md`, `PEC-SC-DATA-001.md` |

> Este documento define el contrato propuesto entre Angular y PEC API. Las rutas nuevas no están implementadas. Se conservan las convenciones verificadas y se marcan como **OPEN** los permisos concretos, contratos externos y reglas que requieren confirmación. No define rutas de CF ni Emblue ni genera código.

## 1. Hallazgos de convenciones actuales

Inspeccionados controllers, DTOs, guards, paginación y filtros globales de `pec-api`:

| Área | Convención verificada | Consecuencia para este contrato |
|---|---|---|
| Rutas | NestJS usa `@Controller('resource')` y métodos REST con `GET`, `POST`, `PATCH`, `PUT` y `DELETE`. Ejemplos: `surveys`, `categories`, `catalogs`, `case-locks`. | Rutas nuevas se expresan relativas al API base; el prefijo de proxy/despliegue no se fija aquí. |
| DTOs | `class-validator`; en ciertos DTOs `class-transformer`; `PartialType`/`OmitType` disponibles. | Request bodies propuestos son DTOs camelCase, validados por tipo, enum, rango, arrays y reglas condicionales. |
| Validación global | `ValidationPipe({ transform: true, whitelist: true })`. | Tipos se transforman y propiedades no declaradas se excluyen. No se verificó `forbidNonWhitelisted`. |
| Paginación | Query params `page` y `perPage`, defaults usados `1` y `10`; Prisma `skip/take`. | Listados nuevos usan `page`/`perPage` y metadatos en `pagination`. Deben validar enteros positivos y un límite de página acordado. |
| Respuesta paginada | Hay respuestas `{ data, pagination: { page, perPage, total } }`; otro servicio agrega `totalPages`. Surveys puede agregar metadatos propios dentro de `pagination`. | Para el recurso nuevo se normaliza a `{ data, pagination: { page, perPage, total, totalPages } }`, compatible con el patrón existente. |
| Respuesta individual | Los controllers suelen devolver el objeto/resultado del service directamente; algunos endpoints de operación devuelven `{ success, message, ... }`. | Recursos Campaign/Execution se devuelven directamente; comandos devuelven el recurso actualizado. |
| Autenticación | `AuthGuard`, `CaslGuard` y `SurveyAccessGuard` están registrados globalmente. `@Public()` excluye rutas de ese flujo. | Los endpoints administrativos son autenticados; no deben decorarse como públicos. |
| Autorización | CASL genera `request.abilities`; muchos controllers llaman `PoliciesGuard` y `@CheckPolicies(Action.Read/Create/Update/Delete, subject)`. | Aplicar políticas de recurso a todas las rutas nuevas. El subject y matriz Campaign todavía deben incorporarse/configurarse. |
| Contexto autenticado | `AuthGuard` adjunta `request.user` (`sspectuser`) y `request.role`; rol trae `user_id`, `enterprise_id` y rol asociado. | El API toma identidad y scope del contexto autenticado; no confía en `createdByUserId` o permisos enviados por Angular. |
| Errores | `GlobalExceptionFilter` devuelve `statusCode`, `timestamp`, `path`, `method`, `message`, `error`, `code` opcional; stack solo en desarrollo. | El contrato conserva esta envoltura. Los códigos funcionales indicados abajo son propuestos, no todos existen hoy. |
| Errores de validación | `ValidationPipe` y excepciones NestJS; el proyecto usa `BadRequestException`, `NotFoundException`, `ForbiddenException`, `UnauthorizedException` y `ConflictException`. | Usar status HTTP estándar y mensaje comprensible en español, evitando payloads con PII. |
| Encuestas existentes | `GET /surveys` ya acepta `page`, `perPage`, `enterpriseId`, `status`, `searchText`, `includeArchived` y otros filtros; su respuesta es paginada. `GET /surveys/:id` consulta una encuesta. | Reutilizar la capacidad existente para selección/consulta, con autorización y scope actual. No inventar un endpoint externo ni mapear `status=1` a `PUBLISHED`. |

El backend global requiere autenticación salvo rutas expresamente `@Public()`. El listado `GET /surveys` no usa un `PUBLISHED` explícito verificable: el schema actual tiene `status Int`, `archived` y `content Json`; el significado de publicación requiere validación del módulo Surveys.

## 2. Convenciones del contrato propuesto

- **Base:** rutas relativas a PEC API, sin asumir prefijo de gateway. Los IDs internos de Campaign, Survey y Execution son enteros PEC; los IDs externos de CF/Analytics y Emblue se transportan como strings opacos.
- **Nombres:** campos nuevos en `camelCase`; valores de enums exactamente como se definieron en FOUND/DATA.
- **Éxito:** `GET` → `200`; `POST` de creación → `201`; `PATCH` → `200`; `DELETE` lógico permitido → `200` con confirmación. Respuesta de un recurso: objeto directo; listas: `{ data, pagination }`.
- **Errores:** envoltura del `GlobalExceptionFilter` actual. `code` es opcional en la implementación actual; los códigos de este contrato son recomendados para que Angular no dependa de textos.
- **Fechas:** `YYYY-MM-DD` para fechas de calendario, `HH:mm` para hora local, ISO-8601 con zona/desplazamiento para timestamps. Zona IANA y regla DST son OPEN; no interpretar hora local en silencio como UTC.
- **Segmentation:** IDs externos seleccionados; no se aceptan labels enviados por Angular como fuente autoritativa. Las etiquetas de catálogo deben resolverse en servidor.
- **PII:** no devolver emails completos, token de correlación, `purchaseId` ni `transactionId` en rutas de Campaign, Preview, Execution o Recipient salvo autorización/decisión específica aprobada. Ninguna respuesta devuelve tokens de correlación.

## 3. Seguridad y permisos

Todos los endpoints de este documento son administrativos e internos para Angular, requieren autenticación PEC y verificación de permisos en backend. Los nombres de acciones describen capacidades; su mapeo final a roles PEC y CASL es **OPEN**.

| Capacidad requerida | Acciones |
|---|---|
| `Campaign:Read` | Listado, resumen, detalle, catálogos de segmentación, Audience Preview, listado/detalle de executions y consulta de recipients autorizada. |
| `Campaign:Create` | Crear Campaign en `DRAFT`. |
| `Campaign:Update` | Editar configuración en estados permitidos y ejecutar `schedule`, `suspend`, `resume`, `cancel`. Validar estado además del permiso. |
| `Campaign:Delete` | `DELETE` únicamente para Campaign `DRAFT` sin historial, según política de soft delete aprobada. |
| `Survey:Read` | Seleccionar Survey existente en el scope autorizado. |

En términos de la convención CASL, se propone usar acciones `Action.Read`, `Action.Create`, `Action.Update` y `Action.Delete` sobre los subjects nuevos de Campaign/Execution/Recipient una vez generados por Prisma. El checkout actual no incluye Campaign en `AppSubjects` ni una matriz de permisos específica. **OPEN:** definir subjects exactos, roles que reciben cada permiso, autorización para datos de recipients y scope por `enterpriseId`.

El API obtiene el usuario actual desde `request.user` y el rol/scope desde `request.role`. El body no puede elegir el creador, ampliar el `enterpriseId` permitido ni actuar sobre Campaign fuera del scope autorizado. Si la existencia del recurso no es visible al usuario, responder de manera que no filtre información entre scopes; regla exacta OPEN.

## 4. DTOs de referencia

Estos DTOs son formas de contrato, no clases implementadas.

### 4.1 `CampaignWriteDto`

```json
{
  "name": "Encuesta post compra",
  "description": "Invitación posterior a la compra",
  "surveyId": 123,
  "postPurchaseOffset": 2,
  "emailTemplateExternalId": "template-ref",
  "segmentation": {
    "divisions": ["DIV-01"],
    "departments": ["DEP-10", "DEP-11"],
    "auxiliaryClassifications": [],
    "companies": ["COMP-01"],
    "formats": [],
    "stores": []
  },
  "schedule": {
    "sendType": "FREQUENCY",
    "startDate": "2026-11-01",
    "endDate": "2027-01-31",
    "sendTime": "09:30",
    "frequency": "WEEKLY",
    "timeZone": "America/Guayaquil"
  }
}
```

- `name`: requerido, string no vacío.
- `description`: opcional.
- `surveyId`: entero positivo de una Survey existente y accesible.
- `postPurchaseOffset`: entero >= 0.
- `emailTemplateExternalId`: opcional en el contrato inicial mientras Emblue no confirme su identificador/selección; su obligatoriedad para schedule es **OPEN**.
- `segmentation`: objeto opcional al guardar un borrador incompleto; para schedule requiere al menos un valor en `divisions`, `departments` y `companies`. Opcionales: `auxiliaryClassifications`, `formats`, `stores`. No se acepta Auxiliary Subclassification.
- IDs de filtros: arrays de strings no vacíos, sin duplicados dentro del mismo filtro. La comprobación de existencia/compatibilidad se hace contra los catálogos disponibles.
- `schedule`: objeto opcional para permitir configurar un `DRAFT`; `sendType` será requerido para persistir la forma si DATA-001 queda así aprobado. Para programar, se exigen las propiedades según SendType.
- `createdByUserId`, `status`, `nextExecutionAt`, `lastExecutionAt`, conteos y campos de auditoría no son editables desde Angular.

**OPEN:** DATA-001 propone `Campaign.sendType` no nullable, pero FOUND permite un borrador antes de programar y no termina de fijar qué campos mínimos debe contener un borrador. Este contrato permite que las fechas estén incompletas en DRAFT; confirmar si `sendType` y segmentation son obligatorios al crear o solo al schedule antes de implementar.

### 4.2 `CampaignScheduleDto`

No recibe estado, `nextExecutionAt` ni `lastExecutionAt`; la acción valida la configuración almacenada.

- `ONE_TIME`: `sendDate` y `sendTime` requeridos. `startDate`, `endDate`, `frequency` deben omitirse.
- `FREQUENCY`: `startDate`, `endDate`, `sendTime` y `frequency` requeridos. `sendDate` debe omitirse.
- La API verifica fechas coherentes, frecuencia permitida para la duración, Survey `PUBLISHED`, filtros requeridos y `postPurchaseOffset >= 0`.
- La zona horaria debe ser explícita al activar calendario, pero el identificador definitivo y si se guarda en Campaign son **OPEN**.

### 4.3 `AudiencePreviewDto`

```json
{
  "campaignId": 123,
  "evaluationDate": "2026-11-03",
  "configuration": {
    "surveyId": 123,
    "postPurchaseOffset": 2,
    "segmentation": {
      "divisions": ["DIV-01"],
      "departments": ["DEP-10"],
      "companies": ["COMP-01"]
    }
  },
  "includeSample": false,
  "sampleSize": 0
}
```

`evaluationDate` es la fecha de negocio (`YYYY-MM-DD`) usada para determinar elegibilidad y se distingue de `calculatedAt`, que es el timestamp en que el servidor calculó la respuesta. La regla es `PurchaseDate + PostPurchaseOffset <= evaluationDate`, interpretada según la zona horaria y regla exacta de corte que se aprueben (**OPEN**). `campaignId` identifica una Campaign existente opcionalmente; `configuration` permite preview de una configuración todavía no guardada. Si se envían ambos, `configuration` se interpreta como la propuesta a evaluar y no actualiza Campaign. Conflictos de propiedad entre ID y payload se validan. `sampleSize` se omite o limita al máximo que establezca el servicio; el límite queda OPEN.

`evaluationDate` es requerido y debe ser una fecha calendario válida en el formato indicado.

## 5. Endpoints de Campaign

### 5.1 Listar campañas

`GET /survey-campaigns`

| Query | Tipo | Default | Descripción |
|---|---|---:|---|
| `page` | integer >= 1 | 1 | Página. |
| `perPage` | integer >= 1 | 10 | Elementos por página; el máximo es configurable/OPEN. |
| `searchText` | string | — | Búsqueda por `Campaign Name`; normalización/case-insensitive recomendada. |
| `surveySearchText` | string | — | Búsqueda por `Survey Name`; normalización/case-insensitive recomendada. |
| `status` | `CampaignStatus` | — | Un estado por consulta, incluido `CANCELLED`. |
| `sendType` | `CampaignSendType` | — | Filtro opcional. |
| `frequency` | `CampaignFrequency` | — | Filtro opcional. |
| `sortBy` | `name`, `status`, `nextExecutionAt`, `createdAt` | `createdAt` | Campo en allowlist. |
| `sortOrder` | `asc`, `desc` | `desc` | Orden. Desempate estable por `id`. |

Respuesta `200`:

```json
{
  "data": [
    {
      "id": 123,
      "name": "Encuesta post compra",
      "survey": { "id": 45, "name": "Experiencia de compra" },
      "sendType": "FREQUENCY",
      "frequency": "WEEKLY",
      "nextExecutionAt": "2026-11-08T09:30:00-05:00",
      "status": "ACTIVE",
      "createdAt": "2026-10-18T16:00:00Z",
      "updatedAt": "2026-10-20T14:00:00Z"
    }
  ],
  "pagination": { "page": 1, "perPage": 10, "total": 1, "totalPages": 1 }
}
```

Cuando se envían ambos campos de búsqueda, se combinan con AND: la Campaign debe coincidir con `searchText` y su Survey con `surveySearchText`. Cada campo solo busca en su nombre correspondiente. Los filtros `status`, `sendType` y `frequency` también se aplican junto con las búsquedas. `status` admite todos los valores de `CampaignStatus`, incluido `CANCELLED`. El listado no incluye configuración completa, emails, recipients ni tokens. Campaign se filtra por scope autorizado del usuario; no se acepta un scope que amplíe permisos.

### 5.2 Consultar resumen del dashboard

`GET /survey-campaigns/dashboard/summary`

Respuesta `200` (conteos sobre el scope autorizado):

```json
{
  "total": 12,
  "byStatus": {
    "DRAFT": 3,
    "SCHEDULED": 2,
    "ACTIVE": 4,
    "SUSPENDED": 1,
    "COMPLETED": 1,
    "CANCELLED": 1
  }
}
```

`total` es el número de Campaigns visibles y no eliminadas lógicamente dentro del scope autorizado; incluye `CANCELLED`. La interfaz presenta seis tarjetas: `Total`, `Active`, `Scheduled`, `Draft`, `Suspended` y `Completed`. `CANCELLED` permanece en `byStatus` y disponible como filtro, sin tarjeta propia.

### 5.3 Crear Campaign

`POST /survey-campaigns`

Request: `CampaignWriteDto`; la API ignora/rechaza campos de lifecycle y auditoría que no sean editables.

Respuesta `201`: recurso Campaign con `status: "DRAFT"`, IDs asignados, Survey resumida, segmentation y schedule configuradas, y timestamps. No inicia envío ni crea Execution.

Validaciones: `name`, Survey existente y visible, offset entero no negativo; el estado de publicación no bloquea el guardado en `DRAFT`. Reglas de completitud del borrador **OPEN** según DTO de referencia.

### 5.4 Consultar detalle

`GET /survey-campaigns/:campaignId`

Respuesta `200`: recurso Campaign completo permitido para consulta: `id`, `name`, `description`, Survey (`id`, `name`, estado funcional disponible), `emailTemplate`, `status`, `sendType`, `frequency`, `nextExecutionAt`, `lastExecutionAt`, calendario, segmentation con IDs/labels, offset, `createdBy`, `createdAt`, `updatedAt`. No incluir email de recipient ni token de correlación.

`campaignId` debe ser entero PEC positivo. Inexistente/no visible: `404` o respuesta equivalente sin fuga cross-scope, según política PEC.

### 5.5 Actualizar Campaign

`PATCH /survey-campaigns/:campaignId`

Request: campos editables de `CampaignWriteDto` en forma parcial. No admite `status`, `nextExecutionAt`, `lastExecutionAt`, `createdByUserId` ni conteos.

Respuesta `200`: Campaign actualizada.

Validaciones de estado:

- `DRAFT`, `SCHEDULED` y `SUSPENDED`: edición conforme al baseline.
- `ACTIVE`: rechazar; primero debe suspenderse.
- `COMPLETED` y `CANCELLED`: solo lectura.
- Cambiar Survey no publicada puede guardarse en DRAFT; una operación posterior de schedule vuelve a validar `PUBLISHED`.
- Al cambiar un filtro padre, limpiar/validar IDs hijos que ya no correspondan. Filtros obligatorios se exigen al programar.

Conflicto de concurrencia/versión y uso de `version` optimista **OPEN**; no aceptar sobrescritura silenciosa si el servicio implementa control de versión.

### 5.6 Schedule

`PATCH /survey-campaigns/:campaignId/schedule`

Request: sin body; programa la configuración guardada. Respuesta `200`: Campaign actualizada con `status: "SCHEDULED"` y `nextExecutionAt` calculada.

Validaciones: transición permitida; Survey existente, accesible y `PUBLISHED`; segmentation obligatoria completa; offset >= 0; SendType y fechas/hora/frecuencia válidas; frecuencia permitida para la duración; Email Template según regla OPEN. `DRAFT` es el estado esperado para la primera programación; **OPEN:** condiciones exactas para schedule de Campaign `SUSPENDED` deben definirse por separado de Resume.

### 5.7 Suspend

`PATCH /survey-campaigns/:campaignId/suspend`

Request sin body. Respuesta `200`: Campaign con `status: "SUSPENDED"`. Desde `SCHEDULED` o `ACTIVE`; campaña suspendida no inicia nuevas ejecuciones. Si ya hay una Execution `PROCESSING`, aplica la política OPEN del documento funcional.

### 5.8 Resume

`PATCH /survey-campaigns/:campaignId/resume`

Request sin body. Respuesta `200`: Campaign reanudada y `nextExecutionAt` según la regla aprobada. Desde `SUSPENDED` solamente. Estado resultante `SCHEDULED` o `ACTIVE` depende de si ya hubo ejecuciones; la fecha vencida durante suspensión y recuperación quedan OPEN según FOUND-001.

### 5.9 Cancel

`PATCH /survey-campaigns/:campaignId/cancel`

Request sin body; respuesta `200`: Campaign `CANCELLED`. Solo desde estados para los que FOUND permite cancelar: `DRAFT`, `SCHEDULED`, `ACTIVE`, `SUSPENDED`. No inicia ejecuciones futuras y conserva historial. Tratamiento de trabajo ya iniciado es OPEN.

### 5.10 Delete de borrador

`DELETE /survey-campaigns/:campaignId`

Permitido solo en `DRAFT` y sin historial de Execution/Recipient. La acción es lógica, según decisión de DATA-001 todavía OPEN; respuesta `200` propuesta:

```json
{ "success": true, "message": "Campaign eliminada." }
```

Si tiene historial o estado diferente de DRAFT: `409 CAMPAIGN_DELETE_NOT_ALLOWED`. No se elimina físicamente una Campaign con historial. Si no se adopta soft delete, cambiar el comportamiento antes de implementación sin eliminar historial.

## 6. Catálogos de segmentación

Estos son endpoints **internos de PEC** que delegan a un adapter de catálogo. No describen endpoints de CF/Analytics.

| Método y ruta | Parámetros requeridos/opcionales | Dependencia funcional |
|---|---|---|
| `GET /survey-campaigns/segmentation/divisions` | `page`, `perPage`, `searchText` | Catálogo raíz de Division. |
| `GET /survey-campaigns/segmentation/departments` | `divisionIds` requerido, `page`, `perPage`, `searchText` | Department dependiente de Division. |
| `GET /survey-campaigns/segmentation/auxiliary-classifications` | `parentIds` opcional según contrato, `page`, `perPage`, `searchText` | Auxiliary Classification opcional; dependencia padre exacta OPEN. |
| `GET /survey-campaigns/segmentation/companies` | `page`, `perPage`, `searchText` | Catálogo raíz Company. |
| `GET /survey-campaigns/segmentation/formats` | `companyIds` según jerarquía, `page`, `perPage`, `searchText` | Format opcional, dependencia exacta OPEN. |
| `GET /survey-campaigns/segmentation/stores` | `companyIds` y/o `formatIds` según contrato, `page`, `perPage`, `searchText` | Store opcional, dependencia exacta OPEN. |

La serialización de arrays repetidos en query (`divisionIds=...&divisionIds=...` o CSV) se debe fijar al implementar y documentar en OpenAPI. Respuesta `200` para todos:

```json
{
  "data": [
    { "externalId": "DEP-10", "label": "Hogar", "parentExternalId": "DIV-01" }
  ],
  "pagination": { "page": 1, "perPage": 10, "total": 1, "totalPages": 1 }
}
```

Las selecciones son multiselect. `Auxiliary Subclassification` no tiene endpoint v1. Cambio de parent invalida/remueve selecciones hijas incompatibles. Error o falta de disponibilidad del catálogo debe devolver fallo trazable, no catálogo vacío como si no existieran valores. **OPEN:** fuente y conexión, claves, compatibilidad de padres, cache/frescura y máximos de paginación. En NestJS, declarar rutas estáticas (`dashboard`, `segmentation`, `audience`) antes de rutas con `:campaignId` para evitar ambigüedad.

## 7. Audience Preview

`POST /survey-campaigns/audience/preview`

Request: `AudiencePreviewDto` con configuración necesaria. Es una consulta de cálculo en backend; no persiste Campaign ni marca compras como procesadas.

Respuesta `200`:

```json
{
  "evaluationDate": "2026-11-03",
  "calculatedAt": "2026-10-18T16:00:00Z",
  "eligiblePurchases": 1250,
  "eligibleCustomers": 1192,
  "sample": []
}
```

Reglas:

- Calcular con segmentation y `PurchaseDate + PostPurchaseOffset <= evaluationDate`, interpretada según la zona y regla de corte que se aprueben. `evaluationDate` es una fecha de evaluación de negocio; `calculatedAt` es el instante de cálculo del servidor y no la sustituye. Zona horaria y corte exacto permanecen **OPEN**.
- Para `campaignId` existente, excluir compras ya procesadas por esa Campaign. La deduplicación interna permanece `campaignId + purchaseId/transactionId`.
- Preview no persiste `CampaignRecipient`, no crea attempts ni modifica estados de Campaign, Execution o Recipient.
- `sample` es opcional y limitada; `email`, token, purchase IDs y payload fuente se omiten. Formato/datos permitidos de muestra **OPEN**.
- Angular nunca recibe la audiencia completa.
- Falla de fuente no devuelve conteos parciales silenciosamente; responde error de integración en formato PEC.

Idempotencia: la operación no modifica estado ni recipients y repetir la misma petición puede recalcular un conteo distinto porque la fuente corporativa cambia. No se requiere `Idempotency-Key` para evitar escrituras; `evaluationDate` identifica el día de negocio evaluado y `calculatedAt` hace visible cuándo se produjo ese snapshot de consulta.

## 8. Ejecuciones y destinatarios

### 8.1 Listar executions por Campaign

`GET /survey-campaigns/:campaignId/executions`

Query: `page`, `perPage`, `status` (`CampaignExecutionStatus`), `fromDate`, `toDate`, `sortBy` (`scheduledAt`, `startedAt`, `finishedAt`, `status`), `sortOrder`. Default page/perPage 1/10; orden `scheduledAt desc`, desempate por `id desc`.

Respuesta paginada `200`. Cada fila muestra `id`, `scheduledAt`, `startedAt`, `finishedAt`, conteos `eligiblePurchases`, `eligibleCustomers`, `sentCount`, `failedCount`, `duplicateCount` y `status`. No incluye snapshot completo ni datos personales.

### 8.2 Detalle de Execution

`GET /survey-campaigns/:campaignId/executions/:executionId`

Respuesta `200`: datos de ejecución, conteos, timestamps, status, error resumido sanitizado y los campos funcionales de snapshot necesarios para consulta, sin PII ni token. Verificar que Execution pertenezca a `campaignId` y scope del usuario; si no, `404` conforme a política contra IDOR.

### 8.3 Listar recipients

`GET /survey-campaigns/:campaignId/executions/:executionId/recipients`

La respuesta representa los outcomes persistentes de esa Execution mediante `CampaignRecipientAttempt` o estructura equivalente. Cada fila corresponde a un outcome/attempt de un `CampaignRecipient`; no es una lista deduplicada del estado actual del ledger.

Query: `page`, `perPage`, `attemptStatus` (`CampaignRecipientStatus`, filtrado sobre el outcome), `sortBy` (`attemptedAt`, `attemptStatus`, `attemptNumber`), `sortOrder`. Respuesta paginada `200` con campos mínimos: `attemptId`, `recipientId`, `campaignRecipientStatus` (estado del ledger), `attemptStatus` (resultado de esta Execution), `attemptNumber`, `providerRequestMade`, `attemptedAt`, `completedAt` y `safeErrorCode`.

`campaignRecipientStatus` y `attemptStatus` son campos distintos: el primero describe la proyección del ledger único Campaign + Purchase; el segundo describe el resultado persistido en la Execution consultada. `SKIPPED_DUPLICATE` solo puede aparecer como `attemptStatus` para documentar una detección local sin crear otro ledger ni implicar una solicitud a Emblue; no es un estado del ledger. Si existen múltiples attempts del mismo Recipient dentro de la misma Execution, se devuelven como filas separadas con su secuencia; este contrato no determina cuándo se autorizan retries ni su política.

Por defecto no devuelve email, `purchaseId`, `transactionId`, token ni payloads. La consulta requiere permiso específico de lectura de recipients; **OPEN:** campos enmascarados adicionales, rol y audit log requerido. Errores incluyen 404 por Campaign/Execution fuera del scope. Paginación obligatoria para controlar exposición y volumen.

### 8.4 Exclusions

No se define endpoint `exclusions` para v1. La exclusión de compras ya procesadas se expresa mediante el ledger `CampaignRecipient`; la detección de duplicado por ejecución se refleja en `duplicateCount` y el outcome `SKIPPED_DUPLICATE` de `CampaignRecipientAttempt` (o equivalente), no como estado del ledger. No se aprobó lista de exclusión manual ni supresión/consentimiento como recurso Campaign. Si se acuerda una política de bajas, definir contrato separado en otro cambio; no inferirla de deduplicación.

## 9. Consulta de Surveys elegibles

Reutilizar las capacidades existentes, sin duplicar SurveyJS ni crear en este documento una ruta de integración externa:

- `GET /surveys?page=1&perPage=10&enterpriseId={id}&searchText={text}&status=1&includeArchived=false` es el listado existente y paginado. Su respuesta incluye `{ data, pagination }` y puede agregar filtros/metadatos específicos.
- `GET /surveys/:id` es el detalle existente.
- La UI presenta encuestas disponibles en el scope autorizado. `status=1` y `includeArchived=false` reflejan filtros existentes; **no equivalen por sí solos a `PUBLISHED`**.
- Al programar, el API debe validar en servidor la regla funcional `PUBLISHED`, además de existencia, archivado y disponibilidad que correspondan.
- **OPEN:** definición técnica de `PUBLISHED` y campos mínimos/response de Survey elegible. No cambiar el significado de `status` existente por conveniencia del nuevo módulo.

## 10. Validaciones por operación

| Código propuesto | Condición | Resultado HTTP esperado |
|---|---|---:|
| `VALIDATION_ERROR` | Tipo/formato inválido, campo requerido, `postPurchaseOffset < 0`, array inválido o query no parseable. | 400 |
| `CAMPAIGN_NOT_FOUND` | ID inexistente o no visible en el scope. | 404 |
| `SURVEY_NOT_FOUND` | Survey inexistente o no accesible para el usuario. | 404 |
| `SURVEY_NOT_PUBLISHED` | Survey existente pero no cumple `PUBLISHED` al intentar schedule. | 400 propuesta; texto exacto depende de mapeo. |
| `INVALID_CAMPAIGN_STATE` | Acción no permitida para estado actual; incluye editar `ACTIVE` sin suspender. | 409 |
| `CAMPAIGN_DELETE_NOT_ALLOWED` | No es `DRAFT` o tiene historial. | 409 |
| `INVALID_SEGMENTATION` | Falta Division/Department/Company al programar, IDs externos no disponibles o jerarquía incompatible. | 400 |
| `INVALID_SCHEDULE` | Campos incompatibles con `ONE_TIME`/`FREQUENCY`, frecuencia no disponible para rango o fechas inválidas. | 400 |
| `AUDIENCE_SOURCE_UNAVAILABLE` | Fuente de catálogos/audiencia no disponible o falla al calcular preview. | 503 propuesta, alineada a fallos de dependencia; detalle no sensible. |
| `FORBIDDEN` | Usuario autenticado sin acción/scope requerido. | 403 |
| `UNAUTHORIZED` | Token/contexto de usuario PEC inválido. | 401 |
| `CAMPAIGN_CONFLICT` | Cambio concurrente, request duplicado incompatible o estado no permite transición. | 409 |

Envoltura de error conforme a `GlobalExceptionFilter` actual:

```json
{
  "statusCode": 409,
  "timestamp": "2026-10-18T16:00:00.000Z",
  "path": "/survey-campaigns/123/suspend",
  "method": "PATCH",
  "message": "La acción no es válida para el estado actual de la campaña.",
  "error": "Conflict",
  "code": "INVALID_CAMPAIGN_STATE"
}
```

`code` puede añadirse cuando el nuevo módulo lance una excepción estructurada compatible; el filtro actual lo conserva si viene en la respuesta. No devolver exception stack, SQL, credenciales, email ni payload de proveedor en respuestas de producción. Errores de DTO pueden traer `message` como lista, conforme a NestJS.

## 11. Idempotencia y reintentos HTTP

- `Audience Preview` es lectura lógica y no reserva recipients.
- `POST /survey-campaigns` puede duplicar Campaign si el cliente repite una petición tras timeout. Se recomienda soportar `Idempotency-Key` para crear borrador; persistencia, expiración y respuesta repetida son **OPEN**, no una convención existente.
- `PATCH` de configuración debe ser repetible con el mismo payload; si se aprueba control optimista por `version`/ETag, responder `409` ante versión obsoleta.
- Comandos `schedule`, `suspend`, `resume`, `cancel` deben ser seguros ante doble click/reintento: no crear dos executions ni adelantar dos veces la frecuencia. Si Campaign ya está en el estado objetivo, devolver el recurso actual sin efectos repetidos o un `409` consistente; elegir política en implementación/API review (**OPEN**).
- Idempotencia de envío se garantiza internamente por Campaign + purchase y se complementa con la clave Emblue solo si el proveedor la soporta. Angular no crea ni proporciona la clave de deduplicación del destinatario.
- La idempotencia de eventos Emblue/DataLab no forma parte de endpoints Angular; queda en el adapter y contrato externo.

## 12. Paginación, filtros y ordenamiento

- Para todas las colecciones (`Campaign`, catálogo y `Execution`/`Recipient`): `page` default 1, `perPage` default 10, respuesta `{ data, pagination: { page, perPage, total, totalPages } }`.
- Validar `page >= 1`, `perPage >= 1`; definir máximo fijo/configurable antes de implementar (**OPEN**, las pipes actuales no muestran límite superior).
- Listas deben filtrar antes de paginar y retornar `total` del mismo filtro.
- `sortBy` solo acepta allowlist documentada por endpoint; `sortOrder` acepta `asc` o `desc`; agregar desempate por ID para paginación estable.
- No aceptar nombres Prisma ni fragmentos SQL en `sortBy`, filtros o campos de búsqueda.
- `GET /survey-campaigns` filtra por `searchText` (Campaign Name), `surveySearchText` (Survey Name), `status`, `sendType`, `frequency`; cuando se aplican ambas búsquedas se combinan con AND. `status` admite todos los estados y `CANCELLED` se consulta con `status=CANCELLED`.
- Las dependencias de catálogo usan filtros de padres y paginación. La semántica de filtros vacíos conserva reglas FOUND: opcionales vacíos no restringen.

## 13. Resumen de operaciones

| Función | Método y ruta |
|---|---|
| Listar Campaigns | `GET /survey-campaigns` |
| Dashboard counts | `GET /survey-campaigns/dashboard/summary` |
| Crear Campaign | `POST /survey-campaigns` |
| Ver Campaign | `GET /survey-campaigns/:campaignId` |
| Editar Campaign | `PATCH /survey-campaigns/:campaignId` |
| Schedule | `PATCH /survey-campaigns/:campaignId/schedule` |
| Suspend | `PATCH /survey-campaigns/:campaignId/suspend` |
| Resume | `PATCH /survey-campaigns/:campaignId/resume` |
| Cancel | `PATCH /survey-campaigns/:campaignId/cancel` |
| Delete DRAFT elegible | `DELETE /survey-campaigns/:campaignId` |
| Preview de audiencia | `POST /survey-campaigns/audience/preview` |
| Catálogos internos | `GET /survey-campaigns/segmentation/{divisions,departments,auxiliary-classifications,companies,formats,stores}` |
| Listar executions | `GET /survey-campaigns/:campaignId/executions` |
| Ver execution | `GET /survey-campaigns/:campaignId/executions/:executionId` |
| Listar recipients | `GET /survey-campaigns/:campaignId/executions/:executionId/recipients` |
| Consultar Surveys existentes | `GET /surveys`, `GET /surveys/:id` (rutas actuales) |
| Exclusions | Sin endpoint v1; usar deduplicación ledger de recipients. |

En `GET /survey-campaigns`, `searchText` busca Campaign Name y `surveySearchText` busca Survey Name; si ambos se envían se combinan con AND. `status` admite todos los valores de CampaignStatus, incluido `CANCELLED`.

## 14. OPENs de contrato

- **OPEN-API-01:** Mapear `PUBLISHED` al modelo y servicio Survey real; el contrato existente usa `status` entero y `archived`, no enum publicado verificado.
- **OPEN-API-02:** Confirmar si Campaign tiene scope `enterpriseId`, cómo se relaciona con usuario autenticado y Survey, y cómo se aplica CASL.
- **OPEN-API-03:** Aprobar roles/abilities y permiso separado para visualizar recipients.
- **OPEN-API-04:** Compleción mínima permitida de `DRAFT` y obligatoriedad/identificador de `emailTemplateExternalId`.
- **OPEN-API-05:** Fuente de catálogos/audiencia, dependencias de padres y representación/serialización de arrays de IDs externos.
- **OPEN-API-06:** Límites de `perPage` y `sampleSize`, allowlists finales de ordenamiento y normalización/case-sensitivity de las búsquedas; los campos, combinación AND y significado de `total` quedan fijados en este contrato.
- **OPEN-API-07:** Zona horaria, interpretación de fecha/hora, duración exacta para catálogo dinámico y comportamiento de End Date.
- **OPEN-API-08:** Política idempotente de comandos ante estado ya objetivo; `Idempotency-Key` y control por `version`/ETag.
- **OPEN-API-09:** Campos enmascarados autorizados de Recipient, registro/auditoría de consultas y tratamiento de errores del proveedor.
- **OPEN-API-10:** Soft delete para DRAFT sin historial; DATA-001 marca `deletedAt` como propuesta, no patrón PEC global verificado.
- **OPEN-API-11:** Comportamiento de suspensión/cancelación con Execution en curso y reanudación tras tiempo vencido.
- **OPEN-API-12:** Códigos de error estables y mapping status HTTP para Survey no publicado y errores de fuente corporativa.

## 15. Criterios de aceptación del contrato

- Las rutas administrativas requieren autenticación y permisos backend; ninguna confía en controles de UI para autorización.
- Los endpoints implementan los estados/transiciones de FOUND-001 y no permiten editar Campaign `ACTIVE` hasta suspenderla.
- Schedule valida `Survey PUBLISHED`; crear/guardar `DRAFT` no rechaza una Survey existente únicamente por no estar publicada.
- El contrato usa `page`/`perPage`, paginación en forma PEC y ordenamientos por allowlist.
- `GET /survey-campaigns` permite buscar por Campaign Name (`searchText`) y Survey Name (`surveySearchText`); cuando ambos están presentes se combinan con AND, junto con `status`. `status` acepta todos los estados, incluido `CANCELLED`.
- Dashboard `total` cuenta Campaigns visibles y no eliminadas lógicamente en el scope autorizado, incluyendo `CANCELLED`; se definen seis tarjetas (`Total`, `Active`, `Scheduled`, `Draft`, `Suspended`, `Completed`) y `CANCELLED` queda en `byStatus` y como filtro, sin tarjeta propia.
- Preview calcula del lado del API, excluye compras procesadas al editar Campaign y no expone audiencia completa.
- Preview requiere `evaluationDate`, la distingue de `calculatedAt`, aplica la regla de fecha de compra aprobada sujeta a zona/corte OPEN, y no persiste recipients ni modifica estado.
- La consulta de recipients por Execution expone outcomes persistidos de `CampaignRecipientAttempt` o equivalente, distingue estado de ledger y estado del outcome, y permite registrar duplicados sin inferir llamada a Emblue ni fijar política de retries.
- Responses de recipients y ejecuciones no exponen PII, purchase IDs ni correlation tokens por defecto.
- No existe endpoint Campaign para borrar historial ni se elimina Campaign con historial.
- Los estados y nombres de enums coinciden con FOUND/DATA; filtros de segmentación conservan multiselect y dependencias, sin Auxiliary Subclassification v1.
- Las respuestas de error conservan la envoltura del filtro global, con códigos funcionales estables cuando se aprueben.
- Encuestas se consultan con capacidades existentes de `/surveys`; el contrato no inventa que `status=1` signifique `PUBLISHED`.
- No se definen rutas de CF/Emblue, controllers, código, migraciones ni schema Prisma.

## 16. Estado del documento

- **Status:** DRAFT
- **Version:** 0.2
- **Fuentes revisadas:** ARCH-001, FOUND-001, DATA-001, controllers/DTOs/guards/pipes y filtro global de `pec-api`.
- **Pendiente:** resolver OPENs de permisos, Survey publication, CF/Analytics, calendario, datos de recipients y contratos de integración antes de implementar.
- **Artefactos deliberadamente no generados:** controllers, DTOs TypeScript, código, migraciones y rutas de proveedores externos.

## 17. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Se añade búsqueda por Survey Name y su combinación con Campaign Name; se define el total del dashboard; se incorpora `evaluationDate` al Preview; se especifica la respuesta de outcomes por Execution distinguiendo ledger, attempts y duplicados. |
