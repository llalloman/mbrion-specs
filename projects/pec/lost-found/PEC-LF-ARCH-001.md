# Especificación: Arquitectura y dominio de Artículos encontrados y declarados perdidos

> Esta especificación es la fuente única de verdad para el desarrollo. Mantenerla actualizada durante el flujo **Idea & Spec → Plan → Tasks → Execute → Validate**. Los planes, tareas, implementación y validación deben derivarse de esta especificación; registrar aquí los cambios de alcance o decisiones.
>
> **Clasificación de información:** cada elemento se identifica como **CONFIRMADO**, **PROPUESTA / INFERIDO** o **PENDIENTE DE DEFINICIÓN**. Los hechos funcionales confirmados se contrastan con el documento funcional oficial del módulo; las propuestas no son decisiones aprobadas. Esta especificación describe arquitectura y dominio; no autoriza implementación.

## 1. Encabezado

| Campo | Valor |
|---|---|
| ID | `PEC-LF-ARCH-001` |
| Título | Arquitectura y dominio de Artículos encontrados y declarados perdidos |
| Versión | `0.3.0` |
| Autor | Mbrion / equipo PEC — por confirmar |
| Fecha | `2026-09-30` |
| Work item / backlog | Pendiente de referencia |

## 2. Problema

**CONFIRMADO:** PEC requiere un módulo transversal para administrar registros de artículos perdidos y encontrados y los procesos asociados: búsqueda nacional, validación de propiedad, custodia, contacto, entrega, vencimientos, reportes, administración y permisos.

**PROPUESTA / INFERIDO:** estos procesos comparten información y evidencia, pero no necesariamente el mismo ciclo de vida. En particular, un reporte de pérdida describe una declaración de una persona y no implica que exista un artículo físico bajo custodia. Un artículo encontrado describe un objeto localizado y puede existir sin que haya un reporte coincidente.

**PENDIENTE DE DEFINICIÓN:** el vocabulario oficial del dominio, los límites entre “perdido”, “olvidado” y “encontrado”, y los procedimientos vigentes por tipo de artículo y local.

## 3. Resultado esperado

Disponer de un modelo de dominio y una arquitectura conceptual revisables que permitan derivar posteriormente especificaciones funcionales y técnicas sin confundir declaraciones, artículos físicos, coincidencias, validaciones, contactos, custodia y entregas. Las decisiones no sustentadas deben permanecer abiertas.

## 4. Alcance

### Incluye

- **CONFIRMADO:** registro inicial; artículo perdido; artículo encontrado; documentos y tarjetas; búsqueda nacional; validación de propiedad; entrega; custodia; contacto; vencimientos; reportes; administración; roles y permisos.
- **PROPUESTA / INFERIDO:** identificar conceptos, responsabilidades de dominio, relaciones conceptuales, dimensiones de estado, integraciones PEC por investigar, reglas de seguridad y preguntas bloqueantes para el diseño posterior.
- **CONFIRMADO:** analizar como hipótesis, sin asumirlos como entidades definitivas, los conceptos preliminares enumerados en la sección 11.

### Fuera de alcance

- Implementar código o modificar `pec-api` o `pec-ui`.
- Crear modelos Prisma, migraciones, endpoints, componentes Angular o contratos implementables.
- Aprobar un algoritmo de búsqueda/coincidencias, plazos de vencimiento, políticas de retención, estructura física de datos o permisos detallados.
- Definir integraciones como existentes o disponibles sin verificar sus capacidades y responsables.

### Supuestos

- **CONFIRMADO:** el módulo pertenece al producto PEC. PEC ya cuenta con capacidades técnicas de autenticación, autorización, identidad, archivos, notificaciones, reportes/exportación y ejecución programada que se deben evaluar para el módulo.
- **PROPUESTA / INFERIDO:** debe mantenerse trazabilidad entre los registros del módulo y el local responsable, sin duplicar innecesariamente identidades o catálogos existentes.
- **PENDIENTE DE DEFINICIÓN:** autoridad, calidad, permisos, contratos y adecuación funcional de cada capacidad PEC; su existencia no implica reutilización directa.

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
- **RB-10 — PENDIENTE DE DEFINICIÓN:** precisar evento técnico de inicio del cómputo, calendario y zona horaria, excepciones, responsables, parametrización y comportamiento ante fallos.
- **RB-11 — PROPUESTA / INFERIDO:** una coincidencia sugerida no prueba propiedad ni autoriza por sí sola una entrega; el documento funcional la mantiene pendiente de validación.
- **RB-12 — PENDIENTE DE DEFINICIÓN:** definir si el reporte de pérdida puede cerrarse/reabrirse y la cardinalidad/resolución de asociaciones entre varios reportes y artículos.

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

