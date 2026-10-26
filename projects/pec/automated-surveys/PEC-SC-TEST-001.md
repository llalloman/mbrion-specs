# PEC-SC-TEST-001 — Estrategia de Pruebas

## Automated Survey Campaigns

| Campo | Valor |
|---|---|
| Documento | PEC-SC-TEST-001 |
| Título | Estrategia de Pruebas - Automated Survey Campaigns |
| Estado | DRAFT |
| Versión | 0.5 |
| Fecha | 2026-10-20 |
| Idioma | Español; nombres técnicos de tests en inglés |

> Define los escenarios y evidencias de QA para Automated Survey Campaigns, alineados con ARCH-001, FOUND-001, DATA-001, API-001, INTEG-001 y SCHED-001. No crea pruebas ni convierte puntos **OPEN** de esos documentos en reglas aprobadas.

## 1. Objetivo

Establecer una estrategia verificable para las reglas de Campaign, segmentación, calendario, ejecución incremental, integraciones, seguridad y consulta administrativa. La estrategia cubre unit, integration, E2E, contract, security, concurrency y failure/recovery. Distingue pruebas deterministas con dependencias simuladas de validaciones que requieren PostgreSQL, infraestructura multi-instancia o contratos reales.

## 2. Documentos de referencia y límites

### 2.1 Baseline

- `PEC-SC-ARCH-001.md`: arquitectura propuesta, scheduler central, claim compartido, privacy, observabilidad y responsabilidades.
- `PEC-SC-FOUND-001.md`: reglas funcionales, criterios `RB-xxx`/`AC-xxx`, estados y audiencia.
- `PEC-SC-DATA-001.md`: entidades conceptuales, snapshots, unique keys, recipients, intentos e historial.
- `PEC-SC-API-001.md`: contratos internos Angular/API, validaciones, permisos, paginación y errores.
- `PEC-SC-INTEG-001.md`: fronteras conceptuales CF/Analytics, Emblue y DataLab/Webhooks.
- `PEC-SC-SCHED-001.md`: recurrencias, cutoff, claim, catch-up, recovery, idempotencia y métricas.
- `PEC-SURVEY-CAMPAIGNS-001.md`: alcance inicial y contexto histórico del módulo.

Si una regla de un documento posterior está marcada **OPEN**, los tests verifican que el sistema no actúe fuera de la política provisional aprobada; el resultado exacto no se fija hasta cerrar esa OPEN. No usar fixtures de prueba para ocultar una contradicción entre documentos: registrar la decisión y actualizar baseline antes de aprobar el test.

### 2.2 Hechos de repositorio verificados

- `pec-api` usa Jest/ts-jest con `*.spec.ts`; dispone de `npm test`, `npm run test:cov` y `npm run test:e2e`. E2E API usa Jest y Supertest desde `test/jest-e2e.json`.
- Los specs de servicios Nest suelen construir `TestingModule` e inyectar mocks. El spec de `SchedulerService` verifica paginación y el uso de `$transaction` con servicios/modelos simulados.
- Los E2E API existentes son limitados: `app.e2e-spec.ts` importa `AppModule` y reemplaza `AuthGuard`; `user.e2e-spec.ts` levanta HTTP con Supertest pero reemplaza `UserService` con memoria. No son evidencia de persistencia PostgreSQL real.
- `pec-ui` usa Jest con `jest-preset-angular` y `config.jest.ts`; conserva además un target Angular/Karma en `angular.json`.
- Ya existe `automated-campaigns.component.spec.ts`; cubre navegación entre lista/formulario, búsqueda + estado, paginación, vacíos y contadores calculados sobre datos mock del componente. No valida backend, persistencia, autorización ni ejecución.
- `pec-ui/package.json` incluye `test:integration` apuntando a `ng run gira-monitors-web:integration`, pero ese target/proyecto no está declarado en el `angular.json` inspeccionado. El script `e2e` llama `ng e2e`; README indica que debe agregarse un runner compatible, y no se encontró un target/runner configurado.
- La configuración actual no demuestra un entorno de integración dedicado con CF/Analytics o Emblue. Sus contratos siguen OPEN en INTEG-001.

Estos hallazgos describen el checkout revisado; se deberán volver a verificar cuando se implemente la funcionalidad o cambien scripts/configuración.

## 3. Clasificación de pruebas

| Tipo | Propósito para Campaigns | Dependencias y límite |
|---|---|---|
| **Unit** | Reglas puras de calendario, segmentación, transiciones, cutoff, deduplicación y mapeo de resultados. | Reloj fijo; mocks de repositorios y adapters; rápida y determinista. |
| **Integration** | Comprobar interacción entre servicios Nest, Prisma/PostgreSQL, transacciones, constraints, controllers/guards y adapters simulados. | Base de datos de QA aislada con migraciones; proveedores externos simulados. No usar datos productivos. |
| **E2E** | Validar recorrido UI → API → DB, permisos reales de prueba, dashboard/detalle y acciones administrativas. | Requiere runner/navegador que el proyecto todavía no tiene configurado para UI; especificar runner y datos de prueba antes de ejecución. |
| **Contract** | Comprobar compatibilidad de adapters PEC con OpenAPI/schema/fixtures aprobados de CF y Emblue, incluyendo webhook. | No se puede cerrar hasta recibir contratos reales. Fixtures conceptuales no equivalen a consumer/provider contract aprobado. |
| **Security** | Autorización, scope, IDOR, PII, secretos, webhook authenticity/replay y protección de correlation reference. | Revisión de seguridad y fixtures sintéticos; criptografía/claims definitivos OPEN. |
| **Concurrency** | Garantizar un claim y un envío lógico por Campaign/slot/compra con varias instancias y transacciones concurrentes. | PostgreSQL real compatible con despliegue; procesos concurrentes independientes. Unit mocks no son evidencia suficiente. |
| **Failure/Recovery** | Simular downtime, leases vencidos, fallos de fuente/proveedor, resultados ambiguos, parcialidad y reanudación. | DB aislada, clock controlado y adapters fault-injectable; pruebas sandbox de Emblue solo cuando haya ambiente/contrato. |

Una suite puede pertenecer a más de una clasificación; usar una categoría primaria y registrar las adicionales en el reporte para evitar duplicar el escenario lógico.

## 4. Niveles, ambientes y datos

### Unit

- Mantener servicios/reglas de dominio aislados de Nest cuando sea posible; para servicios Nest, seguir los patrones `TestingModule` y `jest.fn()` existentes.
- Inyectar reloj/control de tiempo en lugar de depender de `Date.now()` real.
- Probar matrices de estados y límites de fecha con casos parametrizados.
- No validar semántica del proveedor a partir de mocks inventados.

### Integration

- Ejecutar migraciones reales sobre una DB PostgreSQL exclusiva de tests para validar índices únicos, constraints, isolation y transacciones.
- Levantar `CampaignModule`/controller con servicios externos mockeados en nivel de adapter, no con DB simulada cuando el escenario trate claims o idempotencia.
- Sembrar y limpiar fixtures por test; preferir transacción/reset de schema con aislamiento entre workers.
- Incluir casos de validación/paginación HTTP y respuesta PEC, con Auth/CASL de prueba explícitos.

### E2E

- Recorrer `/surveys/automated-campaigns`: dashboard/lista → búsqueda/filtro → detalle → creación/preview → schedule o suspend/resume según estado → regreso/listado.
- Verificar UI y API juntos usando backend/DB QA y proveedores simulados deterministas.
- Usar fixtures autenticados para los roles efectivos `ADMIN`, `SUPERADMIN` y roles no autorizados; el runner E2E UI sigue OPEN.
- No depender de mocks de estado de frontend para afirmar que Campaign se guardó o que una invitación se envió.

### Contract y sandbox

- Versionar fixtures desde contratos aprobados sin secretos ni PII.
- Validar request, response, enums, compatibilidad, timeouts, rechazo, límites y eventos tal como estén documentados.
- Cuando exista sandbox no productivo, hacer una prueba controlada con recipients sintéticos y template de QA; no enviar campañas a clientes reales.
- Hasta recibir el contrato real, etiquetar los resultados como “validación del adapter contra fixture provisional”, no “Emblue compatible”.

### Datos de prueba

- Crear Campaign, Survey, usuarios/roles, catálogos, compras, correos sintéticos y resultados de Survey con identificadores reservados para test.
- Incluir fixtures de compra repetida, cliente con compras múltiples, fechas límite, registros incompletos y eventos repetidos/fuera de orden.
- Nunca copiar emails, tokens, IDs personales o payloads de producción. No registrar valores PII en reportes ni snapshots de test.
- Congelar zona horaria y reloj por caso; al cerrar OPEN de timezone, incluir DST y calendarios regionales.

## 5. Matriz de escenarios — Campaign CRUD, permisos y UI

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-CRUD-01 | Unit + Integration | Crear Campaign DRAFT referenciando Survey existente; campos editables válidos. | Persiste una Campaign separada de Survey; devuelve campos esperados sin que el cliente asigne estado/creador/counters reservados. | P0 | Unit assertion; request/response HTTP; lectura DB. |
| TST-CRUD-02 | Integration | Leer lista/detalle y paginar/filtar por estado con fixtures mixtos. | Filtra antes de paginar; orden estable y `total` consistente; scope aplicado. | P1 | Supertest response y consultas controladas. |
| TST-CRUD-03 | Unit + Integration | Editar DRAFT y SUSPENDED; intentar editar ACTIVE, COMPLETED y CANCELLED. | DRAFT/SUSPENDED válidos actualizan config; ACTIVE solo se edita después de suspender; finales son solo lectura. | P0 | Tabla de transición ejercitada; HTTP 409/response; no-mutación DB. |
| TST-CRUD-04 | Integration | Delete DRAFT sin historial y luego intentar delete con Execution/Recipient histórico. | Solo caso permitido conforme a decisión de soft-delete; historial nunca se borra físicamente. | P1 | Estado/row counts antes-después; regla soft delete resuelta. **OPEN** el mecanismo exacto. |
| TST-AUTH-01 | Security + E2E | Rol distinto a ADMIN/SUPERADMIN efectivo intenta Campaign, selector Survey, catalogos, preview/acciones/recipients. | Backend niega aunque Angular permita request; ADMIN/SUPERADMIN conserva acceso aprobado. | P0 | HTTP autenticado, 401/403 y evidencia CASL; grants de otros modulos sin cambio. |
| TST-AUTH-02 | Security + Integration | Manipular IDs de Campaign/Execution/Survey o IDs externos incompatibles. | Rechazo sin modificacion ni fuga; alcance multiempresa permitido solo para roles autorizados. | P0 | Requests negativos y evidencia de scope/guard. |
| TST-UI-01 | Unit + Integration + E2E | Buscar por `searchText` (Campaign Name), por `surveySearchText` (Survey Name) y por ambos a la vez; aplicar también `status`, paginación y scope. | Cada búsqueda filtra su nombre; cuando ambas están presentes se combinan con AND; status se combina con las búsquedas y el scope, antes de paginar. Incluye `CANCELLED`. | P0 | Jest Angular; requests/responses API; fixtures DB por scope y páginas; E2E cuando haya runner. |
| TST-UI-02 | Unit + Integration + E2E | Dashboard con Campaigns visibles y no eliminadas lógicamente de todos los estados, incluyendo `CANCELLED`. | Presenta exactamente seis cards: Total, Active, Scheduled, Draft, Suspended y Completed. Total incluye las CANCELLED visibles; CANCELLED aparece en `byStatus`/filtro, sin card propia. | P0 | Conteos API comparados con filas DB dentro del scope; prueba UI que verifica seis cards y ausencia de card CANCELLED. |
| TST-UI-03 | Unit + E2E | Campaign Detail con estados/datos mixtos. | Detalle coincide con API y muestra Scheduling, Segmentation, conteos e historial disponible sin inventar aperturas/clicks. | P1 | Fixture DB/API, comparación UI/API y screenshot/artifact. |

