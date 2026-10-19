# Especificación Funcional - Automated Survey Campaigns

| Campo | Valor |
|---|---|
| ID | PEC-SC-FOUND-001 |
| Estado | DRAFT |
| Versión | 0.2 |
| Fecha | 2026-10-18 |
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

**OPEN:** nombres de roles y permisos PEC concretos, acceso a la consulta de destinatarios y permisos para reintentos operativos.

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
- **RB-006 (OPEN):** La representación actual de `PUBLISHED` en PEC y su relación con vigencia/disponibilidad deben confirmarse antes de establecer el mensaje/validación exactos.

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
- **RB-040:** La siguiente ejecución se calcula por calendario: `DAILY` +1 día, `WEEKLY` +7 días, `BIWEEKLY` +14 días, `MONTHLY` +1 mes calendario y `QUARTERLY` +3 meses calendario.
- **RB-041:** `ONE_TIME` queda `COMPLETED` después de su única ejecución, según el resultado terminal registrado.
- **RB-042:** `FREQUENCY` queda `COMPLETED` cuando la siguiente ejecución supera `End Date`.
- **RB-043:** El scheduler es centralizado; no se define un scheduler individual por Campaign.
- **RB-044 (OPEN):** Zona horaria definitiva, horario de verano, manejo de fin de mes y política ante hora ambigua/inexistente requieren definición.

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
- **RB-060 (OPEN):** La disposición exacta de una ejecución que ya está `PROCESSING` al suspender/cancelar aún requiere política funcional; no se presume cancelación inmediata.

### 9.8 Execution lifecycle y recuperación

- **RB-061:** Los estados de ejecución son `PENDING`, `PROCESSING`, `COMPLETED`, `COMPLETED_WITH_ERRORS` y `FAILED`.
- **RB-062:** Una ejecución sigue la transición `PENDING` → `PROCESSING` → estado final.
- **RB-063:** Los estados finales de Execution son históricos e inmutables.
- **RB-064:** Una caída del sistema no debe hacer perder las compras elegibles pendientes.
- **RB-065:** La recuperación procesa elegibles pendientes sin generar ejecuciones atrasadas innecesarias ni una ráfaga de correos por cada intervalo perdido.
- **RB-066:** Una falla total no debe marcar compras elegibles pendientes como procesadas ni provocar que futuras ejecuciones omitan compras que todavía deben considerarse.
- **RB-067:** Una ejecución parcial puede terminar en `COMPLETED_WITH_ERRORS` si conserva sus resultados por destinatario.
- **RB-068 (OPEN):** Política de reintentos, espera, límites, resultados ambiguos del proveedor y definición de “fallo total” se debe acordar con Emblue y operación.

### 9.9 Recipient lifecycle

- **RB-069:** Los estados de Recipient son `PENDING`, `SENT`, `FAILED` y `SKIPPED_DUPLICATE`.
- **RB-070:** No se utiliza un estado genérico `SKIPPED`.
- **RB-071:** El resultado debe permitir distinguir envío exitoso, fallo y duplicado sin reinterpretar el estado de entrega final de Emblue.
- **RB-072 (OPEN):** Confirmar si `SENT` significa aceptado por la API de Emblue o entregado al buzón; eventos y confirmaciones pueden afectar el seguimiento sin cambiar el significado acordado del estado.

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

## 10. Criterios de aceptación

### Casos positivos

- **AC-001:** Dada una Survey PEC existente no publicada, cuando un administrador guarda una Campaign, entonces se guarda en `DRAFT` sin obligar a publicar la Survey.
- **AC-002:** Dada una Campaign válida y una Survey `PUBLISHED`, cuando el administrador la programa, entonces queda `SCHEDULED` y se muestra su próxima ejecución.
- **AC-003:** Dada una configuración válida, cuando se solicita `Audience Preview`, entonces se muestran `Eligible Purchases` y `Eligible Customers` calculados con segmentación, offset y deduplicación existentes.
- **AC-004:** Dada una segmentación con varias selecciones, cuando se evalúa, entonces las selecciones de cada filtro se combinan con OR y los filtros/grupos distintos con AND.
- **AC-005:** Dada una Campaign con `Post-purchase Offset` cero, cuando se calcula elegibilidad, entonces no se añade espera posterior a la fecha de compra.
- **AC-006:** Dada una Campaign `ONE_TIME` con fecha y hora válidas, cuando llega el momento programado, entonces se registra una única ejecución y, al finalizar, la campaña pasa a `COMPLETED`.
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
- **AC-029:** Dado un mes sin el mismo número de día que la fecha de ejecución anterior, cuando aplica recurrencia mensual/trimestral, entonces el cálculo respeta una regla de calendario definida antes de producción. **OPEN:** cuál es la regla para fin de mes.
- **AC-030:** Dado un período de indisponibilidad con varias ejecuciones omitidas, cuando el scheduler se recupera, entonces procesa compras pendientes sin producir una ráfaga de envíos atrasados innecesarios.
- **AC-031:** Dado un fallo parcial de destinatarios, cuando se cierra la ejecución, entonces se conservan los resultados individuales y puede utilizarse `COMPLETED_WITH_ERRORS`.
- **AC-032:** Dada una falla de la fuente corporativa o de Emblue, cuando se procesa la Campaign, entonces queda un error trazable y se conserva el historial. **OPEN:** política exacta de reintento.
- **AC-033:** Dado que una Campaign se suspende mientras una ejecución ya está en curso, cuando se aplica la suspensión, entonces no se asume una cancelación retroactiva de los envíos ya iniciados. **OPEN:** política para el trabajo en curso.
- **AC-034:** Dado un destinatario sin correo válido, duplicado en la fuente o suprimido por una regla de contacto, cuando se prepara la audiencia, entonces se aplica una regla acordada y se contabiliza de forma auditable. **OPEN:** reglas de validez, supresión y conteo.
- **AC-035:** Dado un callback repetido o fuera de orden, cuando PEC lo recibe, entonces no altera incorrectamente la correlación/resultado. **OPEN:** depende del contrato y de identificadores de evento Emblue.

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