- **CA-B01:** Dado que la longitud del número de una tarjeta es menor o igual al formato de visualización definido, cuando se genera una vista o evidencia, entonces se aplica una política segura pendiente de definir sin exponer el número completo.
- **CA-B02:** Dado que existen varios reportes candidatos o varios artículos compatibles, cuando el sistema propone coincidencias, entonces el procedimiento de desempate o revisión humana permanece explícito y no se inventa como decisión aprobada.
- **CA-B03:** Dado que se describe el vencimiento de un documento, artículo general o alimento, cuando se especifica su flujo, entonces se reflejan los plazos confirmados de la regla RB-06 a RB-09 y se dejan pendientes solo las variables técnicas y operativas de RB-10.
- **CA-B04:** Dado que se registra una transición relevante, cuando se especifica la evidencia de auditoría, entonces se conservan quién actuó, cuándo, desde qué local y qué cambió, como requiere el documento funcional; el formato técnico, acceso e integridad siguen por definir.

## 7. Datos personales

- **¿Aplica?** [x] Sí  [ ] No
- **Datos tratados:** **CONFIRMADO:** tipo y número de identificación; nombres y apellidos; celular/teléfono; email; fotografías; archivos/documentos; datos de tarjetas; firmas; respuestas de validación. Los campos funcionales del registro también incluyen descripción del artículo, tema/subtema, lugar/fecha aproximados y datos del local/usuario que registra.
- **Finalidad y uso:** **CONFIRMADO:** registrar declaraciones y hallazgos, buscar coincidencias, validar propiedad, contactar al cliente, documentar entrega/custodia y mantener trazabilidad. Base legal, retención de datos personales y mecanismo técnico de tratamiento: **PENDIENTE DE DEFINICIÓN**.
- **Restricciones de acceso:** **CONFIRMADO:** la matriz funcional por rol/acción de la sección 13; las respuestas de validación no se muestran previamente al cliente; solo el local que posee físicamente el artículo modifica custodia. Visibilidad de cada campo/canal sensible: **PENDIENTE DE DEFINICIÓN**.
- **Restricciones de logs:** **PROPUESTA / INFERIDO:** no registrar números completos de tarjetas, respuestas secretas de validación, documentos íntegros, credenciales ni datos de contacto innecesarios. Definir minimización, enmascaramiento, acceso y retención de logs antes del diseño técnico.
- **Restricciones de QA:** **PROPUESTA / INFERIDO:** usar datos sintéticos o anonimizados; no utilizar tarjetas reales ni respuestas reales de validación. Confirmar datos permitidos, ambientes y manejo de evidencias QA.

## 8. Riesgos técnicos

| Clasificación | Riesgo | Impacto | Mitigación / respuesta |
|---|---|---|---|
| PROPUESTA / INFERIDO | Acoplar el reporte de pérdida al objeto físico encontrado | Registros incoherentes cuando hay reportes sin hallazgo, duplicados o múltiples candidatos | Mantener conceptos separados en el modelo; validar relaciones y cardinalidad con negocio antes de persistencia |
| PROPUESTA / INFERIDO | Colapsar dimensiones de estado en un único estado general | Transiciones inválidas o pérdida de información sobre custodia, contacto, validación y matching | Modelar y revisar dimensiones separadas; definir reglas de coordinación explícitas |
| CONFIRMADO | Exposición de número de tarjeta o respuestas de validación | Fraude, divulgación de información y pérdida de confianza | Respetar las reglas RB-01 y RB-02 en todos los canales; revisar logs y datos QA |
| CONFIRMADO | Cambios de custodia por un local sin posesión física | Pérdida de trazabilidad y responsabilidad sobre el artículo | Vincular la autorización de cambio a la posesión física confirmada; mecanismo operativo pendiente |
| CONFIRMADO | Cerrar entrega sin acta firmada o sin auditoría | Disputa sobre entrega y falta de evidencia | Bloquear cierre sin acta; conservar auditoría de transiciones relevantes |
| PENDIENTE DE DEFINICIÓN | Cómputo, responsables y excepciones de vencimientos no acordados | Retención indebida, descarte prematuro o artículos sin gestionar | Acordar evento de inicio, calendario/zona horaria, responsables, parámetros, fallos y excepciones; aplicar las duraciones confirmadas |
| PROPUESTA / INFERIDO | Reutilizar servicios PEC cuya capacidad/contrato no se haya verificado | Duplicación, dependencia incorrecta o ruptura de controles compartidos | Investigar dueños, contratos, permisos, retención, disponibilidad y fuente de verdad |
| PENDIENTE DE DEFINICIÓN | Búsqueda nacional con criterios o datos no acordados | Falsos positivos/negativos, acceso excesivo o uso de datos incompatibles | Definir atributos, alcance, algoritmo, límites de acceso y revisión humana antes de diseñar interfaces |
| CONFIRMADO | Tratar logs técnicos o campos created/modified como historial funcional suficiente | No habría trazabilidad durable de movimientos de custodia, entregas o destrucción | Diseñar auditoría de negocio durable; el mecanismo físico sigue pendiente |
| PENDIENTE DE DEFINICIÓN | Reutilizar permisos existentes sin comprobar alcance local/empresa y posesión física | Acceso transversal o cambios de custodia por actores sin posesión | Definir autorización contextual y validar pertenencia antes de fijar el mapeo PEC |
| PENDIENTE DE DEFINICIÓN | Ejecutar vencimientos programados sin controles de concurrencia e idempotencia | Alertas, transferencias o disposiciones duplicadas o perdidas | Diseñar jobs nuevos con idempotencia, concurrencia, reintentos y registro de resultados |