## 6. Matriz — Survey, segmentation, offset y Audience Preview

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-SURVEY-01 | Unit + Integration + E2E | Schedule con Survey existente `PUBLISHED`, accesible y elegible. | Campaign puede programarse si el resto de requisitos es válido. | P0 | Contrato Survey real, resultado del schedule y DB. |
| TST-SURVEY-02 | Unit + Integration | Survey inexistente, privada, inactiva, archivada o con ventana fuera de calendario al intentar Schedule. | Rechaza con error estable; no crea Execution ni cambia calendario. Guardar DRAFT sigue permitido. | P0 | Verificar campos y ventana Campaign. |
| TST-SEG-01 | Unit + Integration | Schedule sin Division, Department o Company, individualmente y en conjunto. | Rechaza; ninguna selección obligatoria se interpreta como “sin filtro”. | P0 | Tabla de validación y request/response. |
| TST-SEG-02 | Unit | Multiselect con varios valores en el mismo grupo. | Los valores de ese grupo se combinan OR; un optional vacío no restringe. | P0 | Casos parametrizados con expected purchase IDs sintéticos. |
| TST-SEG-03 | Unit + Integration | Combinar Division, Department, Company y opcionales. | Los grupos diferentes se combinan AND, manteniendo OR intra-grupo. | P0 | Dataset pequeño conocido y consulta/resultado explícito. |
| TST-SEG-04 | Unit + Integration + E2E | Cambio de selección padre que vuelve incompatible una selección hija. | UI limpia selección incompatible; API revalida jerarquía y no acepta combinación inválida. | P1 | Test componente/API y fixture jerárquico. **OPEN** dependencias fuente exactas. |
| TST-SEG-05 | Integration + Contract | Catálogo CF devuelve códigos desconocidos, página repetida, parcial o error. | No presenta fallo de catálogo como lista válida vacía; valida errores y consistencia según contrato. | P1 | Fixture contractual y logs/códigos sanitizados. Contrato CF **OPEN**. |
| TST-OFFSET-01 | Unit + Integration | Offset `0`, `1`, fecha exactamente al límite y fecha un día antes. | Elegible si y solo si segmentación válida y `PurchaseDate + PostPurchaseOffset <= evaluationDate`. `evaluationDate` es fecha de negocio; `eligibilityCutoffAt` es timestamp técnico distinto. No hay rango manual de compra. | P0 | Tabla de tiempo controlado con ambos valores visibles; asserts de igualdad al límite y no elegibilidad previa. Timezone/conversiones OPEN. |
| TST-OFFSET-02 | Unit + Integration | Offset negativo, fraccionario, nulo o ausente al schedule. | Validación rechaza; no crea Execution. | P0 | Error HTTP/DTO y no-mutación DB. |
| TST-PREV-01 | Unit + Integration + Security | Preview de Campaign nueva y existente; incluir purchases ya registradas para Campaign existente y fijar `evaluationDate` de negocio. | Backend evalúa `PurchaseDate + PostPurchaseOffset <= evaluationDate`, excluye compras procesadas y devuelve `Eligible Purchases` + `Eligible Customers`; muestra opcional acotada, sin audiencia completa ni PII/token. Preview no reserva destinatarios. `eligibilityCutoffAt` es el timestamp técnico de una ejecución y no sustituye `evaluationDate`. | P0 | Respuesta redacted, DB sin writes y fixture temporal. Para ejecución, snapshot congela `eligibilityCutoffAt` y `evaluationDate` al iniciar resolución; ambos permanecen iguales entre páginas/recovery. Timezone/conversiones OPEN. |
| TST-PREV-02 | Failure/Recovery + Integration | CF falla/timeout o responde parcialmente durante preview. | Reporta fallo de origen, no un preview vacío que parezca válido; no deja reservas/Execution. | P0 | Fault injection y comprobación de DB/log sanitizado. Contrato fuente OPEN. |

## 7. Matriz — calendario y next execution

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-CAL-01 | Unit + Integration | ONE_TIME con SendDate/SendTime futuros y vencidos; campos de FREQUENCY incompatibles; Campaign de Ecuador sin zona horaria explícita. | Valida campos exclusivos; `nextExecutionAt` es única fecha/hora calculada; no crea ejecución antes del vencimiento. Antes de programación efectiva persiste `America/Guayaquil` por defecto; no infiere la zona del navegador. | P0 | Clock fijo, casos límites, persistencia de `timeZone` y payload. |
| TST-CAL-02 | Unit + Integration | FREQUENCY con Start, End, SendTime y cada Frequency. | Primera fecha corresponde al slot definido; siguiente fecha usa aritmética de calendario y no duración fija para MONTHLY/QUARTERLY. | P0 | Tabla de casos y fechas esperadas contra regla aprobada. |
| TST-CAL-03 | Unit | Catálogo dinámico con duración inclusiva `calendarDayDifference(endDate, startDate) + 1`. | Solo ofrece y acepta las frecuencias permitidas por FOUND-001 en cada límite inclusivo. | P1 | Tests parametrizados para duraciones 1, 6, 7, 13, 14, 29, 30, 89 y 90; fechas reales e inválidas; cálculo sin timestamps ni UTC. |
| TST-CAL-04 | Unit + Integration | MONTHLY iniciado el 31 de enero y el 30 de enero, incluyendo febrero y el mes siguiente. | Ajusta a último día disponible y restaura el día original anclado después del mes corto; no hay desplazamiento acumulativo. | P0 | Asserts de secuencia `31 ene → 28 feb → 31 mar → 30 abr` y `30 ene → 28 feb → 30 mar`, con hora local/UTC. |
| TST-CAL-05 | Unit + Integration | QUARTERLY con mes destino corto y secuencia que cruza año bisiesto. | Avanza tres meses calendario desde StartDate y su índice; ajusta el día faltante sin cambiar el ancla ni sumar 90 días fijos. | P1 | Secuencias trimestrales, incluida `29 feb 2024 → 28 feb 2025`; comprobar hora local y UTC. |
| TST-CAL-06 | Unit + Integration | Validación de frecuencia frente a duración inclusiva; slot de recurrencia en EndDate a SendTime y siguiente slot posterior. | Rechaza frecuencia incompatible en operación efectiva; `DRAFT` incompleto o pendiente de corrección puede guardarse. El slot exactamente en EndDate a SendTime es válido y no se genera otro posterior. La ejecución real sigue bloqueada mientras falten precondiciones. | P0 | Tests unitarios de calendario/local date y validación API/UI; persistencia de `timeZone`; el scheduler real requiere habilitación posterior. |
| TST-CAL-07 | Unit + Integration | `nextExecutionAt` y `lastExecutionAt` entre éxito, partial error y fallo total. | Calendario avanza solo conforme regla aprobada; `lastExecutionAt` tiene semántica estable; fallo total no avanza cutoff exitoso. | P0 | Antes/después DB + eventos de transición; política final OPEN en DATA/SCHED. |
| TST-CAL-08 | Unit + Integration | Zona horaria explícita `America/Guayaquil`, conversión local→UTC, DST y hora inexistente/repetida. | Se conserva la zona de Campaign, no se infiere del navegador y la hora local de negocio se convierte al instante UTC correspondiente. | P1 | Pruebas de conversión local→UTC y persistencia de zona; defecto actual de conversión debe corregirse antes de habilitar programación. DST/reglas de otras zonas permanecen OPEN. |

