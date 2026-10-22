# PEC-SC-SCHED-001 — Diseño de Scheduling y Ejecución

## Automated Survey Campaigns

| Campo | Valor |
|---|---|
| Documento | PEC-SC-SCHED-001 |
| Título | Diseño de Scheduling y Ejecución - Automated Survey Campaigns |
| Estado | DRAFT |
| Versión | 0.4 |
| Fecha | 2026-10-20 |
| Idioma | Español; nombres técnicos en inglés |

> Este documento define el diseño propuesto para planificar y ejecutar campañas. Los requisitos funcionales provienen de FOUND-001; las entidades y restricciones conceptuales, de DATA-001. Las decisiones aún no aprobadas se marcan como **OPEN**. El pseudocódigo y los diagramas son ilustrativos, no código productivo.

## 1. Objetivo y principios

Definir un `Central Scheduler` compatible con NestJS y PostgreSQL para detectar campañas vencidas, reclamar trabajo de manera segura entre instancias, procesar audiencias incrementales y registrar ejecuciones recuperables sin generar envíos repetidos.

Principios:

1. Existe un único tick global de scheduler compartido por todas las Campaigns; no se crea un cron por Campaign.
2. El cron de NestJS dispara el sondeo, pero no ofrece exclusión distribuida ni persistencia del trabajo por sí mismo.
3. La base de datos es la autoridad para reclamar una ejecución y para evitar duplicados entre réplicas.
4. Una ejecución congela el instante técnico `eligibilityCutoffAt` y la fecha de negocio `evaluationDate` usados al iniciar la resolución; no se usa el avance del calendario como prueba de que la audiencia fue procesada.
5. Solo se procesan compras que cumplen simultáneamente segmentación, offset y deduplicación.
6. Llamadas a CF/Analytics y Emblue ocurren fuera de la transacción breve de claim; el progreso y las referencias necesarias se persisten por lote/recipient.
7. Los estados finales de `CampaignExecution` son históricos; retries y recuperaciones no reescriben ejecuciones cerradas.
8. Un fallo total no marca compras pendientes como procesadas ni permite avanzar un cutoff exitoso. Un cutoff global exitoso de Campaign, si se adopta, es una propuesta OPEN y nunca sustituye la deduplicación por compra.

## 2. Fuentes y límites de certeza

### 2.3 Presentación en español

Los estados y modalidades conservan sus valores técnicos en inglés durante la ejecución. La interfaz Angular muestra sus equivalencias en español conforme al mapeo centralizado documentado en FOUND-001. `COMPLETED` se presenta como **Finalizada** para Campaign y Execution; `ONE_TIME`, como **Una sola vez**. Esta regla de presentación no habilita ni implementa el scheduler.

### 2.1 Verificado en código del checkout

- `@nestjs/schedule` está instalado; `ScheduleModule.forRoot()` se registra en `pec-api/src/app.module.ts`.
- `SchedulerService` tiene un cron global cada cinco minutos que calcula/actualiza campos de quejas en páginas de 200 y usa `$transaction` para grupos de updates.
- `CaseLocksScheduler` ejecuta limpieza cada minuto. `SyncService` registra sincronización diaria a las 08:00 mediante `@Cron`.
- El scheduler existente llama servicios Nest dentro del API. No se verificó una cola persistente de jobs ni un mecanismo compartido de claim específico para ejecuciones.
- Helm `pec-api/ci/helm/values.yaml` configura `replicaCount: 1` como default. El chart renderiza un Kubernetes `Deployment` y contiene plantilla de HPA con `minReplicas`/`maxReplicas` configurables cuando `autoscaling.enabled` está activo. Esto no demuestra la cantidad de pods desplegados en ningún ambiente.
- El esquema contiene `case_locks`, campos de vencimiento e índice único `(case_id, is_active)`. El servicio de locks administra edición de casos. No es un claim distribuido reutilizable comprobado para campañas ni se verificó una transacción atómica que cubra su flujo completo.
- Existen usos de Prisma `$transaction`; no se encontró en el código inspeccionado un patrón de `FOR UPDATE SKIP LOCKED`, advisory lock ni lease de ejecuciones de campañas.

### 2.2 Decisiones de baseline

- Scheduler centralizado; sin cron por Campaign.
- Tipos: `ONE_TIME` y `FREQUENCY`.
- Frecuencias: `DAILY`, `WEEKLY`, `BIWEEKLY`, `MONTHLY`, `QUARTERLY`, calculadas por calendario como +1 día, +7 días, +14 días, +1 mes calendario y +3 meses calendario.
- `FREQUENCY` requiere `Start Date`, `End Date`, `Send Time` y `Frequency`; `ONE_TIME` requiere `Send Date` y `Send Time`.
- Campaign queda `SCHEDULED` antes de su primera ejecución; pasa a `ACTIVE` después de ejecutar si todavía existen ocurrencias futuras. Cuando la siguiente ejecución supera `End Date`, la Campaign termina `COMPLETED`.
- Para Campaigns de Ecuador, la zona predeterminada aprobada es `America/Guayaquil`; cada Campaign debe persistir explícitamente `timeZone` antes de Schedule/Resume efectivos. No se infiere la zona del navegador.
- `EndDate` es inclusiva como fecha local de negocio: una ocurrencia exactamente en `EndDate` a `SendTime` es válida; no se generan ocurrencias con fecha local posterior.
- Execution: `PENDING → PROCESSING → COMPLETED | COMPLETED_WITH_ERRORS | FAILED`.
- Una Campaign recurrente obtiene audiencia incremental; no reenvía una compra ya procesada en esa Campaign.
- Compras elegibles por fecha cumplen `PurchaseDate + PostPurchaseOffset <= evaluationDate` (fecha de negocio; no el timestamp técnico de evaluación).
- Deduplicación: `campaignId + purchaseId/transactionId`.
- Historial final de Execution se conserva; no hay eliminación física de Campaign con historial.

### 2.3 Propuestas y OPENs

El claim transaccional/lease, la semántica ante caída a mitad de envío, el cálculo exacto de catch-up, timezone/DST, definición de éxito del cutoff, retry de destinatarios y límites de lotes requieren decisión técnica/operativa. La política de recurrencia al fin de mes está aprobada y se detalla en la sección 4.2. Este documento propone opciones seguras sin etiquetarlas como comportamiento implementado.

## 2.4 Estado de Sprint 2.5 y límites aprobados