## 9. Evidencia esperada

- Revisión de dominio que confirme o corrija la separación entre reporte de pérdida y artículo encontrado.
- Validación de las cardinalidades y escenarios de asociación entre reportes, artículos y coincidencias.
- Matriz acordada de dimensiones de estado, transiciones y reglas de coordinación.
- Matriz de permisos por rol, acción, local y posesión física.
- Confirmación por responsables de los contratos, propietarios, permisos y fuentes de verdad de las capacidades PEC existentes; su existencia técnica ya se verificó en el descubrimiento.
- Contraste con el documento funcional oficial del módulo: `1-Módulo-de-artículos-encontrados-y-declarados-perdidos.docx`; registrar discrepancias o cambios de decisión en esta especificación.
- Confirmación de los parámetros operativos todavía pendientes para vencimientos; las duraciones funcionales y alerta de 3 días están confirmadas.
- Decisiones pendientes sobre contacto, validación, entrega, custodia y conservación de datos, registradas como cambios de esta especificación.
- Revisión de privacidad y seguridad de los datos personales, números de tarjeta, respuestas de validación, logs y datos QA.

## 10. Dependencias

- **PENDIENTE DE DEFINICIÓN:** responsables de negocio y operación local para validar procesos, terminología, vencimientos y excepciones.
- **CONFIRMADO:** PEC cuenta con Keycloak, autorización CASL/PoliciesGuard, estructuras de usuarios/empresas/establecimientos, infraestructura de archivos en Google Cloud Storage, notificaciones in-app/email, reportes/exportación, y scheduler NestJS.
- **PENDIENTE DE DEFINICIÓN:** responsables y contratos de esas capacidades; mapeo de roles, alcance empresa/local, dominio Zone, permisos/retención por archivo, eventos y plantillas de notificación, límites de reportes y auditoría durable.
- **PENDIENTE DE DEFINICIÓN:** responsables de Gestión de Quejas, Redes Sociales y locales para confirmar colaboración, visibilidad y límites de acceso.
- **PENDIENTE DE DEFINICIÓN:** definición de GCSS u otros destinos de custodia/archivo, incluidos responsable, flujo y evidencia, si aplican.
- **PENDIENTE DE DEFINICIÓN:** políticas de privacidad, seguridad, retención y uso de datos personales.

## 11. Modelo conceptual / datos (cuando aplique)

### Principio de modelado

- **CONFIRMADO:** un reporte de pérdida no es necesariamente el mismo concepto que un artículo físico encontrado.
- **PROPUESTA / INFERIDO:** representar por separado la declaración de una persona (`LostItemReport`) y el objeto localizado bajo responsabilidad operativa (`FoundItem`). La relación entre ellos se expresa mediante una o varias coincidencias revisables (`ItemMatch`), no por identidad ni por asociación automática definitiva.
- **PENDIENTE DE DEFINICIÓN:** nombres oficiales, cardinalidades, atributos, identificadores, límites de agregados y autoridad del dato.

### Conceptos candidatos

| Concepto candidato | Responsabilidad conceptual propuesta | Clasificación / dudas |
|---|---|---|
| `LostItemReport` | Declaración de pérdida u olvido, descripción, reportante, fecha/lugar aproximados y canal de registro | **PROPUESTA / INFERIDO.** Confirmar si “perdido” y “olvidado” son tipos distintos y qué datos son obligatorios. |
| `FoundItem` | Artículo físico localizado; descripción, lugar/fecha de hallazgo, local que lo posee y categoría | **PROPUESTA / INFERIDO.** Definir cuándo nace, duplicados, identificadores y propiedad física verificable. |
| `ItemMatch` | Candidato o asociación revisable entre uno o más reportes y artículos | **PROPUESTA / INFERIDO.** Definir algoritmo, criterios, revisión manual, cardinalidad y quién confirma/rechaza. |
| `OwnershipValidation` | Intento/proceso de acreditar propiedad, preguntas/evidencias y resultado protegido | **PROPUESTA / INFERIDO.** Definir preguntas, evidencia, reintentos, responsables, acceso y tratamiento de respuestas. |
| `ContactManagement` | Registro y coordinación de intentos de contacto y resultados | **PROPUESTA / INFERIDO.** Puede ser capacidad/proceso y no entidad independiente; definir canales, responsables y vencimientos. |
| `CustodyMovement` | Evento de recepción, traslado, entrega a otro custodio o cambio de ubicación/responsable | **PROPUESTA / INFERIDO.** Solo el local con posesión física puede cambiar custodia. Definir si traslados entre locales o hacia GCSS aplican. |
| `ItemDelivery` | Proceso de entrega a la persona autorizada, con acta firmada y evidencia | **PROPUESTA / INFERIDO.** Cierre bloqueado sin acta firmada. Definir firmantes, representación y excepciones. |
| `ItemTransfer` | Traslado administrativo/físico entre responsables o centros | **PROPUESTA / INFERIDO.** Puede ser evento de custodia y no concepto separado; definir frontera con `CustodyMovement`. |
| `ItemDisposal` | Disposición final del artículo vencido/no reclamado, con autorización y evidencia | **PROPUESTA / INFERIDO.** Definir plazos, aprobadores, métodos y constancia requerida por categoría. |
| `AuditHistory` | Evidencia durable de acciones y transiciones relevantes | **PROPUESTA / INFERIDO.** El requisito de auditoría de negocio está confirmado, pero PEC no tiene historial genérico de dominio encontrado; mecanismo, alcance, fuente de verdad y retención siguen pendientes. |

