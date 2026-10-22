# Modelo de Datos - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-DATA-001 |
| Estado | DRAFT |
| Versión | 0.4 |
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
| `providerExecutionRef` | String | Sí | Referencia externa Emblue/DataLab si el contrato provee una; forma OPEN. Puede que refs por lote deban vivir en Attempts/Events. |
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
- La regla funcional de programación requiere Survey `PUBLISHED`; en el schema actual el campo visible es `status Int` y `archived Boolean`, y el contenido es `Json`. El significado exacto de publicación no se infiere del schema: requiere validar el servicio/contrato Survey.
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
| OPEN-04 | Campo/regla real que indica Survey `PUBLISHED`, archivado y disponibilidad. | Schema actual muestra `status`, `archived`, `content Json`; semántica publicada no está determinada por el schema aislado. |
| OPEN-05 | DST/horas ambiguas o inexistentes, timezone de campañas fuera de Ecuador, conversión local/UTC y efecto de recovery/downtime sobre EndDate. El default Ecuador `America/Guayaquil`, la persistencia explícita de timezone, el límite local inclusivo y el ajuste mensual/trimestral anclado están aprobados. | Evita ambigüedad restante en calendarización; no cambia los campos físicos actuales. |
| OPEN-06 | Semántica de `lastExecutionAt` y cuándo actualizar el próximo ciclo frente a éxito/fallo. | Impacta dashboard, recovery e idempotencia. |
| OPEN-07 | `executionKey` exacta, granularidad de recovery, claim/lease y política de múltiples réplicas. | Evita doble ejecución y no perder slots. |
| OPEN-08 | `purchaseId` vs `transactionId`, fuente, estabilidad, posibilidad de colisión entre sistemas y tratamiento de IDs nulos. | Define la constraint de deduplicación durable. |
| OPEN-09 | Failed recipient: exclusión/reconsideración, política de retry, cuándo se confirma `firstProcessedAt` y cómo se proyecta estado del ledger. | No fijar la política de reintentos Emblue; una reserva abandonada o falla total no puede excluir permanentemente compras pendientes. |
| OPEN-10 | Si el correo debe persistirse para retries o puede volver a leerse; cifrado y retención. | PII directa y restricción de logs. |
| OPEN-11 | Contrato Emblue: template ID, recipient/message/batch refs, límites, idempotency key, estados y callbacks. | Determina campos de integración y unicidad de provider refs. |
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

## 15. Estado del documento

- **Status:** DRAFT
- **Version:** 0.4
- **Fuentes revisadas:** arquitectura, especificación funcional y `pec-api/prisma/schema.prisma`.
- **Pendiente:** resolver OPENs de CF/Analytics, Emblue, estados Survey, calendario, PII, auditoría y operación antes de definir schema final.
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
