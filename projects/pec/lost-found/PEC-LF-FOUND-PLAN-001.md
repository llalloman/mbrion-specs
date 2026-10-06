# PEC-LF-FOUND-PLAN-001

## Plan técnico — Registro de artículo encontrado (Sprint 1)

| Campo | Valor |
|---|---|
| ID | PEC-LF-FOUND-PLAN-001 |
| Version | 0.12.0 |
| Estado | BORRADOR |
| Fecha | 2026-10-06 |
| Responsable | Walter Molina |
| Fuentes funcionales | PEC-LF-ARCH-001 v1.2.0; PEC-LF-DATA-001 v1.2.0; PEC-LF-FOUND-001 v1.2.0 APPROVED |

Este documento guia la implementacion de Sprint 1 bajo estrategia database-first: PostgreSQL es la fuente del cambio fisico, los scripts SQL versionados se revisan y aplican controladamente solo en la base DEV, y schema.prisma se sincroniza despues. No se usa Prisma Migrate para crear estructuras de Lost & Found. Las decisiones funcionales de las tres Specs prevalecen; lo que aun requiere decision se identifica como **PENDIENTE TECNICO**.
## 1. Alcance y límites

Se diseñan `LostFoundCatalog`, `LostFoundCatalogOption`, `LostFoundZone`, `LostFoundEstablishmentZone`, `FoundItem`, `sspectlffile`, `FoundItemFile`, `CustodyPolicy` y `LostFoundAudit`. Se reutilizan `sspectenterprise`, `sspectestablishment`, `sspectuser`, `DatabaseModule`/`DatabaseService`, Keycloak, CASL, la consulta PEC de colaboradores y `StorageService`/GCS. No se crea otro `PrismaClient`.

Quedan fuera del Sprint 1 `LostItemReport`, `ItemMatch`, `RecoveryClaim`, `OwnershipValidation`, `ContactAttempt`, `CustodyMovement`, `ItemTransfer`, `ItemDelivery`, `ItemDisposal`, `DocumentDetail` y `CardDetail`. El esquema debe permitir futuras FKs hacia `FoundItem` sin crearlas ahora. Tampoco se implementan matching, entrega, transferencia ni disposición.

## 2. Evidencia del checkout y convenciones

La inspeccion de schema.prisma y de la estructura PostgreSQL configurada para este trabajo confirma tablas sspect*, PK enteras autoincrementales, referencias PEC, UserEstablishment con PK compuesta y convenciones de auditoria. La conexion se identifica para esta tarea como la base DEV designada por el usuario: PostgreSQL 15.19, database neondb y esquema public. Una consulta de catalogo confirmo que no existian las diez tablas, funciones ni triggers LF antes de la aplicacion. Los modelos Prisma LF estan anadidos como reflejo ORM del DDL; las migraciones Prisma historicas no se usan para cambios Lost & Found.

AppModule instala AuthGuard, CaslGuard y SurveyAccessGuard globalmente; los controladores PEC usan PoliciesGuard y @CheckPolicies. Para solicitudes Lost & Found, CaslGuard obtiene la membresía y el rol de la empresa enviada (`enterpriseId`, o `pec` en el resumen explícitamente marcado) al construir las abilities. Las demás rutas mantienen la resolución histórica del rol. Los servicios LF vuelven a validar la membresía seleccionada; los perfiles generales consultan establecimientos activos de esa empresa, mientras Admin Local/Gestor Local quedan limitados a establecimientos asignados al usuario.

CollaboratorController ofrece POST /collaborator/code con {codigoReferenciaEmpleado}; CollaboratorService.getCollaboratorCode(code) llama al servicio laboral externo y devuelve response.data sin DTO tipado. La UI espera codigoReferenciaEmpleado, nombreCompleto y cargoNombre; no hay campos nombre/apellido separados garantizados. La pantalla register_found ya existe, pero usa temas y zonas locales, fecha sin hora, establecimiento informativo, restricciones locales de archivos y botones de guardado deshabilitados.

FileManagerController ofrece POST /file-manager/upload, aplica extension/MIME/limite y usa StorageService.uploadFileGCP; su autorizacion se basa en sspectfile. Su respuesta no crea metadata LF. StorageService incorpora ahora operaciones genéricas de borrado y descarga a buffer para el flujo LF.

FOUND-BE-002 implementa `GET /lost-found/catalogs/:code/options` como lista plana ordenada por `sortOrder` e `id`, con `parentId` para reconstruir la jerarquia. El cliente envía la empresa activa y el backend verifica la asociación en `UserEnterpriseRole`; catálogo y opciones se filtran por esa empresa y solo se devuelven registros activos. El código recibido se normaliza sin distinguir mayúsculas/minúsculas y se consulta contra el código canónico almacenado.

La respuesta incluye tipo funcional declarado y efectivo, indicadores `hasChildren`, `hasRequiredChild` y `selectable`. Al no existir un atributo persistido de hijo obligatorio, en esta Task `hasRequiredChild` significa que existe al menos un hijo activo configurado; si en el futuro se requieren hijos opcionales, el modelo funcional/técnico debe ampliarse antes de distinguirlos. El servicio detecta ciclos, padres inactivos/ausentes y niveles inválidos en la lectura y al validar una opción existente. No se insertaron catálogos/opciones base porque no se indicó un enterprise objetivo ni se aprobaron opciones funcionales.

FOUND-BE-003 implementa `GET /lost-found/establishments/:id/zones?enterpriseId=...`. El backend valida la membresía y obtiene el rol de esa empresa antes de comprobar el establecimiento activo, su pertenencia a la empresa y, para Admin Local/Gestor Local, la asignación del usuario. Los perfiles generales pueden consultar cualquier establecimiento activo de la empresa seleccionada. Solo se devuelven vínculos y zonas activos, ordenados por nombre/código/id; `[]` indica que falta configuración y no inserta zonas por defecto. BE-006 reutiliza `LostFoundZoneService.validateZoneForEstablishment({zoneId, establishmentId, enterpriseId})` antes de crear FoundItem.

## 3. Modelo físico propuesto