### Relaciones propuestas

- **PROPUESTA / INFERIDO:** un `LostItemReport` puede no tener coincidencias, tener una o varias candidatas; un `FoundItem` puede no tener reportes asociados o relacionarse con candidatos múltiples. La asociación aceptada debe tener reglas de resolución aún por definir.
- **PROPUESTA / INFERIDO:** una validación de propiedad aplica a una reclamación sobre un artículo encontrado (y posiblemente a una coincidencia); su relación exacta con `ItemMatch` queda pendiente.
- **PROPUESTA / INFERIDO:** un artículo encontrado tiene una secuencia de movimientos de custodia y, si procede, transferencia, entrega o disposición. No se presupone que todas las acciones sean estados mutuamente excluyentes.
- **PROPUESTA / INFERIDO:** contacto y entrega se relacionan con la persona/reclamación y el artículo, con visibilidad de información limitada por rol.
- **PENDIENTE DE DEFINICIÓN:** si documentos y tarjetas son subtipos de artículo, categorías, adjuntos o referencias a un catálogo; y si un documento físico y una tarjeta tienen reglas distintas.

### Límites de dominio propuestos

| Dominio / capacidad | Responsabilidad propuesta | Clasificación |
|---|---|---|
| Declaraciones de pérdida | Registrar y mantener declaraciones de objetos perdidos/olvidados y datos aportados por quien reporta | **PROPUESTA / INFERIDO** |
| Artículos encontrados y registro inicial | Registrar objeto físico, hallazgo, categoría, local poseedor y descripción | **PROPUESTA / INFERIDO** |
| Matching y búsqueda nacional | Encontrar candidatos y permitir revisión autorizada de relaciones entre declaraciones y artículos | **PROPUESTA / INFERIDO.** Privacidad, alcance nacional y algoritmo pendientes. |
| Validación de propiedad | Gestionar acreditación y proteger preguntas/respuestas y evidencias | **PROPUESTA / INFERIDO** |
| Contacto | Coordinar intentos y resultados sin exponer datos innecesarios | **PROPUESTA / INFERIDO** |
| Custodia y vencimientos | Registrar posesión, movimientos, ubicación, plazos y acciones al vencer | **PROPUESTA / INFERIDO.** Política operativa pendiente. |
| Entrega y disposición | Controlar salida por entrega autorizada o disposición documentada | **PROPUESTA / INFERIDO** |
| Administración, permisos y reportes | Configuración operativa, acceso y vistas de seguimiento autorizadas | **PROPUESTA / INFERIDO.** Alcance por rol y datos reportables pendientes. |
| Capacidades PEC compartidas | Identidad, autorización, usuarios, empresas, establecimientos, archivos, notificaciones, exportación y scheduler existen técnicamente en PEC | **CONFIRMADO** por descubrimiento técnico. Reutilización/adaptación se propone por capacidad en la sección 14; contratos, permisos y adecuación siguen pendientes. |

## 12. Estados y transiciones (cuando aplique)

**CONFIRMADO:** el descubrimiento técnico no encontró una máquina de estados genérica reutilizable; los estados de quejas son propios de ese dominio y no se reutilizarán semánticamente para Lost & Found.

**PROPUESTA / INFERIDO:** separar analíticamente reporte, matching, validación, contacto, custodia, entrega y vencimiento en dimensiones de estado distintas. El documento funcional presenta algunos estados en una misma lista de custodia; la separación conceptual aquí propuesta no aprueba todavía entidades ni máquinas de estados técnicas.