Los estados y frecuencias permanecen en inglés dentro de datos, eventos y lógica de scheduling. Si se muestran en interfaz, se usan las etiquetas españolas centralizadas de FOUND-001. La localización no transforma el payload ni cambia fechas técnicas, calendario, enums o transiciones.

- **APPROVED:** Division, Department y Company son obligatorios para programar; filtros opcionales no bloquean DRAFT.
- **MOCK:** el proveedor temporal solo contiene catálogos sintéticos y relaciones de demostración. No representa compras, clientes ni contratos reales CF/Analytics.
- **DEFERRED:** no activar scheduler, resolver audiencia, crear recipients ni enviar correo cuando los catálogos MOCK sean el proveedor configurado. Este documento describe el diseño futuro y no autoriza su ejecución.
- **OPEN:** `Survey.status = 1` no está confirmado como `PUBLISHED`; ningún flujo de scheduling puede considerarse habilitado por ese valor sin evidencia del módulo Surveys.
- **OPEN:** la validación PostgreSQL del Sprint 2.4 sigue pendiente; ninguna sección de diseño constituye evidencia de migración/relaciones probadas en Neon.
- **DEFERRED/BLOCKED:** Schedule y Resume no se habilitan mientras falten el estado técnico de publicación, contratos de audiencia real y las demás precondiciones; el catálogo MOCK no puede autorizar audiencia ni ejecución.

## 3. Modelo temporal

### 3.1 Tres tiempos que no deben confundirse

| Concepto | Campo conceptual | Significado |
|---|---|---|
| Próxima ocurrencia de calendario | `Campaign.nextExecutionAt` | Instante planificado para el siguiente disparo. Se calcula desde la configuración de scheduling; no indica que se procesó toda compra elegible. |
| Última ejecución procesada | `Campaign.lastExecutionAt` | Instante operativo asociado a la última Execution concluida según semántica que apruebe producto. Su significado exacto está OPEN en DATA-001. |
| Instante técnico de evaluación | `CampaignExecution.eligibilityCutoffAt` (propuesto) | Timestamp técnico en que empieza la resolución de audiencia; se congela y conserva en snapshot. No es por sí mismo la fecha de negocio usada en la comparación. |
| Fecha de negocio de elegibilidad | `CampaignExecution.evaluationDate` (propuesto) | Fecha de negocio congelada que participa en `PurchaseDate + PostPurchaseOffset <= evaluationDate`. Zona, conversiones y regla exacta de fecha permanecen OPEN. |

`eligibilityCutoffAt` y `evaluationDate` se congelan juntos al inicio de la resolución y se conservan en el snapshot. Ninguno debe inferirse automáticamente de `nextExecutionAt` o `lastExecutionAt`; tampoco se recalculan entre páginas ni al reiniciar el worker. Persistir un cutoff exitoso global en Campaign sería una optimización propuesta y OPEN, no un watermark obligatorio ni sustituto del ledger `campaignId + purchaseId/transactionId`.

### 3.2 Instante de evaluación y zona horaria

Para una ejecución ordinaria, `eligibilityCutoffAt` es el instante técnico capturado al iniciar la resolución y `evaluationDate` es la fecha de negocio derivada/aprobada para aplicar la regla `PurchaseDate + PostPurchaseOffset <= evaluationDate`. Ambos valores efectivos se persisten antes de leer la primera página y se reutilizan tras reinicio. La programación usa el `timeZone` explícito y persistido de Campaign; para Ecuador, su default es `America/Guayaquil`. La conversión de cutoff/evaluationDate para elegibilidad de audiencia, otros mercados, DST y la regla exacta de corte permanecen **OPEN**.

En recuperación tras downtime, `scheduledAt` original y valores efectivos de evaluación pueden diferir. Se propone congelar el instante técnico y `evaluationDate` al iniciar la resolución de la recuperación, registrar el slot original y conservar ambos en el snapshot. Así se pueden considerar compras cuyo offset ya sea elegible en esa fecha de negocio consolidada. La conversión y semántica exactas deben confirmarse antes de implementación.

## 4. Cálculo de ocurrencias

### 4.1 `ONE_TIME`

1. `nextExecutionAt` corresponde a `SendDate + SendTime` interpretados en la zona horaria acordada.
2. Si `nextExecutionAt > now`, el tick no crea Execution.
3. Si está vencida y Campaign está `SCHEDULED` (o en estado elegible según transición aprobada), se genera una unidad de trabajo idempotente.
4. Tras resultado final, Campaign no vuelve a programarse; la siguiente ejecución es nula y la campaña pasa al estado final funcional aprobado. Baseline establece `COMPLETED` para el fin de una campaña, pero las reglas de estados para falla total/una vez requieren confirmar la transición.

Una Execution `FAILED` es terminal e inmutable y no cuenta como audiencia procesada exitosamente. Si quedan compras o trabajo pendientes de recuperación, Campaign no debe cerrarse como `COMPLETED` mientras la política de recuperación siga sin resolver. La transición de Campaign tras fallo total permanece **OPEN**, especialmente para `ONE_TIME`; un retry no reabre la Execution fallida.

### 4.2 `FREQUENCY`

La primera ocurrencia es `StartDate + SendTime`. Cada ocurrencia se calcula desde la fecha original `StartDate` y su índice de recurrencia, usando aritmética sobre fechas locales de calendario antes de convertirla a UTC:

| Frequency | Avance conceptual |
|---|---|
| `DAILY` | 1 día calendario |
| `WEEKLY` | 7 días calendario |
| `BIWEEKLY` | 14 días calendario |
| `MONTHLY` | 1 mes calendario |
| `QUARTERLY` | 3 meses calendario |

En `MONTHLY` y `QUARTERLY` se conserva el día de `StartDate` como ancla. Si el mes de destino no contiene ese día, la ocurrencia usa el último día disponible de ese mes; la normalización no reemplaza ni desplaza el ancla para ocurrencias posteriores. Aplica a años bisiestos. Ejemplo: `2026-01-31` mensual produce `2026-02-28`, `2026-03-31`, `2026-04-30`. No sustituir meses/trimestres por un número fijo de días.