**Convenciones para todas las tablas LF, reflejadas en DDL PostgreSQL:** PK `INTEGER GENERATED BY DEFAULT AS IDENTITY` (Prisma `Int @id @default(autoincrement())`), salvo la clave técnica compuesta indicada en §6. `created_by_user TEXT NOT NULL`, `created_date TIMESTAMP(3) NOT NULL DEFAULT CURRENT_TIMESTAMP`, `last_modified_by_user TEXT NOT NULL`, `last_modified_date TIMESTAMP(3) NOT NULL`, `version INTEGER NOT NULL DEFAULT 1`, `created_from_ip TEXT NOT NULL`, `updated_from_ip TEXT NOT NULL` reproducen el patrón PEC; su valor lo fija el backend, nunca el cliente. Los registros operativos usan `status INTEGER NOT NULL DEFAULT 1` donde se indica. Los tipos, longitudes y nombres propuestos se verificaron contra el DDL aplicado en DEV. Todas las FKs históricas usan `ON DELETE RESTRICT`/`NO ACTION`; no hay borrado en cascada del artículo o sus evidencias. Configuración referenciada se desactiva/versiona; transacciones y auditoría se conservan.

En las tablas siguientes, `!` significa NOT NULL, `?` nullable; `id` significa PK entera identity; `std` representa los campos PEC anteriores. Una FK marcada con `→` tiene índice para búsquedas y se valida además en la capa de servicio cuando la FK sola no expresa pertenencia/actividad.

### 3.1 `sspectlfcatalog` (`LostFoundCatalog`)

| Columnas propuestas | Restricciones y acceso |
|---|---|
| `id`!, `enterpriseId INTEGER`!, `code VARCHAR(80)`!, `name VARCHAR(200)`!, `description TEXT`?, `active BOOLEAN`! DEFAULT true, `std` | FK `enterpriseId → sspectenterprise`; UNIQUE `(enterpriseId, code)`; CHECK `code <> ''`; índice `(enterpriseId, active)`. Desactivación lógica, sin eliminación si tiene opciones. |

### 3.2 `sspectlfcatalogoption` (`LostFoundCatalogOption`)

| Columnas propuestas | Restricciones y acceso |
|---|---|
| `id`!, `enterpriseId INTEGER`!, `catalogId INTEGER`!, `parentId INTEGER`?, `optionType VARCHAR(20)`?, `functionalType VARCHAR(30)`?, `code VARCHAR(80)`!, `label VARCHAR(200)`!, `value TEXT`?, `sortOrder INTEGER`! DEFAULT 0, `active BOOLEAN`! DEFAULT true, `std` | FKs a empresa, catálogo y padre; UNIQUE `(enterpriseId, catalogId, parentId, code)` **no basta para raíces con NULL**: usar índice único PostgreSQL con `NULLS NOT DISTINCT` si la versión lo soporta, o dos índices parciales raíz/hijo. Índices `(catalogId,parentId,active,sortOrder)` y `(enterpriseId,code)`. CHECK `parentId <> id`, `optionType` en CATEGORY/THEME/SUBTHEME para clasificación. Desactivación, no borrado histórico. |

FK compuesta propuesta `(parentId,catalogId,enterpriseId) → (id,catalogId,enterpriseId)` con UNIQUE auxiliar en destino impide padre de otro catálogo/empresa. Un trigger de integridad o escritura serializada con comprobación transaccional impide ciclos y valida niveles CATEGORY → THEME → SUBTHEME, con máximo tres y terminación válida en cualquier nivel. `functionalType` nullable es la decisión física de este Plan para implementar el tipo funcional aprobado en DATA/FOUND. Mapea una rama de clasificación a tipos usados por CustodyPolicy; los valores iniciales conocidos son DOCUMENT y FOOD. Puede definirse en el nodo final o heredarse del ancestro más cercano que lo defina, siempre dentro de la misma rama. Por ejemplo, una categoría Documentos con `functionalType=DOCUMENT` permite que sus temas Pasaporte y Cédula lo hereden. Nunca se infiere de `label` o de otro texto; se rechazan configuraciones contradictorias de la rama cuando corresponda. Sin `functionalType` en el nodo ni en sus ancestros, se intenta la política GENERAL; GENERAL no necesita almacenarse como `functionalType`. El servicio impide seleccionar un padre cuando un hijo configurado exige selección. Estas validaciones también se ejecutan al administrar opciones; no se confía solo en la UI.

### 3.3 `sspectlfzone` (`LostFoundZone`) y `sspectlfestablishmentzone` (`LostFoundEstablishmentZone`)

| Tabla | Columnas propuestas | Restricciones y acceso |
|---|---|---|
| `sspectlfzone` | `id`!, `code VARCHAR(80)`!, `name VARCHAR(200)`!, `description TEXT`?, `active BOOLEAN`! DEFAULT true, `std` | Catálogo global del dominio LF; UNIQUE `code`; CHECK no vacío; índice `active,name`. No borrar zonas referenciadas. |
| `sspectlfestablishmentzone` | `id`!, `establishmentId INTEGER`!, `zoneId INTEGER`!, `active BOOLEAN`! DEFAULT true, `std` | FKs a establecimiento y zona; UNIQUE `(establishmentId,zoneId)`; índice `(establishmentId,active)` y `(zoneId,active)`. Desactivar el vínculo, no borrarlo si hay historia. |

`FoundItem.zoneId` es NOT NULL. Antes del alta, validar zona y vínculo activos, mismo establecimiento, y que haya al menos una zona habilitada; si no hay, responder que requiere configuración previa. `id_zona` territorial PEC no interviene. La FK compuesta opcional `(currentEstablishmentId,zoneId)` hacia el vínculo exige UNIQUE `(establishmentId,zoneId)` y obliga a que la relación exista; la condición `active` permanece en servicio. **PENDIENTE TÉCNICO:** decidir si esa FK compuesta se aplica al custodio actual, porque transferencias futuras podrían cambiar `currentEstablishmentId` sin cambiar la zona histórica del hallazgo. Para no bloquear Sprint posterior, se recomienda conservar FK simple de `zoneId`, validar vínculo al crear y definir explícitamente cómo se actualiza zona en una transferencia futura.

### 3.4 `sspectlfcustodypolicy` (`CustodyPolicy`)

| Columnas propuestas | Restricciones y acceso |
|---|---|
| `id`!, `enterpriseId INTEGER`!, `policyCode VARCHAR(80)`!, `scopeType VARCHAR(20)`! (`CLASSIFICATION`,`FUNCTIONAL_TYPE`,`GENERAL`), `scopeKey VARCHAR(80)`!, `classificationOptionId INTEGER`?, `functionalType VARCHAR(30)`?, `policyVersion INTEGER`!, `active BOOLEAN`! DEFAULT true, `effectiveFrom TIMESTAMP(3)`!, `localDays INTEGER`?, `centralDays INTEGER`?, `alertDays INTEGER`?, `destinationEstablishmentId INTEGER`?, `responsibleUserId INTEGER`?, `finalAction VARCHAR(30)`?, `requiresTransfer BOOLEAN`! DEFAULT false, `requiresEvidence BOOLEAN`! DEFAULT false, `parameters JSONB`?, `std` | FKs a empresa, clasificación, establecimiento destino y usuario; UNIQUE `(enterpriseId,scopeType,scopeKey,policyVersion)`; índices `(enterpriseId,scopeType,scopeKey,policyVersion DESC)` y `(classificationOptionId)`; CHECK versiones positivas, días no negativos y coherencia de scope/campos. RESTRICT para históricos. |