| Dimensión candidata | Ejemplos de estados/eventos para analizar (no aprobados) | Regla conocida | Decisión pendiente |
|---|---|---|---|
| Reporte de pérdida | Registro con ID Encuentra automático; acciones Guardar, Buscar coincidencias y Cancelar | **CONFIRMADO:** registro inicial y datos funcionales descritos en el documento | Estados posteriores, edición/cancelación, duplicados y cardinalidad con hallazgos |
| Artículo encontrado | Registrado | **CONFIRMADO:** estado inicial indicado | Transiciones posteriores y relación con reportes |
| Custodia local/GCSS | En custodia del local; Pendiente de transferencia; Transferido al GCSS; En custodia GCSS; Pendiente de destrucción; Destruido; Entregado al cliente | **CONFIRMADO:** solo el local que posee físicamente el artículo modifica custodia. Documentos: 1 mes local y 3 meses GCSS antes de destrucción. | Secuencia exacta, recepción/acuse, responsable y excepciones; artículo general y alimentos siguen RB-07/RB-08 |
| Matching | Posible coincidencia, pendiente de validación; revisar fotografías/características y confirmar o descartar | **CONFIRMADO:** existe el concepto funcional de posible coincidencia pendiente de validación | Algoritmo, umbrales, cardinalidad, prioridad y reversión |
| Validación | Resultados: alta, media o baja coincidencia; validación correcta habilita continuar con entrega | **CONFIRMADO:** nunca mostrar previamente las respuestas registradas | Preguntas/configuración, número de intentos y resultado requerido para aprobación |
| Contacto | Gestión de contacto; Cliente contactado. Resultados: Contactado, No contesta, Número incorrecto, Se acercará al local | **CONFIRMADO:** activar correo y/o llamada cuando existan datos del propietario | Frecuencia, tiempos, consentimiento, errores y cierre de seguimiento |
| Entrega | Generar acta automáticamente; cargar acta firmada | **CONFIRMADO:** no cerrar caso sin el archivo del acta firmada | Firmantes admitidos, validez de firma, cancelación, archivo y trazabilidad física |
| Vencimiento / disposición | Documentos: 1 mes local → transferencia pendiente → GCSS 3 meses → destrucción; artículos generales: 3 meses → correspondencia interna; alimentos: hasta cierre del local → desecho con evidencia | **CONFIRMADO:** alerta 3 días antes por sistema/correo/reporte Excel/notificación | Evento técnico de inicio, zona horaria/calendario, excepciones, responsables, parámetros y fallos |

**CONFIRMADO — requisito funcional de auditoría:** registrar quién realizó la acción, cuándo, desde qué local y qué cambió. **PROPUESTA / INFERIDO:** registrar también origen/destino, motivo y referencia a evidencia cuando aplique. Fuente PEC, formato, integridad, acceso y retención quedan pendientes.

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
- **PENDIENTE DE DEFINICIÓN:** alcance territorial exacto de Admin Local; si existen permisos adicionales; mapeo de estos roles a Keycloak/PEC; permisos de solo lectura; y administración de usuarios/configuración.

## 14. Integraciones (cuando aplique)

Los hallazgos siguientes describen la arquitectura actual observada en `pec-api` y `pec-ui`. La clasificación técnica de una capacidad no aprueba por sí sola su contrato ni su uso en el módulo.