La frecuencia permitida también valida la duración inclusiva entre Start y End conforme a FOUND-001: `durationDays = calendarDayDifference(endDate, startDate) + 1`. Ambas fechas cuentan como fechas de calendario, sin diferencias horarias, timestamps ni conversiones UTC. 1–6 días admite DAILY; 7–13 DAILY/WEEKLY; 14–29 DAILY/WEEKLY/BIWEEKLY; 30–89 añade MONTHLY; 90 o más admite todas, incluida QUARTERLY. La ocurrencia se interpreta con el `timeZone` persistido de Campaign.

### 4.3 `EndDate`

- `EndDate` limita ocurrencias de calendario y no filtra compras ni sus fechas.
- `EndDate` es una fecha local inclusiva de negocio evaluada en `Campaign.timeZone`.
- Una ocurrencia programada exactamente en `EndDate` a `SendTime` es válida.
- No se generan ocurrencias cuya fecha local sea posterior a `EndDate`.
- En operación ordinaria, la recurrencia termina cuando no existe una próxima ocurrencia dentro de la ventana; la transición a `COMPLETED` no debe ignorar trabajo de recuperación pendiente.
- **OPEN:** si una ejecución iniciada antes del límite termina después; qué hacer ante downtime o suspensión que atraviese EndDate; catch-up/reanudación tras expiración; conversión local a timestamp y DST. Estos OPENs no cambian la regla de no crear ocurrencias posteriores.

### 4.4 Avance seguro de `nextExecutionAt`

`nextExecutionAt` se calcula desde el slot programado, no desde la hora de finalización del envío. Nunca se deben crear varios slots y varias invitaciones de catch-up solo para alcanzar la fecha actual.

Regla propuesta:

1. Al reclamar el slot debido, conservar su instante como `scheduledAt`.
2. No fijar todavía `Campaign.nextExecutionAt` al futuro como si el trabajo hubiera sido exitoso, salvo que se use un campo separado de claim/cursor y la recuperación pueda probar pendientes.
3. Tras cierre `COMPLETED` o `COMPLETED_WITH_ERRORS` con ledger completo y política de retries aplicada, calcular la primera ocurrencia programada estrictamente posterior al slot procesado que no requiera una ejecución atrasada separada.
4. Si se detectaron slots omitidos por downtime, consolidarlos según estrategia de catch-up en una unidad de recuperación y saltar los slots intermedios, sin múltiples envíos atrasados.
5. Ante fallo total de extracción/envío, no marcar como procesadas compras elegibles pendientes ni avanzar cutoff exitoso. No cerrar Campaign recurrente como `COMPLETED` mientras haya trabajo pendiente de recuperación con política no resuelta. La Campaign puede mantener vencida la programación o pasar a tratamiento operativo según política OPEN.

**OPEN:** persistencia de `nextExecutionAt` mientras el slot está `PROCESSING`, clave de ocurrencia tras una recuperación consolidada y política de avance luego de fallo total/parcial.

## 5. Central Scheduler

### 5.1 Responsabilidades

El `CampaignScheduler` es un coordinador global con un tick configurable. Cada tick:

1. Lee una cantidad limitada de Campaigns vencidas, activas para ejecución y no finalizadas (`SCHEDULED` o `ACTIVE`, `nextExecutionAt <= now`).
2. Ordena candidatos de forma determinista por `nextExecutionAt`, luego ID estable.
3. Reclama cada candidato atómicamente en base de datos y crea/recupera un `CampaignExecution` con clave idempotente.
4. Encola o invoca un worker acotado sin sostener la transacción de claim.
5. Finaliza rápidamente para que el tick no tenga una duración proporcional a toda la audiencia.

No se crea una tarea `@Cron` por Campaign. La frecuencia de sondeo puede ser más corta que una recurrencia de negocio, pero no cambia el calendario de Campaign. Intervalo, `batchSize`, concurrencia máxima y mecanismo de despacho son **OPEN**.

### 5.2 Búsqueda de due Campaigns

Consulta conceptual:

```text
Campaign.status IN (SCHEDULED, ACTIVE)
AND Campaign.nextExecutionAt <= databaseNow
AND (no active claim OR claim expired and safely recoverable)
ORDER BY nextExecutionAt, campaignId
LIMIT configuredBatchSize
```

La lectura de candidatos no constituye un claim: varias instancias pueden observar la misma fila. Cada candidato debe pasar por operación atómica de claim/creación idempotente. La consulta debe usar el índice recomendado `(status, nextExecutionAt)` y limitarse para evitar cargar todas las Campaigns en memoria.

### 5.3 Cadencia y límites de trabajo

- `pollInterval`: configurable, global; no representa frecuencia de negocio.
- `campaignBatchSize`: máximo de campañas reclamadas por tick.
- `workerConcurrency`: límite de ejecuciones concurrentes por pod y/o sistema.
- `audienceBatchSize`: tamaño de página/lote que determine el contrato de CF/Analytics.
- `providerBatchSize`: límite que determine Emblue.

Los valores permanecen **OPEN** hasta medir volumen y obtener SLAs/cuotas externos. Evitar mantener la respuesta HTTP o el request de Angular abierto durante una ejecución completa.

## 6. Claim transaccional y multi-instance idempotency

### 6.1 Riesgo

`@nestjs/schedule` dispara el cron en cada instancia de la aplicación. Si hay más de un pod, cada uno puede leer la misma Campaign vencida. `replicaCount: 1` de Helm es solo un default del chart; HPA puede configurarse, y la cantidad efectiva desplegada no se verificó. Por ello el diseño no puede depender de singleton en memoria.

### 6.2 Unidad idempotente

Cada ocurrencia ordinaria usa una identidad determinista de slot derivada conceptualmente de `campaignId + scheduledAt/slot`, protegida por `UNIQUE(campaignId, executionKey)`. Un tick repetido del mismo slot reutiliza/encuentra el trabajo existente y no crea una nueva Execution. Una recuperación consolidada o retry que requiera nueva Execution utiliza una nueva `executionKey` y referencia explícita a la ejecución original mediante `recoveryOfExecutionId` o equivalente. Algoritmos definitivos de clave y recuperación permanecen OPEN.

### 6.3 Claim corto y compartido

Opciones a decidir en ADR:

- selección de filas con bloqueo transaccional (`FOR UPDATE SKIP LOCKED`) y creación de Execution en la misma transacción;
- transición condicional/compare-and-set sobre Campaign o una fila de lease más unique constraint;
- advisory lock PostgreSQL con protección durable adicional por unique key.

Requisitos mínimos de cualquiera de las opciones:

1. Una sola instancia gana un slot; perdedores continúan sin procesar esa Campaign.
2. La creación de `CampaignExecution(PENDING)` y claim lógico quedan atómicos o protegidos por unicidad; un slot ya registrado no origina otra Execution por cada tick.
3. El lease incluye owner técnico, `claimedAt` y vencimiento; permite recuperar una instancia muerta.
4. La renovación del lease ocurre mientras el worker progresa, sin bloquear otros slots.
5. El claim/lease no se considera prueba de envío ni de audiencia procesada.
6. Las transacciones son breves y finalizan antes de llamadas a fuente externa/Emblue.
7. El sistema protege también la unicidad de `CampaignRecipient` por Campaign + ID de compra.

No usar `case_locks` como substituto automático: ese lock es para edición de casos y su modelo/semántica no definen claims de ejecución de campañas.

## 7. Lifecycle de Campaign y Execution

### 7.1 Campaign

| Situación | Transición funcional |
|---|---|
| Programación válida antes de primer envío | `DRAFT → SCHEDULED`, guardar `nextExecutionAt`. |
| Tiene ejecución concluida y quedan slots dentro de EndDate | `SCHEDULED → ACTIVE`; continuar como ACTIVE. |
| No existe próxima ocurrencia dentro de EndDate o ONE_TIME finalizó según regla | `→ COMPLETED` solo si no hay trabajo pendiente de recuperación cuya política esté sin resolver. |
| Operador suspende | `SCHEDULED/ACTIVE → SUSPENDED`; scheduler no reclama slots nuevos. |
| Suspensión y reanudación | `SUSPENDED → SCHEDULED/ACTIVE` y recalcular próximo slot según regla aprobada. |
| Operador cancela | `→ CANCELLED`; no reclamar ocurrencias futuras. |
| Falla de una Execution | No cambia automáticamente Campaign a un estado no definido. Retención de `nextExecutionAt` y retry requieren política OPEN. |

Una suspensión/cancelación que ocurre mientras Execution ya está `PROCESSING` no cancela mágicamente la llamada externa en vuelo. La política de stop/drain/finish para trabajo en curso es OPEN en FOUND-001.

### 7.2 Execution

| Estado | Significado operativo |
|---|---|
| `PENDING` | Slot vencido y registrado, aún no comenzó el procesamiento. |
| `PROCESSING` | Worker obtuvo/renovó el claim y procesa snapshot, audiencia y lotes. |
| `COMPLETED` | Proceso concluyó sin errores según semántica de resultado aprobada. |
| `COMPLETED_WITH_ERRORS` | Procesamiento concluyó y conserva resultados individuales, con fallos parciales. |
| `FAILED` | Falla total que impide completar la Execution; detalle sanitizado. Es terminal, inmutable y no representa procesamiento exitoso de la audiencia. |

Transición normal: `PENDING → PROCESSING → terminal`. Los estados terminales no se regresan a PROCESSING ni se sobrescriben. Una recuperación de worker con Execution aún `PROCESSING` puede reasumir el claim vencido sobre esa misma Execution si el progreso es durable y continuar es seguro. Una Execution terminal `FAILED` no se reabre. Si se requiere una nueva Execution para retry posterior, lleva nueva `executionKey` y relación explícita `recoveryOfExecutionId` o equivalente, sin mutar la anterior. La política exacta de retry permanece OPEN.

### 7.3 Creación idempotente

Al vencer una ocurrencia, el ganador de claim crea `CampaignExecution` en `PENDING`, snapshot de configuración, valores de evaluación cuando comience la resolución y clave única del slot. La transición a `PROCESSING` se realiza al iniciar worker. Un tick que vuelve a ver el mismo `nextExecutionAt` encuentra la Execution ya asociada al mismo `executionKey`; no registra otra. Si existe una Execution terminal FAILED, no se reabre ni se reemplaza silenciosamente bajo el mismo key.

## 8. Audience incremental, offset y cutoff

### 8.1 Regla obligatoria por ejecución

Cada ejecución procesa únicamente compras que cumplen **todas** estas condiciones:

1. Coinciden con la segmentación efectiva congelada en el snapshot de esa Execution:
   - Division y Department obligatorios;
   - Auxiliary Classification opcional;
   - Company obligatoria;
   - Format y Store opcionales;
   - OR dentro del mismo grupo de multiselect y AND entre grupos.
2. Son elegibles por fecha de negocio congelada para la Execution: `PurchaseDate + PostPurchaseOffset <= evaluationDate`. El timestamp técnico `eligibilityCutoffAt` se registra por separado y no sustituye a `evaluationDate`.
3. No han sido procesadas previamente por esa Campaign, deduplicando con clave lógica `campaignId + purchaseId/transactionId`.
4. Tienen los datos requeridos para intentar envío según políticas de calidad/privacidad acordadas; los inválidos se contabilizan conforme regla OPEN y no se envían.

`PostPurchaseOffset` es entero obligatorio >= 0 expresado en días. No hay filtro manual por rango de fechas de compra. El backend calcula la audiencia; Angular no recibe el universo de destinatarios.

### 8.2 Incrementalidad

En cada ocurrencia, la fuente devuelve compras que cumplen filtros y offset en ese momento. PEC consulta contra su ledger y no reenvía compras ya registradas para la Campaign. Una misma persona puede recibir otra invitación si existe otra compra elegible distinta y el negocio no define una regla adicional de frecuencia por cliente.

La ventana incremental no se calcula como `PurchaseDate > lastExecutionAt`: eso perdería compras antiguas que cumplen offset o compras que llegaron tarde a la fuente. El ledger por purchase ID es la protección de duplicados; cutoff gobierna elegibilidad temporal, no es sustituto de deduplicación.

### 8.3 Registro antes del envío

Para evitar dos workers que procesen la misma compra:

1. Recuperar una página de resultados fuente.
2. Validar/normalizar identidad de compra sin loggear email.
3. Intentar reservar/crear `CampaignRecipient` con uniqueness Campaign + purchase usando reserva temporal.
4. Si conflicto: marcar conteo `duplicateCount`/resultado `SKIPPED_DUPLICATE` asociado al outcome apropiado, sin modificar el Recipient original ni enviar otra solicitud por ese duplicado.
5. Si obtiene reserva, inspeccionar ledger y `CampaignRecipientAttempt`; registrar/retomar outcome pendiente y enviar solo si no existe envío confirmado ni solicitud ambigua sin reconciliar.
6. Persistir aceptación técnica, resultado operativo y referencia disponibles por separado antes de avanzar al siguiente lote; no equiparar aceptación con entrega.

