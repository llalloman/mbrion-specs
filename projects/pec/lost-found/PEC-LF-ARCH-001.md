# Especificación: Arquitectura y dominio de Artículos encontrados y declarados perdidos

> Esta especificación es la fuente única de verdad para el desarrollo. Mantenerla actualizada durante el flujo **Idea & Spec → Plan → Tasks → Execute → Validate**. Los planes, tareas, implementación y validación deben derivarse de esta especificación; registrar aquí los cambios de alcance o decisiones.
>
> **Clasificación de información:** cada elemento se identifica como **CONFIRMADO**, **PROPUESTA / INFERIDO** o **PENDIENTE DE DEFINICIÓN**. Los hechos funcionales confirmados se contrastan con el documento funcional oficial del módulo; las propuestas no son decisiones aprobadas. Esta especificación describe arquitectura y dominio; no autoriza implementación.

## 1. Encabezado

| Campo | Valor |
|---|---|
| ID | `PEC-LF-ARCH-001` |
| Título | Arquitectura y dominio de Artículos encontrados y declarados perdidos |
| Versión | `1.2.0` |
| Autor | Mbrion / equipo PEC — por confirmar |
| Fecha | `2026-09-30` |
| Work item / backlog | Pendiente de referencia |

## 2. Problema

**CONFIRMADO:** PEC requiere un módulo transversal para administrar registros de artículos perdidos y encontrados y los procesos asociados: búsqueda nacional, validación de propiedad, custodia, contacto, entrega, vencimientos, reportes, administración y permisos.

**CONFIRMADO:** los procesos comparten información y evidencia, pero tienen ciclos de vida distintos. Un reporte de pérdida es una declaración y no implica que exista un artículo físico bajo custodia; un artículo encontrado puede existir sin reporte coincidente.

**CONFIRMADO:** el modelo conceptual aprobado separa la declaración de pérdida del artículo encontrado. Los detalles de vocabulario de interfaz no fusionan `LostItemReport` y `FoundItem`.

## 3. Resultado esperado

Disponer de un modelo de dominio y una arquitectura conceptual revisables que permitan derivar posteriormente especificaciones funcionales y técnicas sin confundir declaraciones, artículos físicos, coincidencias, validaciones, contactos, custodia y entregas. Las decisiones no sustentadas deben permanecer abiertas.

## 4. Alcance

### Incluye

- **CONFIRMADO:** registro inicial; artículo perdido; artículo encontrado; documentos y tarjetas; búsqueda nacional; validación de propiedad; entrega; custodia; contacto; vencimientos; reportes; administración; roles y permisos.
- **CONFIRMADO:** documentar conceptos, responsabilidades de dominio, relaciones, dimensiones de estado, integraciones PEC, reglas de seguridad y decisiones destinadas a Specs posteriores.
- **CONFIRMADO:** analizar como hipótesis, sin asumirlos como entidades definitivas, los conceptos preliminares enumerados en la sección 11.

### Fuera de alcance

- Implementar código o modificar `pec-api` o `pec-ui`.
- Crear modelos Prisma, migraciones, endpoints, componentes Angular o contratos implementables.
- Aprobar el algoritmo de búsqueda/coincidencias, políticas de retención, estructura física de datos o permisos detallados. Las duraciones funcionales confirmadas se mantienen como requisitos; los parámetros técnicos de cómputo siguen pendientes.
- Definir integraciones como existentes o disponibles sin verificar sus capacidades y responsables.

### Supuestos

- **CONFIRMADO:** el módulo pertenece al producto PEC. PEC ya cuenta con capacidades técnicas de autenticación, autorización, identidad, archivos, notificaciones, reportes/exportación y ejecución programada que se deben evaluar para el módulo.
- **CONFIRMADO:** mantener trazabilidad entre los registros y el establecimiento responsable, reutilizando referencias de identidad/catálogos PEC cuando esté definido en §14.
- **CONFIRMADO:** la arquitectura reutiliza o extiende las capacidades PEC especificadas en §14 respetando los límites de dominio. Los detalles implementables se describen en Specs posteriores.

## 5. Reglas de negocio

- **RB-01 — CONFIRMADO:** una tarjeta nunca debe mostrar su número completo; solo se muestran los primeros 6 y los últimos 4 dígitos. Debe confirmarse cómo se presentan números que no tengan longitud suficiente y qué otros canales o artefactos aplican.
- **RB-02 — CONFIRMADO:** las respuestas de validación de propiedad no se muestran previamente al cliente.
- **RB-03 — CONFIRMADO:** solo el local que posee físicamente el artículo puede modificar su custodia.
- **RB-04 — CONFIRMADO:** no se puede cerrar una entrega sin acta firmada.
- **RB-05 — CONFIRMADO:** las transiciones relevantes deben conservar auditoría.
- **RB-06 — CONFIRMADO:** documentos permanecen 1 mes en el local; luego pasan a pendiente de transferencia y se transfieren al Gran Centro de Servicio y Solución (GCSS), donde permanecen 3 meses antes de su destrucción.
- **RB-07 — CONFIRMADO:** los artículos generales cumplen 3 meses antes de transferencia mediante correspondencia interna. El destino/persona indicada por el documento funcional es Antonio Mejía; confirmar vigencia de ese responsable antes de parametrizarlo.
- **RB-08 — CONFIRMADO:** alimentos y comida rápida permanecen hasta el cierre del local; después se desechan y se registra evidencia.
- **RB-09 — CONFIRMADO:** emitir alertas automáticas 3 días antes del vencimiento, dentro del sistema, por correo y mediante reporte Excel/notificación.
- **RB-10 — CONFIRMADO:** CustodyPolicy versionada aplica los plazos aprobados y conserva versión aplicada + expirationDate. Los detalles implementables de cómputo, calendario, excepciones, responsables, parametrización y fallos se desarrollan en Specs posteriores.
- **RB-11 — CONFIRMADO:** una coincidencia (incluso `CONFIRMED`) no prueba propiedad ni autoriza por sí sola una entrega; la validación aprobada habilita el proceso, que solo se completa con acta firmada.
- **RB-12 — CONFIRMADO:** `LostItemReport` y `FoundItem` mantienen relación N:M mediante `ItemMatch`; sus cardinalidades están aprobadas. Las reglas de cierre/edición del reporte se detallarán en su Spec funcional.