| Capacidad / evidencia PEC | Hallazgo confirmado | Tratamiento propuesto para Lost & Found | Pendiente / riesgo |
|---|---|---|---|
| Autenticación Keycloak y contexto backend | **CONFIRMADO:** PEC usa Keycloak. `AuthGuard` obtiene `preferred_username` y carga contexto de usuario/rol. | **REUTILIZAR / ADAPTAR:** flujo de autenticación y contexto existente. | **PENDIENTE:** mapeo de `Gestor Local` (no se encontró con ese nombre), autorización contextual por posesión física/local y alcance efectivo por empresa/local. |
| Autorización backend | **CONFIRMADO:** PEC usa CASL y `PoliciesGuard`; existe infraestructura reutilizable para permisos backend. | **EXTENDER / ADAPTAR:** expresar políticas propias del módulo dentro del marco existente. | **PENDIENTE:** roles y acciones concretas, ámbito empresa/local y comprobación contextual de posesión física. |
| Usuarios y empresas | **CONFIRMADO:** existen `UserEnterpriseRole` y contexto de empresas/usuarios. | **ADAPTAR / REUTILIZAR:** referenciar usuarios y empresa existentes como fuentes PEC, tras validar contratos. | **PENDIENTE:** fuente de verdad para reportantes externos y alcance/selección efectiva de empresa. |
| Establecimientos y ubicación | **CONFIRMADO:** existen `UserEstablishment`, establecimientos, empresa, ciudad/región; `id_zona` existe como atributo. No se encontró un dominio completo `Zone`. | **ADAPTAR / REUTILIZAR:** establecimientos y atributos geográficos existentes, si cubren la necesidad. | **PENDIENTE:** zonas: significado, catálogo, jerarquía, autoridad y uso operativo; alcance real del usuario por establecimiento. |
| Catálogos | **CONFIRMADO:** PEC tiene infraestructura técnica de catálogos. | **ADAPTAR / EXTENDER:** evaluar infraestructura para catálogos propios del módulo. | **PENDIENTE:** no asumir que temas, subtemas, estados ni tipos de Lost & Found compartan los catálogos actuales; confirmar semántica, administración y dueño. |
| Archivos | **CONFIRMADO:** existe almacenamiento de archivos con Google Cloud Storage, con endpoints/servicios protegidos y asociaciones específicas a quejas. | **REUTILIZAR / ADAPTAR:** infraestructura de almacenamiento; crear asociación conceptual propia del dominio Lost & Found. | **PENDIENTE:** permisos por recurso, acceso, retención, integridad, borrado y contrato de asociación para fotos, documentos y actas. La asociación a quejas no se reutiliza como asociación de dominio. |
| Notificaciones y correo | **CONFIRMADO:** hay notificaciones in-app y email, con procesadores por evento. | **REUTILIZAR / EXTENDER:** mecanismos y canales existentes; definir nuevos eventos del módulo. | **PENDIENTE:** eventos y plantillas, destinatarios, consentimiento, reintentos, fallos y datos expuestos. |
| Auditoría | **CONFIRMADO:** existen logs técnicos y campos `created`/`modified` en algunos modelos. No se encontró historial genérico de auditoría de dominio. | **PROPUESTA / INFERIDO:** Lost & Found requiere auditoría de negocio durable para custodia, entrega y destrucción; los logs técnicos no la sustituyen. | **PENDIENTE:** mecanismo físico, fuente de verdad, integridad, acceso y retención. |
| Reportes y exportación | **CONFIRMADO:** existe `ReportsService`; `ExcelExportService` usa ExcelJS y el frontend también exporta con `xlsx`. | **REUTILIZAR / EXTENDER:** infraestructura genérica de exportación, sujeto a adecuación. **NO REUTILIZAR:** lógica funcional de reportes de quejas. | **PENDIENTE:** decidir generación backend o frontend, permisos, campos, volumen y protección de datos exportados. |
| Scheduler y trabajos programados | **CONFIRMADO:** NestJS Schedule está habilitado y PEC contiene jobs programados. | **ADAPTAR / REUTILIZAR:** infraestructura scheduler para orquestar los procesos nuevos. **EXTENDER:** implementar jobs de vencimientos Lost & Found como procesos propios. | **PENDIENTE:** idempotencia, concurrencia, reintentos, recuperación y auditoría de ejecuciones. |
| Estados | **CONFIRMADO:** no se encontró máquina de estados genérica reutilizable; existen estados de quejas propios de ese dominio. | **NO REUTILIZAR** semánticamente los estados de quejas. **PROPUESTA / INFERIDO:** definir dimensiones de estado del módulo. | **PENDIENTE:** catálogo, transiciones, coordinación e implementación técnica de estados propios. |
| Frontend y módulo funcional | **CONFIRMADO:** existe Keycloak frontend; módulos lazy-loaded, Angular Material + Fuse + Tailwind, formularios reactivos, servicios con `HttpClient` e infraestructura de archivos. No existe módulo/ruta Lost & Found. | **PROPUESTA / INFERIDO:** implementar el futuro módulo como feature aislada; no extender internamente el módulo de quejas. | **PENDIENTE:** rutas, guards/permisos, flujos, asociación de archivos y diseño detallado. |
| Gestión de Quejas | **CONFIRMADO:** varias capacidades (por ejemplo, asociaciones de archivos y reportes) están ligadas específicamente a quejas. | **NO REUTILIZAR** lógica de dominio de quejas para representar Lost & Found; reutilizar solo infraestructura genérica tras evaluación. | **PENDIENTE:** cualquier referencia o integración de negocio entre quejas y Lost & Found. |
| GCSS / correspondencia | **CONFIRMADO funcionalmente:** los flujos de documentos y artículos generales incluyen transferencia a GCSS/correspondencia interna. | **PROPUESTA / INFERIDO:** modelar como destino/proceso operativo externo al módulo hasta comprobar una integración PEC. | **PENDIENTE:** integración PEC existente (no confirmada), responsables, protocolo, recepción/acuse y evidencia. |

### Implementación Lost & Found existente

**CONFIRMADO:** el descubrimiento no encontró modelos, endpoints, servicios, rutas ni componentes específicos de PEC para artículos perdidos, artículos encontrados, custodia, matching, validación de propiedad, entrega o destrucción/disposición. Este hallazgo describe la revisión realizada y no aprueba crear estructuras de implementación en esta especificación.

### Requerimientos funcionales confirmados de administración y reportes

Estos elementos constan en el documento funcional; su implementación o integración técnica con PEC no se presume.

- **CONFIRMADO — Administración:** catálogos de temas, subtemas, locales, zonas, tipos de identificación, canales, estados, roles, plantillas de correo, preguntas de validación, tipos de documentos y tipos de tarjetas.
- **CONFIRMADO — Parámetros administrables:** tiempo de custodia, tiempo de alertas, destinos y responsables.
- **CONFIRMADO — Reporte general:** ID Encuentra, fecha de registro, comentario, tema, subtema, local, zona, cantidad, estado, responsable y fecha de vencimiento.
- **CONFIRMADO — Reportes:** artículos encontrados/perdidos, pendientes de entrega, próximos a vencer, transferidos, destruidos, documentos y tarjetas; agrupación/filtro por local, zona, fechas, tema y estado; exportación a Excel.
- **PENDIENTE DE DEFINICIÓN:** permisos de cada rol para administrar catálogos/parámetros y consultar/exportar campos o reportes.