`scopeKey` no nulo evita ambigüedad de UNIQUE con NULL (`classificationOptionId` codificado como clave estable en scope CLASSIFICATION, código DOCUMENT/FOOD en FUNCTIONAL_TYPE y `GENERAL` en GENERAL); el servicio comprueba correspondencia con FKs. Cada fila es una versión inmutable: no editar parámetros ni `active` después de insertar. Una nueva configuración, incluida una desactivación, inserta una versión superior; para cada scope se toma la última versión efectiva al instante del registro, y solo si `active=true` se considera aplicable. Una última versión inactiva permite continuar con el nivel de prioridad siguiente. La creación/versionado de políticas serializa por scope y comprueba UNIQUE. No existe `CustodyPolicyVersion`.

### 3.5 `sspectfounditem` (`FoundItem`)

| Columnas propuestas | Restricciones y acceso |
|---|---|
| `id`!, `itemCode VARCHAR(16)`!, `codeYear INTEGER`!, `enterprise_id INTEGER`!, `currentEstablishmentId INTEGER`!, `classificationOptionId INTEGER`!, `zoneId INTEGER`!, `name VARCHAR(200)`!, `description TEXT`?, `foundPlace TEXT`!, `comment TEXT`?, `foundAt TIMESTAMP(3)`!, `foundTimeKnown BOOLEAN`! DEFAULT false, `finderType VARCHAR(20)`!, `finderEmployeeCode VARCHAR(30)`?, `finderName VARCHAR(200)`?, `finderLastName VARCHAR(200)`?, `finderPosition VARCHAR(200)`?, `finderNote TEXT`?, `registeredByUserId INTEGER`!, `status VARCHAR(30)`! DEFAULT 'REGISTERED', `custodyPolicyId INTEGER`!, `expirationDate TIMESTAMP(3)`?, `std` | FKs a empresa, establecimiento, opción, zona, usuario y política; UNIQUE `(enterprise_id,itemCode)`; CHECK formato `LF-F-AAAA-XXXXXX`, `codeYear` igual al año incluido en `itemCode`, `finderType` permitido, status inicial/dominio permitido y campos condicionales del finder. Índices `(enterprise_id,codeYear)`, `(enterprise_id,currentEstablishmentId,status)`, `(enterprise_id,classificationOptionId)`, `(enterprise_id,zoneId)`, `(enterprise_id,custodyPolicyId)`, `(enterprise_id,expirationDate)`. Sin borrado físico. |

La unicidad funcional es por empresa/año, y el año forma parte de `itemCode`; por eso basta UNIQUE `(enterprise_id,itemCode)`. `codeYear` permanece como dato técnico del correlativo, las consultas y la validación del año codificado. El correlativo de seis dígitos no se recicla. `enterprise_id` se deriva del establecimiento autorizado y se valida contra catálogo/política. Para `COLLABORATOR` son obligatorios código, nombre, apellido y cargo; para `CUSTOMER`, nombre y apellido; para `OTHER`, ambos opcionales; `finderNote` siempre opcional. Los CHECK de formato y finder se implementan en SQL de migración si Prisma no los expresa. La fecha del hallazgo es obligatoria, hora opcional. Si `foundTimeKnown=false`, `foundAt` conserva la fecha con un valor técnico de hora que jamás se presenta como hora informada; la normalización y zona horaria concretas se fijan en §5 y pendientes. `expirationDate` puede ser NULL cuando la política no produce fecha al crear.

`name VARCHAR(200) NOT NULL` y `description TEXT NULL` conservan la decisión funcional de FOUND-001 v1.0.0: el nombre es obligatorio y la descripción no fue definida como obligatoria.

### 3.6 `sspectlffile` y `FoundItemFile`

| Tabla | Columnas propuestas | Restricciones y acceso |
|---|---|---|
| `sspectlffile` | `id`!, `enterpriseId INTEGER`!, `objectKey TEXT`!, `originalName TEXT`!, `mimeType VARCHAR(255)`!, `sizeBytes BIGINT`!, `uploadState VARCHAR(20)`! (`STAGED`,`ATTACHED`,`FAILED`), `uploadTokenHash VARCHAR(128)`?, `stagedUntil TIMESTAMP(3)`?, `uploadedByUserId INTEGER`!, `std` | FKs a empresa/usuario; UNIQUE `objectKey`, UNIQUE `uploadTokenHash` cuando no nulo; CHECK `sizeBytes > 0` y estado permitido; índices `(enterpriseId,uploadState,stagedUntil)` y `(uploadedByUserId,uploadState)`. Sin FK a `sspectfile`; borrado físico solo mediante política de retención autorizada. |
| `FoundItemFile` (nombre físico propuesto `sspectlffounditemfile`) | `id`!, `foundItemId INTEGER`!, `fileId INTEGER`!, `filePurpose VARCHAR(20)`! (`PHOTO`,`EVIDENCE`), `std` | FKs a `FoundItem` y `sspectlffile`; UNIQUE `(foundItemId,fileId)` y UNIQUE `(fileId)` para impedir doble asociación; índices `(foundItemId,filePurpose)` y `(fileId)`. RESTRICT. |

`PHOTO` y `EVIDENCE` expresan propósito, no extensiones ni MIME. La asociación requiere metadata LF aceptada, objeto GCS existente, mismo enterprise y token de staging vigente; esas condiciones cruzadas se validan en el servicio/transacción. El nombre físico de asociación sigue el patrón `sspect*` del repositorio y queda sujeto a cotejo con el catálogo PostgreSQL.

### 3.7 `sspectlfaudit` (`LostFoundAudit`)