## 8. Matriz — Scheduler, lifecycles e idempotencia

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-SCH-01 | Unit + Integration | Tick detecta Campaign vencida en SCHEDULED/ACTIVE y campañas futuras/suspendidas/finales. | Solo reclama due elegibles; tick corto, candidatos ordenados y limitados; no se crean crons por Campaign. | P0 | Clock fijo, query spy/DB y configuración de batch. |
| TST-SCH-02 | Integration | Crear una CampaignExecution por `campaignId + executionKey` al repetir tick sobre mismo slot ordinario. | `UNIQUE(campaignId, executionKey)` se cumple; la identidad del slot es determinista; segundo tick encuentra la misma Execution y no crea Execution ni dispatch adicional. | P0 | Constraint PostgreSQL, clave del slot, filas antes/después y conteo de invocaciones adapter. |
| TST-SCH-03 | Integration + Concurrency | Dos o más workers/pods compiten por una Campaign/slot simultáneamente en PostgreSQL. | Exactamente un claim ganador; una Execution/dispatch lógico; perdedores no procesan. | P0 | Prueba concurrente con procesos/conexiones independientes y salida de cada claim. Mocks no bastan. |
| TST-SCH-04 | Unit + Integration | Validar `PENDING → PROCESSING → COMPLETED`, `COMPLETED_WITH_ERRORS` o `FAILED`; intentar saltos/reapertura terminal. | Solo transición válida; estado final histórico no se sobrescribe. | P0 | Tabla de transiciones, HTTP/service rejection y DB history. |
| TST-SCH-05 | Unit + Integration | Campaign DRAFT→SCHEDULED→ACTIVE, suspend/resume/cancel y terminación por EndDate. | Transiciones conforme a FOUND; Campaign suspendida/final no se reclama; ACTIVE se edita solo tras suspend. Schedule/Resume permanecen bloqueados hasta resolver publicación, audiencia real y demás precondiciones; EndDate es inclusiva en fecha local. | P0 | Matriz FOUND, validación de precondiciones y eventos/consulta scheduler una vez habilitados; no habilitar por esta prueba. |
| TST-SCH-06 | Integration + Concurrency | Dos páginas/workers intentan reservar misma compra y mismo Campaign. | Unique recipient key permite un ganador; conflicto se registra como duplicate sin alterar recipient original. | P0 | Constraint DB, conteos y resultado por ID sintético. |
| TST-SCH-07 | Unit + Integration | Una persona tiene compras distintas y luego reaparece la misma compra en otro ciclo. | Nueva purchase elegible puede generar recipient independiente; purchase repetida no se reenvía en misma Campaign. | P0 | Ledger tras ciclos y llamadas al fake Emblue. |
| TST-SCH-08 | Failure/Recovery + Integration | Worker cae con lease activo/expirado antes del envío, entre lotes o tras respuesta del provider pero antes de persistir; incluye reserva expirada. | Lease vencido de Execution PROCESSING puede recuperarse en la misma Execution si es seguro. La reserva solo devuelve ownership; se inspeccionan ledger, Attempts y resultado del provider. No se repiten envíos confirmados; resultado ambiguo no autoriza resend sin reconciliación/idempotencia confirmada. | P0 | Fault points, snapshot/cursor/ledger/Attempt en DB, invocaciones adapter y logs sin PII; contrato Emblue para ambigüedad OPEN. |
| TST-SCH-09 | Failure/Recovery + Concurrency | Downtime omite varios slots de FREQUENCY; recuperación con páginas múltiples y slots que alcanzan o exceden EndDate. | Una recuperación consolidada evalúa `PurchaseDate + PostPurchaseOffset <= evaluationDate`; snapshot conserva `eligibilityCutoffAt` y `evaluationDate`. No crea slot posterior a EndDate ni duplica envíos; FAILED no se reabre. El tratamiento de slots omitidos, recuperación y finalización sigue OPEN. | P0 | Clock controlado, snapshot/keys, relación recovery y conteos. Política de catch-up, recuperación, conversión temporal y finalización al reanudar permanecen OPEN. |
| TST-SCH-10 | Failure/Recovery | Falla total de lectura CF/Analytics o despacho y luego retry/recovery. | El fallo total de lectura no equivale a audiencia vacía ni marca compras pendientes como procesadas; no avanza cutoff exitoso. `FAILED` es terminal e inmutable. Retry que requiere nueva Execution usa nueva `executionKey` y `recoveryOfExecutionId` o relación equivalente; no se crea una por tick repetido. | P0 | Fault injection, compras candidatas no procesadas antes/después, estado/snapshot inmutable, keys y vínculo de recovery. Completitud/paginación y mecanismo fuente OPEN. |
| TST-SCH-11 | Failure/Recovery + Integration | Procesamiento completo conforme a política aprobada con algunos resultados individuales fallidos y otros exitosos. | Solo si el conjunto de trabajo se completó según la política de completitud aprobada, Execution puede quedar `COMPLETED_WITH_ERRORS`; conserva cada outcome. Lectura incompleta no se clasifica automáticamente como parcial completada. Aceptación técnica Emblue, estado operativo `SENT` y evento `DELIVERED` permanecen distintos; significado final de SENT OPEN. | P0 | Estado de completitud de fuente, ledger/Attempts por recipient, conteos reconciliados y evidencia de que webhook no cambia automáticamente Attempt. Políticas/contrato OPEN. |
| TST-SCH-15 | Integration + Failure/Recovery | Recuperación posterior a una Execution terminal `FAILED` que requiera una nueva unidad. | Nunca reabre ni muta la FAILED; usa nueva `executionKey` y referencia `recoveryOfExecutionId` o equivalente. | P0 | Dos filas históricas/snapshots inmutables, unique constraint y vínculo verificable. Algoritmo definitivo OPEN. |
| TST-SCH-12 | Integration | Editar filtros/configuración después de una Execution. | Snapshot histórico no cambia; ejecución nueva usa nueva configuración solo después de transición permitida. | P1 | Comparación snapshot antes/después. |
| TST-SCH-13 | Failure/Recovery | Suspender/cancelar mientras una ejecución está PROCESSING o batches en vuelo. | No se reclama nuevo slot; trabajo actual sigue política explícita (drain/stop/finish), sin efectos contradictorios. | P1 | Prueba con barreras controladas; política de in-flight OPEN. |
| TST-SCH-14 | Integration | Audiencia grande sobre múltiples páginas y worker limitado. | Memoria/control de carga acotado; progreso durable y orden/cursores consistentes; no se duplica por frontera de página. | P1 | Conteos por página, métricas, cursor/ledger y perfil de recursos. Fuente/batch sizes OPEN. |

## 9. Matriz — CF/Analytics, Emblue y contratos

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-CF-01 | Contract + Integration | Request de audiencia contra contrato firmado de CF; incluir multiselect, jerarquía, offset, paginación, `evaluationDate` y `eligibilityCutoffAt`. | Campos/semántica de filtros coinciden; elegibilidad sigue `PurchaseDate + PostPurchaseOffset <= evaluationDate`. El timestamp técnico y fecha de negocio se congelan en snapshot al iniciar resolución y se reutilizan entre páginas/recovery. | P0 | Contract test aprobado y DB/snapshot inspeccionado en varias páginas/reinicio. Timezone/conversiones y contrato CF OPEN. |
| TST-CF-02 | Failure/Recovery + Contract | CF timeout, 5xx, auth error, rate limit, esquema inválido, página repetida o incompleta. | Clasificación diferenciada; fallo total no equivale a audiencia vacía ni marca compras pendientes procesadas. Lectura incompleta no se declara automáticamente `COMPLETED_WITH_ERRORS`; aplica política de completitud cuando se apruebe. | P0 | Fault fixture, Execution/preview outcome, ledger sin marcado indebido, retry metrics y logs sanitizados. Completitud/paginación OPEN. |
| TST-EMB-01 | Contract + Integration | Enviar a Emblue Transactional API con `surveyUrl` requerida y variables opcionales permitidas. | Payload conforme al contrato aprobado y al template; `purchaseId/transactionId` no se entrega como variable visible por defecto. | P0 | Endpoint/request/response individual v2.3 están confirmados parcialmente. Template, variables y flujo completo siguen OPEN; requiere sandbox no productivo para integración. |
| TST-EMB-02 | Failure/Recovery + Contract | Rechazo total/individual, credencial inválida, timeout antes/después del dispatch y rate limit. | Diferencia errores permanentes/transitorios/ambiguos; política de retry e idempotencia no produce duplicados. | P0 | 21 pruebas unitarias mock cubren los casos clasificados por `EmblueService`; semántica contractual, provider ref persistida e idempotencia siguen OPEN. |
| TST-EMB-03 | Contract + Integration | Respuesta aceptada de Emblue seguida de eventos de entrega/opens/clicks/bounces disponibles. | No confunde request aceptada con delivered; mapea solamente estados/eventos documentados y no inventa métricas. | P1 | Fixtures provider-approved y reconciliación recipient/event. |
| TST-EMB-04 | Unit | `EmblueService.sendEmail()` con `HttpService` mock: body v2.3 confirmado y respuesta `activityId`; errores 401/403/429/5xx, timeout, red, response inesperado y validación local. | Usa `POST /emails/send`; outcome ambiguo no reintenta; logs no exponen credenciales, destinatario, HTML ni response. | P0 | Ejecutado en pec-api: 21 pruebas unitarias aprobadas; no realiza tráfico externo. |
| TST-EMB-05 | Manual Integration | Script local NestJS, contexto mínimo `ConfigModule` + `EmblueModule`, dirección autorizada, confirmación explícita y una llamada. | Sin `AppModule`, DB/Prisma, Neon, scheduler ni reintento; resultado limita salida a status, httpStatus y activityId. | P0 | Ejecutado manualmente por el usuario: una llamada devolvió HTTP 200 y `activityId`; no acredita entrega final. Destinatario no registrado. |
| TST-WEB-01 | Contract + Security + Integration | Webhook válido, inválido, firma incorrecta, replay, evento duplicado y evento fuera de orden. | Autenticidad antes de efecto; evento deduplicado/idempotente; transiciones válidas; replay rechazado según mecanismo aprobado. | P0 | Contrato/fixture, respuesta HTTP, event ledger y logs sin payload sensible. Schema/firma OPEN. |
| TST-WEB-02 | Failure/Recovery + Integration | Webhook llega antes/después del cierre, falla persistencia o no se encuentra correlación. | No corrompe Execution terminal ni atribuye a destinatario incorrecto; se registra como evento y no cambia automáticamente el resultado operativo de `CampaignRecipientAttempt`. | P1 | Eventos sintéticos con orden controlado; comparación del estado Attempt y registro Event antes/después. Contrato y regla de reconciliación OPEN. |

El contrato de envío individual emBlue API v2.3 está confirmado parcialmente con evidencia aportada: endpoint, request y response `activityId`. `TST-EMB-04` valida el adapter con mocks; no demuestra compatibilidad integral del proveedor ni se considera prueba Contract completa. La consulta de actividad, idempotencia persistida, procesamiento, entrega y webhook siguen OPEN. No ejecutar el test manual sin autorización explícita.

## 10. Matriz — Security, PII y response correlation

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-SEC-01 | Security | Introducir email/nombre/IDs de compra/tokens/secretos en errores de CF/Emblue y mensajes de excepción. | Logs y respuestas públicas redactan PII/secreto; usar IDs internos/códigos sanitizados; no se registra payload completo. | P0 | Captura controlada de log + response inspeccionados con assertions de ausencia de canary values. |
| TST-SEC-02 | Security + E2E | Campaign/detail/preview/recipient response incluye accidentalmente email, `purchaseId`, `transactionId` o correlation token. | API no expone campos prohibidos por defecto; se deniega consulta sin permiso específico. | P0 | Snapshot/redaction assertion por ruta y rol. |
| TST-SEC-03 | Security + Integration | Uso de `surveyUrl` con referencia válida, manipulada, expirada o desconocida; enlazar `ResponseCorrelation` al `CampaignRecipientAttempt` o equivalente. | Solo referencia validada en servidor atribuye la respuesta correctamente a Campaign, Execution y Recipient. `surveyUrl` no incluye PII ni IDs de compra/transacción. Webhook no modifica automáticamente el Attempt. Reutilización se valida cuando se apruebe su regla. | P0 | Requests sintéticos, relación persistida de correlación → Attempt/Execution/Recipient y negative cases. Token, expiración, revocación y reutilización OPEN. |
| TST-SEC-04 | Security | Webhook/request intenta injection, replay, sobredimensionado o datos extra no reconocidos. | Rechazo o sanitización segura; no se muta estado ni se filtra stack/SQL. | P1 | Respuesta estándar, log sanitizado, controles de tamaño/validación aprobados. |
| TST-SEC-05 | Security + Integration | ADMIN/SUPERADMIN y roles no autorizados consultan Campaign/Execution/recipient; empresa activa distinta y membresia ADMIN secundaria. | Acceso Campaign sigue solo el rol efectivo y es multiempresa; otros roles reciben 403. | P0 | HTTP autenticado/CASL; recipients conservan permiso/campos OPEN. |
| TST-SEC-06 | Security | Cargar base con datos sintéticos y buscar texto por email o ID sensible en endpoints/listado. | Búsqueda/ordenamiento no acepta campos no permitidos; logs y resultados respetan minimización. | P1 | Request invalid tests y log/report redacted. |

## 10.1 Escenarios del proveedor MOCK y alcance multiempresa