## 15. Contratos / API (cuando aplique)

No se definen rutas, endpoints, eventos ni esquemas de API en esta especificación de arquitectura. Su diseño queda **FUERA DE ALCANCE** en esta etapa. Antes de especificarlos, resolver límites de dominio, permisos, relaciones, estados y contratos de las integraciones investigadas.

## 16. Pantallas afectadas (cuando aplique)

**CONFIRMADO:** existe un prototipo funcional de Figma como referencia de UX/UI para el módulo. En `pec-ui`, Keycloak frontend está integrado, los módulos se cargan lazy, se usan Angular Material + Fuse + Tailwind, formularios reactivos y servicios HTTP mediante `HttpClient`; también existe infraestructura de archivos. No existe actualmente módulo ni ruta Lost & Found.

**PROPUESTA / INFERIDO:** el nuevo módulo debe implementarse como feature aislada y no como extensión interna del módulo de quejas.

**OBSERVACIÓN DEL PROTOTIPO — no es comportamiento esperado:** la opción “Búsqueda Nacional” muestra navegación/pantalla incompleta en el prototipo. El comportamiento funcional debe derivarse del documento funcional (buscar artículos de cualquier local, con sus filtros y campos de resultado confirmados), no de esa pantalla incompleta.

Los flujos, vistas finales y perfiles de pantalla se documentarán en especificaciones funcionales posteriores.

## 17. Decisiones técnicas (cuando aplique)

| Decisión / principio | Motivo | Estado / consecuencia |
|---|---|---|
| Separar conceptualmente reporte de pérdida y artículo encontrado | **CONFIRMADO:** un reporte no implica necesariamente un objeto encontrado bajo custodia | Principio arquitectónico confirmado; estructura física y cardinalidades pendientes |
| Analizar dimensiones independientes para matching, validación, contacto y custodia | Son procesos distintos con reglas y responsables potencialmente distintos | **PROPUESTA / INFERIDO:** validar coordinación y estados antes de diseño técnico |
| No definir todavía entidades, tablas, endpoints ni modelos de implementación | Esta especificación es de arquitectura/dominio y varias decisiones base siguen abiertas | Restricción de alcance de esta especificación |
| Mantener protección transversal de tarjeta, respuestas de validación, acta y auditoría | Reglas funcionales confirmadas | Definir controles y contratos después de investigar capacidades PEC |
| No asumir reutilización de servicios PEC por nombre | La capacidad, contrato y dueño deben verificarse | **PENDIENTE DE DEFINICIÓN:** investigación de integraciones |
| Evaluar motores transversales de reglas, automatizaciones, búsqueda/identificación y auditoría/trazabilidad | El documento funcional los recomienda para implementar tiempos, transiciones, transferencias, alertas, coincidencias e historial | **PROPUESTA / INFERIDO:** son recomendaciones de arquitectura, no componentes existentes ni decisiones técnicas aprobadas |
| Aprovechar Keycloak y la infraestructura CASL/PoliciesGuard | Son capacidades técnicas actuales de PEC para autenticación y autorización | **CONFIRMADO** en PEC; adaptación y autorización contextual para Lost & Found pendientes |
| Usar almacenamiento y canales transversales PEC con asociación propia del módulo | GCS, archivos protegidos, notificaciones in-app/email y procesadores por evento ya existen | **PROPUESTA / INFERIDO:** evaluar/adaptar infraestructura; permisos, eventos, retención y contratos siguen pendientes |
| Diseñar auditoría de negocio durable para el módulo | No se encontró historial genérico de auditoría de dominio y los logs técnicos no sustituyen trazabilidad de custodia/entrega/destrucción | **PROPUESTA / INFERIDO:** requisito de dominio confirmado; mecanismo físico final pendiente de diseño |
| Implementar el futuro frontend como feature aislada | No existe actualmente ruta/módulo Lost & Found y la lógica de quejas es específica | **PROPUESTA / INFERIDO:** no anidar el módulo dentro de quejas; aplicar patrones frontend compartidos donde sean adecuados |
| Preservar el estado de trabajo existente de `pec-api` y `pec-ui` | Hay archivos modificados y no versionados en ambos repositorios observados durante el descubrimiento | Antes de cualquier implementación futura, revisar `git status` y preservar íntegramente esos archivos; no sobrescribir ni limpiar cambios ajenos |

## 18. Preguntas pendientes