La operación de reserva y el dispatch externo nunca forman una transacción distribuida. Resultado ambiguo por timeout requiere reconciliación o idempotencia externa; no repetir ciegamente.

### 8.4 Actualización segura del cutoff

- Congelar `eligibilityCutoffAt` (instante técnico), `evaluationDate` (fecha de negocio) y criterios en snapshot al inicio de la resolución para que todas las páginas/lotes usen los mismos valores.
- No recalcular ninguno de esos valores entre páginas ni al reiniciar el worker.
- No marcar exitoso ningún cutoff al crear la Execution ni al emitir la primera página.
- Confirmar completitud de lectura y persistencia de recipients/outcomes antes de considerar exitoso el procesamiento de una Execution.
- Un fallo total de fuente o despacho no marca compras pendientes como procesadas ni avanza un cutoff exitoso. La unicidad del ledger evita crear otro Recipient, pero una reserva no equivale a envío; la recuperación debe revisar Attempts antes de continuar.
- Cualquier cutoff exitoso global de Campaign es una propuesta **OPEN**, no un watermark obligatorio, y nunca sustituye la deduplicación por compra.
- Si la ejecución se completa con errores, conservar cutoff de la ejecución pero mantener fallidos en flujo de retry separado; no reinsertarlos como nuevos recipients de la siguiente ejecución ordinaria.

**OPEN:** si se requiere persistir un cutoff exitoso global de Campaign aparte de `lastExecutionAt`; criterio de completitud ante fuente parcialmente paginada; corrección tardía de compras; transición de Recipient FAILED y criterio de éxito tras `COMPLETED_WITH_ERRORS`. No se establece watermark/lookback obligatorio.

## 9. Catch-up y downtime recovery

### 9.1 Objetivo

Tras una caída, recuperar compras que cumplen los filtros y el offset sin crear una Execution por cada tick/slot perdido ni lanzar una ráfaga de envíos atrasados innecesarios.

### 9.2 Estrategia propuesta: recuperación consolidada

Cuando `nextExecutionAt` quedó atrás en varias ocurrencias:

1. Crear/reclamar una sola unidad de recuperación por Campaign para el backlog vencido.
2. Conservar en snapshot la ocurrencia más antigua pendiente o el rango consolidado de slots (regla exacta OPEN), `recovery=true`, hora real de inicio, `eligibilityCutoffAt` y `evaluationDate` congelados.
3. Resolver una sola audiencia incremental usando `PurchaseDate + Offset <= evaluationDate` y excluir compras presentes en el ledger.
4. No reenviar las mismas compras por cada slot perdido.
5. Al completar según política acordada, avanzar `nextExecutionAt` al siguiente slot futuro de calendario, saltando slots ya absorbidos por recovery.
6. Si falla toda la recuperación, conservar compras pendientes sin marcarlas procesadas y no avanzar cutoff exitoso. No reabrir una Execution terminal `FAILED`; si un retry requiere nueva Execution, usar nueva `executionKey` y relación `recoveryOfExecutionId` o equivalente.

Así se consolida cantidad de ciclos, pero puede cambiar el número/calendario de invitaciones que habría ocurrido en operación normal. El negocio debe aprobar que una compra genere máximo una invitación por Campaign aunque haya slots omitidos; esto se alinea con dedupe Campaign+purchase, pero la política de recuperación como tal sigue OPEN.

### 9.3 No catch-up ilimitado

No ejecutar un bucle que envíe una campaña atrasada repetidamente hasta que `nextExecutionAt > now`. No materializar todas las ocurrencias vencidas como mensajes separados sin decisión de negocio. Si existe ventana máxima de recuperación o backlog demasiado antiguo, pausar/alertar y requerir intervención en vez de crear una ráfaga.

**OPEN:** duración máxima del downtime recuperable, retención de datos fuente, regla para campañas cuyo EndDate ya expiró mientras PEC estaba caído, y si se omite o consolida ejecución al reanudar campaña suspendida.

## 10. Fallos, estado final y retry separation

### 10.1 Clasificación de resultado

- `COMPLETED`: audiencia y lotes terminan; no hay errores por recipient de acuerdo con la respuesta comprobable del proveedor.
- `COMPLETED_WITH_ERRORS`: parte del trabajo terminó correctamente y parte falló; se conservan outcomes por recipient y conteos.
- `FAILED`: fallo total o condición que imposibilita completar la unidad de trabajo. Es terminal, inmutable, no acredita audiencia procesada exitosamente y la causa queda sanitizada.

El significado de éxito (`Emblue accepted` frente a email delivered) depende del contrato Emblue y está OPEN. Una Execution `FAILED` no marca como procesadas las compras pendientes ni avanza cutoff exitoso. Webhooks no deben cambiar retroactivamente una Execution terminal sin una política explícita de reconciliación. Una Campaign recurrente no se cierra `COMPLETED` mientras existan trabajos pendientes de recuperación cuya política no esté resuelta.

### 10.2 Reintentos separados

El retry de fallidos no debe ser una nueva ejecución periódica que vuelva a enviar todos los recipients. Propuesta:

- mantener `CampaignRecipient` y su clave por Campaign + purchase;
- registrar cada retry como `CampaignRecipientAttempt` o entidad/evento histórico vinculada al recipient;
- reintentar solo recipients cuyo último resultado lo permita y para los que exista resultado no ambiguo o reconciliación segura;
- una reserva vencida solo permite recuperar ownership; antes de reenviar, revisar ledger, `CampaignRecipientAttempt` y resultado conocido del proveedor. No reenviar envíos confirmados; los resultados ambiguos requieren reconciliación o idempotencia externa confirmada;
- reutilizar clave interna Campaign + purchase y, si Emblue confirma soporte, su idempotency key;
- aplicar backoff acotado, máximo de intentos y ventana acordados;
- preservar `CampaignExecution` final; una nueva Execution de retry usa nueva `executionKey` y relación `recoveryOfExecutionId` o equivalente conforme DATA/API.

DATA-001 propone tabla de intentos, pero transición tras `FAILED`, permisos de retry y efecto en estado agregado permanecen OPEN. No crear un estado `RETRYING` sin aprobación del enum funcional.

## 11. Batch processing y límites de concurrencia