| ID | Categoría | Escenario | Resultado esperado | Evidencia |
|---|---|---|---|---|
| TST-CAT-01 | Unit + HTTP | Consultar cada uno de los seis catálogos con proveedor MOCK explícito. | Devuelve opciones sintéticas con `id`, `label`, `parentId?`, `active`, `source=MOCK`; no incluye Auxiliary Subclassification. | Test de servicio/controller; respuesta sin datos reales. |
| TST-CAT-02 | Unit + Angular | Cargar opciones con padres, buscar, multiseleccionar, editar y cambiar padres. | Limpia selecciones hijas incompatibles; DRAFT conserva/persiste IDs y los muestra al editar. | Tests de componente/servicio y request/respuesta. |
| TST-CAT-03 | Unit + HTTP | Guardar DRAFT incompleto y enviar IDs inexistentes, inactivos o de relación incompatible. | DRAFT incompleto es válido; IDs inexistentes/incompatibles se rechazan en backend. | Estado persistido/HTTP y aserciones de proveedor. |
| TST-CAT-04 | Security + HTTP | Configurar Analytics no implementado, proveedor ausente o fallo del proveedor. | Falla explícitamente; no activa MOCK como fallback. En producción, configurar MOCK impide iniciar el módulo. | Test de factory/configuración y respuesta de error. |
| TST-CAT-05 | Unit + HTTP | Intentar usar IDs `MOCK-*` bajo Analytics. | Backend rechaza; nunca los interpreta como IDs reales. | Test de validación por proveedor. |
| TST-CAT-06 | Security + HTTP | ADMIN/SUPERADMIN contra campañas y selector Campaign de Surveys de varias empresas; otros roles y membresía ADMIN secundaria. | Acceso multiempresa solo para rol efectivo ADMIN/SUPERADMIN; los demás reciben 403. Cambiar empresa activa no altera alcance. | HTTP autenticado/CASL; el test no modifica grants generales. |
| TST-CAT-07 | Regression | Consultar endpoints generales `/surveys` y otros módulos con los mismos roles. | Se conservan sus permisos previos; el endpoint Campaign no amplía autorización general. | Suites relevantes existentes. |
| TST-CAT-08 | Boundary | Intentar scheduling, Audience Preview real o envío usando MOCK. | No se crea ejecución/recipient ni se solicita correo. | Prueba de no ejecución/fallo explícito; funcionalidades DEFERRED. |

Estas pruebas son evidencia de contrato interno/UI/validación, no compatibilidad CF/Analytics ni evaluación real de audiencia. Relaciones fixture no son reglas CF.

### Trazabilidad de decisiones a trabajo futuro de Sprint 2.5

### Escenarios de localización

| ID | Categoría | Escenario | Resultado esperado | Evidencia |
|---|---|---|---|---|
| TST-I18N-01 | Angular | Renderizar todos los estados Campaign/Execution/Recipient/Attempt, frecuencias y modalidades. | Etiquetas en español según FOUND-001; valores técnicos intactos en controles y modelos. | Tests parametrizados de mapeo. |
| TST-I18N-02 | Angular + HTTP | Filtrar por estado, seleccionar frecuencia/modalidad y guardar. | UI muestra español; solicitud conserva exactamente los enums técnicos, ONE_TIME o FREQUENCY. | Component/service tests con request body y query assertions. |
| TST-I18N-03 | Angular | Renderizar listado, formulario, detalle, botones, filtros, validaciones, vacíos y confirmación. | Todo texto propio del módulo en español; errores no exponen stack/SQL ni códigos internos crudos. | DOM assertions y casos de error. |
| TST-I18N-04 | Angular Material | Mostrar fechas, horas y paginador. | Fechas/horas y controles del paginador en español, con formato acordado. | Tests de locale/Intl y MatPaginatorIntl. |
| TST-I18N-05 | Provider MOCK + Angular | Mostrar opciones sintéticas del proveedor. | Labels españolas y marca MOCK visibles; IDs conservados sin transformación. | Provider assertions y UI/request round trip. |

| Decisiones / regla | SPEC de referencia | Incremento de implementación/QA |
|---|---|---|
| DEC-SC-01, DEC-SC-02, RB-097–RB-099 | FOUND-001, API-001 | T25-01: probar CASL por rol efectivo, endpoint Campaign de Surveys multiempresa y no regresión de `/surveys`. |
| DEC-SC-03, RB-100 | ARCH-001, DATA-001 | T25-02: conservar separación entre Survey.enterpriseId, Company externa e IDs de catálogos; no crear relación PEC inferida. |
| DEC-SC-04–DEC-SC-07, RB-101–RB-103 | ARCH-001, INTEG-001, API-001 | T25-03: seleccionar provider explícitamente, bloquear MOCK en producción y probar error/no fallback; preservar DTO Angular. |
| DEC-SC-08–DEC-SC-10, RB-007–RB-018, RB-104 | FOUND-001, DATA-001, API-001 | T25-04: probar opciones, relaciones sintéticas, búsqueda/multiselect, limpieza padre-hijo y validación backend; DRAFT incompleto y campos obligatorios al programar. |
| DEC-SC-11, RB-105 | ARCH-001, FOUND-001, SCHED-001, INTEG-001 | T25-05: comprobar que MOCK no dispara preview real, execution, recipient ni envío; funcionalidad queda DEFERRED. |
| DEC-SC-12, RB-006 / OPEN-API-01 | FOUND-001, API-001, ARCH-001 | Cerrado para Campaign: probar conjuncion de campos y cobertura de ventana, sin nuevo estado global PUBLISHED. |
| DEC-SC-13, OPEN-TEST-03 | TEST-001, DATA-001 | T25-07: completar verificación PostgreSQL en rama/base aislada y con fixtures sintéticos no destructivos. |
| OPEN de Analytics/Emblue | INTEG-001, API-001 | T25-08: incorporar contratos reales aprobados y contract tests; sin endpoints o jerarquías inventados. |

## 11. Métricas, evidencia y trazabilidad

Cada caso ejecutado debe poder vincularse con un `testId` de esta especificación y una regla/capacidad de FOUND/API/DATA/INTEG/SCHED. Evidencia mínima según nivel:

- **Unit:** nombre del suite/test (`describe`/`it` en inglés según estilo repo), assertion principal, versión/commit probado y resultado.
- **Integration:** request/response relevante sanitizado, estado DB previo/posterior, migración aplicada y configuración de test.
- **E2E:** pasos y rol de prueba, URL/ruta navegada, estado API/DB, screenshot/trace/video sin PII real.
- **Contract:** versión/hash de OpenAPI/schema/fixtures aprobados, proveedor/ambiente no productivo y reporte de compatibilidad.
- **Security:** rol/claim de prueba, request negativo y evidencia de no exposición, redactada.
- **Concurrency:** cantidad de workers/conexiones, barrera de inicio, resultados por worker, unicidad DB y total de envíos observados.
- **Failure/Recovery:** punto de inyección del fallo, momento del reloj, estado persistido antes/después y prueba de que recovery no duplicó.

Indicadores QA recomendados: passed/failed/skipped, coverage de branch de reglas de dominio, pruebas P0 pendientes, defectos abiertos, cobertura de estados/transiciones, invariantes de dedupe/claim y contract OPENs. La cobertura numérica mínima debe establecerse en CI; no inventar un porcentaje como baseline sin política de calidad PEC.

## 12. Definition of Done de QA

La estrategia se considera satisfecha para una versión candidata cuando:

1. Cada regla P0 aplicable tiene al menos una prueba determinista y una evidencia asociada; las pruebas de UI/API necesarias cubren el flujo real.
2. Todas las transiciones válidas e inválidas de Campaign, Execution y Recipient se verifican sin añadir estados no aprobados.
3. Survey `PUBLISHED`, Division/Department/Company requeridos, filtros OR/AND, dependencias, offset y Audience Preview pasan sus casos límite.
4. `ONE_TIME`, las cinco frecuencias y el catálogo dinámico usan duración inclusiva por fechas calendario y cubren límites 1/6/7/13/14/29/30/89/90. EndDate es inclusiva en fecha local, el slot exacto a SendTime es válido y no se genera uno posterior. MONTHLY/QUARTERLY conservan ancla StartDate, ajustan a fin de mes sin drift y consideran años bisiestos. Recovery/downtime y finalización al reanudar siguen OPEN.
5. Campaign persiste zona horaria explícita antes de Schedule/Resume; Ecuador usa `America/Guayaquil` por defecto y nunca se infiere la zona del navegador. Audience Preview y ejecuciones aplican `PurchaseDate + PostPurchaseOffset <= evaluationDate`; congelan `evaluationDate` y `eligibilityCutoffAt` en snapshot y no los cambian entre páginas ni durante recovery. La conversión local→UTC debe representar el instante correcto; DST y reglas de otras zonas siguen OPEN.
6. Integration/concurrency tests en PostgreSQL demuestran `UNIQUE(campaignId, executionKey)`, identidad determinista del slot ordinario, un único claim por slot y deduplicación `campaignId + purchaseId/transactionId` con carreras reales, no solo mocks.
7. Recovery cubre caída en etapas críticas y downtime con varios slots: lease PROCESSING puede recuperarse en la misma Execution si es seguro; FAILED terminal nunca se reabre; retry nuevo enlaza con `recoveryOfExecutionId`; tick repetido no crea Execution/envío duplicado; reserva expirada no autoriza resend ambiguo.
8. Fallo total de CF/Analytics no aparece como audiencia vacía ni procesa pendientes. `COMPLETED_WITH_ERRORS` requiere procesamiento completo conforme a política aprobada; lectura incompleta queda sujeta a política OPEN. No se confunde aceptación técnica de Emblue, `SENT` y `DELIVERED`.
9. Tests de CF/Emblue/Webhooks usan contratos aprobados o permanecen explícitamente bloqueados por OPEN; no se habilita producción con contratos inventados. Webhooks no alteran automáticamente el outcome de Attempt.
10. Los tests de seguridad comprueban permisos backend, aislamiento de scope, ausencia de PII en URLs/logs/responses y ResponseCorrelation correcta Campaign/Execution/Recipient mediante Attempt o equivalente.
11. Dashboard y Campaign Detail presentan datos persistidos y estados verdaderos; Total incluye CANCELLED visibles/no eliminadas, hay exactamente seis tarjetas, búsqueda `searchText` + `surveySearchText` en AND y paginación/scope tras aplicar filtros. CANCELLED es filtro y no tarjeta.
12. Suites existentes y nuevas, build relevante, CI y Quality Gate pasan. El reporte separa unit, integration, E2E, contract, security y concurrency; una suite mock no se reporta como prueba de DB/proveedor real.
13. Ningún fallo P0/P1 aplicable queda aceptado sin disposición escrita; Contract/Failure tests externos bloqueados hasta contratos aprobados; tests son reproducibles, aislados, sin credenciales ni PII reales; documentación de OPENs y ambiente coincide con el commit evaluado.

## 12.1 Pruebas nuevas requeridas por Sprint 2.9.1

Estas pruebas son requisitos para el sprint de implementación; no se ejecutaron en esta actualización documental.