1. **Terminología:** ¿“perdido” y “olvidado” son categorías distintas? El documento usa ambos términos; falta acordar vocabulario y relación sin fusionar `LostItemReport` y `FoundItem`.
2. **Reporte de pérdida:** ¿Qué campos son obligatorios/opcionales y quién puede corregir, cancelar, cerrar o reabrir el reporte?
3. **Artículo encontrado:** ¿Cuál es el momento exacto de creación/recepción física, cómo se detectan duplicados y qué identifica un artículo sin serie/código?
4. **Relaciones:** ¿Un reporte puede asociarse con varios artículos y viceversa? ¿Quién confirma o disputa una coincidencia?
5. **Matching:** ¿Qué algoritmo, atributos, umbrales, priorización y revisión humana producen el nivel de coincidencia? Los filtros/columnas de búsqueda nacional constan en el documento funcional, pero la pantalla Figma está incompleta.
6. **Tarjetas:** ¿Cómo aplicar el formato primeros 6/últimos 4 a números cortos, capturas, documentos y canales? ¿Se conserva el número completo y, si es así, con qué controles?
7. **Documentos y tarjetas:** ¿Se modelan como categorías/subtipos de `FoundItem`, adjuntos o referencias? ¿Cómo se gestionan autocompletado e integración de datos cuando exista?
8. **Validación de propiedad:** ¿Quién configura preguntas y respuestas, cuántos intentos hay y qué combinación de respuestas/nivel (alta, media, baja) califica como válida o inconclusa?
9. **Roles:** ¿Cuál es el alcance territorial exacto de Admin Local? ¿Hay permisos adicionales? ¿Cómo se mapean los cuatro roles a Keycloak/PEC? ¿Hay permisos de solo lectura y quién administra usuarios/configuración?
10. **Posesión/custodia:** ¿Cómo se acredita la posesión física y quién registra la recepción? ¿Qué protocolo confirma cada traslado local↔GCSS y cambio de responsable?
11. **Estados:** ¿Cuáles son las transiciones autorizadas y sus condiciones para reporte, matching, validación, contacto, custodia, entrega y vencimiento? ¿Cómo se coordinan las dimensiones separadas?
12. **Entrega/acta:** ¿Qué tipos de firma son válidos, quiénes deben firmar, cómo se carga/archiva el acta y cuál es el destinatario/proceso de envío físico a Experiencia del Cliente (el documento deja el nombre del destinatario como “xxx”)? ¿Qué cancelaciones se permiten?
13. **Contacto:** ¿Cuándo se envían correo y gestión telefónica, con qué frecuencia/reintentos y reglas de consentimiento? ¿Cómo se cierran los resultados y se gestionan números erróneos?
14. **Vencimientos — pendientes operativos/técnicos:** confirmar evento de inicio del cómputo, calendario/zona horaria, excepciones, responsables, parámetros y comportamiento ante fallos. Plazos por tipo y alerta 3 días antes están confirmados en RB-06 a RB-09.
15. **Transferencia/disposición:** ¿Qué acuse/evidencia se conserva para correspondencia interna, GCSS y destrucción/desecho? ¿Quién es actualmente responsable de correspondencia (el documento nombra a Antonio Mejía) y qué aprobaciones aplican?
16. **GCSS:** confirmar el nombre institucional vigente, propietario, localización, protocolo de recepción/custodia, consultas y retención de evidencias.
17. **Auditoría:** PEC no tiene un historial genérico de auditoría de dominio encontrado. ¿Qué mecanismo durable se diseñará para guardar quién, cuándo, desde qué local y qué cambió? ¿Qué acceso, integridad y retención requiere?
18. **Integraciones PEC:** ¿Quiénes son propietarios y cuáles son los contratos/garantías de Keycloak, CASL/PoliciesGuard, usuarios/empresas/establecimientos, GCS/archivos, notificaciones, reportes/exportación y scheduler para el uso del módulo?
19. **Privacidad:** ¿Cuál es la base legal, retención, acceso, eliminación y manejo de incidentes para los datos personales confirmados? ¿Qué datos pueden utilizarse en QA?
20. **Reportes y administración:** los campos, reportes, catálogos y parámetros funcionales están enumerados en la sección 14. ¿Qué roles pueden administrarlos, consultarlos y exportarlos, y qué restricciones de datos aplican?
21. **Aprobación y backlog:** ¿Qué work item, responsables de negocio/operación, aprobadores y fecha de aprobación deben asociarse a esta especificación?

## 19. Estado de aprobación

- **Estado:** Borrador
- **Aprobadores:** Pendiente de identificar
- **Fecha y observaciones:** 2026-09-30 — documento de arquitectura y dominio para revisión; propuestas e inferencias no aprobadas y preguntas abiertas.

## 20. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.1.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Primera especificación transversal de arquitectura y dominio; distingue confirmado, propuesto y pendiente. |
| 0.2.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Reclasifica vencimientos, alertas, permisos, datos personales, reportes/administración y observación de Figma según el documento funcional oficial; mantiene pendientes los aspectos técnicos y operativos no definidos. |
| 0.3.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Incorpora hallazgos del descubrimiento técnico de pec-api/pec-ui; clasifica capacidades PEC existentes, límites de reutilización, ausencia de implementación Lost & Found y preservación del estado de trabajo. |