| Columnas propuestas | Restricciones y acceso |
|---|---|
| `id`!, `enterprise_id INTEGER`!, `entityType VARCHAR(80)`!, `entityId INTEGER`!, `eventType VARCHAR(80)`!, `userId INTEGER`!, `username VARCHAR(255)`!, `establishmentId INTEGER`!, `eventAt TIMESTAMP(3)`! DEFAULT CURRENT_TIMESTAMP, `previousValue JSONB`?, `newValue JSONB`?, `observations TEXT`?, `std` | FKs a empresa, usuario y establecimiento; índices `(enterprise_id,entityType,entityId,eventAt)` y `(enterprise_id,eventType,eventAt)`. Append-only y RESTRICT; ambos snapshots son nullable en el modelo genérico. |

La auditoría se escribe en la misma transacción que FoundItem. Para `FOUND_ITEM_CREATED`, `previousValue=NULL` y `newValue` es obligatorio a nivel de `LostFoundAuditService`, con campos pertinentes y sin datos sensibles innecesarios; otros eventos podrán omitir cualquiera de los snapshots según su contrato. JSONB es coherente con los campos Prisma `Json` ya existentes, pero la política exacta de anonimización/acceso del snapshot se revisa antes de migrar.

## 4. DDL PostgreSQL y sincronizacion Prisma

PostgreSQL es la fuente de los cambios fisicos. FOUND-BE-001 crea scripts versionados en pec-api/sql/lost-found/, separados por dependencia. Se revisan y ejecutan en orden unicamente contra la base DEV designada para la tarea. La aplicacion se hace como una unidad transaccional para evitar un estado parcial; no se incluyen INSERT funcionales ni se modifica data existente.

Orden de ejecucion:

1. 001_lf_catalog.sql ? catalogos y opciones.
2. 002_lf_zones.sql ? zonas y vinculos con establecimientos.
3. 003_lf_custody_policy.sql ? politicas inmutables.
4. 004_lf_item_counter.sql ? correlativo tecnico por empresa/ano.
5. 005_lf_found_item.sql ? articulo encontrado.
6. 006_lf_files.sql ? metadata LF y asociacion de archivos.
7. 007_lf_audit.sql ? auditoria.

La consulta previa confirmo que los diez nombres de tabla y los nombres de funciones/triggers estaban libres. Los siete scripts pasaron la prueba transaccional con ROLLBACK y no dejaron objetos persistentes. Tras revisar el DDL, se aplicaron en una unica transaccion a DEV, sin INSERT ni cambios de data funcional. El resultado fue 10 tablas, 49 indices, 50 constraints y 3 triggers. La comparacion de 161 columnas contra schema.prisma no encontro diferencias. Se ejecuto prisma format y prisma validate correctamente. La generacion de Prisma Client se intento, pero fallo con EPERM al reemplazar query_engine-windows.dll.node; queda pendiente reintentar cuando se libere el archivo bloqueado.

El proyecto opera actualmente bajo estrategia database-first. El historial Prisma Migrate no se utiliza para aplicar cambios de Lost & Found; las migraciones historicas pendientes no bloquean FOUND-BE-001 y no se reconcilian dentro de esta Task. La situacion del historial queda como deuda tecnica general del repositorio.

## 5. CustodyPolicy, fechas y clasificación

Validar `classificationOptionId` activo, enterprise, catálogo y cadena padre. Una rama puede acabar en CATEGORY, THEME o SUBTHEME. Resolver el tipo funcional desde `LostFoundCatalogOption.functionalType` del nodo seleccionado o del ancestro definido más cercano, nunca por palabras del nombre o `label` del artículo; en ausencia de mapeo, intentar GENERAL. Para cada scope consultar la versión efectiva más alta y decidir: específica por clasificación, luego tipo funcional, luego GENERAL. Si no hay ninguna activa, responder error de configuración y no insertar artículo. Empates de versión se impiden por UNIQUE. Registrar el `id` exacto seleccionado como `custodyPolicyId`.

Propuesta de cómputo: las reglas basadas en días usan parámetros de la versión aplicada y fecha/hora de inicio definida en esa política; trabajar con una zona horaria empresarial/establecimiento explícita y persistir el resultado como instante inequívoco. Para DOCUMENT se calcula inicialmente el hito local (30 días); los 90 días en GCSS y la destrucción son hitos posteriores de custodia, no acciones de Sprint 1. Para GENERAL se calcula el hito de 90 días. Para FOOD, si el cierre local puede obtenerse de una fuente aprobada, usar ese cierre; si no hay hora de cierre confiable, guardar `expirationDate=NULL` y dejar el hito operativo pendiente, sin inventar hora ni omitir `custodyPolicyId`. `foundTimeKnown=false` obliga a mostrar solo la fecha del hallazgo; el valor horario técnico no es evidencia de una hora informada.

FOUND-BE-004 implementa `CustodyPolicyService.resolveForFoundItem({ enterpriseId, classificationOptionId, registrationAt })`. El servicio reutiliza `LostFoundCatalogService.validateOption`, consulta exclusivamente la empresa indicada y resuelve la última versión efectiva por `policyVersion` para cada scope, incluyendo versiones inactivas al decidir si el scope sigue habilitado. La prioridad es CLASSIFICATION → FUNCTIONAL_TYPE → GENERAL; una versión inactiva o inexistente permite continuar al siguiente scope. Si no hay política activa, el servicio bloquea el alta con `LOST_FOUND_CUSTODY_POLICY_NOT_CONFIGURED`. Valida coherencia de scope, versión y días. Las filas de política ya son inmutables en PostgreSQL mediante trigger; esta Task no expone administración ni actualiza versiones.

El proceso PEC fija `TZ=UTC`; el cálculo inicial suma `localDays × 24 horas` al instante de registro del servidor, nunca a `foundAt`. DOCUMENT usa el hito local configurado y GENERAL el plazo configurado en `localDays`. FOOD devuelve `expirationDate=NULL` porque no existe fuente PEC confiable para el cierre del local; el helper acepta un instante de cierre solo desde una futura integración backend confiable. No se insertan políticas iniciales: sin política activa para alguno de los scopes aplicables, el alta queda bloqueada.

**PENDIENTE TÉCNICO:** confirmar si los días de custodia significan períodos transcurridos de 24 horas o días calendario locales. La implementación vigente adopta 24 horas como estrategia explícita para usar un instante inequívoco en UTC. También siguen pendientes una fuente de cierre verificable para FOOD y la representación de `foundAt` cuando se conoce solo la fecha. Estas decisiones no cambian los plazos ni la prioridad funcional aprobados.

## 6. itemCode y concurrencia