| ID | Tipo | Escenario | Resultado esperado |
|---|---|---|---|
| TST-REC-01 | Unit + Integration | Slots vencidos de varios ciclos, catch-up dentro de 30 días. | Una sola recovery por Campaign, snapshot identifica rango/slots cubiertos; no se reproduce envío por cada slot. |
| TST-REC-02 | Boundary + Integration | Antigüedad del backlog justo bajo/en/sobre 30 días (incluyendo valor configurable). | Dentro del límite se recupera consolidado; sobre el límite no se descarta ni avanza el trabajo y se emite alerta/intervención. |
| TST-REC-03 | Integration | Resume antes y después de EndDate inclusiva. | Antes: recovery consolidada y próximo slot futuro. Después: no slot ordinario nuevo, solo recovery de slots originales válidos; completa cuando no quede trabajo operativo. |
| TST-REC-04 | Concurrency | Dos workers reclaman mismo Campaign/slot a la vez. | Una Execution/claim; unicidad y claim atómico prevalecen. |
| TST-REC-05 | Failure/Recovery | Lease expira y worker anterior intenta dispatch/escritura tras adquirir otro worker. | Fencing/CAS rechaza worker rezagado; el progreso ya persistido no se duplica. |
| TST-REC-06 | Integration | Suspend concurrente con claim y con batch ya iniciado. | No comienza claim/batch posterior a Suspend; el batch iniciado se drena y persiste. |
| TST-REC-07 | Unit + Integration | Retry de transitorios confirmados, permanentes y ambiguos. | Máximo 3 retries automáticos además del inicial, backoff exponencial+jitter, nueva Execution relacionada solo con elegibles; permanente/ambiguo sin retry automático. |
| TST-REC-08 | Failure/Recovery | Timeout después de dispatch sin respuesta concluyente. | Estado ambiguous conservado; no resend automático hasta reconciliación/idempotencia documental. |
| TST-REC-09 | Integration | Campaign llega a fin de calendario con Execution/recipient pendiente, processing o recuperable. | No pasa COMPLETED hasta resolver el trabajo; terminales inmutables. `lastExecutionAt` usa `finishedAt` de la última terminal incluyendo FAILED. |
| TST-REC-10 | Contract | Respuesta API aceptada, proceso asíncrono y evento de entrega simulados separadamente. | No se confunden aceptación, procesamiento y entrega efectiva; no se asume que emBlue soporta idempotencia. |

## 13. OPENs de prueba

| ID | Pendiente | Efecto en QA |
|---|---|---|
| OPEN-TEST-01 | Runner E2E UI y configuración CI de navegador. | Definir herramienta, comando, fixtures y artefactos antes de exigir E2E Angular. |
| OPEN-TEST-02 | `test:integration` Angular apunta a proyecto/target no encontrado en `angular.json`. | Corregir o retirar script como trabajo separado cuando se apruebe; no asumir que corre hoy. |
| OPEN-TEST-03 | Ambiente PostgreSQL aislado para integración Campaign, cadena de migraciones y fixtures no destructivos. | Sprint 2.4 PostgreSQL continúa pendiente. No ejecutar TRUNCATE/DROP ni modificar filas existentes; usar rama/base aislada antes de probar. |
| OPEN-TEST-04 | Contrato real, ambiente sandbox y datos sintéticos de CF/Analytics. | Contract/source integration queda pendiente. |
| OPEN-TEST-05 | Credenciales no productivas, templates, sandbox y contratos externos restantes de Emblue; la llamada de envío individual v2.3 está confirmada parcialmente. | El adapter tiene pruebas unitarias mock; contract/send real, webhook y reconciliación no se certifican sin estas dependencias. |
| OPEN-TEST-06 | Permisos específicos de recipients/operaciones; el acceso Campaign básico está aprobado solo para ADMIN/SUPERADMIN efectivos y alcance multiempresa. | Probar rol efectivo, denegación a otros roles, independencia de empresa activa y no regresión de otros módulos. |
| OPEN-TEST-07 | TZ local uniforme entre replicas API, acceso despues de availableTo; conversion local a UTC/DST y reglas para otras zonas. | Probar la conjuncion Campaign status=1, isPublic=true, archived=false y cobertura del calendario. Schedule/Resume siguen bloqueados por audiencia/catalogos/plantilla. |
| OPEN-TEST-08 | Reconciliación/idempotencia Emblue, firma/replay/event ID de webhook, códigos para clasificar errores, y reglas de completitud de lectura/fallo parcial. | Probar no-resend ante ambiguity; contrato/reconciliación externos quedan bloqueados. `FAILED` terminal no se reabre. |
| OPEN-TEST-09 | Umbral CI de coverage y calidad, métricas de contract tests y Quality Gate PEC aplicable. | DoD requiere usar Quality Gate vigente sin inventar porcentaje nuevo. |
| OPEN-TEST-10 | Validación PostgreSQL aislada de `executionKey`, `claimGeneration`, recuperación de reservas/Attempts y constraints nuevas. `nextRetryAt` no se requiere: retry queda en `CampaignExecution.scheduledAt`. | Las pruebas unitarias mock verifican CAS/fencing y selección de retry; falta confirmar concurrencia, unique index y transacciones en PostgreSQL local aislado, sin limpiar datos existentes. |
| OPEN-TEST-11 | Token de ResponseCorrelation: emisión, expiración, revocación, reutilización y cardinalidad de respuestas. | Probar casos válidos/negativos en cuanto el mecanismo se apruebe; no inventar reglas de reuso. |

## 14. Estado del documento

- **Status:** DRAFT
- **Version:** 0.5
- **Tests escritos/ejecutados:** 21 pruebas unitarias mock de `EmblueService.sendEmail()` aprobadas y una ejecución manual NestJS reportada con HTTP 200/`activityId`; esta evidencia confirma aceptación API para una llamada, no procesamiento o entrega final.
- **Pendiente:** ejecutar suites de catálogos/scope y validación PostgreSQL aislada. `/activities/find`, idempotencia persistida, publicación Survey, calendario, scheduler y recuperación operativa siguen OPEN. Esta actualización documental no ejecuta pruebas.

### Actualización Sprint 3.1A — pruebas del control de ejecuciones

`campaign-execution.service.spec.ts` ejecuta 13 pruebas unitarias con mocks: creación idempotente, contienda de dos workers simulada mediante CAS, lease vencido, fencing inválido/worker rezagado, ordinal interno duplicado, renovación/liberación de lease, slots de recovery preservados, marcador `DISPATCHING`, aceptación con `activityId` sin inferir `SENT`, inmutabilidad terminal y retry selectivo que excluye accepted/ambiguous.

Resultado local: 13/13 pruebas focalizadas y 84 pruebas de Campaign aprobadas; build NestJS y `prisma validate` aprobados. La prueba de dos workers es un mock, no demuestra serialización real de PostgreSQL. No había `CAMPAIGN_TEST_DATABASE_URL`; no se ejecutó integración PostgreSQL, no se aplicó la migración y no se usó Neon. Generación Prisma quedó bloqueada por `EPERM` del motor Windows.

## 15. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Alineación de elegibilidad temporal con `evaluationDate` y snapshot dual; criterios de fallos totales/parciales y distinción Emblue; pruebas de identidad de ejecución y recovery; cobertura explícita de tarjetas/búsqueda/paginación; ResponseCorrelation vinculada a Attempt; actualización de DoD y OPENs sin cerrar decisiones pendientes. |
| 0.3 | 2026-10-20 | Añade escenarios de catálogos MOCK, relaciones, DRAFT, autorización multiempresa y regresión; mantiene PostgreSQL pendiente con fixtures no destructivos y aclara que MOCK no prueba contratos externos ni habilita scheduler/envíos. Estado DRAFT. |
| 0.4 | 2026-10-20 | Agrega pruebas para etiquetas españolas de estados, frecuencias, modalidades, textos, fechas, paginador, MOCK y preservación exacta de payloads. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Se cierran los casos de conteo inclusivo y límites exactos de duración; se separa la validación de frecuencia de los OPENs sobre ocurrencias efectivas en EndDate. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Formaliza pruebas para `America/Guayaquil`, zona explícita sin inferencia del navegador y EndDate local inclusiva; mantiene OPEN publicación, conversión local→UTC, DST, catch-up y recuperación. Estado DRAFT conservado. |
| 0.4 | 2026-10-21 | Agrega casos deterministas MONTHLY/QUARTERLY con ancla StartDate, ajuste a fin de mes, años bisiestos y ausencia de drift acumulativo; mantiene OPEN recovery/downtime. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Añade escenarios requeridos para consolidación/backlog 30 días, Suspend/Resume/EndDate, fencing multiworker, retry selectivo y separación de aceptación/procesamiento/entrega; cierra OPENs funcionales de recovery. Sin pruebas ejecutadas. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra 21 pruebas unitarias mock de `sendEmail()` y el script manual NestJS preparado, pendiente de autorización/ejecución. No confunde acceptance con entrega ni cierra `activities/find` o idempotencia. Estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra la ejecución manual reportada por el usuario: una solicitud NestJS aceptada HTTP 200 con `activityId`; sin destinatario ni afirmación de entrega. Estado DRAFT conservado. |

### Validación Sprint 3.1A.1 — resultados comprobados (2026-10-23)

`prisma validate`, `npm run build`, ESLint focalizado y 84 pruebas unitarias Campaign pasaron localmente. `prisma generate` volvió a fallar con `EPERM` al renombrar `query_engine-windows.dll.node`; no se detuvo ningún proceso. La consulta de esquema de solo lectura confirmó en la base de desarrollo los enums, columnas, default e índice de la migración. La migración no está registrada en `_prisma_migrations`. La suite PostgreSQL no se ejecutó: crea y actualiza empresas, usuarios, encuestas y campañas sin limpieza posterior; no hay `CAMPAIGN_TEST_DATABASE_URL` y no se autorizó escribir en la base persistente. La contienda entre workers permanece verificada solo por mocks, no por PostgreSQL. Estado DRAFT conservado.

El registro incompleto 20250907231907_add_user_establishment_table es metadato historico bajo Database-First; no bloquea Prisma Client ni DDL manual y no se modifico.


### Convención Database-First y verificación (2026-10-23)

Las pruebas unitarias y build no aplican DDL. Las pruebas de integración con escrituras requieren PostgreSQL aislado, preparado con SQL manual y esquema sincronizado; no ejecutan migraciones Prisma ni usan la base de desarrollo persistente. La ausencia de una fila en _prisma_migrations no afecta el Prisma Client. Después de generar el cliente, prisma validate, prisma generate, build NestJS, ESLint focalizado y 84 pruebas unitarias Campaign pasaron. La suite PostgreSQL se omitió porque crea y actualiza fixtures sin limpieza y no existe CAMPAIGN_TEST_DATABASE_URL. DRAFT conservado.

### Sprint 3.1A.2 — seguridad y preparación PostgreSQL (2026-10-23)

La suite `survey-campaigns.integration.spec.ts` escribe empresas, usuario, encuestas, campañas, ejecuciones, destinatarios e intentos; también actualiza una Survey sintética archivándola. No usa datos de clientes. La suite ahora asigna un `runTag` UUID a las filas sintéticas y, en `afterAll`, elimina por IDs rastreados y en orden de claves foráneas. No utiliza TRUNCATE, DROP, reset de secuencias ni limpieza global. Una terminación forzada del proceso antes de `afterAll` puede dejar fixtures identificados por `IT-CAMPAIGN-<UUID>`; por eso se exige una base descartable/local dedicada.

La integración se rechaza salvo que `CAMPAIGN_TEST_DATABASE_URL` apunte a `localhost`/loopback, puerto 5432 y base `pec_campaigns_test`. Antes de crear fixtures se consulta y verifica `current_database`, `current_schema`, dirección/puerto del servidor, `current_user` y propietario de la base; se requiere esquema `public`, loopback y rol/propietario `campaign_test`. Se elimina el opt-in a Neon. Estas comprobaciones no constituyen autorización para usar una base remota ni sustituyen la revisión de la conexión local antes de ejecutar.