## 6. Criterios de aceptación

Los criterios de esta especificación verifican el resultado de arquitectura y dominio. No constituyen autorización de implementación.

### Positivos

- **CA-P01:** Dado que se revisa el modelo conceptual, cuando se describe un reporte de pérdida y un artículo encontrado, entonces ambos aparecen como conceptos distintos y se explica cómo podrían relacionarse sin presuponer que son el mismo objeto.
- **CA-P02:** Dado el diseño de estados, cuando se documenta el ciclo de vida, entonces custodia, matching, contacto y validación se analizan como dimensiones separadas y se justifican las relaciones entre ellas.
- **CA-P03:** Dado que se documenta el alcance, cuando se revisan integraciones y permisos, entonces cada punto sin evidencia aparece como propuesta o pregunta pendiente y no como capacidad confirmada.
- **CA-P04:** Dado que se revisan reglas de datos sensibles, cuando se describe tarjeta y validación de propiedad, entonces constan el enmascaramiento confirmado y la no exposición previa de respuestas.

### Negativos

- **CA-N01:** Dado que existe un reporte de pérdida sin artículo localizado, cuando se representa el dominio, entonces no se crea ni se presupone un artículo físico en custodia.
- **CA-N02:** Dado que existe una coincidencia entre una declaración y un artículo, cuando se describe su efecto, entonces no se trata la coincidencia como validación de propiedad ni como autorización de entrega.
- **CA-N03:** Dado que una persona consulta el proceso de validación, cuando todavía no ha respondido, entonces las respuestas correctas o históricas no se le revelan.
- **CA-N04:** Dado que no existe acta firmada, cuando se evalúa el cierre de entrega, entonces el cierre queda impedido.
- **CA-N05:** Dado que un local no posee físicamente el artículo, cuando intenta modificar su custodia, entonces la operación no está permitida.

### Casos de borde

- **CA-B01:** Dado que la longitud del número de una tarjeta es menor o igual al formato de visualización definido, cuando se genera una vista o evidencia, entonces se aplica la regla segura de enmascaramiento sin exponer el número completo; el tratamiento de números cortos se precisará en la Spec funcional.
- **CA-B02:** Dado que existen varios reportes candidatos o varios artículos compatibles, cuando el sistema propone coincidencias, entonces el procedimiento de desempate o revisión humana permanece explícito y no se inventa como decisión aprobada.
- **CA-B03:** Dado que se describe el vencimiento de un documento, artículo general o alimento, cuando se especifica su flujo, entonces se reflejan los plazos confirmados de la regla RB-06 a RB-09 y se detallan las variables técnicas y operativas indicadas en RB-10 para las Specs posteriores.
- **CA-B04:** Dado que se registra una transición relevante, cuando se especifica la evidencia de auditoría, entonces se conservan quién actuó, cuándo, desde qué local y qué cambió, como requiere el documento funcional; el formato técnico, acceso e integridad siguen por definir.

## 7. Datos personales

- **¿Aplica?** Sí.
- **Datos tratados CONFIRMADOS:** tipo/número de identificación, nombres/apellidos, celular/teléfono, email, fotografías, archivos/documentos, datos de tarjetas, firmas y respuestas de validación; además, descripción del artículo, tema/subtema, lugar/fecha aproximados y datos de establecimiento/usuario que registra.
- **Finalidad CONFIRMADA:** registrar declaraciones y hallazgos, buscar coincidencias, validar propiedad, contactar al cliente, documentar entrega/custodia y mantener trazabilidad.
- **Acceso CONFIRMADO:** aplicar matriz funcional de §13 y las reglas de protección de tarjeta/respuestas de validación; custodia modificable solo por establecimiento poseedor.
- **Diseño de privacidad:** la especificación funcional/técnica precisará base y avisos de privacidad, acceso por campo/canal, retención, datos completos, logs, QA e incidentes. Esta arquitectura no aprueba una base legal ni un mecanismo técnico.
- **PROPUESTA / INFERIDO:** minimizar datos sensibles en logs y utilizar datos sintéticos o anonimizados en QA.
## 8. Riesgos técnicos

