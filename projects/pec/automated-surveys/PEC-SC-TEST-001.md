# PEC-SC-TEST-001 — Estrategia de Pruebas

## Automated Survey Campaigns

| Campo | Valor |
|---|---|
| Documento | PEC-SC-TEST-001 |
| Título | Estrategia de Pruebas - Automated Survey Campaigns |
| Estado | DRAFT |
| Versión | 0.2 |
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
- Usar los roles de prueba que finalmente se mapeen a PEC; hasta entonces, los E2E autenticados de Campaign quedan **OPEN**.
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
| TST-AUTH-01 | Security + E2E | Usuario sin permiso administra Campaign, preview, acciones o recipients. | Backend niega la operación aunque UI oculte/deshabilite acciones. | P0 | Prueba 401/403, audit/log sin PII, matriz de roles aprobada. **OPEN** roles. |
| TST-AUTH-02 | Security + Integration | Manipular `campaignId`, `executionId`, `surveyId` o IDs externos fuera del scope autorizado. | No hay lectura ni modificación cross-scope; respuesta evita filtrar existencia cuando aplique la política IDOR. | P0 | Requests negativos y evidencia de scope/guard. **OPEN** scope final. |
| TST-UI-01 | Unit + Integration + E2E | Buscar por `searchText` (Campaign Name), por `surveySearchText` (Survey Name) y por ambos a la vez; aplicar también `status`, paginación y scope. | Cada búsqueda filtra su nombre; cuando ambas están presentes se combinan con AND; status se combina con las búsquedas y el scope, antes de paginar. Incluye `CANCELLED`. | P0 | Jest Angular; requests/responses API; fixtures DB por scope y páginas; E2E cuando haya runner. |
| TST-UI-02 | Unit + Integration + E2E | Dashboard con Campaigns visibles y no eliminadas lógicamente de todos los estados, incluyendo `CANCELLED`. | Presenta exactamente seis cards: Total, Active, Scheduled, Draft, Suspended y Completed. Total incluye las CANCELLED visibles; CANCELLED aparece en `byStatus`/filtro, sin card propia. | P0 | Conteos API comparados con filas DB dentro del scope; prueba UI que verifica seis cards y ausencia de card CANCELLED. |
| TST-UI-03 | Unit + E2E | Campaign Detail con estados/datos mixtos. | Detalle coincide con API y muestra Scheduling, Segmentation, conteos e historial disponible sin inventar aperturas/clicks. | P1 | Fixture DB/API, comparación UI/API y screenshot/artifact. |