La tabla auxiliar técnica `sspectlfitemcounter(enterprise_id,codeYear,nextValue,exhausted)` tiene PK `(enterprise_id,codeYear)`, FK a enterprise y CHECK de rango `1..999999`. `nextValue` es el siguiente correlativo emitible. FOUND-BE-005 implementa `ItemCodeService.reserveNextCode({ enterpriseId, registrationAt, tx })`; recibe el cliente de la transacción externa, crea el contador si falta, bloquea la fila con SQL parametrizado `SELECT ... FOR UPDATE`, valida su estado y actualiza dentro de la misma transacción. Una fila agotada conserva `nextValue=999999`; emitir ese último número marca `exhausted=true`, y las reservas siguientes fallan con `LOST_FOUND_ITEM_CODE_EXHAUSTED`. El formato se centraliza como `LF-F-AAAA-XXXXXX`; el año usa `registrationAt.getUTCFullYear()`, nunca `foundAt`. La unicidad por enterprise/código también queda protegida por el UNIQUE de FoundItem. Un estado imposible o fuera de rango falla cerrado sin corregirlo.

La prueba de concurrencia se ejecutó en una base temporal aislada PostgreSQL 17: 12 transacciones simultáneas para el mismo enterprise/año produjeron las secuencias únicas 1..12. El test eliminó y verificó la limpieza de su contador temporal. Usó una fila enterprise y la estructura de la tabla contador; no escribió en DEV/Neon ni creó FoundItem. Si una transacción externa hace rollback, también revierte el avance del contador y puede reutilizarse el número si nunca se confirmó ni mostró como registro exitoso. No se expone código antes del commit.

Una secuencia global evita duplicados pero no entrega un correlativo propio por empresa/año; múltiples secuencias dinámicas complican despliegue y mantenimiento. La tabla de contadores cumple directamente el scope aprobado y la prueba real confirma serialización. El año se obtiene del instante de registro en UTC, no del `foundAt` histórico. **PENDIENTE TÉCNICO:** confirmar si UTC es la zona oficial que debe regir el cambio de año cuando se defina timezone empresarial/establecimiento.

## 7. Archivos y consistencia GCS/PostgreSQL

FOUND-BE-007 implementa `POST /lost-found/files?enterpriseId=...` con permiso `lostFoundFile` y `multipart/form-data` (`file`, `purpose`). La empresa se valida contra la asociación del usuario. Extensiones, MIME y límite técnico salen de `PecFileUploadPolicy`, compartida con `FileManagerController`; el endpoint actual de FileManager conserva sus validaciones y comportamiento. Lost & Found no usa `sspectfile`: persiste su metadata en `sspectlffile` y el vínculo en `FoundItemFile`. `StorageService` conserva GCS como almacenamiento físico.

Flujo implementado: validar el archivo antes de GCS, subirlo con una clave aleatoria `lost-found/{enterpriseId}/{uuid}`, insertar metadata STAGED y devolver token criptográficamente aleatorio. Solo se almacena SHA-256(token + propósito), de modo que PHOTO/EVIDENCE queda ligado al token sin alterar DDL. El token vence según `LOST_FOUND_FILE_STAGING_TTL_MINUTES` (default técnico de 30 minutos para DEV), y se liga al usuario y enterprise autenticados. En el POST de FoundItem, el servicio verifica propietario, enterprise, estado, vencimiento y existencia GCS; dentro de la misma transacción cambia STAGED a ATTACHED, invalida el hash y crea `FoundItemFile`. No se acepta `objectKey` del cliente. La descarga valida el acceso al FoundItem y su asociación antes de obtener bytes de GCS.

GCS queda fuera de la transacción ACID. Si falla el upload, no se crea metadata. Si falla el insert STAGED después de subir, se intenta borrar el objeto; un fallo de compensación se registra en el log para seguimiento. Si falla la creación del FoundItem o la asociación, PostgreSQL revierte contador, artículo y vínculos; el archivo permanece STAGED hasta vencer. `findExpiredStagedFiles` y `cleanupExpiredStagedFiles` dejan lógica idempotente: excluyen filas asociadas, pasan expirados a FAILED y eliminan/reintentan el borrado en GCS. No se incorporó scheduler porque el prompt excluye esa tarea si no hay job simple ya existente. `StorageService` ahora ofrece borrado controlado y descarga a buffer, genéricos y reutilizables. **PENDIENTE TÉCNICO:** configurar el TTL de producción, política de retención y operación periódica del cleanup; confirmar el procedimiento para investigar huérfanos cuyo insert metadata y compensación de GCS fallen ambos.

## 8. Contratos HTTP y DTOs propuestos

Las rutas siguen el patrón NestJS de controladores PEC. No existen aún estas rutas LF.

| Método y ruta propuesta | Contrato/responsabilidad |
|---|---|
| `POST /lost-found/found-items?enterpriseId=...` | `CreateFoundItemDto`; el contexto de empresa se valida contra la membresía y el establecimiento. Crea y devuelve `FoundItemResponseDto`, id y itemCode. |
| `GET /lost-found/found-items/:id?enterpriseId=...` | Lee dentro de la empresa seleccionada y autorizada; incluye clasificación, zona, política aplicada y archivos accesibles. |
| `GET /lost-found/catalogs/:code/options?enterpriseId=...` | Consulta jerárquica activa por empresa asociada; para clasificación devuelve `id,parentId,optionType,code,label,hasRequiredChild`. El código acepta mayúsculas/minúsculas. |
| `GET /lost-found/establishments/:id/zones?enterpriseId=...` | Zonas LF activas con vínculo activo y establecimiento permitido por la empresa y el rol; lista vacía indica configuración requerida. |
| `POST /lost-found/files?enterpriseId=...` | Multipart `file` + propósito PHOTO/EVIDENCE, valida membresía y devuelve token de staging, no clave GCS editable. |
| `GET /lost-found/found-items/:id/files/:fileId?enterpriseId=...` | Descarga protegida con validación de empresa, relación y permiso del recurso. |

No se propone `PUT/PATCH` de FoundItem en Sprint 1: la Spec aprobada cubre creación; editar estado, política o custodia requiere flujo propio. No exponer administración de políticas a usuarios registradores. La API administrativa de catálogo/zonas/políticas se diseña en tareas separadas si es indispensable para configuración inicial, con permisos distintos de registro.

