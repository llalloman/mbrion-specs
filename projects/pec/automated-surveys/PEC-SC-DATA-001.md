# Modelo de Datos - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-DATA-001 |
| Estado | DRAFT |
| Versión | 0.5 |
| Fecha | 2026-10-20 |
| Idioma | Español; entidades, campos, enums y nombres técnicos en inglés |
| Documentos base | `PEC-SC-ARCH-001.md`, `PEC-SC-FOUND-001.md` |

> Este documento propone un **modelo lógico**, no un schema Prisma definitivo. Los tipos expresan intención conceptual; el mapeo físico y las restricciones finales requieren resolver los puntos **OPEN**. No se modifica el modelo existente de Survey en esta propuesta.

## 1. Objetivo y principios

Definir los datos que permiten configurar campañas, conservar selecciones multiselect, ejecutar calendarios, registrar resultados por compra/destinatario y atribuir respuestas con seguridad.

Principios:

- Las etiquetas españolas son de presentación; no se almacenan en lugar de enums, campos o IDs técnicos. La normalización de idioma no modifica modelos, enums, columnas ni contratos Prisma/PostgreSQL.

- Reutilizar `sspectsurvey` y `sspectresult`; no duplicar encuestas ni respuestas.
- Mantener configuración editable de `Campaign` separada del historial inmutable de `CampaignExecution`.
- Deduplicar por Campaign + identificador de compra, no por cliente.
- Mantener `CampaignRecipient` como ledger único por Campaign + Purchase y conservar un resultado persistente por Execution y Recipient mediante `CampaignRecipientAttempt` o mapeo físico equivalente obligatorio.
- No crear catálogos locales duplicados para jerarquías que pertenecen a CF/Analytics; persistir IDs externos seleccionados y snapshots descriptivos mínimos.
- No exponer PII en URL ni guardar datos personales completos sin necesidad operacional justificada.
- Usar identificadores y convenciones compatibles con PostgreSQL/Prisma y con las relaciones existentes PEC.
- Reservar decisiones dependientes de contratos no entregados como **OPEN**.

## 2. Convenciones verificadas en Prisma

Inspección de `pec-api/prisma/schema.prisma`:

| Convención encontrada | Evidencia / implicación para la propuesta |
|---|---|
| Provider PostgreSQL y Prisma Client | `datasource db` usa `postgresql`; los campos conceptuales se expresan como tipos Prisma habituales. |
| PK predominante | Modelos de negocio usan `Int @id @default(autoincrement())`, por ejemplo `sspectsurvey.sspecrpksur`, `sspectuser.sspecrpkuse` y `sspectresult.sspecrpkres`. `CaseLock` usa `BigInt`; relaciones compuestas también existen. |
| UUID | No se identificó uso de `@default(uuid())` en los modelos inspeccionados. No se propone cambiar a UUID sin una necesidad probada. |
| Timestamps | Convención frecuente: `created_date DateTime @default(now())`, `last_modified_date DateTime @updatedAt`; algunos modelos tienen variantes. |
| Auditoría | Muchos modelos incluyen `created_by_user`, `last_modified_by_user`, `created_from_ip`, `updated_from_ip` y `version`. Es una convención existente, con diferencias de nullability entre modelos. |
| Estados/soft delete | Es común `status Int @default(1)`; Survey además tiene `archived Boolean @default(false)`. No se observó una convención global de `deletedAt`. |
| Enums | Hay enums Prisma para dominios acotados, como `activityStatus`, `documentType`, `LockType` y `FileType`; otros estados continúan como enteros. |
| Relaciones | Relaciones usan scalar FK + `@relation(fields: ..., references: ...)`; las reglas `onDelete` varían (Cascade, SetDefault o sin regla explícita). |
| Índices/uniqueness | Se usan `@@index`, `@@unique` y PK compuestas; a veces hay nombres físicos con `map`. |
| Nombres | Conviven modelos y campos legacy en snake_case y nuevos modelos como `CaseLock`, `EnterpriseTopicRelation`, `dashboard_config`; se usan `@@map`/`@map` cuando es necesario. |
| Survey existente | `sspectsurvey` tiene PK entera, `content Json`, `enterpriseId`, `status Int`, `archived Boolean` y auditoría. Relaciona `sspectresult[]`. |
| Survey result existente | `sspectresult` tiene PK entera, FK `surveyId`, respuestas en `content Json`, timestamps y relaciones opcionales de establecimiento. Tiene índices por encuesta/establecimiento/fecha. |
| Usuario existente | `sspectuser` tiene PK entera `sspecrpkuse`, `username`, `email`, `name` y auditoría. `UserEnterpriseRole` muestra relaciones por FK y PK compuesta. |

Los nombres anteriores son los verificados en el checkout actual. No se infiere de ellos el contrato de datos externo ni que un catálogo PEC equivalga a un catálogo corporativo CF.

## 3. Modelo conceptual / ER textual

```text
sspectuser (sspecrpkuse) 1 ───── N Campaign (createdByUserId)
sspectsurvey (sspecrpksur) 1 ── N Campaign (surveyId)
sspectenterprise (sspecrpkent) — no tiene relación Campaign establecida por la decisión multiempresa aprobada

Campaign 1 ── N CampaignDivision
Campaign 1 ── N CampaignDepartment
Campaign 1 ── N CampaignAuxiliaryClassification
Campaign 1 ── N CampaignCompany
Campaign 1 ── N CampaignFormat
Campaign 1 ── N CampaignStore

Campaign 1 ── N CampaignExecution
CampaignExecution 1 ── N CampaignRecipientAttempt (resultado por execution-recipient)
Campaign 1 ── N CampaignRecipient (ledger único Campaign + Purchase)
CampaignRecipient 1 ── N CampaignRecipientAttempt
CampaignExecution 0..1 ── N CampaignExecution (recoveryOfExecutionId)

CampaignRecipientAttempt 1 ── N ResponseCorrelation (invitación originada por ese intento; cardinalidad OPEN)
ResponseCorrelation N ── 0..1 sspectresult (respuesta PEC existente)

CampaignExecution 1 ── 0..N CampaignProviderEvent (si se requiere inbox de eventos)
```

Las relaciones externas con catálogos CF/Analytics se representan con identificadores externos, no con FKs a maestros locales hasta que se confirme identidad y autoridad de cada catálogo.

## 4. Entidades

### 4.1 `Campaign`

**Propósito:** configuración vigente y editable de una campaña. Conserva una referencia a Survey, filtros y calendario. No debe ser la fuente del historial de ejecuciones.

| Campo conceptual | Tipo conceptual | Null | Uso / sensibilidad |
|---|---|---:|---|
| `id` | Entero autoincremental | No | PK; convención predominante PEC. |
| `surveyId` | Entero FK a `sspectsurvey.sspecrpksur` | No | Survey PEC existente; no almacenar una segunda definición de Survey. |
| `createdByUserId` | Entero FK a `sspectuser.sspecrpkuse` | No propuesto | Relación navegable con el usuario que crea Campaign. |
| `name` | String | No | Nombre funcional de campaña. |
| `description` | String | Sí | Descripción. |
| `status` | `CampaignStatus` | No | Estado de lifecycle; separado de `status Int` legacy. |
| `sendType` | `CampaignSendType` | No | `ONE_TIME` o `FREQUENCY`. |
| `frequency` | `CampaignFrequency` | Sí | Requerido solo para `FREQUENCY`; nulo para `ONE_TIME`. |
| `postPurchaseOffset` | Entero | No | Días, >= 0. Requiere validación de negocio y, si el motor lo permite, constraint de rango. |
| `sendDate` | Fecha | Sí | Obligatorio solo para `ONE_TIME`. |
| `startDate` | Fecha | Sí | Obligatorio solo para `FREQUENCY`. |
| `endDate` | Fecha | Sí | Obligatorio solo para `FREQUENCY`. |
| `sendTime` | Hora local | Sí | Obligatorio para ambos tipos. Tipo físico `TIME`/equivalente y zona quedan OPEN. |
| `timeZone` | String | Sí | Nullable mientras DRAFT no se programa; debe persistirse explícitamente antes de Schedule/Resume efectivos. Para Campaigns de Ecuador, default aprobado `America/Guayaquil`; no se infiere del navegador. Otras zonas, DST y conversión a instantes siguen sujetos a OPEN. |
| `nextExecutionAt` | Timestamp | Sí | Próximo instante absoluto calculado; nulo si no está programada o ya no tiene siguiente ciclo. |
| `lastExecutionAt` | Timestamp | Sí | Instante de última ejecución concluida o procesada, semántica exacta OPEN. |
| `emailTemplateExternalId` | String | Sí | Identificador externo de plantilla Emblue, cuando el contrato lo confirme. |
| `emailTemplateNameSnapshot` | String | Sí | Etiqueta visible opcional para que un cambio de nombre externo no altere la vista histórica/configurada. |
| `deletedAt` | Timestamp | Sí | Propuesta de soft delete para Delete funcional permitido en DRAFT sin historial; patrón no verificado como convención global PEC. |
| `created_by_user` | String | No recomendado conforme a patrón | Auditoría textual PEC; redundante con `createdByUserId`, se conserva solo si es requisito de auditoría común. |
| `created_date` | Timestamp | No | Auditoría, default `now()`. |
| `last_modified_by_user` | String | No recomendado conforme a patrón | Actor de la última edición, sujeto a convención común del API. |
| `last_modified_date` | Timestamp | No | Auditoría, actualizado en cambios de Campaign. |
| `version` | Entero | No | Control/auditoría optimista si PEC lo aplica al nuevo módulo. |
| `created_from_ip` / `updated_from_ip` | String | Pendiente | Incluir solo si la política de auditoría PEC los requiere; no guardar IP si no aplica. |

**Relaciones:** Survey requerida; `sspectuser` como creador; sin FK Campaign a `sspectenterprise` bajo la decisión multiempresa. Tiene colecciones de selecciones, ejecuciones y recipients.

**Índices/constraints recomendados:** PK `id`; índice por `(status, nextExecutionAt)` para dashboard/scheduler; `(surveyId, status)`; `(createdByUserId, created_date)` si se usa filtro de creador. `postPurchaseOffset >= 0`. Validación mutuamente excluyente: `ONE_TIME` requiere `sendDate`, no requiere `startDate/endDate/frequency`; `FREQUENCY` requiere `startDate/endDate/frequency`, no requiere `sendDate`; ambos requieren `sendTime`. Constraint de nombre único no está aprobado.

La duración de `FREQUENCY` para validar su frecuencia permitida se deriva inclusivamente de las fechas `startDate` y `endDate` como valores de calendario (`endDate - startDate + 1` días calendario), sin alterar campos persistidos ni introducir timestamps. `EndDate` también es inclusiva como fecha local de negocio: una ocurrencia en esa fecha a `SendTime` es válida y no se generan ocurrencias posteriores. Para `MONTHLY` y `QUARTERLY`, cada fecha local se calcula desde `StartDate` y el índice; conserva el día original, o usa el último día disponible del mes destino, sin desplazar el ancla posterior. Esto incluye años bisiestos y ocurre antes de convertir a UTC. `DRAFT` puede conservar configuración incompleta o pendiente de corrección; la validación completa corresponde a la acción efectiva de programación.