Se prepararon escenarios PostgreSQL reales para claim concurrente con un único ganador, generación de fencing, takeover de lease expirado, rechazo de worker obsoleto, renovación, unicidad/idempotencia de executionKey, inmutabilidad terminal, progreso y activityId, retry selectivo sin ambiguos y recovery con slots originales. Estas pruebas requieren la base aislada anterior; no se ejecutaron en Sprint 3.1A.2 porque `CAMPAIGN_TEST_DATABASE_URL` no está configurada. Ninguna consulta de integración ni fixture PostgreSQL se ejecutó.

La concurrencia PostgreSQL, restricciones y transacciones permanecen pendientes de evidencia. Pruebas unitarias mock no sustituyen dichas comprobaciones. Para ejecutar la suite se requiere una instancia PostgreSQL local aislada, base propiedad del rol `campaign_test`, esquema preparado manualmente conforme al SQL Database-First vigente, URL temporal de pruebas y credenciales sin privilegios sobre otras bases. El runner de integración no aplica DDL.

Validación de despliegue Database-First: el entrypoint conserva una ruta condicional que llama `prisma migrate deploy` cuando `RUN_MIGRATIONS=true`; el chart Helm inspeccionado no establece esa variable. Se recomienda eliminar/bloquear esa ruta en una actualización posterior autorizada o hacer que `RUN_MIGRATIONS=true` termine el arranque con error explícito; no se modificó el despliegue en este sprint.

## Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.5 | 2026-10-23 | Documenta el guard conservador para PostgreSQL local, fixtures UUID con limpieza acotada, escenarios de concurrencia preparados y resultados pendientes al no existir URL de pruebas. Registra el riesgo condicional de `RUN_MIGRATIONS=true`; estado DRAFT conservado. |
| 0.5 | 2026-10-23 | Registra 15/15 pruebas PostgreSQL, 108/108 pruebas Campaign, build/lint y correcciones del guard, fixtures e idempotencia concurrente; conserva DRAFT. |

Aclaración del árbol actual: `prisma/migrations/20261018000000_add_automated_survey_campaigns/migration.sql` sigue existiendo en el checkout. El DDL adicional de controles está en `prisma/manual-sql/campaign_execution_controls.sql`, pero eso no elimina la migración inicial de Campaign ni el resto del historial. Si `RUN_MIGRATIONS=true`, Prisma podría intentar aplicar cualquier migración pendiente de esa cadena contra la base configurada. El riesgo refuerza la recomendación de bloquear la ruta automática bajo Database-First; no se cambió ni movió este archivo.

### Sprint 3.1A.3 — disponibilidad de PostgreSQL y regresión general (2026-10-23)

Comprobación de solo lectura del equipo: Docker Desktop está instalado (CLI 29.8.0), pero el servicio `com.docker.service` figura `Stopped` y el cliente no puede conectarse al engine (`docker version`/`docker ps`: acceso denegado al named pipe). No hay proceso/servicio PostgreSQL ni comandos `psql`/`postgres` disponibles. El puerto local 5432 no tiene listener y la conexión a `127.0.0.1:5432` falla. `CAMPAIGN_TEST_DATABASE_URL` no está configurada. Por tanto, no se pudo verificar identidad, propietario ni esquema; no se crearon recursos ni se ejecutó DDL/DML. Las 15 pruebas PostgreSQL continúan sin evidencia real.

No debe iniciarse el `docker-compose.yml` del proyecto para esta validación: inicia también la API, lee `.env`, declara `restart: always`, conserva un volumen `db` y publica PostgreSQL en 5435. Se requiere autorización para iniciar Docker Desktop y provisionar una instancia de pruebas separada, solo enlazada a loopback, con base `pec_campaigns_test`, propietario `campaign_test` y almacenamiento propio. No existe hoy un SQL bootstrap completo aprobado para una base vacía: `campaign_execution_controls.sql` es incremental y la migración Campaign inicial depende de tablas PEC existentes. Antes de probar se debe preparar/revisar el esquema de test completo; no cargar datos de Neon ni usar el compose actual.

Regresión Jest ejecutada con la suite PostgreSQL excluida: 95 suites, 72 aprobadas, 23 fallidas; 1.124 pruebas, 1.063 aprobadas y 61 fallidas. Agrupación observada: 9 suites no llegaron a ejecutar pruebas por errores de compilación TypeScript o resolución de módulos; 20 pruebas fallaron por providers Nest no registrados en módulos de test; las otras 41 fallaron por expectativas/mocks, fixtures, resultados y un caso de timeout en suites de Complaints, Product, Reports, Metrics, Surveys reports, Country, Configs, Auth, Keycloak y Notifications. Los errores de compilación incluyen fixtures `codCollaborator`/DTO incompatibles, propiedades esperadas ausentes, helper con aridad distinta y resolución de `roles.constants`.

No falló una suite específica de Campaign en esa corrida; los tests Campaign focalizados pasaron 90/90 en 8 suites, build NestJS pasó y ESLint focalizado pasó. Esto no demuestra causalidad ni permite etiquetar los 61 fallos como preexistentes: falta una comparación reproducible con el commit base previo a los cambios Campaign. No se modificaron módulos ajenos.

La revisión de `docker-entrypoint.sh` confirma que `RUN_MIGRATIONS=true` activa `prisma migrate deploy`; el chart inspeccionado no define esa variable, pero overrides externos podrían hacerlo. Además, `20261018000000_add_automated_survey_campaigns/migration.sql` permanece en `prisma/migrations`; Prisma podría procesar migraciones pendientes de toda la cadena. Recomendación Database-First: eliminar la rama de deploy automático o hacer fail-closed al detectar `RUN_MIGRATIONS=true`, mantener `prisma generate` separado y añadir una comprobación CI de chart/overrides. No se modificó el despliegue.

Dictamen: Sprint 3.1A no se cierra todavía. Falta ejecutar las pruebas reales contra PostgreSQL local aislado y obtener resultado concurrente. Los SPECS continúan versión 0.5 / DRAFT.

### Sprint 3.1A.3 — validación PostgreSQL aislada (2026-10-23)

Esta evidencia sustituye el bloqueo operativo descrito en la nota Sprint 3.1A.3 anterior. Se utilizó el contenedor dedicado `pec-campaign-test-db-sprint313`, imagen PostgreSQL 15.4 Alpine, publicado solo en `127.0.0.1:5432`, sin volumen persistente ni datos reales. La base `pec_campaigns_test`, esquema `public` y rol/propietario `campaign_test` se verificaron antes de los fixtures. No se utilizó Neon ni Prisma Migrate.

El esquema de prueba se preparó aplicando al contenedor recién creado SQL DDL generado desde el `schema.prisma` actual (178 sentencias estructurales; sin DML, DROP ni TRUNCATE). Se verificaron las diez tablas físicas `sspectcampaign*`, tipos y restricciones requeridos, incluidos `claim_generation DEFAULT 0 NOT NULL` y el índice único de `attempt_key`.

La suite de integración PostgreSQL ejecutó sus 15 pruebas: 15 aprobadas. Cubrió restricciones FK/UNIQUE, relaciones Campaign/Survey entre empresas, dos workers con un solo claim, incremento de fencing, takeover de lease expirado, rechazo de worker obsoleto, renovación, repetición concurrente de `executionKey`, inmutabilidad terminal, progreso/`activityId`, retry solo para fallo transitorio confirmado y recovery con slots originales. Tras la suite, consultas de solo lectura confirmaron cero filas en Campaign, Execution, Recipient, Attempt y ResponseCorrelation, y ausencia de fixtures sintéticos en Country, Enterprise, User y Survey.

La validación encontró y corrigió tres defectos limitados a Campaign: el guard rechazaba la IP privada reportada por PostgreSQL dentro de Docker aunque la URL del cliente fuera loopback; los fixtures omitían campos y la relación Country obligatorios por el schema; y dos `upsert` simultáneos con la misma clave podían devolver `P2002` en lugar de la fila ganadora. Se agregó cobertura unitaria para el guard y para la carrera de unicidad; el caso de fencing ahora utiliza un Recipient sintético válido.

Validación posterior: 9 suites Campaign, 108 pruebas aprobadas (incluye las 15 PostgreSQL); `npm run build` aprobado; ESLint focalizado aprobado. La suite general, excluyendo la integración ejecutada separadamente, reportó 95 suites (72 aprobadas, 23 fallidas) y 1.127 pruebas (1.066 aprobadas, 61 fallidas). Los fallos se ubican en suites ajenas a Campaign: errores de compilación/resolución, dependencias no provistas y aserciones en Complaints, Product, Reports, Metrics, Surveys report, Country, Configs, Auth/Keycloak y Notifications. No se hizo una corrida comparable del commit base; no se atribuyen como preexistentes ni se afirma causalidad de Campaign.

Para esta fase, la verificación de PostgreSQL aislado y concurrencia queda satisfecha. Permanecen fuera de alcance y OPEN los flujos de scheduler, envío automático, `/activities/find`, idempotencia externa de emBlue, entrega final, publicación real de Survey y la protección de despliegue `RUN_MIGRATIONS=true`. SPECS conservan v0.5 / DRAFT.

### Sprint 3.1B — evidencia de pruebas de checkout

`pnpm exec jest src/survey-campaigns --runInBand` reportó 10 suites aprobadas, 108 pruebas aprobadas, 1 suite de integración omitida y 17 pruebas omitidas. El build NestJS y ESLint focalizado finalizaron correctamente. Los tests unitarios cubren lifecycle, blockers, dry-run y backlog; los tests PostgreSQL están en `survey-campaigns.integration.spec.ts`, pero no se ejecutaron aquí. Docker Engine denegó acceso y no había listener en `localhost:5432`; no se intentó Neon. Por tanto, no hay evidencia de concurrencia/reinicio PostgreSQL para 3.1B. No se enviaron correos y el scheduler no invoca emBlue. Mantener criterios PostgreSQL abiertos hasta una corrida aislada.

| 0.5 | 2026-10-23 | Registra resultados de build, lint y 108 pruebas unitarias Campaign; 17 pruebas de integración omitidas por falta de PostgreSQL local accesible. DRAFT conservado. |

### Sprint 3.1B.1 — cobertura PostgreSQL adicional preparada (2026-10-23)

Se amplió `survey-campaigns.integration.spec.ts` con fixtures identificados por `runTag` sintético y limpieza por IDs: Schedule/Suspend concurrentes con verificación de `version`, Resume posterior a EndDate con una recovery que contiene solo slots originales y `nextExecutionAt = null`, y rechazo de Resume cuando el backlog supera 30 días sin alterar estado/cursor. El test de ticks concurrentes ahora también simula un tick posterior al reinicio y verifica que el cursor avanzado evita una Execution duplicada. Estas pruebas compilan, pero no fueron ejecutadas contra PostgreSQL.

La suite local `pnpm exec jest src/survey-campaigns --runInBand` pasó 108 pruebas en 10 suites; 20 pruebas de la única suite PostgreSQL quedaron omitidas. Build NestJS y ESLint focalizado del archivo de integración pasaron. Docker service `com.docker.service` figura Stopped (Manual), el engine rechazó la consulta del contenedor y no hay listener en 5432; no se inició Docker ni se modificó ningún servicio. `psql` no está disponible en PATH. Identidad/configuración del contenedor no verificables en esta sesión. DRAFT conservado.