## 6. Matriz — Survey, segmentation, offset y Audience Preview

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-SURVEY-01 | Unit + Integration + E2E | Schedule con Survey existente `PUBLISHED`, accesible y elegible. | Campaign puede programarse si el resto de requisitos es válido. | P0 | Contrato Survey real, resultado del schedule y DB. |
| TST-SURVEY-02 | Unit + Integration | Survey inexistente, archivada, no publicada o fuera de disponibilidad al intentar schedule. | Rechaza con error estable y claro; no crea Execution ni altera calendario. Guardar DRAFT con Survey existente no publicada sigue la regla de API-001. | P0 | Response/error code, estado DB y regla `PUBLISHED` mapeada. **OPEN** campo técnico. |
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
| TST-CAL-01 | Unit + Integration | ONE_TIME con SendDate/SendTime futuros y vencidos; campos de FREQUENCY incompatibles. | Valida campos exclusivos; `nextExecutionAt` es única fecha/hora calculada; no crea ejecución antes del vencimiento. | P0 | Clock fijo, casos límites y persistencia. |
| TST-CAL-02 | Unit + Integration | FREQUENCY con Start, End, SendTime y cada Frequency. | Primera fecha corresponde al slot definido; siguiente fecha usa aritmética de calendario y no duración fija para MONTHLY/QUARTERLY. | P0 | Tabla de casos y fechas esperadas contra regla aprobada. |
| TST-CAL-03 | Unit | Catálogo dinámico para rangos 1–6, 7–13, 14–29, 30–89 y >=90 días. | Solo ofrece y acepta las frecuencias permitidas por FOUND-001 en cada límite del rango. | P1 | Tests parametrizados en días frontera (incluye cambio 6/7, 13/14, 29/30, 89/90). |
| TST-CAL-04 | Unit + Integration | MONTHLY iniciado día 29/30/31 y transición febrero/marzo; siguiente cálculo repetido. | Resultado sigue la política de month-end acordada, determinista y sin drift silencioso. | P0 | Tabla de calendario; aprobación de política OPEN antes de fijar exact expected. |
| TST-CAL-05 | Unit + Integration | QUARTERLY en fin de trimestre/mes y año bisiesto; múltiples ciclos. | Avanza tres meses calendario, respeta anclaje y no suma 90 días fijos. | P1 | Calendario sintético; regla de anclaje OPEN antes de aprobación. |
| TST-CAL-06 | Unit + Integration | Frecuencia permitida/no permitida por duración; frecuencia cae exactamente en EndDate y la siguiente la supera. | Rechaza combinación no permitida; permite slot final según regla de inclusión acordada y marca Campaign COMPLETED cuando no hay siguiente slot válido. | P0 | Matriz de ventanas, fecha límite y transición; inclusión temporal OPEN. |
| TST-CAL-07 | Unit + Integration | `nextExecutionAt` y `lastExecutionAt` entre éxito, partial error y fallo total. | Calendario avanza solo conforme regla aprobada; `lastExecutionAt` tiene semántica estable; fallo total no avanza cutoff exitoso. | P0 | Antes/después DB + eventos de transición; política final OPEN en DATA/SCHED. |
| TST-CAL-08 | Unit + Integration | Zona horaria, DST y hora inexistente/repetida una vez definida. | Misma entrada genera resultado determinista y visible; nunca se interpreta silenciosamente como UTC si se configuró hora local. | P1 | Table tests por timezone; timezone/DST OPEN. |