**Uniqueness:** ninguna unicidad natural aprobada. `id` es PK. No asumir unicidad global por nombre, Survey o propietario.

**Lifecycle:** `DRAFT` editable; `SCHEDULED` editable según la especificación funcional; `ACTIVE` debe suspenderse antes de editar; `SUSPENDED` editable/reanudable; `COMPLETED/CANCELLED` solo lectura. Delete lógico solo según condiciones funcionales y permisos.

**Históricos:** cambios de filtros/calendario de Campaign no alteran los snapshots de `CampaignExecution` ya iniciados/finalizados.

**Sensibilidad:** descripción y configuración en general no son PII; `createdByUserId` y campos de auditoría identifican personal interno.

### 4.2 Entidades multiselect de Campaign

Los seis modelos conservan solo las selecciones efectuadas para una Campaign. No replican atributos ni jerarquías completas de CF/Analytics. Se propone usar un identificador externo canónico serializado como texto hasta conocer tipo, longitud y formato reales. `externalSystem` identifica el catálogo de origen si hay más de una fuente.

| Entidad | Campo de FK | Identificador externo | Regla funcional |
|---|---|---|---|
| `CampaignDivision` | `campaignId` | `divisionExternalId` | Requerida: al menos una selección para programar. |
| `CampaignDepartment` | `campaignId` | `departmentExternalId` | Requerida: al menos una selección para programar. |
| `CampaignAuxiliaryClassification` | `campaignId` | `classificationExternalId` | Opcional; cero filas no restringen audiencia. |
| `CampaignCompany` | `campaignId` | `companyExternalId` | Requerida: al menos una selección para programar. |
| `CampaignFormat` | `campaignId` | `formatExternalId` | Opcional; cero filas no restringen audiencia. |
| `CampaignStore` | `campaignId` | `storeExternalId` | Opcional; cero filas no restringen audiencia. |

Campos conceptuales aplicables a cada una:

| Campo | Tipo | Null | Notas |
|---|---|---:|---|
| `campaignId` | Entero FK a Campaign | No | Parte de PK/unique; `onDelete: Cascade` solo si borrado lógico no aplica y se valida ciclo de vida. |
| `<kind>ExternalId` | String canónico | No | ID externo de CF/Analytics. Tipo real OPEN. |
| `externalSystem` | String/código | No propuesto | Fuente del ID; no fijar nombre ni catálogo hasta confirmar integración. |
| `labelSnapshot` | String | Sí | Nombre mostrado al configurar; permite mostrar selección aun si el catálogo externo cambia. |
| `parentExternalId` | String | Sí | Solo donde ayude a conservar relación de jerarquía elegida; no es sustituto del catálogo autoritativo. |

**PK y uniqueness:** PK compuesta conceptual `(campaignId, externalSystem, <kind>ExternalId)`; impide duplicar la misma selección. Si Prisma/longitud real no admite PK compuesta conveniente, evaluar surrogate `Int` + `@@unique` equivalente; decisión física OPEN.

**Índices:** `(campaignId)` cubierto por PK compuesta; índice por `(externalSystem, <kind>ExternalId)` solo si se necesita búsqueda inversa o reportes. No indexar todos los nombres de snapshot por defecto.

**Lifecycle:** filas se agregan/remueven al editar Campaign en estados permitidos. No son historial autoritativo: el snapshot de cada Execution conserva la selección efectiva de esa ejecución.

**Sensibilidad:** IDs externos y etiquetas corporativas pueden ser datos internos. No contienen correo por diseño. Clasificación de confidencialidad OPEN con el propietario de fuente.

#### `CampaignDivision`

**Propósito:** registrar cada `Division` seleccionada. **Campos:** `campaignId` (entero, requerido), `externalSystem` (código/string, requerido), `divisionExternalId` (string, requerido), `labelSnapshot` (string, nullable). **Relación:** N:1 Campaign. **Índices/constraints:** unicidad Campaign + sistema + division ID; no FK a un maestro local. **Lifecycle/histórico:** editable en Campaign; se copia a `CampaignExecution.configurationSnapshot`. **Datos sensibles:** no PII; nombres e IDs son corporativos.

#### `CampaignDepartment`

**Propósito:** registrar cada `Department` seleccionada. **Campos:** `campaignId`, `externalSystem`, `departmentExternalId`, `labelSnapshot` y `parentExternalId` (el último nullable, snapshot opcional de Division padre). **Relación:** N:1 Campaign. **Índices/constraints:** unicidad Campaign + sistema + department ID; validar compatibilidad con Division en el catálogo/servicio autoritativo. **Lifecycle/histórico:** editable según estado y snapshot por Execution. **Datos sensibles:** IDs/etiquetas corporativos.

#### `CampaignAuxiliaryClassification`

**Propósito:** selecciones opcionales de `Auxiliary Classification`. **Campos:** `campaignId`, `externalSystem`, `classificationExternalId`, `labelSnapshot` nullable, `parentExternalId` nullable. **Relación:** N:1 Campaign. **Índices/constraints:** unicidad Campaign + sistema + classification ID. **Lifecycle/histórico:** cero o más; snapshot por Execution. **Datos sensibles:** IDs/etiquetas corporativos. No modela Subclassification.

#### `CampaignCompany`

**Propósito:** registrar cada `Company` comercial seleccionada. **Campos:** `campaignId`, `externalSystem`, `companyExternalId`, `labelSnapshot` nullable. **Relación:** N:1 Campaign. **Índices/constraints:** unicidad Campaign + sistema + company ID. No relacionar automáticamente con `sspectenterprise`: en PEC ese modelo representa Enterprise/tenant y no se verificó equivalencia con Company de CF. **Lifecycle/histórico:** selección editable en Campaign y congelada en cada snapshot. **Datos sensibles:** corporativos.

#### `CampaignFormat`

**Propósito:** selecciones opcionales de `Format` comercial. **Campos:** `campaignId`, `externalSystem`, `formatExternalId`, `labelSnapshot` nullable, `parentExternalId` nullable. **Relación:** N:1 Campaign. **Índices/constraints:** unicidad Campaign + sistema + format ID. No relacionar automáticamente con `sspectformat`: identidad entre formato PEC y CF/Analytics OPEN. **Lifecycle/histórico:** editable/snapshot. **Datos sensibles:** corporativos.

#### `CampaignStore`

**Propósito:** selecciones opcionales de `Store`. **Campos:** `campaignId`, `externalSystem`, `storeExternalId`, `labelSnapshot` nullable, `parentExternalId` nullable. **Relación:** N:1 Campaign. **Índices/constraints:** unicidad Campaign + sistema + store ID. No relacionar automáticamente con `sspectestablishment`: equivalencia entre Store externo y establecimiento PEC OPEN. **Lifecycle/histórico:** editable/snapshot. **Datos sensibles:** corporativos.

**Regla común de filtros:** selección dentro de un filtro se evalúa con OR; grupos distintos, con AND. `Auxiliary Subclassification` no tiene entidad ni campo en v1. Un cambio de padre que invalide hijos se refleja removiendo las selecciones incompatibles de las filas corrientes; las ejecuciones históricas retienen su snapshot.

#### Referencias sintéticas MOCK

Las filas de selección continúan guardando arrays de IDs externos y sus snapshots existentes; la decisión MOCK no requiere cambio de modelo ni migración. Cuando el proveedor sea MOCK, IDs `MOCK-*` y etiquetas `[MOCK]` son valores sintéticos aislados. `parentExternalId` solo conserva la relación de demostración servida por el mismo proveedor, no una jerarquía real confirmada. Al sustituir el proveedor, backend debe validar el namespace y pertenencia con Analytics; no reinterpretar automáticamente IDs MOCK.

### 4.3 `CampaignExecution`

**Propósito:** historial de cada ejecución programada/recuperada y su configuración efectiva. Es inmutable una vez que entra en estado final; nunca debe representar la Campaign mutable actual como si fuera su configuración histórica.

| Campo conceptual | Tipo | Null | Uso / sensibilidad |
|---|---|---:|---|
| `id` | Entero autoincremental | No | PK siguiendo modelo PEC predominante. |
| `campaignId` | Entero FK Campaign | No | Campaña ejecutada. |
| `executionKey` | String | No | Clave idempotente lógica por Campaign + slot/recuperación; algoritmo OPEN. |
| `status` | `CampaignExecutionStatus` | No | Estado funcional. |
| `scheduledAt` | Timestamp | No | Instante programado que da origen a la ejecución. |
| `startedAt` | Timestamp | Sí | Inicio real. |
| `finishedAt` | Timestamp | Sí | Fin real. |
| `recovery` | Boolean | No propuesto, default false | Indica una ejecución que consolida trabajo pendiente tras downtime/fallo; no representa una ejecución por cada frecuencia perdida. Semántica de transición OPEN. |
| `recoveryOfExecutionId` | Entero FK CampaignExecution | Sí | Referencia a la ejecución original que se recupera; nulo para ejecución ordinaria. Una recuperación puede consolidar trabajo omitido y no crea una ejecución por cada frecuencia perdida. Política de enlace y granularidad OPEN. |
| `configurationSnapshot` | JSON | No | Snapshot efectivo, inmutable: IDs y etiquetas seleccionadas, offset, SendType/frecuencia/fechas/hora/zona, Survey ID + versión observada, plantilla/referencia, y parámetros de ejecución. No incluir lista completa de audiencia ni email. |
| `surveyVersionSnapshot` | Entero | Sí | Valor de versión de `sspectsurvey` al inicio, si esa versión cambia con ediciones. |
| `surveyContentHash` | String | Sí | Hash para identificar contenido observado, si se aprueba; no sustituye al contenido ni garantiza reproducibilidad. |
| `eligiblePurchases` | Entero | No | Conteo de compras candidatas de esta ejecución, default 0. |
| `eligibleCustomers` | Entero | No | Conteo de clientes distintos, default 0. |
| `sentCount` | Entero | No | Conteo según semántica de `SENT` que confirme Emblue, default 0. |
| `failedCount` | Entero | No | Conteo de fallos, default 0. |
| `duplicateCount` | Entero | No | Conteo de candidatos duplicados, default 0. |
| `errorSummary` | String | Sí | Error general sanitizado; no payload ni PII completa. |
| `providerExecutionRef` | String | Sí | El envío individual emBlue v2.3 devuelve `activityId` como referencia de aceptación. Su mapeo/persistencia por Execution o Attempt y cualquier referencia por lote siguen OPEN; no implica entrega. |
| `claimOwner` | String | Sí | Identificador técnico de la réplica/worker que obtuvo claim; sin secreto. |
| `claimedAt` | Timestamp | Sí | Inicio de lease/claim. |
| `claimExpiresAt` | Timestamp | Sí | Vencimiento para recuperar claim abandonado; duración OPEN. |
| `created_date` | Timestamp | No | Timestamp de creación del historial. |

**Relaciones:** N:1 Campaign; opcional N:1 `recoveryOfExecutionId` hacia la ejecución original; 1:N `CampaignRecipientAttempt`; potencialmente 1:N `CampaignProviderEvent`. Cada Attempt apunta a la ejecución que produjo el resultado o la detección de duplicado.