| Riesgo | Impacto | Respuesta arquitectónica |
|---|---|---|
| Acoplar LostItemReport con el objeto FoundItem | No representaría reportes sin hallazgos ni candidatos múltiples | Mantener conceptos independientes relacionados mediante ItemMatch. |
| Colapsar los ciclos en un estado único | Pérdida de trazabilidad y transiciones incompatibles | Mantener dimensiones y entidades de proceso separadas según §§11–12. |
| Exponer tarjeta o respuestas de validación | Fraude o divulgación de información | Aplicar RB-01/RB-02 a todos los canales y controlar acceso a referencias y respuestas. |
| Cambiar custodia desde un local sin posesión | Custodia física y registro quedarían inconsistentes | Validar establecimiento poseedor en backend y registrar CustodyMovement. |
| Cerrar entrega sin acta/auditoría | Entrega disputable y falta de trazabilidad | Bloquear cierre sin acta; conservar LostFoundAudit y proceso estructurado. |
| Ejecutar políticas de vencimiento con cómputo o jobs mal definidos | Retención o disposición incorrecta, alertas duplicadas/perdidas | CustodyPolicy versionada; los detalles técnicos de cómputo y operación se documentan en Specs posteriores. |
| Reutilizar entidades o reglas PEC fuera de su semántica | Acoplamiento indebido con Quejas o zona geográfica | Reutilizar infraestructura compartida; conservar dominios y asociaciones Lost & Found propios. |
| Búsqueda nacional/matching con exposición o falsos positivos | Acceso a datos no pertinentes o reclamaciones mal dirigidas | Resultados mínimos seguros, revisión de ItemMatch y algoritmo definido en la Spec de matching. |
## 9. Evidencia esperada

- Trazabilidad de las trece capacidades funcionales con la arquitectura y las responsabilidades transversales.
- Revisión de separación `LostItemReport`/`FoundItem`, relaciones ItemMatch/RecoveryClaim y cardinalidades lógicas DATA-001.
- Revisión de las dimensiones de estado, regla de custodia física, transferencia, entrega y disposición.
- Matriz de permisos funcional por rol y restricción contextual de custodia.
- Confirmación de las capacidades PEC citadas en §14 mediante el descubrimiento técnico.
- Revisión de privacidad y controles para datos personales, tarjetas, respuestas, archivos, logs y QA en las Specs correspondientes.
## 10. Dependencias

- **CONFIRMADO:** Keycloak/CASL/PoliciesGuard, usuarios, empresas, establecimientos, infraestructura técnica de catálogo PEC, StorageService/GCS, correo/notificaciones, scheduler y exportación son capacidades existentes descritas en §14. Los catálogos maestros funcionales Lost & Found son independientes del catálogo actual de otros módulos; las demás capacidades se reutilizan/adaptan según sus límites de dominio.
- **CONFIRMADO:** los destinos GCSS/correspondencia se integran mediante establecimientos, usuarios PEC, ItemTransfer y evidencia.
- **CONFIRMADO:** servicio externo de autocompletado del cliente existe y se contempla como integración con fallback manual.
- **PENDIENTE DE DEFINICIÓN:** mecanismo técnico de firma, matching exacto, transcripción y contrato del servicio de cliente, además del detalle asignado a Specs funcionales posteriores (ver §18).
## 11. Modelo conceptual / datos (cuando aplique)

### Principios y conceptos

- **CONFIRMADO:** `LostItemReport` es la declaración de pérdida y `FoundItem` el artículo físico. Son conceptos separados y pueden existir de forma independiente.
- **CONFIRMADO:** `ItemMatch` persistente relaciona reportes y artículos N:M; `LostItemReport 1:N ItemMatch` y `FoundItem 1:N ItemMatch`. Estados controlados POSSIBLE, CONFIRMED y DISCARDED; admite origen manual o automático y conserva score, nivel y método conceptuales. Confirmar match no confirma propiedad ni entrega.
- **CONFIRMADO:** Búsqueda Nacional es query/capacidad, no entidad persistente; filtra texto, niveles de clasificación categoría/tema/subtema derivados de `LostFoundCatalogOption.parentId`, fecha, local, estado y número de identificación del cliente/reportante; resultados seguros sin información sensible.
- **CONFIRMADO:** `RecoveryClaim` modela que una persona reclame un `FoundItem`; puede nacer desde reporte+ItemMatch o directamente sin reporte. Un artículo puede tener varios reclamos.
- **CONFIRMADO:** `OwnershipValidation`, `ContactAttempt` e `ItemDelivery` se asocian con `RecoveryClaim`. Custodia conserva custodio actual en FoundItem y movimientos históricos en CustodyMovement. ItemTransfer es proceso propio distinto de movimiento.
- **CONFIRMADO:** Finder no es entidad inicial; datos y snapshot quedan en FoundItem. DocumentDetail y CardDetail son detalles opcionales del artículo.

### Límites de dominio

| Dominio/capacidad | Responsabilidad | Estado |
|---|---|---|
| Reportes de pérdida | Declaración con texto o voz, datos snapshot del reportante, lugar/fecha aproximados y archivos | CONFIRMADO |
| Artículos encontrados | Registro del objeto físico, detalle, zona interna y custodio actual | CONFIRMADO |
| Búsqueda/matching | Consulta nacional segura y asociación ItemMatch revisable/manual o automática | CONFIRMADO; pesos/umbrales exactos en Spec de matching |
| Reclamo y validación | RecoveryClaim, preguntas, referencias privadas, respuestas y resultados | CONFIRMADO |
| Contacto | Intentos MANUAL/AUTOMATIC asociados al reclamo y resultados controlados | CONFIRMADO |
| Custodia y transferencia | Custodio actual, historial de movimientos, transferencia con despacho/recepción y políticas de vencimiento | CONFIRMADO |
| Entrega/disposición | Entrega con validación y acta; disposición final con evidencia | CONFIRMADO |
| Administración | Catálogos Lost & Found, jerarquía/opciones, zonas, preguntas, políticas y tipos propios según §14 | CONFIRMADO |
| Reportes | Consultas sobre modelo transaccional y exportación Excel | CONFIRMADO |
| Capacidades PEC compartidas | Usuarios, empresas, locales, infraestructura técnica de catálogo PEC, archivos, notificaciones, seguridad y exportación | Existencia técnica confirmada; no reutilizar `sspectcatalog` como catálogo funcional Lost & Found. Usar `LostFoundCatalog` / `LostFoundCatalogOption` propios. |