### 11.1 Por Campaign

El worker consume CF/Analytics paginando; no carga una audiencia completa en memoria ni en Angular. Para cada página registra progreso durable suficiente para retomar. `pageSize`/cursor se define al conocer el contrato fuente.

### 11.2 Hacia Emblue

Agrupar mensajes solo dentro del límite real de Emblue. Si la respuesta es por elemento, actualizar recipient/attempt individualmente. Si es únicamente por lote, la clasificación de fallos parciales depende del contrato. No declarar `SENT` por éxito de un lote si no se puede determinar qué elementos fueron aceptados según la regla aprobada.

### 11.3 Control de carga

- límite de Campaigns en proceso por réplica y global;
- límite de requests de lectura concurrentes a CF/Analytics;
- límite de lotes de envío concurrentes a Emblue;
- pausa/circuito ante rate limits o fallas sostenidas;
- fairness para que una audiencia muy grande no monopolice el worker;
- backpressure cuando el origen o proveedor no puede consumir al ritmo de PEC.

Valores, orden de prioridades, concurrencia máxima, mecanismo de cola y manejo de cuotas son OPEN hasta conocer audiencia máxima, ventana de envío y límites de proveedor.

## 12. Concurrencia con acciones administrativas

- La edición de Campaign activa requiere suspender primero conforme FOUND-001.
- El worker utiliza snapshot inmutable: una edición posterior no cambia filtros ni calendario de la ejecución en curso.
- Si operador suspende/cancela después del claim, el worker verifica el estado en puntos seguros antes de iniciar nuevos batches; el comportamiento de los batches ya enviados/en vuelo está OPEN.
- La operación Resume debe recalcular próximo slot sin duplicar el `executionKey` de un slot ya reclamado.
- Control optimista de versión/actualización condicional para Campaign y suspensión/claim concurrentes se recomienda; mecanismo concreto OPEN.
- Una Campaign que pasa a COMPLETED/CANCELLED nunca debe volver al conjunto de candidatos por un `nextExecutionAt` residual.

## 13. Pseudocódigo técnico ilustrativo

### 13.1 Tick global y claim

```text
onGlobalSchedulerTick(now):
    candidates = findDueCampaigns(
        status in [SCHEDULED, ACTIVE],
        nextExecutionAt <= now,
        order by nextExecutionAt, campaignId,
        take = configuredBatchSize
    )

    for candidate in candidates:
        claim = database.transaction:
            lock/recheck candidate state and due slot
            if another unexpired claim owns candidate:
                return NOT_CLAIMED

            slotKey = deterministicKey(candidate.id, candidate.nextExecutionAt)
            execution = insert PENDING with unique(campaignId, slotKey)
                or load existing execution for same slot
            if existing execution is PROCESSING with expired lease and safe durable progress:
                recover that same execution
            if existing execution is terminal:
                do not reopen or create another execution for this slot on this tick
            persist short lease(owner, claimedAt, expiresAt)
            return execution and lease

        if claim acquired:
            dispatch bounded worker(execution.id)
```

El método concreto para bloquear filas y ejecutar `insert-if-absent` debe seleccionarse con el motor/version PostgreSQL disponible. El pseudocódigo no afirma que Prisma tenga un helper específico para `SKIP LOCKED`.

### 13.2 Worker por Execution

```text
processExecution(executionId):
    execution = load nonterminal execution and immutable campaign snapshot
    acquireOrRenewLease(execution)
    transition PENDING -> PROCESSING once
    if execution has no frozen evaluation values:
        eligibilityCutoffAt = capture technical evaluation timestamp once
        evaluationDate = derive business evaluation date under approved rules
        persist both values in immutable snapshot before source query
    else:
        reuse stored eligibilityCutoffAt and evaluationDate

    sourceCursor = resume durable cursor, if supported
    while source has next bounded page:
        assert lease is valid
        page = source.read(snapshot.filters, evaluationDate, snapshot.offset, sourceCursor)

        for purchase in page:
            if not matches segmentation: continue
            if purchase.purchaseDate + offset > evaluationDate: continue
            if missing/invalid required identity: record sanitized exclusion outcome; continue

            recipient, reservation = reserveOrLoad unique(campaignId, purchaseId/transactionId)
            if reservation is owned by another live worker:
                continue
            if ledger/attempts show purchase already processed or send confirmed:
                record duplicate count/outcome
                continue
            if reservation expired:
                take ownership and inspect attempts/provider result before continuing
            if provider result is ambiguous:
                reconcile or use confirmed external idempotency; do not resend blindly

            persist reservation and PENDING attempt before external send
            reservation is not proof of successful send or processed purchase
            add recipient to bounded provider batch

        persist source cursor and recipient reservations
        send batch through provider outside database transaction
        persist technical acceptance, operational recipient outcomes and external references distinctly
        never resend confirmed sends; ambiguous outcomes require confirmed reconciliation/idempotency
        stop/continue only according to transient/permanent/ambiguous classification

    if all work completed without recipient errors:
        finalize execution COMPLETED
    else if work completed with per-recipient failures:
        finalize execution COMPLETED_WITH_ERRORS
    else if work cannot complete:
        finalize execution FAILED with sanitized summary

    in short database transaction:
        persist final counts and lastExecutionAt semantics
        advance scheduling only under approved rule; FAILED never advances successful cutoff
        keep Campaign non-COMPLETED while unresolved pending recovery work exists
        set nextExecutionAt or Campaign COMPLETED if next slot exceeds EndDate and no unresolved recovery remains
        release lease
```

No external HTTP call debe ocurrir dentro de los bloques de transacción descritos. La respuesta incierta del proveedor queda en estado reconciliable y no se trata automáticamente como fallo reintentable.

### 13.3 Recovery de lease vencido

```text
recoverStaleWork(now):
    stale = find PROCESSING executions with expired lease (bounded)
    for execution in stale:
        transaction:
            atomically claim expired lease for this same execution
            keep immutable snapshot, eligibilityCutoffAt, evaluationDate, cursor and recipient keys
        for each reserved recipient:
            inspect CampaignRecipient ledger, CampaignRecipientAttempt history and known provider result
            expired reservation grants ownership recovery only; it does not authorize resend
            never resend a confirmed send
            reconcile ambiguous result or use only confirmed external idempotency before any resend
        resume only unfinished work whose outcome is known and not already confirmed
    if a later retry requires a new execution:
        create a new executionKey and link recoveryOfExecutionId to the original
```