**Índices/constraints:** PK; unique `(campaignId, executionKey)`; índices `(status, scheduledAt)`, `(campaignId, scheduledAt)`, `(status, claimExpiresAt)` para scheduler, detalle e inspección de ejecuciones atascadas. Counts no negativos. `finishedAt` requerido en estados terminales (constraint/validación a definir). No mantener lock transaccional mientras se llama a Emblue.

**Lifecycle:** `PENDING` → `PROCESSING` → `COMPLETED`, `COMPLETED_WITH_ERRORS` o `FAILED`. Estados terminales y snapshot inmutables: una recuperación posterior no reabre ni altera la Execution final. Un claim vencido mientras sigue `PROCESSING` puede recuperarse en la misma Execution; después de un estado terminal, el trabajo recuperado puede registrarse en una nueva Execution relacionada mediante `recoveryOfExecutionId`. La elección y transiciones precisas son OPEN, sin crear una ejecución por cada frecuencia omitida. Evitar ejecuciones duplicadas entre réplicas con unicidad de `executionKey` y claim compartido.

**Históricos:** snapshot JSON conserva la configuración efectiva en el momento de esa ejecución, aunque Campaign se edite luego. Los conteos y los timestamps reflejan el resultado real; correcciones administrativas no reescriben ejecuciones terminadas.

**Sensibilidad:** snapshot debe contener solo configuración; `errorSummary` y referencias externas se sanitizan. Claim owner puede identificar infraestructura interna.

### 4.4 `CampaignRecipient`

**Propósito:** ledger de una compra/destinatario por Campaign para incrementalidad y deduplicación. Una persona puede tener más de un registro en una Campaign si corresponde a compras distintas.

| Campo conceptual | Tipo | Null | Uso / sensibilidad |
|---|---|---:|---|
| `id` | Entero autoincremental | No | PK. |
| `campaignId` | Entero FK Campaign | No | Scope de deduplicación. |
| `firstExecutionId` | Entero FK CampaignExecution | No | Ejecución que registró inicialmente la compra elegible. |
| `purchaseSource` | String/código | No | Sistema autoritativo de la clave de compra. |
| `purchaseExternalId` | String | No | ID canónico de compra/transaction, tipo real OPEN. |
| `purchaseIdType` | String/enumeración conceptual | Sí | Distingue `purchaseId` de `transactionId` si la fuente admite ambas clases; catálogo OPEN. |
| `customerExternalId` | String | Sí | ID mínimo para contar clientes/deduplicar conteos; no usarlo como clave de envío salvo decisión. |
| `purchaseDateSnapshot` | Timestamp/fecha | Sí | Fecha observada para explicar elegibilidad; retención/precisión OPEN. |
| `eligibleAt` | Timestamp | Sí | Primer instante en que cumple offset, sujeto a timezone/corte. |
| `status` | `CampaignRecipientStatus` | No | Estado operacional/proyección del ledger según resultados reales; no se cambia al detectar un duplicado en otra Execution. La granularidad y mutación por retry son OPEN. `SKIPPED_DUPLICATE` se conserva como outcome de `CampaignRecipientAttempt`, no como estado que reescriba el Recipient original. |
| `reservedAt` | Timestamp | Sí | Momento de la reserva temporal para coordinar procesamiento concurrente; no significa que la compra ya fue procesada ni enviada. |
| `reservationOwner` | String | Sí | Identificador técnico no sensible del worker que mantiene la reserva, si el mecanismo lo requiere. |
| `reservationExpiresAt` | Timestamp | Sí | Vencimiento de reserva para recuperación tras abandono; duración/política OPEN. |
| `firstProcessedAt` | Timestamp | Sí | Primer momento en que se registra un resultado de procesamiento que, según la política acordada, cuenta como procesado. No se establece al reservar. Semántica para fallos/reintentos OPEN. |
| `lastAttemptAt` | Timestamp | Sí | Última solicitud real de envío intentada; no se actualiza por una detección `SKIPPED_DUPLICATE`. |
| `emailSnapshot` | String protegido/cifrado | Sí | Solo persistir si retry no puede recuperar destinatario desde fuente. PII de alto cuidado; cifrado/retención OPEN. Preferir no persistir si se puede operar sin él. |
| `recipientCorrelationId` | String opaco | Sí | Referencia interna sin PII, si se requiere unir eventos y response correlation. |
| `created_date` | Timestamp | No | Momento de creación del ledger. |

**Relaciones:** N:1 Campaign; N:1 primera CampaignExecution; 1:N Attempts; ResponseCorrelation se vincula a Attempt/invitación para obtener Campaign + Execution + Recipient. Cardinalidad de respuestas OPEN.

**Índices/constraints:** unique `(campaignId, purchaseSource, purchaseExternalId)` para idempotencia funcional; índices para estado y vencimiento de reserva; índice `(firstExecutionId)`; opcional índice sobre `customerExternalId` solo si soporta conteos/consultas autorizadas. Unicidad no incluye `customerExternalId`.

**Lifecycle:** una fila es la identidad/ledger única de una compra dentro de una Campaign. La reserva temporal (claim) es distinta del procesamiento confirmado y del resultado real de envío. Un claim abandonado vence y puede ser recuperado; no debe excluir permanentemente una compra pendiente. Resultado, retry y condición para considerar procesada una compra siguen OPEN según política Emblue/operativa. Un duplicado detectado en otra ejecución no crea otra fila Recipient: se registra como resultado `SKIPPED_DUPLICATE` asociado al Recipient existente y a la Execution que detectó el duplicado; ese resultado no constituye solicitud ni intento de envío a Emblue.

**Históricos:** identidad de compra y primera ejecución no se reescriben. Envíos, fallos y outcomes de duplicado por Execution permanecen en Attempts; detectar un duplicado no cambia `lastAttemptAt` ni el estado del Recipient original. La edición de Campaign no altera los datos base del recipient.

**Sensibilidad:** identificadores de cliente/compra son datos corporativos potencialmente personales; `emailSnapshot` es PII directa. Restringir lecturas, cifrar si se persiste y fijar retención antes de producción.

### 4.5 `CampaignRecipientAttempt` (trazabilidad obligatoria a nivel de dominio)

**Propósito:** conservar persistentemente por ejecución y recipient cada resultado individual, incluyendo intentos de envío, fallos, reintentos y duplicados detectados, sin reescribir el historial. Este concepto es obligatorio en el modelo de dominio para consultar los recipients y resultados de cada Execution. Su tabla/relaciones/campos físicos definitivos permanecen OPEN. El nombre `Attempt` no significa que cada fila haya generado una solicitud a Emblue: un duplicado puede registrarse como resultado de evaluación sin llamada externa.

| Campo conceptual | Tipo | Null | Notas |
|---|---|---:|---|
| `id` | Entero autoincremental | No | PK. |
| `campaignRecipientId` | Entero FK CampaignRecipient | No | Apunta al ledger único existente. Para duplicado apunta al Recipient original; no crea otro Recipient. |
| `executionId` | Entero FK CampaignExecution | No | Ejecución que produjo el intento/resultado. |
| `attemptNumber` | Entero | No | Secuencia dentro del recipient o execution, alcance exacto OPEN. |
| `status` | `CampaignRecipientStatus` | No | Outcome por ejecución-recipient: `PENDING`, `SENT`, `FAILED` o `SKIPPED_DUPLICATE`. `SKIPPED_DUPLICATE` registra detección/contabilización, no un envío ni una nueva llamada al proveedor. |
| `providerRequestMade` | Boolean | No propuesto | Distingue una solicitud de envío real de un outcome local como `SKIPPED_DUPLICATE`; contrato/mapeo físico OPEN. |
| `providerBatchExternalId` | String | Sí | External ID de lote si Emblue lo devuelve. |
| `providerMessageExternalId` | String | Sí | External ID por mensaje/destinatario si existe. |
| `idempotencyKey` | String | Sí | Clave usada con Emblue si el contrato la soporta; nunca asumir que existe. |
| `attemptedAt` | Timestamp | Sí | Momento del intento. |
| `completedAt` | Timestamp | Sí | Fin/resultado. |
| `errorCode` | String | Sí | Código del proveedor si es seguro y estable. |
| `errorSummary` | String | Sí | Resumen sanitizado. |

**Relaciones:** N:1 Execution y N:1 Recipient obligatorias. Una identidad `(executionId, campaignRecipientId)` puede representar el outcome de esa compra en esa ejecución; los envíos repetidos autorizados pueden requerir más de un Attempt bajo esa pareja. **Índices:** `(campaignRecipientId, attemptNumber)`, `(executionId, status)`, unicidad de secuencia según scope acordado; unique opcional por external message ID + provider. **Lifecycle:** un registro puede evolucionar de `PENDING` a su outcome; una vez asentado, ese outcome es inmutable. Cada retry autorizado crea un registro posterior y no sobrescribe resultados anteriores. **Sensibilidad:** no almacenar payload completo ni correo; IDs externos se tratan como internos. La existencia conceptual y persistencia por Execution/Recipient son obligatorias; solo el diseño físico final y la granularidad de filas son OPEN.

**Consistencia de conteos:** `CampaignExecution.duplicateCount` cuenta outcomes `SKIPPED_DUPLICATE` registrados para esa ejecución, no solicitudes ni filas nuevas de `CampaignRecipient`. `sentCount` y `failedCount` agregan únicamente outcomes de envío conforme a la semántica acordada. Una colisión de unicidad no crea un nuevo Recipient; el resultado de duplicado se vincula al Recipient existente y no se manda a Emblue por el solo hecho de contabilizarlo. No contar la misma ocurrencia duplicada más de una vez dentro de la Execution.

### 4.6 `ResponseCorrelation`

**Propósito:** correlacionar cada invitación con Campaign, Execution y Recipient y enlazarla, cuando haya respuesta, con `sspectresult` usando una referencia opaca. La correlación se vincula conceptualmente a `CampaignRecipientAttempt` (que referencia la Execution y el Recipient); vincular directamente a `CampaignExecution` es alternativa física si conserva esa misma triple trazabilidad. No incluir PII ni identificadores de compra en la URL.

| Campo conceptual | Tipo | Null | Uso / sensibilidad |
|---|---|---:|---|
| `id` | Entero autoincremental | No | PK. |
| `campaignRecipientAttemptId` | Entero FK CampaignRecipientAttempt | No | Intento de envío que originó la invitación; permite derivar Execution, Campaign y Recipient. Alternativa de relación directa a Execution más Recipient queda abierta al definir el modelo físico. |
| `surveyResultId` | Entero FK `sspectresult.sspecrpkres` | Sí | Nulo hasta que se registre respuesta; luego referencia al resultado existente. |
| `tokenHash` | String | No | Hash de referencia opaca, no token utilizable en claro; detalle de hashing OPEN. |
| `createdAt` | Timestamp | No | Emisión de referencia. |
| `expiresAt` | Timestamp | Sí | Fecha de vencimiento, política OPEN. |
| `consumedAt` | Timestamp | Sí | Si se define consumo único. |
| `created_date` | Timestamp | No | Auditoría según convención PEC. |