### Integridad conceptual

`ItemMatch.CONFIRMED` no equivale a validación aprobada. La validación aprobada habilita entrega, pero no la completa. La entrega requiere acta firmada. Transferir requiere recepción registrada. Entrega y disposición completadas son cierres alternativos; no se elimina la historia.
## 12. Estados y transiciones (cuando aplique)

**CONFIRMADO:** PEC no tiene una máquina de estados genérica reutilizable; los estados de Quejas son propios de ese dominio y no se reutilizan semánticamente. Lost & Found mantiene dimensiones separadas, controladas por dominio; no se permite crear estados libres ni reducir todo a un status único.

Dimensiones confirmadas: artículo/custodia, matching, RecoveryClaim, validación, contacto, entrega, transferencia y disposición. Los estados funcionales requeridos siguen representados: Registrado; En custodia del local; Gestión de contacto; Cliente contactado; Entregado al cliente; Pendiente de transferencia; Transferido al GCSS; En custodia GCSS; Pendiente de destrucción; Destruido; Posible coincidencia.

| Dimensión | Estados/resultados funcionales | Regla de transición |
|---|---|---|
| Artículo | Registrado | Estado inicial del registro de artículo encontrado. |
| Custodia | En custodia del local; Pendiente de transferencia; Transferido al GCSS; En custodia GCSS; Pendiente de destrucción; Destruido; Entregado al cliente | Solo el establecimiento poseedor modifica custodia; los movimientos y transferencias conservan historia. |
| Matching | POSSIBLE; CONFIRMED; DISCARDED | Match solo propone relación; no es validación de propiedad. |
| Validación | HIGH_MATCH; MEDIUM_MATCH; LOW_MATCH | No revelar respuestas/referencias antes de que el reclamante responda; validación requerida habilita entrega. |
| Contacto | Gestión de contacto; Cliente contactado; CONTACTED; NO_ANSWER; WRONG_NUMBER; WILL_VISIT_STORE | Cada intento pertenece a RecoveryClaim y clasifica MANUAL/AUTOMATIC; máximo tres automáticos. |
| Entrega | Entregado al cliente | Requiere validación superada y acta firmada cargada. |
| Transferencia | Despacho y recepción según flujo de custodia | No se completa como recibida sin confirmar recepción. |
| Disposición/vencimiento | Pendiente de destrucción; Destruido; desecho con evidencia | Entrega o disposición completada cierra el ciclo operativo; plazos y alerta siguen la CustodyPolicy. |

Los detalles implementables de transiciones se desarrollarán en las Specs funcionales sin cambiar estas dimensiones y reglas.
## 13. Roles y permisos (cuando aplique)

**CONFIRMADO — matriz funcional base del documento oficial:**

| Función | Admin Local | Redes Sociales | Gestor Local | Gestión de Quejas |
|---|---:|---:|---:|---:|
| Ver casos | Sí | No | Sí | Sí |
| Buscar nacional | Sí | No | Sí | Sí |
| Registrar artículo perdido | Sí | Sí | Sí | Sí |
| Registrar artículo encontrado | Sí | Sí | Sí | Sí |
| Validar propiedad | Sí | No | Sí | Sí |
| Contactar cliente | Sí | No | Sí | Sí |
| Registrar resultado | Sí | No | Sí | Sí |
| Entregar artículo | Sí | No | Sí | Sí |
| Generar acta | Sí | No | Sí | Sí |
| Cargar acta | Sí | No | Sí | Sí |
| Gestionar custodia | Sí | No | Sí | Sí |
| Gestionar vencimientos | Sí | No | Sí | Sí |
| Gestionar correspondencia | Sí | No | Sí | Sí |

- **CONFIRMADO:** solo el local que posee físicamente el artículo puede modificar su custodia, además de la matriz funcional por rol.
- **CONFIRMADO:** aplicar la matriz funcional por rol mediante Keycloak + CASL/PoliciesGuard; la regla de posesión se valida en backend. Admin Local queda dentro del ámbito PEC que le corresponda. El detalle implementable de roles, ámbitos y administración se especificará en el diseño funcional/técnico.

## 14. Integraciones (cuando aplique)

Hallazgos técnicos PEC confirmados y resolución arquitectónica para Lost & Found:

| Capacidad | Hecho PEC confirmado | Resolución de arquitectura |
|---|---|---|
| Autenticación/autorización | Keycloak; AuthGuard obtiene preferred_username y contexto de usuario/rol; CASL + PoliciesGuard | Reutilizar/extender. Roles funcionales y autorización contextual por establecimiento/custodia aplican también en backend. |
| Usuarios/empresa/establecimiento | UserEnterpriseRole, UserEstablishment, `sspectuser`, `sspectenterprise`, `sspectestablishment`; empresa, ciudad/región | Reutilizar referencias PEC. Guardar enterprise_id en raíces/transaccionales principales; consistencia con establecimiento. |
| Zonas | PEC tiene `id_zona` regional pero no un dominio Zone completo | LostFoundZone + LostFoundEstablishmentZone para ubicaciones internas. No reutilizar `id_zona` territorial. |
| Catálogos | Existe infraestructura técnica de catálogos PEC | Lost & Found tendrá `LostFoundCatalog` + `LostFoundCatalogOption` independiente; no reutilizar el catálogo funcional actual. Clasificación jerárquica categoría → tema → subtema; estados controlados no son catálogo libre. |
| Archivos | StorageService y Google Cloud Storage, servicios/endpoints protegidos; asociaciones actuales específicas a Quejas | Reutilizar almacenamiento. `sspectlffile` y asociaciones Lost & Found propias; no añadir FKs del módulo a `sspectfile`. |
| Notificación/email | Canales in-app/email y procesadores por evento | Reutilizar mecanismo/canales y extender con eventos del módulo; plantillas y configuración Lost & Found propias. |
| Auditoría | Hay logs técnicos y campos created/modified en algunos modelos; no historial genérico de auditoría de dominio encontrado | `LostFoundAudit` durable de negocio requerido, además de CustodyMovement/ItemTransfer/ItemDelivery/ItemDisposal. Logs no sustituyen auditoría. |
| Reportes/Excel | ReportsService; ExcelExportService con ExcelJS; frontend exporta con xlsx | Reutilizar/extender exportación PEC, no lógica de reportes de Quejas. Consultar modelo transaccional; preferir generación backend. |
| Scheduler | NestJS Schedule habilitado y jobs programados existentes | Reutilizar/adaptar infraestructura; jobs de vencimiento Lost & Found son propios. |
| Estados | No hay máquina genérica; estados de Quejas son de su dominio | Estados Lost & Found controlados por dominio y separados por dimensión. |
| Frontend | Keycloak, lazy modules, Angular Material + Fuse + Tailwind, reactive forms, HttpClient y archivos | Feature Lost & Found aislada; no incorporarla dentro de Quejas. |
| GCSS/correspondencia | GCSS y correspondencia son destinos definidos funcionalmente | GCSS se referencia como `sspectestablishment`; transferencia/correspondencia es proceso ItemTransfer con usuarios PEC y evidencia. |
| Autocompletado de cliente | Existe servicio externo de datos del cliente | Integración contemplada con fallback manual y controles de privacidad. Contrato técnico: pendiente real #4. |
| Transcripción | Registro perdido admite voz y archivo | Guardar audio por metadata propia del módulo; proveedor/mecanismo: pendiente real #3. |

### Implementación Lost & Found existente

**INTERFAZ EXISTENTE — CONFIRMADO:** `pec-ui` ya contiene un módulo visual Lost & Found con navegación y pantallas para resumen, registro de artículo perdido, registro de artículo encontrado, contactos y vencimientos. Los formularios incluyen avances de autocompletado de colaborador y cliente; el registro de pérdida incluye selector de establecimiento. Resumen, contactos y vencimientos utilizan datos mock. La clasificación usa listas locales de tema/subtema y el registro de encontrado usa zonas locales fijas. Esta interfaz es parcial y no constituye una implementación funcional de punta a punta.

**INTEGRACIÓN FUNCIONAL PENDIENTE:** los formularios de pérdida y hallazgo tienen «Guardar» deshabilitado, no persisten registros ni llaman a una API Lost & Found. El registro de encontrado muestra el local de forma informativa, pero todavía no selecciona ni envía `establishmentId`; su autocompletado de colaborador aún no persiste el snapshot. Falta conectar catálogo jerárquico, zonas por establecimiento, archivos con metadata Lost & Found, políticas de custodia, auditoría y permisos backend. Búsqueda nacional, custodia y reportes no tienen operación completa.

**BACKEND ESPECÍFICO INEXISTENTE:** `pec-api` aún no contiene modelos Prisma, migraciones, endpoints ni servicios de dominio Lost & Found. Existen capacidades PEC reutilizables: Keycloak/AuthGuard, CASL/PoliciesGuard, DatabaseModule/DatabaseService, servicios de establecimientos y colaboradores, y StorageService/GCS. La implementación debe reutilizar `DatabaseModule`/`DatabaseService`; no crear un `PrismaClient` adicional por módulo.

### Administración y reportes funcionales

**CONFIRMADO:** administración comprende catálogos Lost & Found y opciones con jerarquía/orden/activación, establecimientos, zonas internas, tipos de identificación, estados controlados, roles, plantillas propias de correo, preguntas de validación y CustodyPolicy versionada. Clasificación, tipos de documento/tarjeta y canales son catálogos administrables propios; usuarios responsables referencian `sspectuser`. La interfaz puede especializarse por `catalog.code`, sin pantalla obligatoria por catálogo.

**CONFIRMADO:** reportes de encontrados/perdidos, pendientes de entrega, próximos a vencer, transferidos, destruidos, documentos y tarjetas; filtros por establecimiento, zona, fechas, clasificación (categoría/tema/subtema derivada de `LostFoundCatalogOption`) y estado; exportación Excel. No se crean tablas de reporting iniciales.
## 15. Contratos / API (cuando aplique)

No se definen rutas, endpoints, eventos ni esquemas de API en esta especificación de arquitectura. Su diseño queda **FUERA DE ALCANCE** en esta etapa. Antes de especificarlos, resolver límites de dominio, permisos, relaciones, estados y contratos de las integraciones investigadas.