| 0.5 | 2026-10-23 | Añade cobertura PostgreSQL preparada para concurrencia de lifecycle, Resume al vencer EndDate, límite de backlog y repetición de tick tras reinicio; ejecución pendiente hasta disponer del contenedor aislado. DRAFT conservado. |

### Sprint 3.1B.1 — resultado PostgreSQL y regresión Campaign (2026-10-23)

Resultado verificado: `pnpm exec jest src/survey-campaigns --runInBand` con `CAMPAIGN_TEST_DATABASE_URL` temporal, solo en el proceso, dirigido a `pec_campaigns_test`: 11 suites y 128/128 pruebas aprobadas; la integración PostgreSQL aporta 20/20. `pnpm run build` aprobado. ESLint focalizado aprobado para los servicios lifecycle/scheduler/execution, controller y la prueba PostgreSQL. `git diff --check` aprobado en `pec-api` y `mbrion-specs`.

El test de Suspend falló primero porque el fixture usaba una hora fija que hacía vencer el lease. Se cambió el fixture a tiempos relativos; luego la prueba aislada y la suite completa pasaron. Las consultas de limpieza confirmaron cero registros de prueba en Campaign, Execution, Recipient, Attempt, ResponseCorrelation, Survey, Enterprise, User y Country. No quedan defectos de prueba conocidos en esta suite. No se atribuyen resultados a la suite general de Jest.

| 0.5 | 2026-10-23 | Registra 20/20 pruebas PostgreSQL y 128/128 pruebas Campaign, build/lint focalizados y limpieza verificada de fixtures sintéticos; DRAFT conservado. |
## Registro de validación Sprint 3.1C

Se añadieron verificaciones unitarias de preflight: Survey activa no equivale a publicada, se muestran bloqueos de MOCK/audiencia/plantilla y el endpoint HTTP queda bajo `Read Campaign` (incluye denegación a roles no autorizados). Resultado local: 10 suites / 110 pruebas unitarias Campaign aprobadas (se excluyó explícitamente `survey-campaigns.integration.spec.ts`), build NestJS aprobado y ESLint focalizado aprobado. No se ejecutó PostgreSQL en este incremento, ni se accedió a Neon. La consulta no persiste cambios ni llama emBlue.

| 0.5 | 2026-10-23 | Añade escenarios de preflight, preservación de bloqueos y autorización HTTP; no se usan servicios externos, Neon ni correos. DRAFT conservado. |
## Sprint 3.1C.1 — pruebas independientes de contrato

Se añaden pruebas unitarias puras para detectar variables requeridas ausentes, claves no declaradas y escapar valores para contexto de texto HTML. Resultado: `pnpm exec jest src/survey-campaigns --runInBand --no-coverage --testPathIgnorePatterns=integration.spec.ts`, 11 suites y 113 pruebas aprobadas. Build NestJS y ESLint focalizado también aprobados. No se ejecutó la suite PostgreSQL. No se analiza la sintaxis de placeholders de un proveedor, no se renderiza una plantilla y no se sanea HTML completo. El preflight conserva publicación/URL/audiencia/plantilla OPEN o BLOCKED; una interfaz o fixture MOCK no las convierte en READY.

Pruebas de aceptación aún necesarias: publicación confirmada versus estado activo; ventana de Campaign cubierta por disponibilidad Survey; enlace final en navegador/API; paginación y estabilidad de audiencia; elegibilidad y duplicados; consentimiento/supresión/correo; render de plantilla aprobada, enlace individual y correlación opaca. Todas deben usar datos sintéticos o mocks. Pruebas contractuales externas esperan el contrato oficial de Data-Lab y de plantilla.

| 0.5 | 2026-10-23 | Registra pruebas independientes de validación de variables/escape y criterios pendientes por integración; no se hacen llamadas externas ni envíos. DRAFT conservado. |


## Sprint 3.1C.1 - publicacion de Survey para Campaign

Regla funcional aprobada para Automated Survey Campaigns: una Survey es elegible solo si `status = 1`, `isPublic = true` y `archived = false`. `status=1` aislado no demuestra publicacion ni cambia el significado global de Survey. La ruta publica existente conserva su comportamiento, incluido el fallback para Surveys genericas; Campaign exige `isPublic=true` y no utiliza ese fallback.

El gate lee `content.availableFrom` y `content.availableTo` si estan configurados. Valida formato e intervalo y verifica que el calendario Campaign completo, desde inicio local hasta fin local inclusivo, quede cubierto. Usa la zona IANA de Campaign para el calendario y el parser server-local ya usado por la ruta publica para interpretar los limites Survey. Mantener TZ consistente entre replicas del API es un requisito operativo; acceso a respuestas despues de `availableTo` y enlace individual Campaign siguen pendientes.

Preflight reporta `SURVEY_PUBLICATION=READY` unicamente con la conjuncion aprobada; `SURVEY_CALENDAR_AVAILABILITY` distingue READY, BLOCKED y OPEN. Schedule y Resume vuelven a validar antes de mutar estado. No se modifica Prisma, no se cambia la ruta Surveys, y audiencia real, catalogos reales, plantilla y enlace individual siguen bloqueando la activacion. Evidencia local 2026-10-23: Campaign unitarias sin PostgreSQL, 11 suites / 122 pruebas aprobadas; SurveysService, 1 suite / 58 pruebas aprobadas; build NestJS aprobado. ESLint en CampaignLifecycleService, sus pruebas, helper de disponibilidad y HTTP Campaign aprobado. ESLint del archivo SurveysService reporta hallazgos de estilo/uso fuera del diff; el import `format` ya estaba sin uso en HEAD. No se ejecutaron pruebas PostgreSQL, no se accedio a Neon, no se envio correo ni se activo Cron. Se conserva v0.5 / DRAFT.

## Sprint 3.1C.2 - cobertura del validador concreto

Pruebas para regla activa/pública/no archivada, inactiva, privada, archivada, genérica privada, fecha disponible futura, fecha vencida, calendario fuera de `availableTo`, configuración incompleta y revalidación al reanudar. También se verifica que enlace individual permanezca OPEN y que audiencia/plantilla/catálogos MOCK sigan bloqueando operaciones. Evidencia local 2026-10-23: 34 pruebas focalizadas aprobadas, build NestJS aprobado y ESLint focalizado aprobado. No se ejecutaron pruebas de integración PostgreSQL, DDL/DML, Cron ni envíos. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Registra regla Campaign de Survey, gate temporal y cobertura de pruebas; DRAFT conservado. |

## Sprint 3.1C.3 - aceptación futura de proveedor CF/Data-Lab

No se agregan pruebas de contrato externo porque no existe especificación verificable y no se hicieron llamadas a servicios ni consultas de datos. Antes de aceptar un adapter real deberán existir fixtures sintéticos/redacted que prueben IDs/relaciones de cada catálogo, filtro por padres, paginación estable/repetida, límites de página y errores de fuente; resolución de compra con clave/source/type, fecha y offset; correo ausente/inválido; duplicados; exclusiones/consentimiento; alcance Company multiempresa y minimización de PII. Verificar que fallo de fuente no se transforme en página vacía, que MOCK no habilite programación y que no se llame emBlue. v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Añade criterios de aceptación de contrato para catálogos y audiencia, sin pruebas externas ni datos personales. DRAFT conservado. |
## Sprint 3.1C.4 - pruebas de contrato BI propuestas

No se ejecutaron pruebas externas ni de base de datos para este diseno. Preparar, tras confirmacion del esquema, fixtures sinteticos con identificadores y correos reservados de ejemplo; nunca cargar registros reales. Casos de contrato: claves estables, IDs como strings opacos, relaciones padre-hijo devueltas por BI (sin asumir jerarquia), vigencia/activo, division y departamento obligatorios, empresa/formato/local/subclasificacion opcionales, OR por filtro y AND entre filtros, deduplicacion segun unidad aprobada, fechas/cutoff, correo ausente/invalido, filtro de supresion/consentimiento, paginas repetibles con mismo load_id, cursor invalido/expirado, fuente incompleta y fallo de lectura.

Probar la publicacion atomica de los dos datasets y que un batch vacio solo se acepte con manifiesto/senal de completitud acordada. Verificar que respuesta Angular no contiene PII, que el adapter rechaza batch sin publicar y que no llama a emBlue ni activa scheduler. El DDL sigue siendo propuesta no ejecutada. SPECS v0.5 / DRAFT.

| 0.5 | 2026-10-23 | Anade plan de pruebas sinteticas de contrato BI; no se ejecutaron pruebas ni se usaron datos reales. DRAFT conservado. |
## Sprint 3.1C.4.1 - cobertura de factura consolidada

Agregar pruebas sinteticas que cubran: una factura con varios productos y divisiones elegibles crea exactamente un candidato; mezcla de filas elegibles/no elegibles sigue creando uno; ninguna fila elegible crea cero; productos repetidos no duplican; facturas diferentes del mismo customer crean dos candidatos; misma factura repetida en Campaign diferente es independiente; origen distinto con igual numero de factura no colisiona; source/id vacios o inestables se rechazan; agrupacion no divide una factura entre paginas. Verificar que cada Recipient conserva source + invoice ID, que los Attempts siguen enlazados por FK y que `load_id` queda en el snapshot de Execution al implementar el adapter. Reglas de contacto/supresion requieren casos una vez aprobadas. No se ejecutaron pruebas en este cambio de diseno. v0.5 / DRAFT.
## Sprint 3.1D - evidencia PostgreSQL provisional

Se ejecutaron `campaign-postgres-staging.integration.spec.ts` y `survey-campaigns.integration.spec.ts` contra el contenedor local aislado `pec_campaigns_test`, propietario/rol `campaign_test`, PostgreSQL 15.4, tras validar URL, identidad del servidor y presencia de las tablas. Resultado: 2 suites y 27/27 pruebas aprobadas. Cubren los seis catalogos, filtro de padres, OR/AND, consolidacion de facturas antes de paginar, facturas distintas del mismo cliente, igual numero con distinto origen, preview sin PII, correos ausentes/invalidos, referencias incompatibles, fecha invalida, cursor ligado al snapshot, inconsistencias y recorrido sintetico Campaign → Audience → Execution → Recipient → Attempt. No se invoco emBlue.

Fixtures de test usan UUID `load_id` por ejecucion y cleanup limitado a IDs creados por la suite. Los seeds iniciales son 8 filas de catalogo y 10 filas de compra con IDs `MOCK-*` y correos reservados `example.test`; no contienen datos reales. No se uso Neon ni Prisma Migrate. La evidencia valida esta implementacion local, no un contrato BI externo. DRAFT conservado.

| 0.5 | 2026-10-23 | Registra pruebas PostgreSQL locales 27/27 y aislamiento de fixtures; DRAFT conservado. |

## Sprint 3.1D.1 - aislamiento de destino

Las pruebas PostgreSQL siguen apuntando unicamente a `pec_campaigns_test` y a los scripts `bi_campaign_staging_provisional_local_test.sql` / `_local_test_seed.sql`. Los scripts sin sufijo quedaron destinados a desarrollo Neon; no son parte de la suite y no se ejecutaron. `NODE_ENV=production` se mantiene como destino rechazado. DRAFT conservado.