**Relaciones:** N:1 `CampaignRecipientAttempt`; Campaign, Execution y Recipient se obtienen de sus relaciones. Opcional N:1 `sspectresult`. El resultado mantiene Survey existente; no se crea otra entidad de respuesta ni se copia `content Json`.

**Índices/constraints:** `tokenHash` único; índice por `campaignRecipientAttemptId`. `surveyResultId` unique solo si se confirma que un resultado puede atribuirse a una única invitación. No imponer cardinalidad única por Recipient sin aprobación. Validar que Survey del `sspectresult` corresponde al Survey de Campaign; puede ser constraint lógica en servicio.

**Lifecycle:** asociado a la invitación transaccional emitida y vinculado a `sspectresult` cuando llega la respuesta. La invitación se puede correlacionar con Campaign/Execution/Recipient mediante el Attempt relacionado. Un duplicado detectado sin envío no produce invitación ni correlación. La referencia no debe poder revelar Campaign/Recipient/PII por inspección de URL.

**Datos sensibles:** token en claro no se persiste; hash y relaciones son sensibles por correlación. Acceso restringido y retención OPEN. No registrar URL completa/token.

**OPEN:** cardinalidad de respuestas, uno o varios enlaces por recipient, reutilización, caducidad, invalidación/reenvío, protección replay, validación Survey/tenant y cambios necesarios al flujo público de Survey.

### 4.7 `CampaignProviderEvent` (opcional, según contrato)

**Propósito:** inbox idempotente para eventos DataLab/Webhooks si el contrato ofrece un identificador estable y PEC necesita conservar el procesamiento.

| Campo conceptual | Tipo | Null | Notas |
|---|---|---:|---|
| `id` | Entero autoincremental | No | PK. |
| `provider` | String/código | No | Identifica Emblue/proveedor, sin asumir valores definitivos. |
| `providerEventExternalId` | String | Sí | ID externo; null solo si el contrato no lo proporciona. |
| `recipientAttemptId` | Entero FK CampaignRecipientAttempt | Sí | Correlación si se puede resolver. |
| `eventType` | String | No | Tipo del evento según contrato externo. |
| `receivedAt` | Timestamp | No | Recepción local. |
| `processedAt` | Timestamp | Sí | Fin de aplicación del evento. |
| `payloadHash` | String | Sí | Integridad/reconciliación sin guardar payload con PII. |
| `processingStatus` | String/enum por definir | No | Ciclo interno de ingesta, distinto de estados Recipient. |

**Índices/constraints:** unique `(provider, providerEventExternalId)` cuando el ID externo exista; índice `(processingStatus, receivedAt)`. **Lifecycle:** append-only o estado de procesamiento monotónico; no sobrescribir histórico. **Datos sensibles:** no conservar el payload original sin una política explícita. **OPEN:** contrato, firma, IDs, replay, tipos de evento y retención.

## 5. Enums conceptuales

Se propone usar enums Prisma para estados/catálogos funcionales nuevos, aunque varios modelos legacy PEC usan `status Int`. No se debe mapear un enum nuevo a los enteros legacy sin equivalencias explícitas.

| Enum | Valores | Uso |
|---|---|---|
| `CampaignStatus` | `DRAFT`, `SCHEDULED`, `ACTIVE`, `SUSPENDED`, `COMPLETED`, `CANCELLED` | Lifecycle de Campaign. Delete lógico no es estado funcional; `deletedAt`/mecanismo separado es OPEN. |
| `CampaignSendType` | `ONE_TIME`, `FREQUENCY` | Determina campos de calendario aplicables. |
| `CampaignFrequency` | `DAILY`, `WEEKLY`, `BIWEEKLY`, `MONTHLY`, `QUARTERLY` | Frecuencia solo cuando SendType es `FREQUENCY`. |
| `CampaignExecutionStatus` | `PENDING`, `PROCESSING`, `COMPLETED`, `COMPLETED_WITH_ERRORS`, `FAILED` | Estado de ejecución. |
| `CampaignRecipientStatus` | `PENDING`, `SENT`, `FAILED`, `SKIPPED_DUPLICATE` | Estado/resumen de intento conforme al baseline. No usar `SKIPPED` genérico. |

**OPEN:** compatibilidad de naming/casing Prisma/PostgreSQL, estrategia de migración evolutiva de valores, y si recipient state se mantiene como projection o solo como status de Attempt. No hay que introducir enums de proveedor hasta disponer del contrato.

## 6. Relación con Survey y User existentes

### Survey

- FK conceptual `Campaign.surveyId` → `sspectsurvey.sspecrpksur`.
- La encuesta, su contenido SurveyJS y las respuestas permanecen en `sspectsurvey` y `sspectresult`.
- La regla funcional Campaign requiere status=1, isPublic=true y archived=false; disponibilidad opcional permanece en `content` JSON. No se agrega estado/columna ni se redefine status global.
- La acción de borrado de Survey y su relación con Campaign debe impedir referencias huérfanas. Comportamiento `Restrict`/archivado se decide al definir migración; no cascader ejecuciones históricas.
- **OPEN:** el modelo `sspectsurvey` no provee versiones históricas de contenido evidentes. Confirmar si Campaign execution solo necesita snapshot de versión/hash o si debe preservarse la definición de encuesta inmutable mediante el flujo actual.

### User / CreatedBy

- Referenciar `sspectuser.sspecrpkuse` mediante `createdByUserId` para integridad y navegación, no duplicar tabla de usuarios.
- Los campos PEC `created_by_user` / `last_modified_by_user` son strings de auditoría usados ampliamente. Definir si se mantienen además de la FK para conservar las convenciones de auditoría; evitar que sean dos fuentes contradictorias del usuario.
- Si el usuario se desactiva/elimina, la Campaign y los historiales deben conservar su referencia/auditoría. Estrategia de `onDelete` apropiada (restrict o FK nullable con snapshot de nombre) es OPEN; no cascader Campaign por eliminación de User.
- La vinculación del usuario creador al `enterpriseId`/permisos existentes debe validarse con el modelo de roles de PEC.

## 7. AudienceExclusion

No se recomienda una entidad independiente `AudienceExclusion` para v1:

- La exclusión requerida de compras previamente procesadas se obtiene del ledger `CampaignRecipient`, con unicidad Campaign + purchase.
- Duplicar una tabla separada para la misma función crearía dos fuentes de verdad para idempotencia.
- Supresión por consentimiento, bajas de Emblue, exclusión manual o listas de no contacto no está especificada funcionalmente ni tiene contrato fuente confirmado.

**OPEN:** si negocio o Emblue exige supresión persistente, definir su propietario, vigencia, granularidad (cliente/email/compra), origen y sincronización. Añadir un mecanismo separado solo con esa regla aprobada; no asumir que un `SKIPPED_DUPLICATE` representa baja de comunicaciones.

## 8. Decisiones de normalización

- `Campaign` guarda la configuración vigente; tablas de asociación normalizadas guardan los valores multiselect y facilitan edición/filtrado sin duplicar los maestros externos.
- Cada asociación usa ID externo y fuente, más una etiqueta snapshot opcional para mostrar selecciones. El maestro completo sigue siendo propiedad de CF/Analytics.
- `Auxiliary Subclassification` no se modela en v1.
- `CampaignExecution.configurationSnapshot` congela un JSON acotado de la configuración que produjo la ejecución. Evita replicar seis tablas snapshot adicionales y aísla el historial de ediciones actuales. El JSON no debe ser audiencia completa ni contener PII.
- `CampaignRecipient` funciona como ledger/idempotency key durable por Campaign + Purchase. `CampaignRecipientAttempt` (o estructura física equivalente) es obligatorio para preservar cada outcome individual por Execution/Recipient, incluidos duplicados detectados, sin crear recipients nuevos al reintentar o al hallar una colisión.
- `ResponseCorrelation` se normaliza por separado del `sspectresult.content Json`, para no alterar el payload de SurveyJS ni exponer identificadores en contenido de respuesta; se vincula a la invitación que deriva de Campaign + Execution + Recipient.
- No duplicar Survey, User, Survey Result ni catálogos externos.
- Si reportes futuros necesitan filtrar snapshots JSON con alta selectividad, reevaluar snapshot normalizado por ejecución; no agregar tablas sin necesidad medida.

## 9. Estrategia de snapshots e historial

Al iniciar una ejecución, congelar como mínimo:

- `campaignId`, `surveyId` y la versión/hash observada de Survey si se acuerda;
- nombre/descripción de Campaign que se mostró al operador;
- IDs externos y labels seleccionados para Division, Department, Auxiliary Classification, Company, Format y Store;
- `postPurchaseOffset`;
- `sendType`, `frequency`, `startDate`, `endDate`, `sendDate`, `sendTime`, `timeZone` y `scheduledAt` efectivos;
- referencia externa/nombre de plantilla y variables habilitadas, sin valores PII por destinatario;
- versión de reglas o formato del snapshot para futura lectura (`snapshotVersion` conceptual).

Reglas del snapshot:

1. Es inmutable después de iniciar/finalizar una ejecución.
2. Editar o suspender Campaign solo cambia la configuración vigente y no actualiza snapshots anteriores.
3. IDs/labels de los filtros son el criterio aplicado; el snapshot no incluye la audiencia materializada completa.
4. No copiar el contenido íntegro de SurveyJS en Campaign. Si se necesita reproducir exactamente la encuesta invitada luego de cambios en `sspectsurvey.content`, resolver OPEN de versionado de Survey.
5. `CampaignRecipientAttempt` registra persistentemente outcomes individuales por Execution y Recipient; `CampaignExecution` agrega conteos derivados de esos outcomes. `SKIPPED_DUPLICATE` cuenta en `duplicateCount`, no crea otro Recipient ni implica request a Emblue. Corregir una lectura/reporting no muta estado final histórico sin un evento/auditoría explícita.
6. La retención de `emailSnapshot`, tokens/correlaciones, provider refs y resultados se debe acordar antes de producción.

## 10. Idempotencia y múltiples réplicas

### Deduplicación de negocio

- Clave funcional: `(campaignId, purchaseSource, purchaseExternalId)` equivalente a Campaign + `purchaseId/transactionId`.
- Un cliente puede tener varios recipients dentro de una Campaign si son compras distintas.
- El ledger Recipient se reserva antes de solicitar envío; reservar no equivale a procesar ni enviar. Una colisión se registra en la Execution actual como outcome `SKIPPED_DUPLICATE` ligado al Recipient ya existente, sin cambiar su estado ni crear otro Recipient.
- `duplicateCount` cuenta esas detecciones/outcomes únicos dentro de la Execution; no se cuentan como `sentCount`/`failedCount`, ni como request/Attempt real al proveedor. Una detección de duplicado por sí sola no origina llamada a Emblue.
- La reserva tiene owner/expiración conceptuales. Reserva vencida o abandono no debe excluir permanentemente una compra que siga pendiente; podrá volver a considerarse al recuperarse la reserva. Falla total tampoco debe marcar como procesadas las compras pendientes. La semántica de cuándo se registra `firstProcessedAt` y qué outcome confirma procesamiento permanece sujeta a política de retry, no se fija aquí.
- Fallos y reintentos reutilizan la misma identidad Recipient y agregan un nuevo Attempt. No se crea una identidad nueva de Campaign + Purchase.
- **OPEN:** si `FAILED` mantiene exclusión permanente hasta retry autorizado o queda elegible nuevamente tras una regla temporal. No tomar la decisión solamente por la constraint.