Si no es seguro continuar, la ejecución se finaliza `FAILED` con razón sanitizada y se requiere retry explícito según política. No se resetea una Execution terminal.

## 14. Diagramas de secuencia textuales

### 14.1 Sondeo, claim y envío

```text
Nest Scheduler (cada intervalo global)
  -> CampaignScheduler: tick(now)
  -> PostgreSQL: listar due Campaigns en páginas pequeñas
  -> PostgreSQL: transacción breve, revalidar slot + claim + crear Execution única PENDING
  <- PostgreSQL: claim ganado / ya reclamado
  -> CampaignExecutionWorker: procesar Execution
  -> PostgreSQL: marcar PROCESSING y congelar eligibilityCutoffAt + evaluationDate en snapshot
  -> CF/Analytics: leer página según filtros + offset + evaluationDate
  <- CF/Analytics: purchases candidatas
  -> PostgreSQL: reservar ledger único Campaign + Purchase; revisar Attempts antes de continuar
  -> Emblue: solicitud transaccional individual o agrupada solo si el contrato lo permite
  <- Emblue: aceptación técnica/resultado operativo según contrato OPEN
  -> PostgreSQL: guardar aceptación, outcome del Attempt y eventos de entrega por separado
  -> Worker: no repetir envíos confirmados; reconciliar resultados ambiguos antes de cualquier resend
  -> PostgreSQL: finalizar Execution y actualizar estado/calendario con regla aprobada
```

### 14.2 Dos réplicas compiten por el mismo slot

```text
Replica A                      PostgreSQL                       Replica B
   |-- lee Campaign due --------->|<-------- lee Campaign due --|
   |-- intenta claim en tx ------>|<------ intenta claim en tx --|
   |                              | serializa/unique constraint  |
   |<-- claim ganado -------------|------------ claim perdido -->|
   |-- crea/usa única Execution   |               no envía       |
   |-- procesa fuera de lock      |                               |
```

El aislamiento efectivo debe probarse bajo la configuración de producción. La lectura duplicada es admisible; la doble ejecución y doble recipient no.

### 14.3 Caída y catch-up consolidado

```text
Downtime: pasan varios slots de FREQUENCY sin worker
Al volver:
  Scheduler -> detecta Campaign due y slots omitidos
  -> crea/reclama una sola Execution recovery (no una por slot atrasado)
  -> congela eligibilityCutoffAt técnico y evaluationDate de negocio
  -> consulta compras elegibles contra evaluationDate y filtra ledger Campaign + purchase
  -> recupera reservas inspeccionando Attempts/resultados del proveedor; guarda progreso durable
  -> si termina según política: avanza nextExecutionAt a siguiente slot futuro
  -> si falla totalmente: FAILED queda terminal, no procesa compras pendientes ni avanza cutoff exitoso
  -> Campaign no queda COMPLETED mientras haya recuperación pendiente sin política resuelta
  -> no crea una ejecución por tick/slot perdido ni una ráfaga de envíos
```

La consolidación no define si se usa el primer slot omitido, el último o un rango en la idempotency key: OPEN.

## 15. Observabilidad, métricas y logs

### 15.1 Métricas operativas sugeridas

- `campaign_scheduler_tick_total`, duración del tick y candidatos examinados/reclamados/omitidos;
- ejecuciones por estado, edad de la Execution `PROCESSING` más antigua y leases vencidos;
- tiempo de resolución de audiencia, páginas leídas, compras elegibles, duplicadas y excluidas agregadas;
- recipients pendientes, aceptados/rechazados por proveedor cuando comprobables y fallidos;
- latencia/error rate de CF/Analytics y Emblue, retries/ambiguous outcomes y backlog de recuperación;
- diferencia entre `now` y `nextExecutionAt` para medir retraso operativo;
- duración desde `PENDING` a `PROCESSING` y tiempo total de Execution.

No son métricas aprobadas de producto ni garantizadas por el stack actual. Stack de métricas, nombres/exportadores, dashboards, alertas y umbrales quedan OPEN.

### 15.2 Logs sanitizados

Registrar `campaignId`, `executionId`, `executionKey` opaca/no sensible, `requestId`/correlation ID, pod/versión, etapa, duración, status, conteos, código normalizado y número de intento. No registrar email, nombre, payload fuente/proveedor, token de atribución, token/API key, URL completa con correlación ni `purchaseId`/`transactionId` en texto de log.

El checkout cuenta con `requestDiagnostics` y sanitización de logger para valores sensibles; son apoyo, no autorización para volcar payloads completos. Los jobs deben emitir su propio contexto técnico aunque no nazcan de una request HTTP. El identificador de correlación de job no debe reutilizar ni exponer secretos.

Alertar por executions atascadas, lease expirado recurrente, crecimiento de backlog, errores sostenidos de fuente/proveedor, recuperación cercana a EndDate y resultados ambiguos que requieran conciliación. Retención y umbrales OPEN.

## 16. Riesgos y mitigaciones

| Riesgo | Mitigación propuesta |
|---|---|
| Cada pod ejecuta el cron | Claim durable compartido, unique execution key y unique Campaign + purchase. |
| Worker muere después de llamar Emblue, antes de guardar outcome | La reserva expirada solo permite recuperar ownership; inspeccionar Attempt y resultado del proveedor. Nunca repetir envío confirmado; resultado ambiguo requiere reconciliación o idempotencia externa confirmada. |
| Fuente cambia entre páginas | Snapshot de filtros, `eligibilityCutoffAt` y `evaluationDate`, cursor consistente si contrato lo ofrece; de lo contrario documentar limitación y regla de relectura. |
| Downtime crea avalancha de campañas o envíos | Batch global, límites de concurrencia, una recuperación consolidada por Campaign y backpressure. |
| Retry usa nueva identidad y duplica compra | Reutilizar Recipient por Campaign + purchase; guardar nuevo Attempt. |
| nextExecutionAt avanza aunque fuente falló totalmente | No marcar compras pendientes como procesadas ni avanzar un cutoff exitoso. Un cutoff global de Campaign es opcional y OPEN; no sustituye deduplicación por compra. |
| Campaign editada mientras corre worker | Snapshot inmutable y suspensión obligatoria antes de editar ACTIVE. |
| EndDate pasó durante downtime | Respetar ocurrencias válidas hasta el límite local inclusivo si esa representación se confirma; timezone, conversión y recuperación tras expiración OPEN; no fabricar múltiples ejecuciones. |
| Métricas/logs revelan PII | Agregación, sanitización, allowlist de campos y acceso/retención limitados. |