**OPEN:** transición exacta al reanudar y tratamiento de la próxima fecha vencida mientras estuvo suspendida; no inferir ejecuciones atrasadas adicionales.

### 11.2 Campaign Execution

| Estado actual | Evento | Estado siguiente |
|---|---|---|
| `PENDING` | Comienza procesamiento | `PROCESSING` |
| `PROCESSING` | Todos los resultados completan según el contrato | `COMPLETED` |
| `PROCESSING` | Hay resultados parciales con errores | `COMPLETED_WITH_ERRORS` |
| `PROCESSING` | Falla total | `FAILED` |

Los estados finales son históricos e inmutables. Una recuperación que requiera nuevo procesamiento debe ser trazable según la política de retries pendiente; no cambia retrospectivamente un estado final.

### 11.3 Campaign Recipient

| Estado | Significado funcional |
|---|---|
| `PENDING` | Recipient identificado como elegible, aún sin resultado de envío registrado. |
| `SENT` | Resultado de envío positivo según la semántica que confirme Emblue. |
| `FAILED` | Fallo de envío registrado. |
| `SKIPPED_DUPLICATE` | Compra duplicada para la misma Campaign; no se envía otra invitación. |

**OPEN:** transición desde `FAILED` durante retry y distinción entre aceptación de solicitud y entrega definitiva.

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
- La representación real del estado funcional `PUBLISHED` se mapeará con el módulo Survey antes de habilitar programación.
- La zona horaria y las reglas de calendario serán acordadas por negocio antes de producción.
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
| OPEN-04 | Zona horaria, corte de ejecución, horario de verano, límites de fecha y regla de fin de mes. |
| OPEN-05 | Política ante fecha de ejecución omitida, suspensión y reanudación de una recurrencia. |
| OPEN-06 | Contrato final de Emblue, autenticación, lote, límites, respuestas e idempotencia. |
| OPEN-07 | Eventos de DataLab/Webhooks, firma, orden, deduplicación, retries y semántica de entrega. |
| OPEN-08 | Política de retries, backoff, fallos ambiguos y definición de fallo total/parcial. |
| OPEN-09 | Mecanismo de Response Attribution, expiración y respuesta a enlaces reenviados. |
| OPEN-10 | Roles PEC y permisos para administración, preview, consulta de recipients y operaciones. |
| OPEN-11 | Reglas de correo inválido, destinatario duplicado, supresión/consentimiento y sus conteos. |
| OPEN-12 | Retención, acceso y eliminación de PII, recipients, compras procesadas y eventos. |
| OPEN-13 | Volumen máximo, ventana de envío, muestras de preview y límites operativos. |
| OPEN-14 | Representación de `PUBLISHED` y validación relacionada con disponibilidad de Survey. |

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
- El documento queda en estado **DRAFT**, versión **0.2**.

## 18. Estado del documento

- **Status:** DRAFT
- **Version:** 0.2
- **Base funcional:** baseline descrita en `PEC-SC-ARCH-001.md`.
- **Aprobación:** pendiente de revisión funcional y resolución de los OPENs aplicables.

## 19. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Se formalizan búsqueda por `Campaign Name` y `Survey Name`, filtro por `Status` y su cobertura en AC-015; se aclara el acceso de `CANCELLED` mediante filtro sin card propia; se reemplaza la regla de recuperación por una garantía neutral que evita marcar compras elegibles pendientes como procesadas u omitirlas en ejecuciones futuras. |