### Idempotencia de ejecución

- `CampaignExecution.executionKey` identifica el scheduled execution o unidad consolidada de recuperación. Un unique `(campaignId, executionKey)` permite que varias réplicas compitan sin registrar duplicado.
- Una recuperación se enlaza a la Execution original mediante `recoveryOfExecutionId` o relación equivalente; sus outcomes/attempts se asocian a la Execution que los produjo. Estados terminales permanecen inmutables; un claim expirado antes de un resultado final puede recuperarse en esa misma Execution, mientras el tratamiento de una falla terminal queda en una nueva Execution relacionada. No se crea una Execution independiente por cada frecuencia perdida. La regla exacta de recuperación es OPEN.
- El scheduler global puede dispararse en más de una réplica. La ejecución solo continúa si una réplica obtiene claim/lease compartido y vigente.
- Los campos `claimOwner`, `claimedAt`, `claimExpiresAt` son conceptuales; mecanismo final (row lock, advisory lock, lease) sigue siendo ADR/OPEN de arquitectura.
- No sostener transacción de DB durante la llamada a Emblue. Guardar antes/después los estados y provider refs necesarios para reconciliar timeout ambiguo.
- Si Emblue ofrece idempotency keys, almacenar por request/Attempt su clave y external ID; no asumir que el contrato las ofrece.
- Eventos DataLab/Webhooks se deduplican por `(provider, providerEventExternalId)` si el proveedor expone una clave estable.

**OPEN:** regla exacta de `executionKey` para recovery consolidado y reintento; si un claim expirado reanuda la misma Execution o crea otra enlazada; duración/renovación de lease; resultado de envío incierto; existencia y alcance de idempotencia Emblue.

## 11. Índices y constraints recomendados

| Modelo | Índices / constraints conceptuales |
|---|---|
| `Campaign` | `(status, nextExecutionAt)`; `(surveyId, status)`; offset >= 0; invariantes condicionales de campos por `sendType`. No agregar `enterpriseId` a Campaign: el alcance multiempresa aprobado deriva de CASL, no de la empresa activa PEC. |
| Cada `Campaign*` selector | PK/unique `(campaignId, externalSystem, externalId)`; opcional índice externo inverso si se consulta. |
| `CampaignExecution` | unique `(campaignId, executionKey)`; `(status, scheduledAt)`; `(campaignId, scheduledAt)`; `(status, claimExpiresAt)`; conteos >= 0. |
| `CampaignRecipient` | unique `(campaignId, purchaseSource, purchaseExternalId)`; `(campaignId, status)`; `(firstExecutionId)`; índice de reservas vencidas; conteo por cliente indexado solo con caso probado. |
| `CampaignRecipientAttempt` | Obligatorio conceptualmente para trazabilidad Execution/Recipient; `(executionId, status)`; `(campaignRecipientId, attemptNumber)`; unicidad por Execution + Recipient para outcomes únicos de duplicado; unicidad del número de intento en el scope acordado; unique provider message ref cuando sea estable. |
| `ResponseCorrelation` | unique `tokenHash`; índice `campaignRecipientAttemptId`; unique `surveyResultId` solo si cardinalidad 1:1 queda aprobada. |
| `CampaignProviderEvent` | unique `(provider, providerEventExternalId)` si ID no nulo; `(processingStatus, receivedAt)` para reintentos de ingesta. |

Estas recomendaciones se deben validar con consultas reales y explain plans; no se especifica tamaño, longitud física, collation ni estrategia de particionado porque volumen y claves externas están pendientes.

## 12. Impacto en modelos PEC existentes

| Modelo existente | Relación propuesta | Cambios requeridos en etapa de implementación |
|---|---|---|
| `sspectsurvey` | `Campaign.surveyId` referencia `sspecrpksur`. | Añadir relación inversa Prisma. No duplicar contenido. Validar publicación/vigencia en el servicio. |
| `sspectresult` | `ResponseCorrelation.surveyResultId` referencia `sspecrpkres`. | Añadir relación inversa opcional o dejar relación navegable desde el nuevo modelo; no alterar `content Json` para guardar tokens. |
| `sspectuser` | `Campaign.createdByUserId` referencia `sspecrpkuse`. | Añadir relación inversa Campaign; definir `onDelete` sin cascada de historial. |
| `sspectenterprise` | Campaign no tiene FK al enterprise PEC; puede asociarse a Surveys de distintas empresas conforme a política multiempresa. | No filtrar por empresa activa ni tratar Company de CF como Enterprise PEC. |
| `sspectformat`, `sspectestablishment` | No se relacionan en v1 hasta comprobar equivalencia con `Format`/`Store` externos. | Si se confirma identidad/autoridad, reevaluar FKs para evitar IDs duplicados. |

## 13. OPENs

| ID | Punto por resolver | Motivo |
|---|---|---|
| OPEN-01 | Contrato CF/Analytics, source codes, IDs/tipos/longitudes, jerarquías y fecha de actualización. | Determina claves de selecciones y deduplicación. |
| OPEN-02 | Equivalencia o no de Company/Format/Store con `sspectenterprise`/`sspectformat`/`sspectestablishment`. | Evita relaciones FK incorrectas o duplicación de maestros. |
| OPEN-03 | El alcance multiempresa ya está aprobado; validar que la implementación no filtre por enterprise activo. | No amplía permisos generales de Surveys ni de otros módulos. |
| OPEN-04 | **CLOSED para Campaign:** el gate usa `status=1`, `isPublic=true`, `archived=false`; disponibilidad vive en `content.availableFrom/availableTo`. Esto no establece un estado global `PUBLISHED`. |
| OPEN-05 | DST/horas ambiguas o inexistentes, timezone de campañas fuera de Ecuador, conversión local/UTC y efecto de recovery/downtime sobre EndDate. El default Ecuador `America/Guayaquil`, la persistencia explícita de timezone, el límite local inclusivo y el ajuste mensual/trimestral anclado están aprobados. | Evita ambigüedad restante en calendarización; no cambia los campos físicos actuales. |
| OPEN-06 | Cuándo avanzar el próximo ciclo ante fallos parciales y cuál es el cutoff exitoso de audiencia. `lastExecutionAt` queda definido como `finishedAt` de la terminal más reciente, incluso FAILED. | Impacta dashboard, recovery e idempotencia. |
| OPEN-07 | Forma física exacta del `executionKey`, estrategia PostgreSQL/Prisma para claim y el campo/token de fencing. La clave debe ser determinista y versionada; claim atómico + lease + protección de worker rezagado son reglas APPROVED. | Elegir una implementación compatible con la versión PostgreSQL/Prisma usada. |
| OPEN-08 | `purchaseId` vs `transactionId`, fuente, estabilidad, posibilidad de colisión entre sistemas y tratamiento de IDs nulos. | Define la constraint de deduplicación durable. |
| OPEN-09 | Cuándo se confirma `firstProcessedAt`, cómo se proyecta el estado del ledger y qué errores Emblue están confirmados como transitorios/permanentes/ambiguos. Política APPROVED: hasta 3 intentos automáticos de retry además del inicial, solo transitorios confirmados, backoff exponencial con jitter, retries en nueva Execution relacionada, sin retry ambiguo/permanente. | No excluir permanentemente compras pendientes por una reserva abandonada o fallo total. |
| OPEN-10 | Si el correo debe persistirse para retries o puede volver a leerse; cifrado y retención. | PII directa y restricción de logs. |
| OPEN-11 | Contrato Emblue: template ID, recipient/message/batch refs, límites, soporte/alcance de idempotency key, estados, consulta/reconciliación y callbacks. | Determina campos de integración y unicidad de provider refs; no asumir garantías no documentadas. |
| OPEN-12 | Cardinalidad de Attempt/invitación ↔ ResponseCorrelation ↔ `sspectresult`; token único, reutilizable, múltiples respuestas. | La atribución debe derivar Campaign + Execution + Recipient; cardinalidad y constraints no se fijan sin aprobación. |
| OPEN-13 | Duración y política del token opaco, hash, expiración, consumo y replay. | Atribución segura de encuesta. |
| OPEN-14 | Necesidad, valor legal/funcional y retención de `CampaignProviderEvent`; formato y firma de webhook. | No persistir eventos innecesarios o PII. |
| OPEN-15 | Política común de auditoría PEC para nuevos modelos: strings actor/IP, `version`, `status Int` vs enums y soft delete. | Schema actual combina convenciones; evitar incompatibilidad. |
| OPEN-16 | Delete de Campaign DRAFT sin historial: mecanismo lógico, visibilidad y auditoría; no existe `deletedAt` global verificado. | Debe reconciliarse con acción Delete y enums funcionales. |
| OPEN-17 | Requerimiento de snapshot de Survey/plantilla para reproducibilidad exacta. | SurveyJS content y template externos pueden cambiar tras scheduling. |
| OPEN-18 | Volumen, retención, estrategia de particionado o purga de attempts/events/recipients. | Afecta costos e índices. |

## 14. Propuesta de mapeo Prisma

Esta sección es conceptual, no un schema ejecutable.

| Concepto | Mapeo Prisma propuesto |
|---|---|
| PK nueva | Usar `Int @id @default(autoincrement())` como primera opción por compatibilidad con los modelos PEC existentes; evaluar `BigInt` solo ante volumen/ID contractual que lo justifique. No usar UUID sin requisito. |
| Modelos nuevos | Nombres de aplicación `Campaign`, `CampaignDivision`, `CampaignExecution`, `CampaignRecipientAttempt` (concepto obligatorio; implementación física equivalente permitida), etc.; decidir nombres físicos con `@@map` para seguir la estrategia del módulo y convenciones PostgreSQL. El repo mezcla PascalCase, camelCase y nombres legacy, por lo que naming físico queda OPEN. |
| Relaciones internas | Scalar FK entero + `@relation(fields: ..., references: ...)`, referenciando `sspectsurvey`, `sspectuser` y `sspectresult` existentes. Definir `onDelete` explícitamente según retención/historial; no usar cascadas que borren ejecuciones. |
| Enums | Declarar los cinco enums funcionales nuevos de esta especificación. Mantener los valores aprobados y no reutilizar `status Int` para duplicar su semántica. |
| Fechas | `DateTime` para instantes; fecha calendario y hora local deben mapearse con tipos PostgreSQL compatibles (`DATE`/`TIME` si Prisma/proyecto lo permite) o representación acordada; no persistir `SendTime` como fecha completa ficticia. |
| JSON | `configurationSnapshot` como `Json` no nullable y versionado conceptualmente; solo valores de configuración, sin audiencia completa ni PII. |
| IDs CF/Emblue | Campos String conceptuales con longitud/normalización pendientes. No FKs Prisma a sistemas externos. External refs de Emblue nullable hasta contrato. |
| Multiselect | Modelos join con PK compuesta o `Int` surrogate + `@@unique` equivalente; decidir después de revisar límites reales y convención física. |
| Constraints | PK, `@@unique`, `@@index` Prisma más reglas condicionales (SendType, offset >=0, estados/timestamps) que podrían requerir validación de servicio y/o SQL CHECK en migración. Prisma schema no expresa todas las constraints de dominio en todas las versiones. |
| Auditoría | Evaluar estándar `created_by_user`, `created_date`, `last_modified_by_user`, `last_modified_date`, `version` e IP según política PEC, equilibrándolo con FK `createdByUserId` y con la inmutabilidad de snapshots/historial. |
| Soft delete | `deletedAt DateTime?` es una opción conceptual, no convención existente comprobada. Resolver con Product/PEC si DRAFT Delete debe usar marca, `status` legado o mecanismo común. |