## 17. OPENs

| ID | Pendiente |
|---|---|
| OPEN-SCHED-01 | Mecanismo de claim: row lock/`SKIP LOCKED`, conditional update/lease o advisory lock; validar Postgres/Prisma y concurrencia. |
| OPEN-SCHED-02 | Duración/renovación del lease, detección de stale work y propietario del worker. |
| OPEN-SCHED-03 | Intervalo global, batch size, límites por pod/global, workers/cola y fairness. |
| OPEN-SCHED-04 | Default Ecuador `America/Guayaquil`, persistencia explícita en Campaign y no inferencia del navegador están aprobados. Siguen OPEN otras zonas de negocio, DST, horas inexistentes/repetidas y conversión de fechas locales a instantes. |
| OPEN-SCHED-05 | **Cerrado en Sprint 2.8.1:** `MONTHLY`/`QUARTERLY` conservan el día de `StartDate`; cuando no existe, usan el último día del mes sin cambiar el ancla. Las ocurrencias se calculan desde StartDate y el índice, incluidos años bisiestos. |
| OPEN-SCHED-06 | Regla base de EndDate cerrada: fecha local inclusiva; el slot en EndDate a SendTime es válido y no hay slots posteriores. Siguen OPEN ejecución en curso que cruza el límite, downtime/suspensión/recovery al expirar, finalización al reanudar, timezone de otros mercados y conversión/DST. |
| OPEN-SCHED-07 | Algoritmo definitivo de `executionKey` para slot ordinario, catch-up consolidado y retry; la unicidad por Campaign + key es requerida, retries nuevos enlazan la ejecución original. |
| OPEN-SCHED-08 | Si se adopta cutoff exitoso global (propuesta opcional), su persistencia/criterio ante fuente parcial, `FAILED` y `COMPLETED_WITH_ERRORS`; nunca reemplaza ledger/deduplicación Campaign + Purchase. |
| OPEN-SCHED-09 | Catch-up: primer/último slot, ventana máxima, reanudación de suspendida y qué hacer al expirar EndDate. |
| OPEN-SCHED-10 | Estado de Campaign tras fallo total y comportamiento de próximo slot, pausa y retry. |
| OPEN-SCHED-11 | Lifecycle de `CampaignRecipient`/Attempt, expiración de reservas, retry de FAILED y reconciliación de resultado ambiguo Emblue; una reserva expirada no autoriza resend. |
| OPEN-SCHED-12 | Control de pausa/cancelación mientras batches están en vuelo. |
| OPEN-SCHED-13 | Paginación estable, consistencia y límite de lectura de CF/Analytics. |
| OPEN-SCHED-14 | Tamaño de lotes, límites, idempotencia, latencia y estado por destinatario de Emblue. |
| OPEN-SCHED-15 | Significado de `lastExecutionAt` y si avanza en recuperación, partial errors o solo resultado final. |
| OPEN-SCHED-16 | Métricas, stack, alertas, SLOs, retención y acceso a logs/eventos. |

## 18. Criterios de cierre del diseño

Antes de implementar el scheduler productivo:

- seleccionar y probar un claim compartido que impida doble ejecución con múltiples réplicas;
- persistir `Campaign.timeZone` antes de programar (default Ecuador `America/Guayaquil`) y evaluar EndDate como fecha local inclusiva; definir y probar la conversión a instantes, otros mercados, DST y casos OPEN de recovery/reanudación;
- congelar `eligibilityCutoffAt` técnico y `evaluationDate` de negocio en el snapshot al iniciar audiencia y reutilizarlos tras reinicios;
- garantizar que `FAILED` sea terminal, no procese compras pendientes ni avance cutoff exitoso, y no cierre Campaign con recuperación pendiente sin política resuelta;
- garantizar `UNIQUE(campaignId, executionKey)`, identidad determinista del slot y que ticks repetidos no creen nuevas ejecuciones; recovery nuevo enlazado con nueva key cuando corresponda;
- recuperar reservas inspeccionando ledger, Attempts y resultados del proveedor; no repetir envíos confirmados ni reintentar resultados ambiguos sin reconciliación/idempotencia confirmada;
- documentar contratos CF/Analytics y Emblue que soportan paginación, límites, idempotencia y resultados ambiguos;
- definir recuperación y retry sin mutar estados históricos ni duplicar purchases;
- definir límites de carga, métricas, alertas y reglas de sanitización/retención;
- comprobar que una caída simulada entre reserva, dispatch y persistencia no produce reenvíos ciegos.

La existencia de `@nestjs/schedule`, transacciones Prisma en otros flujos o un `replicaCount` default de uno no satisface por sí sola estos criterios.

## 19. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Aclaración de EndDate inclusivo y calendario; inmutabilidad de FAILED y recuperación pendiente; claves idempotentes y relación de retries; separación de `eligibilityCutoffAt`/`evaluationDate`; recuperación segura de reservas y resultados ambiguos; cutoff global opcional, no watermark obligatorio. |
| 0.3 | 2026-10-20 | Se aclara que diseño de scheduler queda DEFERRED con proveedor MOCK; catálogos sintéticos no crean audiencia ni recipients; `PUBLISHED` y validación PostgreSQL permanecen OPEN. El documento permanece DRAFT. |
| 0.4 | 2026-10-20 | Se agrega la regla de presentación en español de estados y modalidades; valores técnicos en inglés. El documento permanece DRAFT. |
| 0.4 | 2026-10-21 | Se cierra la duración inclusiva de recurrencias para validar la frecuencia permitida; el conteo usa fechas de calendario y no resuelve timezone, DST ni el límite de ocurrencia EndDate. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se aprueba timezone explícito por Campaign, default `America/Guayaquil` para Ecuador y EndDate local inclusiva, válida a SendTime y sin ocurrencias posteriores. Recovery, downtime, finalización al reanudar, DST y conversión permanecen OPEN; Schedule/Resume bloqueados por publicación y audiencia. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se aprueba la regla MONTHLY/QUARTERLY de anclaje a StartDate y ajuste al último día del mes sin desplazamiento acumulativo, también en años bisiestos; la operación se realiza sobre calendario local antes de UTC. Estado DRAFT conservado. |