## 8. Matriz — Scheduler, lifecycles e idempotencia

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-SCH-01 | Unit + Integration | Tick detecta Campaign vencida en SCHEDULED/ACTIVE y campañas futuras/suspendidas/finales. | Solo reclama due elegibles; tick corto, candidatos ordenados y limitados; no se crean crons por Campaign. | P0 | Clock fijo, query spy/DB y configuración de batch. |
| TST-SCH-02 | Integration | Crear una CampaignExecution por `campaignId + executionKey` al repetir tick sobre mismo slot ordinario. | `UNIQUE(campaignId, executionKey)` se cumple; la identidad del slot es determinista; segundo tick encuentra la misma Execution y no crea Execution ni dispatch adicional. | P0 | Constraint PostgreSQL, clave del slot, filas antes/después y conteo de invocaciones adapter. |
| TST-SCH-03 | Integration + Concurrency | Dos o más workers/pods compiten por una Campaign/slot simultáneamente en PostgreSQL. | Exactamente un claim ganador; una Execution/dispatch lógico; perdedores no procesan. | P0 | Prueba concurrente con procesos/conexiones independientes y salida de cada claim. Mocks no bastan. |
| TST-SCH-04 | Unit + Integration | Validar `PENDING → PROCESSING → COMPLETED`, `COMPLETED_WITH_ERRORS` o `FAILED`; intentar saltos/reapertura terminal. | Solo transición válida; estado final histórico no se sobrescribe. | P0 | Tabla de transiciones, HTTP/service rejection y DB history. |
| TST-SCH-05 | Unit + Integration | Campaign DRAFT→SCHEDULED→ACTIVE, suspend/resume/cancel y terminación por EndDate. | Transiciones permitidas; Campaign suspendida/final no se reclama; ACTIVE se edita solo tras suspend. | P0 | Matriz FOUND, eventos de estado y consulta scheduler. |
| TST-SCH-06 | Integration + Concurrency | Dos páginas/workers intentan reservar misma compra y mismo Campaign. | Unique recipient key permite un ganador; conflicto se registra como duplicate sin alterar recipient original. | P0 | Constraint DB, conteos y resultado por ID sintético. |
| TST-SCH-07 | Unit + Integration | Una persona tiene compras distintas y luego reaparece la misma compra en otro ciclo. | Nueva purchase elegible puede generar recipient independiente; purchase repetida no se reenvía en misma Campaign. | P0 | Ledger tras ciclos y llamadas al fake Emblue. |
| TST-SCH-08 | Failure/Recovery + Integration | Worker cae con lease activo/expirado antes del envío, entre lotes o tras respuesta del provider pero antes de persistir; incluye reserva expirada. | Lease vencido de Execution PROCESSING puede recuperarse en la misma Execution si es seguro. La reserva solo devuelve ownership; se inspeccionan ledger, Attempts y resultado del provider. No se repiten envíos confirmados; resultado ambiguo no autoriza resend sin reconciliación/idempotencia confirmada. | P0 | Fault points, snapshot/cursor/ledger/Attempt en DB, invocaciones adapter y logs sin PII; contrato Emblue para ambigüedad OPEN. |
| TST-SCH-09 | Failure/Recovery + Concurrency | Downtime omite varios slots de FREQUENCY; recuperar cuando el API vuelve con páginas múltiples. | Una recuperación consolidada evalúa `PurchaseDate + PostPurchaseOffset <= evaluationDate`. `eligibilityCutoffAt` y `evaluationDate` quedan congelados en snapshot y no cambian entre páginas ni durante recovery. No crea una Execution por slot/tick atrasado ni duplica envíos; FAILED no se reabre. | P0 | Clock controlado, snapshot y keys, relación recovery, cantidad de executions/envíos y nextExecutionAt. Catch-up, timezone y conversiones OPEN. |
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
| TST-EMB-01 | Contract + Integration | Enviar a Emblue Transactional API con `surveyUrl` requerida y variables opcionales permitidas. | Payload conforme al contrato aprobado y al template; `purchaseId/transactionId` no se entrega como variable visible por defecto. | P0 | Contract test + sandbox no productivo con dirección sintética. Endpoint/schema aún OPEN. |
| TST-EMB-02 | Failure/Recovery + Contract | Rechazo total/individual, credencial inválida, timeout antes/después del dispatch y rate limit. | Diferencia errores permanentes/transitorios/ambiguos; política de retry e idempotencia no produce duplicados. | P0 | Resultado por recipient, provider ref y prueba contra sandbox/documentación. Cuotas/semántica OPEN. |
| TST-EMB-03 | Contract + Integration | Respuesta aceptada de Emblue seguida de eventos de entrega/opens/clicks/bounces disponibles. | No confunde request aceptada con delivered; mapea solamente estados/eventos documentados y no inventa métricas. | P1 | Fixtures provider-approved y reconciliación recipient/event. |
| TST-WEB-01 | Contract + Security + Integration | Webhook válido, inválido, firma incorrecta, replay, evento duplicado y evento fuera de orden. | Autenticidad antes de efecto; evento deduplicado/idempotente; transiciones válidas; replay rechazado según mecanismo aprobado. | P0 | Contrato/fixture, respuesta HTTP, event ledger y logs sin payload sensible. Schema/firma OPEN. |
| TST-WEB-02 | Failure/Recovery + Integration | Webhook llega antes/después del cierre, falla persistencia o no se encuentra correlación. | No corrompe Execution terminal ni atribuye a destinatario incorrecto; se registra como evento y no cambia automáticamente el resultado operativo de `CampaignRecipientAttempt`. | P1 | Eventos sintéticos con orden controlado; comparación del estado Attempt y registro Event antes/después. Contrato y regla de reconciliación OPEN. |

Hasta que contratos CF/Emblue/Webhook estén disponibles, no se declara aprobado el bloque Contract. Es válido desarrollar mocks de adapter para pruebas internas, pero no publicar su resultado como compatibilidad externa demostrada.

## 10. Matriz — Security, PII y response correlation