## 14.1 Recuperación, claims y retries (decisiones APPROVED / propuesta física)

Reglas de dominio APPROVED: slots vencidos se consolidan en una única unidad de recovery por Campaign; backlog automático inicial máximo 30 días configurable, y excederlo genera alerta/intervención sin descartar ni marcar datos procesados. Executions terminales son inmutables. Retry crea una nueva Execution relacionada y contiene solo recipients elegibles; hasta 3 intentos automáticos de retry además del inicial para errores transitorios confirmados, con backoff exponencial y jitter. Errores permanentes y ambiguos no se reintentan automáticamente. `lastExecutionAt` es `finishedAt` de la terminal más reciente, incluyendo `FAILED`.

Propuesta técnica, no aplicada al schema:

| Necesidad | Encaje/cambio mínimo propuesto |
|---|---|
| Slot ordinario determinista | `executionKey = v1:slot:<scheduledAtUtc>`; guardar zona y fecha/hora local originales en `configurationSnapshot`. |
| Recovery consolidado | `executionKey = v1:recovery:<firstSlotUtc>:<lastSlotUtc>`; snapshot inmutable incluye rango y lista de `slotKeys` cubiertos. El FK singular `recoveryOfExecutionId` no representa varios slots fuente; el snapshot permite trazabilidad inicial, y una tabla join `CampaignExecutionSlot` solo si se exige consulta relacional de cada slot. |
| Retry de fallo terminal | `executionKey = v1:retry:<sourceExecutionId>:<retryOrdinal>`; `recoveryOfExecutionId` apunta a la Execution fuente. Asegurar unicidad de intento por Recipient/Execution, y no reusar Attempt/outcome terminal. |
| Worker rezagado | Añadir `claimGeneration Int NOT NULL DEFAULT 0` a `CampaignExecution` como fencing token monotónico (o equivalente CAS robusto); cada toma/renovación incrementa generación y todas las escrituras del worker verifican owner, lease y generación vigentes. |
| Retry diferido | La nueva Execution relacionada reutiliza `scheduledAt` como instante programado y `CampaignRecipientAttempt.attemptNumber` como ordinal; no se agrega `nextRetryAt`. Base/tope, jitter y clasificador contractual permanecen OPEN. |
| Resultado externo | Se agregan outcome y retry class de Attempt independientes de `CampaignRecipientStatus`, además de `providerActivityId`; `ACCEPTED` significa aceptación API y no cambia Recipient a `SENT`. |

La selección del mecanismo exacto de claim (transacción/row lock o actualización condicional atómica) debe validarse con la versión PostgreSQL/Prisma y prueba concurrente real. La unicidad actual `(campaignId, executionKey)` y ledger `(campaignId, purchaseSource, purchaseExternalId)` se conservan.

## 15. Estado del documento

- **Status:** DRAFT
- **Version:** 0.5
- **Fuentes revisadas:** arquitectura, especificación funcional y `pec-api/prisma/schema.prisma`.
- **Pendiente:** resolver OPENs de CF/Analytics, Emblue, estados Survey, calendario, PII, auditoría y operación antes de habilitar ejecución productiva. Los cambios de Sprint 3.1A son locales y aditivos; no se aplicaron a PostgreSQL.
- **Alcance documental:** este documento sigue definiendo el modelo logico; los modelos Prisma Campaign existen en checkout. No se ejecutan migraciones ni se modifica la base desde esta revision.

## 16. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Se hace obligatoria la trazabilidad persistente por Execution y Recipient; se relaciona la atribución con la invitación/Attempt; se distinguen duplicados, reservas temporales, procesamiento y envío; se explicita la relación de recuperación con ejecuciones originales y la consolidación de frecuencias omitidas. |
| 0.3 | 2026-10-20 | Se elimina el scope conceptual Campaign por enterpriseId; se distingue Enterprise PEC de Company externa y se mantiene el almacenamiento de IDs de catálogos como referencias externas, incluidos los IDs sintéticos MOCK. El documento permanece DRAFT. |
| 0.4 | 2026-10-20 | Se aclara que las etiquetas españolas son de presentación y no sustituyen enums, propiedades, entidades, IDs ni valores persistidos. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se documenta la duración inclusiva derivada de fechas calendario para validar frecuencias y la separación entre borrador editable y validación efectiva de programación. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se documenta `America/Guayaquil` como default de Campaign para Ecuador, la persistencia explícita de `timeZone`, EndDate local inclusiva y la prohibición de inferir zona del navegador; recovery y conversión horaria permanecen OPEN. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Documenta recurrencias MONTHLY/QUARTERLY ancladas a StartDate, ajuste al último día sin drift y aritmética local previa a UTC, incluidos años bisiestos. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Formaliza recuperación consolidada, backlog 30 días, estados terminales inmutables, retry selectivo y semántica de `lastExecutionAt`; propone fencing, keys versionadas y campos/outcomes faltantes sin modificar Prisma. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra `activityId` como referencia de aceptación del adapter emBlue, sin mapeo persistido ni inferencia de entrega; `/activities/find` permanece OPEN. Estado DRAFT conservado. |

### Actualización Sprint 3.0 — referencia de envío emBlue

El response confirmado de `POST /v2.3/emails/send` contiene `activityId` string. El adapter lo devuelve como referencia de aceptación técnica junto con el código HTTP. No se persiste todavía: el schema actual no define una asignación aprobada de esta referencia por solicitud/recipient ni una reconciliación. `CampaignExecution.providerExecutionRef` permanece una posibilidad conceptual, no un mapeo implementado. No inferir entrega final desde `activityId`. Consulta `POST /activities/find` e idempotencia externa/persistida siguen OPEN. Estado DRAFT conservado.

### Actualización Sprint 3.1A — persistencia y servicio interno

El schema local agrega `CampaignExecution.claimGeneration` como generación monotónica de fencing y en `CampaignRecipientAttempt`: `attemptKey` interno único nullable para compatibilidad con filas previas, `providerOutcome`, `retryClass` y `providerActivityId`. Los enums nuevos no alteran `CampaignRecipientStatus`. La clave externa `idempotencyKey` no se utiliza ni se interpreta como garantía de emBlue.

No se crean tablas adicionales. Los slots cubiertos por una recovery se conservan en `configurationSnapshot.execution.coveredSlots`; el retry se programa en una nueva Execution relacionada mediante `recoveryOfExecutionId` y `scheduledAt`, con nuevos Attempts y `attemptNumber` incrementado solo para la selección elegible. `ACCEPTED` conserva el `activityId`, pero no marca Recipient como `SENT`; `AMBIGUOUS` no admite retry class ni retry automático.

El SQL 20261023000000_add_campaign_execution_controls fue aplicado manualmente en desarrollo segun confirmacion del usuario. prisma validate paso; prisma generate fallo inicialmente por EPERM y luego se completo correctamente. No se ejecuto prueba PostgreSQL con escrituras; DRAFT conservado.

### Validación Sprint 3.1A.1 — esquema de desarrollo (2026-10-23)

El usuario ejecutó manualmente el SQL de la migración en la base de desarrollo. Una transacción `READ ONLY` confirmó ambos enums y sus valores, `claim_generation INTEGER NOT NULL DEFAULT 0`, las cuatro columnas nullable de Attempt y el índice único sobre `attempt_key`, en coincidencia con `schema.prisma` y el SQL local. La fila no existe en `_prisma_migrations`; Prisma aún no reconoce la migración. No se ejecutó reconciliación ni migración. `prisma validate` pasa; `prisma generate` sigue fallando por `EPERM`. No se probaron escrituras ni concurrencia en esa base. Estado DRAFT conservado.

El registro incompleto 20250907231907_add_user_establishment_table es metadato historico y no es bloqueo bajo Database-First; no se modifico.


### Convención Database-First y estado (2026-10-23)

Esta sección supersede notas anteriores que sugirieran prisma migrate resolve o consideraran _prisma_migrations un bloqueo. PostgreSQL es la fuente de verdad. Los cambios físicos se entregan como SQL revisable para ejecución manual; luego se sincroniza schema.prisma y se ejecuta prisma generate. No se aplican cambios con migrate dev/deploy/resolve, db push o migrate reset. La migración Campaign no figura en el historial Prisma, pero esto no afecta el ORM. El SQL de referencia está en prisma/manual-sql/campaign_execution_controls.sql. prisma validate, prisma generate, build NestJS y 84 pruebas unitarias Campaign pasaron después de regenerar el cliente. No se corrieron pruebas PostgreSQL con escrituras. Estado DRAFT conservado.

### Sprint 3.1B — uso del modelo existente

El lifecycle usa `Campaign.status`, `nextExecutionAt` y `version`; las Executions ordinarias y de recovery usan `executionKey`, `recoveryOfExecutionId`, `configurationSnapshot.coveredSlots` y los claims/leases existentes. El avance de calendario y creación de Execution se realizan en una transacción serializada por fila Campaign. No se propusieron columnas, tablas ni SQL nuevos. Las pruebas PostgreSQL de Sprint 3.1B quedaron omitidas en el checkout actual por falta de conexión local; no se afirma validación concurrente de esta fase. DRAFT conservado.

| 0.5 | 2026-10-23 | Confirma que Sprint 3.1B reutiliza el modelo persistido sin DDL adicional; las pruebas PostgreSQL de esta fase quedan pendientes. DRAFT conservado. |

### Sprint 3.1B.1 — validación de persistencia

La suite PostgreSQL ejecutó 20 casos contra el contenedor aislado ya existente. Confirmó integridad/constraints y operaciones Campaign, incluida la concurrencia lifecycle/scheduler y la persistencia de recuperación. Tras los fixtures, consultas SELECT comprobaron cero filas de prueba en las tablas Campaign y entidades PEC auxiliares creadas para el test. No se aplicó DDL, Prisma Migrate ni cambios al modelo; base principal y Neon no fueron usados. DRAFT conservado.
## Registro de validación Sprint 3.1C

No se modifica el modelo persistido. El preflight consulta `sspectsurvey.status`, `archived` e `isPublic` junto con el calendario de Campaign. La regla aprobada para Campaign considera elegible la Survey únicamente con la conjunción `status === 1`, `isPublic === true`, `archived === false`; status aislado no basta. El endpoint público conserva su fallback para encuestas genéricas y revisa `content.availableFrom` / `content.availableTo` al solicitarlo. No hay estado/fecha persistido de publicación ni relación que resuelva destinatario, token o enlace individual de Campaign. Los contratos de audiencia/plantilla permanecen OPEN.