`CreateFoundItemDto` acepta `establishmentId`, `classificationOptionId`, `zoneId`, `name`, `description?`, `comment?`, `foundAt` (fecha y hora si conocida), `foundTimeKnown`, `foundPlace`, `finderType`, `finderEmployeeCode?`, `finderName?`, `finderLastName?`, `finderPosition?`, `finderNote?`, `fileTokens?` con propósito funcional. Nombre, lugar y fecha son obligatorios; las reglas condicionales del finder siguen FOUND-001. Para COLLABORATOR se envía el código/selección y el backend obtiene y valida el snapshot, sin confiar en nombres editables del cliente. El contrato debe rechazar o ignorar explícitamente campos de solo servidor: `itemCode`, `enterprise_id`, `registeredByUserId`, status, `custodyPolicyId`, `expirationDate` y auditoría. Se recomienda rechazar propiedades no permitidas de forma uniforme. La empresa y el usuario se derivan de la sesión y del establecimiento autorizado.

`FoundItemResponseDto` expone `id`, `itemCode`, `enterpriseId`, `currentEstablishmentId`, `classificationOptionId`, `zoneId`, datos de hallazgo y `foundTimeKnown`, finder permitido, estado `REGISTERED`, `custodyPolicyId`, `expirationDate`, archivos asociados y fecha de creación; omitir horas inferidas cuando `foundTimeKnown=false`. La política visible puede resumirse por código/versión, sin permitir modificarla. Validar números/fechas/enum en DTO y pertenencia/actividad en servicio. Respuestas de error distinguibles: 400 datos inválidos, 403 acceso, 404 recurso no visible, 409 conflicto de unicidad/token y 422 configuración insuficiente; armonizar con el filtro de errores existente antes de fijar códigos finales.

Ejemplo conceptual, sin campos confiables de servidor:

```json
{
  "establishmentId": 123,
  "classificationOptionId": 456,
  "zoneId": 789,
  "name": "Bolso",
  "foundPlace": "Entrada principal",
  "foundAt": "2026-10-05",
  "foundTimeKnown": false,
  "finderType": "CUSTOMER",
  "finderName": "Ana",
  "finderLastName": "Pérez",
  "finderNote": "Entregó el artículo en servicio al cliente",
  "fileTokens": [{ "token": "<token-de-staging>", "purpose": "PHOTO" }]
}
```

La forma final de fecha/hora, nombres de propiedades y tokens se cerrará junto al DTO y cliente; el ejemplo expresa las reglas aprobadas, no un endpoint ya existente.

## 9. Servicios, transacción y seguridad

`FoundItemService` orquesta; `LostFoundCatalogService` valida jerarquía y sirve opciones; `LostFoundZoneService` verifica vínculo; `CustodyPolicyService` selecciona versión y calcula vencimiento con estrategia por regla; `ItemCodeService` reserva correlativo; `LostFoundFileService` controla staging/asociación/limpieza; `LostFoundAuditService` construye el evento. Todos reciben `DatabaseService` o el cliente transaccional derivado; no instancian Prisma.

FOUND-BE-006 implementa `POST /lost-found/found-items?enterpriseId=...` y `GET /lost-found/found-items/:id?enterpriseId=...`, con DTO estricto y contexto de empresa seleccionada validado contra `UserEnterpriseRole`; el usuario se deriva de AuthGuard. Reutiliza el acceso local de FOUND-BE-003 y revalida empresa/local/zona/clasificación en la transacción, además de reservar el código mediante FOUND-BE-005 antes del insert. `foundTimeKnown=false` acepta solo `YYYY-MM-DD`, persiste medianoche UTC y responde solo la fecha; con hora conocida exige instante ISO con zona horaria. CUSTOMER y OTHER siguen sus reglas de campos; status, política, vencimiento y código son derivados del backend. Las 13 pruebas focalizadas pasan; `nest build` y ESLint focalizado pasan. No se ha ejecutado contra DEV porque no hay fixture de FoundItem aislado que permita verificar rollback sin dejar datos.

FOUND-BE-008 escribe `FOUND_ITEM_CREATED` inmediatamente después de insertar FoundItem y asociar los archivos, dentro del mismo `$transaction` de Prisma. `LostFoundAuditService.recordEvent({ tx, ... })` recibe el cliente transaccional existente; usa usuario, username, IP, enterprise y establecimiento del contexto autenticado/validado y genera timestamp en backend. El registro exige `newValue`, guarda `previousValue` como SQL NULL y propaga cualquier fallo para que se reviertan artículo, contador y asociaciones. No se exponen operaciones update/delete ni endpoints de auditoría.

El snapshot se construye explícitamente con `itemCode`, `enterpriseId`, `currentEstablishmentId`, `classificationOptionId`, `zoneId`, `name`, `foundPlace`, `foundAt`, `foundTimeKnown`, `finderType`, `registeredByUserId`, `status`, `custodyPolicyId`, `expirationDate`, cantidad y propósitos de archivos asociados. Las fechas se normalizan a ISO; una sanitización defensiva elimina claves que incluyan token, objectKey, hash, secret, password o authorization. No se incluye nombre/apellido/cargo del finder ni claves o identificadores de GCS; la política formal de minimización de estos snapshots queda pendiente. Las pruebas unitarias focalizadas pasan y `nest build` valida el código. La atomicidad real se basa en el `$transaction` PostgreSQL de Prisma; no se hizo escritura de prueba en DEV.

La rama COLLABORATOR reconsulta el servicio laboral usando el codigo, pero la forma consumida actualmente solo garantiza un `nombreCompleto` y no permite separar nombre/apellido de forma fiable. La rama falla cerrada con `LOST_FOUND_COLLABORATOR_SNAPSHOT_UNRESOLVED`; no persiste el articulo ni acepta el nombre/cargo del cliente. Por este pendiente, FOUND-BE-006 continua IN PROGRESS y no satisface su Definition of Done.

Secuencia: autenticar y resolver usuario/empresa/establecimiento permitido; validar DTO y finder; verificar clasificación y zona; resolver política y vencimiento; validar archivos STAGED; iniciar transacción PostgreSQL; reservar correlativo, insertar FoundItem, asociar metadata aceptada, insertar `FOUND_ITEM_CREATED`; confirmar; devolver resultado. La auditoría fallida revierte toda la operación. Las verificaciones sensibles se repiten o protegen dentro de la transacción para evitar cambios de configuración concurrentes; FK/UNIQUE/CHECK son última barrera. Reintentos por conflicto se acotan; no se expone código hasta confirmar. No hay llamada GCS dentro del bloqueo de contador. La carga ya se completó en staging.