| ID | Categoría primaria / adicional | Escenario y precondiciones | Resultado esperado | Prioridad | Evidencia requerida |
|---|---|---|---|---|---|
| TST-SEC-01 | Security | Introducir email/nombre/IDs de compra/tokens/secretos en errores de CF/Emblue y mensajes de excepción. | Logs y respuestas públicas redactan PII/secreto; usar IDs internos/códigos sanitizados; no se registra payload completo. | P0 | Captura controlada de log + response inspeccionados con assertions de ausencia de canary values. |
| TST-SEC-02 | Security + E2E | Campaign/detail/preview/recipient response incluye accidentalmente email, `purchaseId`, `transactionId` o correlation token. | API no expone campos prohibidos por defecto; se deniega consulta sin permiso específico. | P0 | Snapshot/redaction assertion por ruta y rol. |
| TST-SEC-03 | Security + Integration | Uso de `surveyUrl` con referencia válida, manipulada, expirada o desconocida; enlazar `ResponseCorrelation` al `CampaignRecipientAttempt` o equivalente. | Solo referencia validada en servidor atribuye la respuesta correctamente a Campaign, Execution y Recipient. `surveyUrl` no incluye PII ni IDs de compra/transacción. Webhook no modifica automáticamente el Attempt. Reutilización se valida cuando se apruebe su regla. | P0 | Requests sintéticos, relación persistida de correlación → Attempt/Execution/Recipient y negative cases. Token, expiración, revocación y reutilización OPEN. |
| TST-SEC-04 | Security | Webhook/request intenta injection, replay, sobredimensionado o datos extra no reconocidos. | Rechazo o sanitización segura; no se muta estado ni se filtra stack/SQL. | P1 | Respuesta estándar, log sanitizado, controles de tamaño/validación aprobados. |
| TST-SEC-05 | Security + Integration | Usuario lee Campaign/Execution/recipient sin scope o permiso CASL correspondiente. | 401/403/404 conforme política, sin bypass por URL directa o Angular. | P0 | Matriz roles/abilities y HTTP E2E autenticado. OPEN hasta mapping de roles. |
| TST-SEC-06 | Security | Cargar base con datos sintéticos y buscar texto por email o ID sensible en endpoints/listado. | Búsqueda/ordenamiento no acepta campos no permitidos; logs y resultados respetan minimización. | P1 | Request invalid tests y log/report redacted. |

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
4. `ONE_TIME`, las cinco frecuencias, catálogo dinámico, `nextExecutionAt`, month-end, quarterly y EndDate tienen expected results acordados; calendario OPEN no se maquilla con expectations arbitrarias.
5. Audience Preview y ejecuciones aplican `PurchaseDate + PostPurchaseOffset <= evaluationDate`; congelan `evaluationDate` y `eligibilityCutoffAt` en snapshot y no los cambian entre páginas ni durante recovery. Timezone/conversiones permanecen OPEN hasta aprobación.
6. Integration/concurrency tests en PostgreSQL demuestran `UNIQUE(campaignId, executionKey)`, identidad determinista del slot ordinario, un único claim por slot y deduplicación `campaignId + purchaseId/transactionId` con carreras reales, no solo mocks.
7. Recovery cubre caída en etapas críticas y downtime con varios slots: lease PROCESSING puede recuperarse en la misma Execution si es seguro; FAILED terminal nunca se reabre; retry nuevo enlaza con `recoveryOfExecutionId`; tick repetido no crea Execution/envío duplicado; reserva expirada no autoriza resend ambiguo.
8. Fallo total de CF/Analytics no aparece como audiencia vacía ni procesa pendientes. `COMPLETED_WITH_ERRORS` requiere procesamiento completo conforme a política aprobada; lectura incompleta queda sujeta a política OPEN. No se confunde aceptación técnica de Emblue, `SENT` y `DELIVERED`.
9. Tests de CF/Emblue/Webhooks usan contratos aprobados o permanecen explícitamente bloqueados por OPEN; no se habilita producción con contratos inventados. Webhooks no alteran automáticamente el outcome de Attempt.
10. Los tests de seguridad comprueban permisos backend, aislamiento de scope, ausencia de PII en URLs/logs/responses y ResponseCorrelation correcta Campaign/Execution/Recipient mediante Attempt o equivalente.
11. Dashboard y Campaign Detail presentan datos persistidos y estados verdaderos; Total incluye CANCELLED visibles/no eliminadas, hay exactamente seis tarjetas, búsqueda `searchText` + `surveySearchText` en AND y paginación/scope tras aplicar filtros. CANCELLED es filtro y no tarjeta.
12. Suites existentes y nuevas, build relevante, CI y Quality Gate pasan. El reporte separa unit, integration, E2E, contract, security y concurrency; una suite mock no se reporta como prueba de DB/proveedor real.
13. Ningún fallo P0/P1 aplicable queda aceptado sin disposición escrita; Contract/Failure tests externos bloqueados hasta contratos aprobados; tests son reproducibles, aislados, sin credenciales ni PII reales; documentación de OPENs y ambiente coincide con el commit evaluado.