| 0.5 | 2026-10-23 | Registra los campos de Survey verificados y confirma que no se requiere cambio Prisma para el preflight; publicación Campaign, audiencia y enlace individual siguen OPEN. DRAFT conservado. |
## Sprint 3.1C.1 — campos y puertos candidatos

El modelo actual persiste filtros como IDs externos y `postPurchaseOffset`; `CampaignRecipient` contempla `purchaseSource`, `purchaseExternalId`, `purchaseIdType`, `customerExternalId`, `purchaseDateSnapshot`, `eligibleAt` y `emailSnapshot`. Estos nombres identifican espacio disponible, no demuestran cómo Analytics mapea sus datos ni qué fecha determina elegibilidad. No se consultaron filas ni datos personales.

Se propone `CampaignAudiencePageRequest` con selección configurada, offset, fecha de evaluación, cursor y tamaño de página como interfaz interna. Los cursores/tamaños no son campos confirmados de Data-Lab. El resultado candidato usa referencias de compra/cliente y email; acceso, minimización, retención, filtros de consentimiento, deduplicación y reglas de fecha quedan OPEN. No devolver audiencia completa a Angular ni persistir la audiencia total.

Para plantillas se propone definición versionada con `externalId`, `subject`, `html` y declaración de variables con contexto; el registro actual `emailTemplateExternalId` no acredita una plantilla válida. `opaqueCorrelationReference` no especifica formato ni generación del token. No requiere DDL en esta fase.

| 0.5 | 2026-10-23 | Mapea campos existentes a puertos candidatos sin confirmar mapeos fuente ni elegibilidad; no cambia Prisma. DRAFT conservado. |


## Sprint 3.1C.1 - publicacion de Survey para Campaign

Regla funcional aprobada para Automated Survey Campaigns: una Survey es elegible solo si `status = 1`, `isPublic = true` y `archived = false`. `status=1` aislado no demuestra publicacion ni cambia el significado global de Survey. La ruta publica existente conserva su comportamiento, incluido el fallback para Surveys genericas; Campaign exige `isPublic=true` y no utiliza ese fallback.

El gate lee `content.availableFrom` y `content.availableTo` si estan configurados. Valida formato e intervalo y verifica que el calendario Campaign completo, desde inicio local hasta fin local inclusivo, quede cubierto. Usa la zona IANA de Campaign para el calendario y el parser server-local ya usado por la ruta publica para interpretar los limites Survey. Mantener TZ consistente entre replicas del API es un requisito operativo; acceso a respuestas despues de `availableTo` y enlace individual Campaign siguen pendientes.

Preflight reporta `SURVEY_PUBLICATION=READY` unicamente con la conjuncion aprobada; `SURVEY_CALENDAR_AVAILABILITY` distingue READY, BLOCKED y OPEN. Schedule y Resume vuelven a validar antes de mutar estado. No se modifica Prisma, no se cambia la ruta Surveys, y audiencia real, catalogos reales, plantilla y enlace individual siguen bloqueando la activacion. Se conserva v0.5 / DRAFT.

## Sprint 3.1C.2 - lectura del estado Survey

Sin cambio de esquema: el validador Campaign lee los campos existentes `status`, `isPublic`, `archived` y `content`. Elegibilidad confirmada exige exactamente `status === 1`, `isPublic === true`, `archived === false`. No se añade campo ni flujo de publicación. Los límites opcionales `content.availableFrom/availableTo` se validan como fechas calendario y deben cubrir la ventana programada. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra regla Campaign de Survey, gate temporal y cobertura de pruebas; DRAFT conservado. |

## Sprint 3.1C.3 - correspondencia pendiente de fuente real

| Campo requerido por Campaign | Campo PEC/fuente observado | Estado del mapeo |
|---|---|---|
| Division / división externa | `workgroupName` y `workAreaReferenceCode` aparecen en `syncRegions()` para áreas de trabajo/zonas; el primero se persiste como `sspectregion.name`. | Sin equivalencia confirmada con Division CF/Data-Lab. |
| Department / departamento externo | No identificado en respuesta/uso del código inspeccionado. | Sin asignar. |
| Company CF | `codigoCompania=1` es parámetro fijo al sincronizar locales; `sspectenterprise` es Enterprise PEC independiente. | Sin catálogo multiempresa ni equivalencia confirmada. |
| Auxiliary Classification | No identificada en fuentes del checkout. | Sin asignar; permanece fuera de V1 según alcance. |
| Format | `establecimiento.nombreEstablecimiento` se usa para resolver por nombre `sspectformat`; no se conserva ahí ID externo de Format. | Reutilización PEC parcial; identidad/origen CF no probado. |
| Store | `codigoReferencia` alimenta `sspectestablishment.code`; `nombreLocal`, dirección, estado operativo y formato se sincronizan. Consulta observada restringida a `codigoCompania=1`. | Reutilización PEC parcial para los locales sincronizados; no prueba cobertura completa multiempresa ni contrato Campaign. |
| Purchase external ID / source / type | Campaign espera `purchaseSource`, `purchaseExternalId`, `purchaseIdType`; no se encontró ledger/feed de compras de cliente en modelos PEC inspeccionados. | Sin asignar; Data-Lab debe definir clave estable, namespace y tipo. |
| Purchase date | `sspectcomplaint.invoiceDate` y `dateOfPurchase` son fechas de queja/reclamo, no evidencia de transacción comprada. | No reutilizar como fecha de elegibilidad. Fuente de compra pendiente. |
| Customer external ID / email | `ClientService.findPersonContact()` requiere documento individual y devuelve tipo no tipado; no hay consulta masiva de contactos asociada a compras. `CampaignRecipient` admite IDs/email snapshots opcionales. | Contrato de destinatario y fuente de email sin asignar. |
| Filtros jerárquicos | Campaign persiste IDs externos como strings y snapshots label/parent; MOCK define relaciones sintéticas. | Fuente real, claves y relaciones no verificadas. |

La búsqueda en el repositorio no encontró scripts SQL de vistas/procedimientos ni un cliente Data-Lab/Analytics de audiencia. Esta constatación se limita al checkout inspeccionado; no demuestra que CF carezca de tales fuentes. No leer datos reales ni inferir catálogo por similitud de nombres. No se requieren cambios Prisma en este diagnóstico. v0.5 / DRAFT.

El port `CampaignAudiencePageRequest` incluye selección, `postPurchaseOffsetDays`, `evaluationDate`, `cursor` y `pageSize`; no define corte reproducible (`eligibilityCutoffAt`/snapshot), timezone asociado a evaluationDate, orden/cursor estable ni forma de distinguir agotamiento normal de una fuente incompleta. `CampaignRecipientCandidate` no lleva `purchaseIdType` aunque `CampaignRecipient` lo persiste. No se asigna email ni identificador PEC de contacto como equivalencia de cliente corporativo. Validar si la fuente devuelve correo/contacto autorizado junto con la compra o si se requiere un segundo lookup permitido.

| 0.5 | 2026-10-23 | Documenta mapeos observados y campos de fuente real aún sin asignar; no consulta datos ni cambia esquema. DRAFT conservado. |
## Sprint 3.1C.4 - DDL propuesto de staging BI (OPEN)

El DDL revisable esta en `pec-api/prisma/manual-sql/bi_campaign_staging_proposal.sql`. Define, solo como propuesta, `bi_integration.bi_segmentation_catalog` (snapshot por `load_id`, tipo/id/label, padre tipado, Company, active, vigencia y timestamps de fuente/carga/publicacion) y `bi_integration.bi_customer_purchases` (identidad de registro fuente, purchase source/id/type, purchase line opcional, fecha/hora, customer, campos de segmentacion/producto y marcador de publicacion). IDs externos son TEXT opacos; el batch se propone UUID. Sin FKs entre dimensiones porque sus codigos y relaciones reales no estan confirmados.

Claves propuestas: `(load_id, catalog_type, external_id)` y `(load_id, source_record_external_id)`. Indices soportan filtros y keyset por `source_record_external_id`; unicidad del ledger PEC sigue siendo `(campaignId, purchaseSource, purchaseExternalId)`. La tabla de compras hace Division, Department y Company requeridos, alineado a filtros obligatorios, sujeto a que BI confirme estos datos en cada fila. Las dimensiones opcionales corresponden a clasificacion auxiliar, formato y local; Auxiliary Subclassification queda excluida.

Propuesta de lote: cargar ambos snapshots bajo el mismo load_id, validar, y en una transaccion publicar filas de ambas tablas; lectores fijan un unico load_id para todas las paginas. Este marcador por fila no representa correctamente una fuente vacia; si vacio es valido, hace falta una relacion manifiesto o senal atomica equivalente. Retencion, snapshot completo vs incremental, frecuencia, claves, significado de vigencia/fechas, grano (compra vs linea) y proteccion de `recipient_email` quedan OPEN. El email nullable del SQL es sensible y no aprobado; preferir resolverlo en backend desde fuente autorizada. No cambiar Prisma ni ejecutar DDL. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Agrega diseno de datos propuesto para dos tablas BI y senala limites de publicacion por lote; DDL no ejecutado. DRAFT conservado. |

OPEN de grano: el ledger PEC tiene unicidad por `(campaignId, purchaseSource, purchaseExternalId)`, mientras el staging incluye `source_record_external_id` y `purchase_line_external_id` para poder representar una linea o registro fuente. Si BI entrega una fila por producto/linea, el adaptador debe agrupar a compra antes de persistir, o se requiere una decision/cambio de modelo para envio por linea. No se debe escoger esa semantica implicitamente.
## Sprint 3.1C.4.1 - unidad de invitacion por factura (APPROVED)

Decision funcional aprobada: una invitacion por cada factura/compra elegible y Campaign. El staging `bi_customer_purchases` pasa a grano de asignacion factura-linea-producto-division: una factura puede producir varias filas, productos y divisiones. Debe llevar `invoice_source` y `invoice_external_id` estables y no vacios; se exige `invoice_line_external_id` y `product_external_id` para poder evaluar productos. Se conserva `source_record_external_id` como clave de fila unica por lote. La identidad factura+origen se indexa, no se declara unica porque varias filas por factura son esperadas; el consumidor agrupa las coincidencias y deduplica la invitacion por factura.

La audiencia agrupa por identidad `(Campaign, invoice_source, invoice_external_id)` y califica la factura si al menos una fila producto/division satisface todos los filtros seleccionados; multiples IDs dentro de la misma dimension son OR y dimensiones diferentes son AND. Al persistir, `purchaseSource` recibe `invoice_source`, `purchaseExternalId` recibe `invoice_external_id` y `purchaseIdType` puede recibir `invoice_id_type` una vez confirmado el vocabulario BI. La restriccion actual `CampaignRecipient` unica por `(campaignId, purchaseSource, purchaseExternalId)` ya implementa deduplicacion por Campaign e identidad de compra: permite facturas distintas del mismo cliente y no deduplica globalmente por customer/email. No se requiere cambio estructural del ledger PEC para esta regla. Campaign->Execution->Recipient->Attempt se conserva con las relaciones existentes; Recipient retiene origen e ID externo de factura como referencia trazable a la fuente. `configurationSnapshot` puede registrar `load_id` al integrar el proveedor, sin agregar columna ahora.