## Sprint 3.1E - pruebas aisladas de invitación (2026-10-23)

Pruebas unitarias cubren emisión opaca y almacenamiento hash, Attempt inexistente, expiración, URL inválida, enlace consumido/repetido, Survey coincidente y asociación de resultado una sola vez. El fixture HTML valida HTTPS y escape de atributo. No se usaron Neon, datos de clientes, endpoints externos ni emBlue. Build NestJS y ESLint focalizado forman parte de la validación de entrega. Las pruebas no prueban el flujo público extremo a extremo, que sigue bloqueado por la incompatibilidad de rutas/submit documentada en API-001. v0.5 / DRAFT.
## Sprint 3.1F - Evidencia y cobertura de invitación (2026-10-23)

Cobertura unitaria: emisión con hash y expiración, relación token/Survey, rechazo de expiración/consumo, respuesta guardada con asociación y pérdida de carrera por compare-and-set; Angular prueba apertura y envío mediante el token reutilizando SurveyJS. Se prepara una prueba PostgreSQL concurrente con fixtures sintéticos en la suite protegida CAMPAIGN_TEST_DATABASE_URL: exactamente una respuesta debe confirmar, un solo SurveyResult queda persistido y la correlación queda consumida. No se ejecuta si no existe la URL local aislada validada. Los resultados de ejecución se mantienen separados de este criterio preparado. v0.5 / DRAFT.

Evidencia de ejecución Sprint 3.1F: Jest Campaign focalizado aprobó 13 suites / 138 pruebas; la prueba Angular de FormComponent aprobó 1 suite / 9 pruebas. Build NestJS y ESLint focalizado aprobaron. La suite PostgreSQL protegida no se ejecutó: `CAMPAIGN_TEST_DATABASE_URL` no estaba configurada, por lo que no hubo conexión ni escrituras. La prueba concurrente queda preparada, no validada contra PostgreSQL en esta entrega. El build Angular a su carpeta `dist` habitual encontró EPERM al intentar retirar `dist/fuse/3rdpartylicenses.txt`; no se eliminó el archivo ni se detuvo ningún proceso. El build de producción a una carpeta de salida temporal terminó correctamente (109.660 s), con advertencias preexistentes de presupuesto de bundle, dependencias CommonJS y hojas de estilo externas ausentes. El workspace Angular no tiene target `lint`. Se conserva v0.5 / DRAFT.

## Sprint 3.1F.1 - validación transaccional pendiente (2026-10-24)

En esta sesión, la prueba unitaria de `CampaignInvitationCorrelationService` pasó: 1 suite / 9 pruebas. La suite PostgreSQL de Campaign se omitió: 1 suite / 7 pruebas omitidas porque `CAMPAIGN_TEST_DATABASE_URL` no estaba definida. No se verificaron concurrencia, rollback ni aislamiento en PostgreSQL real. Docker Engine respondió acceso denegado; `com.docker.service` figura detenido. Aunque el puerto loopback 5432 tiene listener `com.docker.backend`, no se pudo inspeccionar el contenedor que atiende ese puerto ni verificar su configuración; por ello no se conectó ni se configuró URL alguna. No se usó Neon y no hubo escrituras. v0.5 / DRAFT.
## Sprint 3.1F.1 - compatibilidad y evidencia de pruebas (2026-10-24)

Se cubren Survey v1 elegible, versión 2, encuesta que declara `isQueja`, versión desconocida, preflight bloqueado, Schedule/Resume rechazados y apertura/guardado de invitación. Resultado de la suite unitaria focalizada: 14 suites, 157 pruebas aprobadas. Build NestJS: aprobado. ESLint sobre toda `src/survey-campaigns` no pasa por discrepancias Prettier de CRLF en archivos de la carpeta; no se atribuyen automáticamente a este cambio. La prueba PostgreSQL protegida no se ejecutó: `CAMPAIGN_TEST_DATABASE_URL` no estaba configurada y Docker Engine no permitió verificar el contenedor. No hubo conexión a Neon ni escrituras. Concurrencia, rollback y reuso concurrente del token siguen sin certificación real. v0.5 / DRAFT.
## Sprint 3.1G-A - cobertura de simulación sin envíos (2026-10-24)

Pruebas unitarias cubren: deduplicación de factura repetida entre páginas, diferentes facturas, validez de snapshot, rechazo de providers MOCK/no configurados, bloqueo en producción y para estados no DRAFT, gates de Survey, descarte de email inválido, ausencia de PII en respuesta y side effects reportados en cero. Prueba HTTP verifica autorización por lectura Campaign. Build NestJS y ESLint focalizado (reglas distintas de Prettier) aprobados; la suite completa Campaign se registra tras la corrida. No se ejecutó PostgreSQL aislado, Neon, emBlue ni scheduler. La simulación no prueba persistencia, rollback, concurrency ni vínculo ResponseCorrelation/SurveyResult; se requiere PostgreSQL local verificado para esas pruebas. SPECS v0.5 / DRAFT.
Evidencia Sprint 3.1G-A: suite unitaria Campaign aprobada con 15 suites / 165 pruebas; build NestJS aprobado; ESLint focalizado aprobado al desactivar únicamente la regla Prettier que detecta formato CRLF en el checkout. La suite PostgreSQL no se ejecutó en esta fase; no se usaron Neon, emBlue ni scheduler. Resultado de esta simulación no prueba persistencia ni atomicidad. Estado DRAFT conservado.


## Sprint 3.1G-B - simplificación de conexión Campaign (2026-10-25)

Pruebas locales verifican selección por entorno, rechazo de producción/entorno desconocido, identidad Neon fail-closed y consultas del proveedor mediante los nombres cualificados del esquema `bi_integration`. Las pruebas unitarias/build no prueban la lectura funcional de Neon. No se consultó Neon en este cambio porque la identidad de la rama no estaba confirmada; las pruebas de integración Neon siguen condicionadas a esa verificación. No hubo DDL, DML, migraciones, scheduler ni envíos.

Para la validación independiente de rama se requieren solo Project ID, Branch ID, propietario esperado de la base y credencial de API Neon (`CAMPAIGN_PROVISIONAL_NEON_PROJECT_ID`, `CAMPAIGN_PROVISIONAL_NEON_BRANCH_ID`, `CAMPAIGN_PROVISIONAL_NEON_DATABASE_OWNER`, `CAMPAIGN_PROVISIONAL_NEON_API_KEY`). El endpoint se extrae del host de `DATABASE_URL`, y la base y el rol de esa misma URL; estos datos de identidad no seleccionan proveedor ni crean otra conexión. **Registro de revisión:** v0.5, 2026-10-25. Se registra la decisión de conexión única/esquemas separados y selección automática protegida. Estado formal `DRAFT` conservado.


## Sprint 3.1G-B.1 - auditoria de configuracion Campaign (2026-10-25)

Inspeccion local de nombres de variables, sin leer ni publicar valores: `pec-api/.env` y `.env.local` contienen `DATABASE_URL` y `ENV`. `AppModule` carga globalmente `.env.local`, `.env.preprod`, `.env.prod` y `.env`; el guard lee `process.env`. No se encontro `DIRECT_URL` ni una identidad Neon reutilizable en esos archivos. `PROJECT_ID_GCS` identifica GCP y `API_KEY_STAR` pertenece a otra integracion; no son sustitutos de identidad Neon. El inventario local no prueba variables inyectadas externamente al proceso desplegado.

En el checkout, `CampaignNeonTargetGuard` aun exige Project ID, Branch ID, propietario esperado y credencial API Neon. Ninguno aparece en los archivos locales. El error 503 se origina al faltar esa configuracion: el guard falla antes de llamar a la API de Neon y antes de leer el lote en `bi_integration`. Endpoint ID, base y rol se derivan del `DATABASE_URL`; no se requiere otra conexion ni un `DIRECT_URL`. No se modifico `DATABASE_URL`.

No se retiro la verificacion de rama: `ENV` distingue el entorno PEC, pero no identifica una rama Neon; el hostname solo permite derivar el endpoint. La verificacion API Project/Branch/Endpoint y la comprobacion SQL posterior (base, esquema, rol, propietario y primary) siguen siendo controles distintos. El mecanismo equivalente no puede configurarse con los datos que hoy existen en los archivos inspeccionados. Dato pendiente para una lectura segura: una fuente confiable, ya disponible para NestJS, que relacione la conexion actual con el Project ID y Branch ID de desarrollo y permita verificar esa relacion en Neon. Hasta resolverlo, la lectura funcional falla cerrada. Los SPECS permanecen v0.5 / DRAFT.
## Separación de pruebas de lectura y de escritura (2026-10-25)

Las consultas de catálogos en desarrollo son lecturas mediante el `DatabaseService` y la `DATABASE_URL` de PEC, permitidas solo tras autorizar el destino según ARCH/INTEG. La API administrativa Neon no debe ser una dependencia del endpoint en la arquitectura objetivo. Esto no cambia la regla para pruebas transaccionales: cualquier fixture con INSERT/UPDATE/DELETE/concurrencia usa PostgreSQL local aislado o una rama Neon dedicada a pruebas; nunca la rama de desarrollo compartida. La URL de pruebas no puede sustituir silenciosamente `DATABASE_URL` en el proceso normal.

Las pruebas unitarias no demuestran conectividad funcional con Neon. La prueba real de lectura se informa por separado y no debe escribir datos. Estado formal `DRAFT`; versión v0.5.

## Actualización funcional 2026-10-25 — Fuentes y correspondencia de segmentación

Fuentes implementadas: División, Departamento y Clasificación auxiliar se consultan desde `bi_integration.bi_segmentation_catalog`; Empresa, Formato y Local se consultan desde los modelos activos PEC `sspectenterprise`, `sspectformat` y `sspectestablishment`. Las relaciones PEC verificadas son Formato → Empresa mediante `enterpriseId` y Local → Formato mediante `formatId`. Las compras siguen en `bi_integration.bi_customer_purchases` y se consolidan por `(invoice_source, invoice_external_id)` antes de paginar.

El esquema PEC no contiene una columna de mapeo explícita a `company_external_id`, `format_external_id` o `store_external_id` de BI. La implementación solo acepta igualdad textual exacta entre IDs; no compara por nombre/código ni similitud. Si falta igualdad exacta, preview y operaciones dependientes deben bloquearse e informar la correspondencia faltante. Este estado no declara validado el mapeo BI.

`GET /survey-campaigns/:campaignId/audience-preview?page=N` devuelve conteos de clientes con identificador, facturas elegibles/contactables, registros excluidos por correo ausente/inválido y filas consolidadas. La página contiene etiquetas generadas para cliente/factura y fecha, sin correo, ID de cliente ni identificador real de factura. Preview es de solo lectura. Mantener el estado formal DRAFT y versión 0.5.

Los datos BI persistidos y el seed ya ejecutado no se modificaron. El reemplazo por ejemplos sintéticos representativos (taxonomía general de abarrotes/productos y clientes ficticios) requiere una operación de carga autorizada; no ejecutar ni modificar datos en esta entrega. OPEN: confirmar que los IDs externos BI coinciden con las PK PEC o entregar un mapeo autorizado, además de reglas de contactabilidad/exclusión.