## 16. Pantallas afectadas (cuando aplique)

**CONFIRMADO:** existe un prototipo de Figma como referencia de UX/UI. En `pec-ui` ya existe un módulo visual Lost & Found con rutas y pantallas parciales para resumen, registro de pérdida, registro de hallazgo, contactos y vencimientos. Usa la infraestructura frontend PEC (Keycloak, carga lazy, Angular Material, Fuse, Tailwind y formularios reactivos), pero todavía requiere integración con APIs y persistencia reales. El estado concreto se detalla en §14.

**CONFIRMADO:** la feature Lost & Found se mantiene aislada del módulo de Quejas. Sprint 1 adapta la interfaz existente a los contratos backend y al modelo aprobado; no parte desde cero en UI.

**OBSERVACIÓN DEL PROTOTIPO — no es comportamiento esperado:** la opción “Búsqueda Nacional” muestra navegación/pantalla incompleta en el prototipo. El comportamiento funcional debe derivarse del documento funcional (buscar artículos de cualquier local, con sus filtros y campos de resultado confirmados), no de esa pantalla incompleta.

Los flujos, vistas finales y perfiles de pantalla se documentarán en especificaciones funcionales posteriores.

## Cobertura funcional completa del requerimiento

Las trece capacidades funcionales confirmadas y sus límites arquitectónicos son:

| # | Capacidad | Resolución arquitectónica confirmada |
|---:|---|---|
| 1 | Registro inicial | Registrar identidad del caso, canal, local/usuario y archivos mediante capacidades PEC con asociaciones propias. |
| 2 | Artículo perdido | `LostItemReport` es declaración independiente; admite descripción de texto o voz, snapshots del reportante y adjuntos. |
| 3 | Artículo encontrado | `FoundItem` representa el objeto físico; custodio actual, zona interna, finder snapshot y política aplicada quedan asociados al artículo. |
| 4 | Documentos y tarjetas | `DocumentDetail`/`CardDetail` opcionales; número de tarjeta nunca completo, se muestran solo primeros seis y últimos cuatro dígitos. |
| 5 | Búsqueda nacional | Query nacional con filtros de texto, jerarquía de clasificación (categoría/tema/subtema), fecha, local, estado e identificación del reportante; los niveles se derivan de `LostFoundCatalogOption.parentId`; resultados seguros. |
| 6 | Matching y validación de propiedad | `ItemMatch` persistente/manual o automático; match no acredita propiedad. `RecoveryClaim` inicia validaciones protegidas. |
| 7 | Entrega del artículo | `ItemDelivery` asociada a reclamación; requiere validación aprobada y acta firmada cargada. |
| 8 | Custodia y transferencia | CustodyMovement conserva historia; ItemTransfer es proceso propio; solo establecimiento poseedor modifica custodia. GCSS es establecimiento PEC. |
| 9 | Contacto automático | ContactAttempt pertenece a RecoveryClaim, clasifica manual/automático y guarda resultados controlados; PEC in-app/email son canales integrados. |
| 10 | Vencimientos | CustodyPolicy versionada: documentos 30 días local + 90 días GCSS; generales 90 días; alimentos hasta cierre y desecho con evidencia; alerta tres días antes. |
| 11 | Reportes | Consultas transaccionales de encontrados/perdidos, entrega, próximos a vencer, transferidos, destruidos, documentos y tarjetas; filtros por establecimiento/zona/fechas y clasificación (categoría/tema/subtema derivada de la jerarquía); Excel. |
| 12 | Administración | Catálogos y opciones Lost & Found independientes, con jerarquía, orden y activación/desactivación; zonas internas propias; tipo de identificación PEC; estados controlados; roles, plantillas propias, preguntas, CustodyPolicy, destinos y responsables PEC. |
| 13 | Roles y permisos | Matriz funcional §13, Keycloak + CASL/PoliciesGuard y autorización backend contextual; custodia restringida al local poseedor. |

### Responsabilidades transversales

**CONFIRMADO:** reglas de negocio, automatizaciones, búsqueda/identificación y auditoría/trazabilidad son cuatro responsabilidades arquitectónicas. No se decide que deban ser cuatro servicios físicos independientes.

### Datos personales y seguridad

**CONFIRMADO:** se procesan identificadores, nombres, contacto, fotos, archivos/documentos, tarjetas, firmas y respuestas de validación. Se enmascaran tarjetas y no se exponen las respuestas de validación antes de que responda el cliente. La matriz por rol se documenta en §13.

## 17. Decisiones técnicas (cuando aplique)