OPEN: si una factura puede participar en varias Campaigns queda pendiente de confirmacion; actualmente el modelo permite un Recipient distinto por Campaign. Contactabilidad, consentimiento, supresiones y exclusiones siguen pendientes. Las filas del mismo invoice deben consolidarse antes de paginar resultados de Recipient para no emitir mas de una invitacion. v0.5 / DRAFT.
## Sprint 3.1D - tablas provisionales locales

El SQL ejecutado solo en PostgreSQL local aislado esta en `pec-api/prisma/manual-sql/bi_campaign_staging_provisional_local_test.sql`; los fixtures sinteticos estan en `bi_campaign_staging_provisional_seed.sql`. Se crearon tablas PostgreSQL normales `bi_integration.bi_segmentation_catalog` y `bi_integration.bi_customer_purchases`, fuera de Prisma. No se agregaron modelos ni se alteraron tablas del ledger.

El catalogo conserva snapshot (`load_id`), tipo/ID/etiqueta, padre tipado, empresa, vigencia, activo y timestamps. Compras conserva `source_record_external_id`, identidad de factura (`invoice_source`, `invoice_external_id`, `invoice_id_type`), linea, fechas, cliente/email sinteticos, empresa/division/departamento, dimensiones opcionales, producto y timestamps. La unicidad de fila es por snapshot + registro fuente; la relacion padre se declara dentro del catalogo y las referencias de compras se validan en el proveedor.

Solo se leen lotes publicados conjuntamente, con un `load_id` y timestamp de publicacion comunes. El cursor queda fijado al snapshot y la consulta agrupa por origen + factura antes de paginar. La restriccion existente del ledger `(campaignId, purchaseSource, purchaseExternalId)` implementa deduplicacion por Campaign + factura; no requiere cambio. La fuente es provisional, no DDL/contrato BI aprobado. OPEN: manifiesto para lote legitimo vacio, protocolo de inmutabilidad, retencion, contacto y reuso de factura entre campañas. DRAFT conservado.

| 0.5 | 2026-10-23 | Registra DDL provisional aplicado solo en PostgreSQL local; sin cambios Prisma/ledger ni aprobacion BI. |

## Sprint 3.1E - uso del ledger de correlación (2026-10-23)

El esquema existente `ResponseCorrelation` ya ofrece FK a `CampaignRecipientAttempt`, `tokenHash` único, `surveyResultId`, `expiresAt` y `consumedAt`; permite trazar Attempt → Recipient → factura y asociar una respuesta sin guardar el token en claro. No se requieren cambios Prisma ni DDL. La creación y asociación se implementan detrás de un servicio interno; no se cambió el ledger de Campaign ni las tablas generales de Surveys. La ejecución del DDL/seed de staging en Neon fue reportada por el usuario, no comprobada mediante consulta en esta entrega. v0.5 / DRAFT.
## Sprint 3.1F - Persistencia de respuesta correlacionada (2026-10-23)

Se reutiliza el esquema existente ResponseCorrelation: tokenHash, campaignRecipientAttemptId, surveyResultId, expiresAt y consumedAt. En una transacción se crea sspectresult y se actualiza condicionalmente la correlación solo si no fue consumida ni venció; si la actualización no afecta una fila, se revierte también la creación del resultado. El FK existente conserva Campaign → Execution → Recipient → Attempt → ResponseCorrelation → SurveyResult, y Recipient conserva identidad de compra. No se requiere DDL ni cambio Prisma. v0.5 / DRAFT.

## Sprint 3.1F.1 - evidencia de transacción (2026-10-24)

El diseño reutiliza ResponseCorrelation y SurveyResult existentes y persiste ambos cambios en una transacción Prisma con consumo condicional de un solo uso. La unidad verifica el flujo y el caso de compare-and-set perdido; falta una corrida PostgreSQL real que confirme rollback y una sola fila confirmada bajo carrera. Sin cambio de esquema ni DDL en esta fase. v0.5 / DRAFT.
## Sprint 3.1F.1 - compatibilidad y alcance de persistencia (2026-10-24)

La decisión de compatibilidad se determina con los campos ya existentes `sspectsurvey.version` y `content.isQueja`; no requiere cambios de esquema, SQL ni Prisma. Solo se correlacionan respuestas de Survey v1 de satisfacción. No se ejecutaron escrituras ni pruebas contra PostgreSQL en esta validación; la atomicidad bajo concurrencia y rollback sigue pendiente de evidencia de la base local aislada. v0.5 / DRAFT.
## Sprint 3.1G-A - simulación sin persistencia de invitaciones (2026-10-24)

La simulación consulta el ledger Campaign y staging provisional en modo lectura. Identidad de deduplicación en memoria: `(purchaseSource, purchaseExternalId)` dentro de la Campaign; corresponde a la unicidad existente Campaign + compra de `CampaignRecipient`. Para una ejecución futura, se propone crear/reutilizar Recipient por esa clave y registrar cada Attempt con su `attemptKey`; el token opaco se emitiría asociado al Attempt y solo su hash quedaría en `ResponseCorrelation`, mientras el valor claro existiría transitoriamente para preparar el correo. Esa orquestación durable no se implementa aquí y requiere validar atomicidad/integración PostgreSQL antes de afirmar consistencia. No hubo cambios Prisma/SQL ni inserciones a ResponseCorrelation. v0.5 / DRAFT.


## Sprint 3.1G-B - simplificación de conexión Campaign (2026-10-25)

Las tablas provisionales `bi_integration.bi_segmentation_catalog` y `bi_integration.bi_customer_purchases` residen en la misma base PEC y se acceden con el `DatabaseService` existente mediante nombres cualificados. No se crea otra conexión, se cambia `DATABASE_URL` ni se modifica el DDL. La prueba de identidad Neon debe completarse antes de consultar la rama de desarrollo.

Para la validación independiente de rama se requieren solo Project ID, Branch ID, propietario esperado de la base y credencial de API Neon (`CAMPAIGN_PROVISIONAL_NEON_PROJECT_ID`, `CAMPAIGN_PROVISIONAL_NEON_BRANCH_ID`, `CAMPAIGN_PROVISIONAL_NEON_DATABASE_OWNER`, `CAMPAIGN_PROVISIONAL_NEON_API_KEY`). El endpoint se extrae del host de `DATABASE_URL`, y la base y el rol de esa misma URL; estos datos de identidad no seleccionan proveedor ni crean otra conexión. **Registro de revisión:** v0.5, 2026-10-25. Se registra la decisión de conexión única/esquemas separados y selección automática protegida. Estado formal `DRAFT` conservado.


## Sprint 3.1G-B.1 - auditoria de configuracion Campaign (2026-10-25)

Inspeccion local de nombres de variables, sin leer ni publicar valores: `pec-api/.env` y `.env.local` contienen `DATABASE_URL` y `ENV`. `AppModule` carga globalmente `.env.local`, `.env.preprod`, `.env.prod` y `.env`; el guard lee `process.env`. No se encontro `DIRECT_URL` ni una identidad Neon reutilizable en esos archivos. `PROJECT_ID_GCS` identifica GCP y `API_KEY_STAR` pertenece a otra integracion; no son sustitutos de identidad Neon. El inventario local no prueba variables inyectadas externamente al proceso desplegado.

En el checkout, `CampaignNeonTargetGuard` aun exige Project ID, Branch ID, propietario esperado y credencial API Neon. Ninguno aparece en los archivos locales. El error 503 se origina al faltar esa configuracion: el guard falla antes de llamar a la API de Neon y antes de leer el lote en `bi_integration`. Endpoint ID, base y rol se derivan del `DATABASE_URL`; no se requiere otra conexion ni un `DIRECT_URL`. No se modifico `DATABASE_URL`.

No se retiro la verificacion de rama: `ENV` distingue el entorno PEC, pero no identifica una rama Neon; el hostname solo permite derivar el endpoint. La verificacion API Project/Branch/Endpoint y la comprobacion SQL posterior (base, esquema, rol, propietario y primary) siguen siendo controles distintos. El mecanismo equivalente no puede configurarse con los datos que hoy existen en los archivos inspeccionados. Dato pendiente para una lectura segura: una fuente confiable, ya disponible para NestJS, que relacione la conexion actual con el Project ID y Branch ID de desarrollo y permita verificar esa relacion en Neon. Hasta resolverlo, la lectura funcional falla cerrada. Los SPECS permanecen v0.5 / DRAFT.
## Consolidación DatabaseService / esquemas (2026-10-25)

Campaign usa el `DatabaseService`/PrismaClient existente y la única `DATABASE_URL` de PEC. Las lecturas de staging se califican con `bi_integration.bi_segmentation_catalog` y `bi_integration.bi_customer_purchases`; no se configura una conexión independiente por esquema. El rol de ejecución debe contar solo con los privilegios requeridos para lectura de esas tablas. Los privilegios de base de datos no identifican por sí mismos la rama Neon.

La validación de destino y entorno es un control de seguridad de arquitectura descrito en ARCH/INTEG. La API administrativa de Neon no es parte del acceso normal a datos objetivo. Hasta contar con una autorización confiable del destino que no dependa de `ENV` o hostname solamente, se conserva el guard actual y el fallo cerrado antes de las consultas funcionales. Estado formal `DRAFT`; versión v0.5.

## Actualización funcional 2026-10-25 — Fuentes y correspondencia de segmentación

Fuentes implementadas: División, Departamento y Clasificación auxiliar se consultan desde `bi_integration.bi_segmentation_catalog`; Empresa, Formato y Local se consultan desde los modelos activos PEC `sspectenterprise`, `sspectformat` y `sspectestablishment`. Las relaciones PEC verificadas son Formato → Empresa mediante `enterpriseId` y Local → Formato mediante `formatId`. Las compras siguen en `bi_integration.bi_customer_purchases` y se consolidan por `(invoice_source, invoice_external_id)` antes de paginar.

El esquema PEC no contiene una columna de mapeo explícita a `company_external_id`, `format_external_id` o `store_external_id` de BI. La implementación solo acepta igualdad textual exacta entre IDs; no compara por nombre/código ni similitud. Si falta igualdad exacta, preview y operaciones dependientes deben bloquearse e informar la correspondencia faltante. Este estado no declara validado el mapeo BI.

`GET /survey-campaigns/:campaignId/audience-preview?page=N` devuelve conteos de clientes con identificador, facturas elegibles/contactables, registros excluidos por correo ausente/inválido y filas consolidadas. La página contiene etiquetas generadas para cliente/factura y fecha, sin correo, ID de cliente ni identificador real de factura. Preview es de solo lectura. Mantener el estado formal DRAFT y versión 0.5.

Los datos BI persistidos y el seed ya ejecutado no se modificaron. El reemplazo por ejemplos sintéticos representativos (taxonomía general de abarrotes/productos y clientes ficticios) requiere una operación de carga autorizada; no ejecutar ni modificar datos en esta entrega. OPEN: confirmar que los IDs externos BI coinciden con las PK PEC o entregar un mapeo autorizado, además de reglas de contactabilidad/exclusión.