Aplicar `AuthGuard`/Keycloak, `CaslGuard` y `@UseGuards(PoliciesGuard)` con `@CheckPolicies` explícito para crear, leer, subir y descargar; añadir sujeto LF y permisos a los roles funcionales Admin Local, Redes Sociales, Gestor Local y Gestión de Quejas. En el código actual existen `admin_local`, `redes_sociales` y `gestor_quejas`; no se encontró constante `gestor_local`. **PENDIENTE TÉCNICO:** confirmar mapeo de Gestor Local en Keycloak/PEC sin sustituir un rol por otro. Verificar membresía del usuario en la empresa seleccionada y establecimientos habilitados, estado del usuario/local, clasificación y política del mismo enterprise, zona/vínculo activos, y acceso al artículo/archivo en lectura. La capacidad de lectura/listado entre locales para cada rol también requiere matriz técnica. Evitar usar `request.role` seleccionado de otra empresa como autorización del artículo. No confiar en restricciones del frontend ni en ID/clave GCS aportada por el cliente.

## 10. Consulta de colaboradores y adaptación de UI

Reutilizar `POST /collaborator/code` y `CollaboratorService.getCollaboratorCode(code)` como capacidad de búsqueda. En creación, backend reconsulta/valida el código elegido; persiste `finderEmployeeCode`, `finderName`, `finderLastName`, `finderPosition` como snapshot inmutable, con todos los datos obligatorios. Cambios posteriores del colaborador no alteran el artículo. El contrato actual sin DTO tipado solo garantiza el cuerpo del request y `response.data`; `nombreCompleto`/`cargoNombre` se infieren del consumo UI, no de una respuesta formal verificada. **PENDIENTE TÉCNICO:** confirmar respuesta real y fuente fiable para separar nombre/apellido, identidad del colaborador y autorización de uso del servicio. Si falla o faltan datos obligatorios, no guardar como COLLABORATOR. El fallback manual sería una propuesta sujeta a aprobación; no habilitarlo por defecto ni inventar datos.

Roberto puede adaptar `register_found` existente: cargar establecimientos permitidos, elegir `establishmentId`, consultar catálogo jerárquico y enviar solo `classificationOptionId`, consultar zonas del local y exigir `zoneId`, incorporar hora opcional y `foundTimeKnown`, ajustar campos CUSTOMER/OTHER y `finderNote`, conservar autocompletado de colaborador con validación backend, usar staging LF para PHOTO/EVIDENCE, consumir POST y presentar `itemCode`/errores. Las listas locales de tema/zona y los límites locales de archivos deben ceder al contrato real de PEC; el botón Guardar se habilita según validaciones y configuración. No se rediseña la pantalla.

## 11. Verificación prevista

| Nivel | Casos mínimos |
|---|---|
| Unit | Jerarquía flexible, padre/ciclo/catálogo ajeno; zona inactiva o no vinculada; reglas COLLABORATOR/CUSTOMER/OTHER; prioridad y versión de CustodyPolicy; política ausente; año/formato/agotamiento itemCode; cálculo de expiración y FOOD sin cierre. |
| Integración PostgreSQL/GCS simulado | Creación con FKs y audit en una transacción, rollback si falla audit/asociación, UNIQUE y dos altas concurrentes mismo enterprise/año, aislamiento de empresas, staging de un solo uso, fallo/compensación de GCS y limpieza de huérfanos. |
| E2E API/UI | Registro válido con código y auditoría; sin zonas; clasificación inválida; sin política; archivo rechazado sin asociación; usuario sin permiso; fecha sola sin hora mostrada; snapshot colaborador completo. |

Los siete scripts DDL se ejecutaron en una transaccion de prueba revertida y despues se aplicaron de forma controlada en DEV. Se verificaron 10 tablas y la correspondencia de sus 161 columnas con schema.prisma. La prueba real de concurrencia del correlativo sigue pendiente para FOUND-BE-010.

## 12. Tasks propuestas

| Task | Entregable acotado |
|---|---|
| FOUND-BE-001 | Modelo fisico PostgreSQL + scripts DDL versionados + aplicacion controlada en DEV + sincronizacion posterior de schema.prisma. |
| FOUND-BE-002 | DONE (2026-10-05): servicio, validación jerárquica y consulta segura de catálogos/opciones. Sin administración CRUD ni seed funcional. |
| FOUND-BE-003 | DONE (2026-10-05): consulta segura de zonas activas por establecimiento y validador reutilizable de zona/vínculo. Sin CRUD ni carga inicial. |
| FOUND-BE-004 | DONE (2026-10-05): selección de versión efectiva e inmutable por prioridad y empresa; cálculo del primer hito desde `registrationAt`; FOOD queda sin fecha cuando no hay fuente de cierre. Sin CRUD HTTP ni carga de políticas iniciales. |
| FOUND-BE-005 | DONE (2026-10-05): reserva transaccional con `FOR UPDATE`, formato `LF-F-AAAA-XXXXXX`, rango y agotamiento seguros, aislamiento enterprise/año; prueba concurrente real 12/12 en PostgreSQL 17 temporal aislado. Sin FoundItem POST. |
| FOUND-BE-006 | IN PROGRESS (2026-10-05): POST/GET, validaciones, persistencia transaccional y finder CUSTOMER/OTHER implementados. El snapshot COLLABORATOR falla cerrado hasta confirmar un contrato laboral con nombres/apellidos separados; DoD pendiente. |
| FOUND-BE-007 | DONE (2026-10-05): política PEC compartida, `POST /lost-found/files`, token opaco/hash ligado a usuario/enterprise/propósito y TTL configurable, metadata STAGED, asociación transaccional STAGED→ATTACHED con FoundItem, descarga autorizada y servicio de cleanup sin scheduler. 28 pruebas focalizadas y `nest build` pasan; sin escritura de datos en BD DEV. TTL de producción y ejecución periódica de cleanup siguen pendientes técnicos. |
| FOUND-BE-008 | DONE (2026-10-05): `LostFoundAuditService.recordEvent` append-only, snapshot explícito y sanitizado, `FOUND_ITEM_CREATED` dentro de la transacción de FoundItem después de las asociaciones; un error se propaga para rollback. Pruebas focalizadas y build pasan; sin escritura de prueba en DEV. Política formal de minimización pendiente. |
| FOUND-BE-009 | CASL/PoliciesGuard por empresa, establecimiento y recurso; mapeo de roles. |
| FOUND-BE-010 | Pruebas unitarias, integración PostgreSQL/GCS simulado y E2E de reglas de aceptación. |
| FOUND-FE-001 | IN PROGRESS (2026-10-05): formulario existente conectado a establecimientos, clasificación, zonas, staging y POST/GET reales; E2E DEV pendiente. |