| Decisión | Clasificación | Alcance |
|---|---|---|
| Separar LostItemReport y FoundItem; relacionarlos N:M mediante ItemMatch | CONFIRMADO | Modelo conceptual; no son identidad ni una autorización de entrega. |
| Búsqueda Nacional como query; ItemMatch persistente con estados controlados y match manual/automático inicial basado en reglas | CONFIRMADO | Filtros y resultados seguros en §11; pesos/umbrales exactos son pendiente real #2. |
| Dimensiones independientes de estado; no reutilizar estados de Quejas ni permitir valores libres | CONFIRMADO | Artículo/custodia, match, RecoveryClaim, validación, contacto, entrega, transferencia y disposición. |
| Mantener custodio actual en FoundItem, movimientos históricos separados e ItemTransfer como proceso propio | CONFIRMADO | Actualización consistente; GCSS es establecimiento PEC. |
| RecoveryClaim como raíz para contactos, validaciones y entrega | CONFIRMADO | Puede originarse por coincidencia o directamente sobre artículo. |
| CustodyPolicy versionada y almacenamiento de versión aplicada + expirationDate | CONFIRMADO | Parámetros funcionales de vencimiento están en RB-06 a RB-10. |
| Usar catálogo maestro jerárquico propio de Lost & Found y mantener lógica/metadata de dominio propia | CONFIRMADO | `LostFoundCatalog` + `LostFoundCatalogOption` son independientes del catálogo funcional PEC actual; otras capacidades PEC (Keycloak/CASL, usuarios, establecimientos, StorageService/GCS, notificaciones, scheduler y Excel) se reutilizan/adaptan según §14; no lógica de Quejas. |
| Exigir LostFoundAudit de negocio durable además de eventos transaccionales | CONFIRMADO | Los logs técnicos PEC no sustituyen auditoría de dominio. |
| Mantener ARCH-001 arquitectónico y DATA-001 lógico | CONFIRMADO | Esta spec no aprueba tablas, columnas, modelos Prisma, endpoints, migraciones o componentes. |
| Preservar el estado de trabajo existente de pec-api y pec-ui | CONFIRMADO | Antes de implementación futura, revisar git status y preservar archivos modificados/no versionados. |
## 18. Preguntas pendientes

La consolidación de §21 y `PEC-LF-DATA-001 §22` resolvió las preguntas de dominio, cardinalidades, políticas, responsabilidades, estados, integraciones y reportes que aparecían en versiones anteriores. Permanecen abiertos únicamente:

1. Mecanismo técnico exacto para capturar, validar, almacenar y verificar las firmas del acta (DELIVERY-001).
2. Algoritmo, pesos y umbrales exactos de matching.
3. Proveedor y detalles técnicos de transcripción de audio.
4. Contrato técnico del servicio externo de autocompletado de cliente.
5. Decisiones de detalle propias de las Specs funcionales posteriores, sin reabrir las decisiones consolidadas.
## 19. Estado de aprobación

- **Estado:** APPROVED
- **Aprobador:** Walter Molina
- **Fecha de aprobación:** 2026-09-30

## 20. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.1.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Primera especificación transversal de arquitectura y dominio; distingue confirmado, propuesto y pendiente. |
| 0.2.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Reclasifica vencimientos, alertas, permisos, datos personales, reportes/administración y observación de Figma según el documento funcional oficial; mantiene pendientes los aspectos técnicos y operativos no definidos. |
| 0.3.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Incorpora hallazgos del descubrimiento técnico de pec-api/pec-ui; clasifica capacidades PEC existentes, límites de reutilización, ausencia de implementación Lost & Found y preservación del estado de trabajo. |
| 0.4.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Completa la cobertura de los trece módulos funcionales, estados requeridos, administración, reportes, reglas y componentes transversales; añade matriz final de trazabilidad y refuerza el límite entre ARCH-001 y DATA-001. |
| 0.5.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Consolidación de decisiones arquitectónicas y de modelo aprobadas durante revisión funcional/técnica. Se cierran relaciones, responsabilidades y requisitos que antes aparecían como pendientes; se mantienen únicamente los pendientes enumerados en §18. |
| 1.0.0 | 2026-09-30 | Walter Molina | Aprobación de arquitectura y dominio Lost & Found por Walter Molina. |
| 1.1.0 | 2026-10-03 | Walter Molina | Se adopta catálogo maestro jerárquico independiente de Lost & Found para clasificación y catálogos simples administrables. |
| 1.2.0 | 2026-10-04 | Walter Molina | Se actualiza el estado real de implementación: pec-ui ya contiene interfaz parcial Lost & Found; backend de dominio y persistencia continúan pendientes. |

## Matriz de cobertura del requerimiento

| Requisito | Sección de resolución | Estado |
|---|---|---|
| Registro inicial | Cobertura funcional §1; modelo §11 | Cubierto |
| Artículo perdido | Cobertura funcional §2; modelo §11 | Cubierto |
| Artículo encontrado | Cobertura funcional §3; DATA-001 §§6, 11 | Cubierto |
| Documentos/tarjetas y enmascaramiento | Reglas RB-01; cobertura §4; DATA-001 §6.11 | Cubierto |
| Búsqueda nacional y filtros | Cobertura §5; modelo §11 | Cubierto |
| Matching y validación | Cobertura §6; estados §12; DATA-001 §§6.3–6.5 | Cubierto; algoritmo/pesos/umbrales exactos en §18 |
| Entrega, acta y firmas | Reglas RB-04; cobertura §7; DATA-001 §6.9 | Cubierto; mecanismo técnico de firma en §18 |
| Custodia, transferencias y GCSS | Reglas RB-03, RB-06–RB-08; cobertura §8; DATA-001 §§6.7–6.8 | Cubierto |
| Contacto | Cobertura §9; estados §12; DATA-001 §6.6 | Cubierto |
| Vencimientos y alertas | Reglas RB-06–RB-10; cobertura §10; DATA-001 §6.13 | Cubierto; detalles implementables en Specs posteriores |
| Reportes y Excel | Cobertura §11; integración §14 | Cubierto |
| Administración | Cobertura §12; integración §14; DATA-001 §8 | Cubierto |
| Roles y permisos | Matriz §13; integración §14 | Cubierto |
| Reglas, automatizaciones, búsqueda/identificación, auditoría/trazabilidad | Responsabilidades transversales §16; auditoría §§11, 14 y 17 | Cubierto |
| Voz y autocompletado | DATA-001 §6; integración §14 | Cubierto; transcripción y contrato externo en §18 |
| Figma y Búsqueda Nacional incompleta | Pantallas §16 | Observación, no requerimiento |