## 13. OPENs de prueba

| ID | Pendiente | Efecto en QA |
|---|---|---|
| OPEN-TEST-01 | Runner E2E UI y configuración CI de navegador. | Definir herramienta, comando, fixtures y artefactos antes de exigir E2E Angular. |
| OPEN-TEST-02 | `test:integration` Angular apunta a proyecto/target no encontrado en `angular.json`. | Corregir o retirar script como trabajo separado cuando se apruebe; no asumir que corre hoy. |
| OPEN-TEST-03 | Base PostgreSQL descartable, migraciones de campaña y estrategia paralela de reset. | Necesaria para claims, unique constraints, transacciones y carreras. |
| OPEN-TEST-04 | Contrato real, ambiente sandbox y datos sintéticos de CF/Analytics. | Contract/source integration queda pendiente. |
| OPEN-TEST-05 | Contrato real, credenciales no productivas, templates y sandbox de Emblue. | Contract/send/webhook tests no se certifican sin esta dependencia. |
| OPEN-TEST-06 | Roles, CASL abilities y scope de Campaign/recipient. | Matriz de pruebas de autorización requiere definición funcional. |
| OPEN-TEST-07 | `PUBLISHED` técnico, timezone/conversiones de `evaluationDate`/`eligibilityCutoffAt`, DST/month-end/representación de EndDate y catch-up exacto. | No fijar expected tests temporales o de publicación hasta decisión aprobada. |
| OPEN-TEST-08 | Retry, resultado ambiguo, reconciliación/idempotencia Emblue, firma/replay/event ID de webhook y reglas de completitud de lectura/fallo parcial. | Bloque failure/recovery y contract permanece bloqueado hasta definir semántica y contratos; `FAILED` terminal no se reabre. |
| OPEN-TEST-09 | Umbral CI de coverage y calidad, métricas de contract tests y Quality Gate PEC aplicable. | DoD requiere usar Quality Gate vigente sin inventar porcentaje nuevo. |
| OPEN-TEST-10 | Algoritmo definitivo de `executionKey`, modelo físico de `recoveryOfExecutionId` o equivalente y política de recuperación de reservations/Attempts. | Las pruebas verifican invariantes ya acordadas, pero el contrato físico y el flujo de retry requieren decisión; no autorizar resend ambiguo. |
| OPEN-TEST-11 | Token de ResponseCorrelation: emisión, expiración, revocación, reutilización y cardinalidad de respuestas. | Probar casos válidos/negativos en cuanto el mecanismo se apruebe; no inventar reglas de reuso. |

## 14. Estado del documento

- **Status:** DRAFT
- **Version:** 0.2
- **Tests escritos/ejecutados:** ninguno; este documento define estrategia solamente.
- **Pendiente:** cerrar OPENs de contratos, permisos, calendario, runners y persistencia antes de convertir cada escenario en implementación automatizada.

## 15. Historial de cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 0.2 | 2026-10-18 | Alineación de elegibilidad temporal con `evaluationDate` y snapshot dual; criterios de fallos totales/parciales y distinción Emblue; pruebas de identidad de ejecución y recovery; cobertura explícita de tarjetas/búsqueda/paginación; ResponseCorrelation vinculada a Attempt; actualización de DoD y OPENs sin cerrar decisiones pendientes. |