## 13. PENDIENTE TÉCNICO para revisión del Plan

1. Confirmar el nombre exacto almacenado para `Gestor Local` y completar la matriz de lectura/administración LF para todos los roles y recursos. La resolución por empresa seleccionada ya está implementada en el contexto Lost & Found; las rutas PEC ajenas a LF conservan su selección histórica de rol.
2. FOUND-BE-006: confirmar contrato real de colaboradores y separación fiable de nombres/apellidos; hasta entonces la creación COLLABORATOR falla cerrado.
3. Confirmar si los días de custodia son períodos de 24 horas o días calendario locales; identificar una fuente verificable de cierre para FOOD y definir la representación de `foundAt` fecha sola.
4. Confirmar timezone oficial que debe regir el cambio de año para `itemCode`; la implementación usa UTC y conserva la regla de rollback que permite reutilizar una reserva de una transacción no confirmada.
5. FOUND-BE-007: fijar el TTL de producción y política de retención, y programar la invocación periódica de `cleanupExpiredStagedFiles`; el default de 30 minutos solo es técnico para DEV.
6. Decidir la integridad histórica de `zoneId` ante futura transferencia de custodio; no bloquearla con una FK compuesta improcedente.
7. Cerrar DTO y códigos de error con el filtro HTTP actual, valores administrables iniciales de `functionalType`, permisos de configuración y demás datos iniciales antes de habilitar registro.
8. FOUND-BE-008: aprobar política de minimización/retención para snapshots funcionales, especialmente si eventos futuros requieren datos personales del finder. El evento actual omite nombres, apellidos y cargo.

Estos pendientes son técnicos o de integración; no reabren FOUND-001 v1.0.0 ni autorizan cambiar sus reglas funcionales. El Plan permanece **BORRADOR**.

## 14. Historial

| Versión | Fecha | Autor | Cambio |
|---|---|---|---|
| 0.1.0 | 2026-10-05 | Walter Molina | Primer borrador del Plan técnico implementable de Sprint 1 para PEC-LF-FOUND-001. |
| 0.2.0 | 2026-10-05 | Walter Molina | Ajustes técnicos previos a implementación: límite del correlativo itemCode, simplificación de unicidad, functionalType jerárquico, description nullable y snapshots JSONB opcionales en LostFoundAudit. |
| 0.3.0 | 2026-10-05 | Walter Molina | Se adopta estrategia database-first: scripts DDL PostgreSQL aplicados controladamente en DEV y sincronizacion posterior de schema.prisma; Prisma Migrate historico queda fuera de FOUND-BE-001. |
| 0.4.0 | 2026-10-05 | Walter Molina | FOUND-BE-002: consulta de catalogos por enterprise autenticado, jerarquia plana, seleccionabilidad y herencia de functionalType; no se cargan opciones funcionales. |
| 0.5.0 | 2026-10-05 | Walter Molina | FOUND-BE-003: consulta de zonas activas y vinculos por establecimiento, validacion reutilizable y acceso al local segun las relaciones PEC disponibles. |
| 0.6.0 | 2026-10-05 | Walter Molina | FOUND-BE-004: servicio interno de selección de políticas por prioridad, versión y empresa, con cálculo del primer vencimiento; FOOD sin fuente de cierre queda sin fecha. |
| 0.7.0 | 2026-10-05 | Walter Molina | FOUND-BE-005: reserva transaccional de itemCode con bloqueo de fila, agotamiento seguro y prueba concurrente PostgreSQL aislada. |
| 0.8.0 | 2026-10-05 | Walter Molina | FOUND-BE-006 parcial: endpoints POST/GET, DTO, validaciones y alta transaccional CUSTOMER/OTHER; COLLABORATOR queda bloqueado por contrato de nombres no verificable. |
| 0.9.0 | 2026-10-05 | Walter Molina | FOUND-BE-007: política de archivo PEC compartida, staging GCS con token hasheado, asociación transaccional, descarga autorizada y cleanup reutilizable. |
| 0.10.0 | 2026-10-05 | Walter Molina | FOUND-BE-008: evento `FOUND_ITEM_CREATED` append-only y transaccional, snapshot funcional explícito con serialización y sanitización defensiva. |
| 0.11.0 | 2026-10-05 | Walter Molina | FOUND-FE-001 parcial: integración del formulario existente con selector autorizado, clasificación y zonas dinámicas, staging de archivos y POST FoundItem. Registro COLLABORATOR bloqueado por discrepancia real en el contrato de nombres; E2E DEV pendiente. |
| 0.12.0 | 2026-10-06 | Walter Molina | Se alinea el contexto de permisos y datos de Lost & Found con la empresa seleccionada y asociada al usuario; Admin Local/Gestor Local quedan limitados a locales asignados. Se actualiza el contrato de consulta de zonas y catálogo. |

## 15. Ejecución parcial de FOUND-FE-001

El frontend consume las rutas de catálogo, zonas, staging, FoundItem y descarga con `enterpriseId` igual a la empresa activa asociada al usuario. `EstablishmentPickerComponent` usa el resumen marcado `lostFoundContext=true` y `pec` con esa empresa; solo se envía el `classificationOptionId` final junto con `establishmentId` y `zoneId`. El formulario conserva fecha sola cuando la hora se desconoce y serializa hora conocida con offset local explícito. Los archivos se suben individualmente a staging, y únicamente los tokens recibidos se incluyen en `fileTokens`.

Discrepancia observada en el contrato vigente: `POST /collaborator/code` presenta `nombreCompleto` y `cargoNombre`, pero `CreateFoundItemDto` y el servicio FoundItem requieren un snapshot con `finderName`, `finderLastName` y `finderPosition` obtenibles/validados por backend. El servicio backend no puede separar con fiabilidad nombre y apellido y rechaza el registro COLLABORATOR con 422. El frontend conserva el autocomplete y bloquea el submit para ese tipo, sin inferir ni enviar nombres. Customer requiere nombre/apellido separados; OTHER los permite opcionales.

`FOUND-FE-001` permanece **IN PROGRESS** hasta completar validación manual E2E autenticada contra DEV y resolver el contrato de COLLABORATOR. No se escribió en la base de desarrollo durante esta tarea; por ello el alta y su posterior GET no se ejecutaron. El E2E requiere comprobar que DEV tenga catálogo de clasificación con opciones activas, zonas activas vinculadas a un establecimiento autorizado, y al menos una `CustodyPolicy` efectiva para la clasificación, tipo funcional o GENERAL. No se inventó ni insertó esa configuración.