## 21. Resumen de decisiones arquitectónicas

Este resumen reúne las decisiones desarrolladas en las secciones anteriores. ARCH-001 define cobertura y responsabilidades arquitectónicas; DATA-001 conserva el modelo lógico. Ninguna sección aprueba tablas, columnas, endpoints, modelos Prisma, migraciones o implementación.

### Decisiones confirmadas

- **Dominio:** `LostItemReport` es la declaración de pérdida y `FoundItem` el artículo físico; son conceptos separados e independientes. `ItemMatch` persistente establece una relación N:M y solo representa candidato. Match confirmado no equivale a propiedad validada ni entrega. `RecoveryClaim` registra el reclamo y puede existir sin reporte de pérdida.
- **Búsqueda e identificación:** Búsqueda Nacional es una query, no entidad. Incluye filtros de texto, clasificación (categoría/tema/subtema derivados de `LostFoundCatalogOption.parentId`), fecha, local, estado y número de identificación del reportante; los resultados deben excluir información sensible. ItemMatch admite flujos manual y automático; matching inicial basado en reglas, con score/nivel/método conceptuales.
- **Custodia y transferencias:** el artículo conserva establecimiento custodio actual; los movimientos guardan historia. `ItemTransfer` es un proceso distinto que, al recibirse, refleja el movimiento y cambio de custodio. GCSS es un establecimiento PEC. Solo el establecimiento con posesión física puede cambiar custodia.
- **Reclamo, contacto y validación:** `RecoveryClaim` puede tener varios intentos de contacto y validación; contacto separa MANUAL/AUTOMATIC, resultados controlados, máximo tres intentos automáticos y N manuales. Las referencias de validación no se exponen antes de recibir la respuesta. Validación requerida habilita entrega, no la ejecuta.
- **Entrega/disposición:** `ItemDelivery` final válida es cero o una por reclamación; requiere validación y acta firmada. `ItemDisposal` cierra el ciclo sin entrega, con tipos destruido/desechado y evidencia; artículo e historia se conservan.
- **Especialización y zonas:** documentos y tarjetas son detalles opcionales de FoundItem; nunca se muestra PAN completo, solo primeros seis/últimos cuatro dígitos. Finder no es entidad inicial: snapshot en FoundItem. Zona significa ubicación interna del establecimiento, mediante catálogo Lost & Found y asociación a locales, no `id_zona` geográfico.
- **PEC:** usar catálogos maestros propios Lost & Found (`LostFoundCatalog` / `LostFoundCatalogOption`); no reutilizar `sspectcatalog` como catálogo funcional. Reutilizar/adaptar establecimientos, usuarios, Keycloak/CASL, StorageService/GCS, notificaciones/correo, scheduler y exportación según DATA-001. La clasificación y demás configuraciones son propias del módulo; no reutilizar semántica de Quejas. Metadata/asociaciones de archivos son propias del dominio (`sspectlffile` lógico), separadas de `sspectfile`.
- **Auditoría y estados:** auditoría genérica durable de negocio complementa movimientos/transferencias/entregas/disposiciones; logs técnicos no la sustituyen. Estados son dimensiones separadas, controladas por dominio y no reutilizan semántica de Quejas.
- **Identidad y responsables:** ID Encuentra e itemCode son identificadores funcionales generados, independientes de PK física. Actores responsables referencian usuarios PEC; no hay persona/entidad Responsible ni texto libre.
- **Políticas y reportes:** CustodyPolicy se versiona; cambios no recalculan históricos y se conserva `expirationDate` aplicada. Plazos confirmados: documentos 30 días en local + 90 días en GCSS + destrucción; generales 90 días + correspondencia/transferencia configurada; alimentos hasta cierre + desecho con evidencia; alertas tres días antes. Reportes consultan modelo transaccional sin tablas de reporting iniciales, con filtros y exportación Excel; reutilizar infraestructura PEC, no la lógica de Quejas.
- **Integraciones funcionales:** voz para registrar pérdidas almacenada como archivo; transcripción e integración externa de autocompletado de cliente se contemplan con fallback manual y controles de privacidad. Figma es referencia UX; pantalla incompleta de Búsqueda Nacional es solo observación.
- **Cobertura obligatoria:** registro inicial, artículo perdido, artículo encontrado, documentos/tarjetas, búsqueda nacional, validación, entrega, custodia, contacto automático, vencimientos, reportes, administración y roles/permisos; transversalmente reglas, automatizaciones, búsqueda/identificación y auditoría/trazabilidad. Estos cuatro componentes son responsabilidades lógicas, no una decisión de cuatro servicios.

### Pendientes reales

1. Mecanismo técnico exacto para firmas del acta (DELIVERY-001).
2. Algoritmo, pesos y umbrales exactos de matching.
3. Proveedor y detalles de transcripción de audio.
4. Contrato técnico del servicio externo de autocompletado de cliente.
5. Detalle propio de las Specs funcionales posteriores, sin reabrir ni contradecir decisiones aprobadas.

Este apartado resume las decisiones arquitectónicas registradas en sus secciones originales. El documento está en estado **APPROVED**.
