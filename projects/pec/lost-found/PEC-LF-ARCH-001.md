# Especificación: Arquitectura y dominio de Artículos encontrados y declarados perdidos

> Esta especificación es la fuente única de verdad para el desarrollo. Mantenerla actualizada durante el flujo **Idea & Spec → Plan → Tasks → Execute → Validate**. Los planes, tareas, implementación y validación deben derivarse de esta especificación; registrar aquí los cambios de alcance o decisiones.
>
> **Clasificación de información:** cada elemento se identifica como **CONFIRMADO**, **PROPUESTA / INFERIDO** o **PENDIENTE DE DEFINICIÓN**. Los hechos funcionales confirmados se contrastan con el documento funcional oficial del módulo; las propuestas no son decisiones aprobadas. Esta especificación describe arquitectura y dominio; no autoriza implementación.

## 1. Encabezado

| Campo | Valor |
|---|---|
| ID | `PEC-LF-ARCH-001` |
| Título | Arquitectura y dominio de Artículos encontrados y declarados perdidos |
| Versión | `0.4.0` |
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
- Aprobar el algoritmo de búsqueda/coincidencias, políticas de retención, estructura física de datos o permisos detallados. Las duraciones funcionales confirmadas se mantienen como requisitos; los parámetros técnicos de cómputo siguen pendientes.
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

**CONFIRMADO — estados funcionales requeridos:** Registrado; En custodia del local; Gestión de contacto; Cliente contactado; Entregado al cliente; Pendiente de transferencia; Transferido al GCSS; En custodia GCSS; Pendiente de destrucción; Destruido; Posible coincidencia.

**PROPUESTA / INFERIDO:** no representar todos estos conceptos en un único campo `status`. Separar analíticamente custodia, matching, validación, contacto, entrega y vencimiento. Esta separación no aprueba todavía modelos físicos ni máquinas de estados técnicas.

| Dimensión | Estados/eventos funcionales y tratamiento | Regla conocida | Decisión pendiente |
|---|---|---|---|
| Registro de artículo | `Registrado` | **CONFIRMADO:** estado inicial del registro de artículo encontrado | Transiciones posteriores y su coordinación con custodia |
| Custodia | En custodia del local; Pendiente de transferencia; Transferido al GCSS; En custodia GCSS; Pendiente de destrucción; Destruido | **CONFIRMADO:** solo el local con posesión física puede gestionar la custodia. Plazos según RB-06 a RB-08. | Secuencia exacta, recepción/acuse, responsable y excepciones |
| Matching | Posible coincidencia; pendiente de validación; confirmar o descartar tras revisar características/fotografías | **CONFIRMADO:** existe el concepto funcional de posible coincidencia | Algoritmo, umbrales, cardinalidad, prioridad y reversión |
| Validación | Resultado de coincidencia alta, media o baja; validación correcta permite continuar a entrega | **CONFIRMADO:** nunca mostrar previamente las respuestas registradas | Preguntas configurables, intentos y criterio de validación satisfactoria |
| Contacto | Gestión de contacto; Cliente contactado. Resultados: Contactado, No contesta, Número incorrecto, Se acercará al local | **CONFIRMADO:** activar correo y/o llamada cuando existan datos del propietario | Frecuencia, tiempos, consentimiento, errores y cierre de seguimiento |
| Entrega | Entregado al cliente, después de completar la evidencia exigida | **CONFIRMADO:** generar acta y cargar el acta firmada; no cerrar sin ella | Firmantes admitidos, validez de firma, cancelación, archivo y trazabilidad física |
| Vencimiento / disposición | Documentos: 1 mes local → transferencia → GCSS → 3 meses → destrucción; artículos generales: 3 meses → correspondencia interna; alimentos: hasta cierre del local → desecho con evidencia | **CONFIRMADO:** alertas 3 días antes por sistema, correo y reporte Excel/notificación | Evento técnico de inicio, zona horaria/calendario, excepciones, responsables, parámetros y fallos |

**CONFIRMADO — requisito funcional de auditoría:** registrar quién realizó la acción, cuándo, desde qué local y qué cambió. **PROPUESTA / INFERIDO:** registrar también origen/destino, motivo y referencia a evidencia cuando aplique. Logging técnico existe en PEC, pero no reemplaza la auditoría de negocio durable. Fuente, formato, integridad, acceso y retención quedan pendientes.

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

## Cobertura funcional completa del requerimiento

Esta sección refleja los trece módulos del documento funcional y su tratamiento de arquitectura. `ARCH-001` describe capacidades, datos de negocio a nivel conceptual, reglas y dependencias; no define columnas físicas, claves foráneas, modelos Prisma ni tablas. El diseño lógico de entidades y relaciones corresponde a `PEC-LF-DATA-001`; el detalle de comportamiento implementable corresponderá a especificaciones funcionales posteriores.

En las siguientes subsecciones, **REUTILIZAR PEC**, **ADAPTAR / EXTENDER PEC**, **NUEVO** y **PENDIENTE DE DEFINICIÓN** califican el tratamiento arquitectónico. Reutilizar una infraestructura no significa reutilizar la lógica de negocio de Quejas.

### 1. Registro inicial

- **Objetivo funcional:** iniciar un registro de pérdida o hallazgo y proporcionar un identificador de seguimiento del módulo.
- **Datos principales:** ID Encuentra generado por el sistema; tipo de registro; fecha; canal; actor que registra; datos básicos de la persona; descripción y clasificación del artículo; local y ubicación/fecha relacionadas; archivos o fotografías cuando apliquen.
- **Reglas relevantes:** conservar la distinción entre declaración de pérdida y objeto físico encontrado; aplicar permisos del rol; proteger datos personales y números de tarjeta.
- **Dependencias:** registros de pérdida/encontrado, usuarios y locales, catálogos funcionales, archivos y auditoría.
- **Tratamiento arquitectónico:** **ADAPTAR / EXTENDER PEC** para autenticación, contexto de usuario/local y carga de archivos; **NUEVO** para flujo, identificador ID Encuentra y reglas de registro; **PENDIENTE DE DEFINICIÓN** para formato del identificador y datos obligatorios por tipo.
- **Capacidades PEC candidatas:** Keycloak, CASL/PoliciesGuard, `sspectuser`, `sspectenterprise`, `sspectestablishment`, `UserEstablishment`, catálogos, GCS/`sspectfile`.

### 2. Registro de artículo perdido

- **Objetivo funcional:** registrar la declaración de una persona que informa que perdió u olvidó un artículo, aunque todavía no haya un objeto encontrado.
- **Datos principales:** ID Encuentra; tipo/número de identificación; nombres; medios de contacto; tipo y descripción del artículo; tema/subtema; lugar y fecha aproximados; local/canal de registro; fotografías o archivos cuando correspondan.
- **Reglas relevantes:** no crear por implicación un artículo físico en custodia; preservar la declaración aunque no haya coincidencia; una coincidencia requiere validación antes de habilitar la entrega.
- **Dependencias:** registro inicial, catálogos, búsqueda nacional, coincidencias, validación, contacto y privacidad.
- **Tratamiento arquitectónico:** **NUEVO** para la capacidad de declaración y su ciclo funcional; **REUTILIZAR PEC** para identidad/locales y mecanismos genéricos; **ADAPTAR / EXTENDER PEC** para asociación de archivos; **PENDIENTE DE DEFINICIÓN** para campos obligatorios, edición/cierre y tipo de reportante externo.
- **Capacidades PEC candidatas:** `sspectuser`, `sspectenterprise`, `sspectestablishment`, `UserEstablishment`, catálogos, GCS/`sspectfile`, logging técnico como complemento y no como historial de dominio.

### 3. Registro de artículo encontrado

- **Objetivo funcional:** registrar un objeto físico localizado y bajo responsabilidad de un local.
- **Datos principales:** ID Encuentra; descripción, tema/subtema, fecha y lugar del hallazgo; categoría; local que lo posee; cantidad y responsable cuando aplique; fotografías/archivos; estado inicial `Registrado`.
- **Reglas relevantes:** el hallazgo es independiente de cualquier reporte de pérdida; solo el local que tiene posesión física puede modificar la custodia; registrar las transiciones relevantes.
- **Dependencias:** registro inicial, local, catálogos, custodia, búsqueda nacional, vencimientos, contacto, entrega y auditoría.
- **Tratamiento arquitectónico:** **NUEVO** para el registro físico y su ciclo; **REUTILIZAR PEC** para referencias de usuario/empresa/establecimiento; **ADAPTAR / EXTENDER PEC** para archivos, notificaciones y scheduler; **PENDIENTE DE DEFINICIÓN** para clasificación y ubicación detallada.
- **Capacidades PEC candidatas:** `sspectenterprise`, `sspectestablishment`, `sspectuser`, `UserEstablishment`, catálogos, GCS/`sspectfile`, notificaciones y scheduler NestJS.

### 4. Registro especial de documentos y tarjetas

- **Objetivo funcional:** capturar los datos diferenciados requeridos para documentos y tarjetas encontrados.
- **Datos principales:** documentos: cédula, licencia, pasaporte y carné; tarjetas: bancaria, afiliación y regalo; tipo, datos identificativos disponibles, titular, datos adicionales y archivos/fotografías cuando apliquen.
- **Reglas relevantes:** mostrar únicamente primeros 6 y últimos 4 dígitos de tarjetas; nunca mostrar la numeración completa; cuando existan datos de la persona propietaria, activar la gestión de contacto; el autocompletado depende de una integración disponible.
- **Dependencias:** artículo encontrado, tipos administrables, protección de datos, archivos, contacto y posibles fuentes externas de consulta.
- **Tratamiento arquitectónico:** **NUEVO** para captura y reglas específicas del módulo; **ADAPTAR / EXTENDER PEC** para archivos y contacto; **PENDIENTE DE DEFINICIÓN** para autocompletado, fuente/contrato, tratamiento de números cortos, persistencia de valores completos y clasificación conceptual como categoría/detalle.
- **Capacidades PEC candidatas:** GCS/`sspectfile`, correo/notificaciones; no se confirma un catálogo PEC equivalente de tipos de documento/tarjeta ni una integración de autocompletado.

### 5. Búsqueda nacional

- **Objetivo funcional:** localizar candidatos de artículos registrados en cualquier local PEC.
- **Datos principales:** filtros por texto libre, tema, subtema, fecha, local, estado e identificación; resultados con nombre/datos no sensibles, tema/subtema, fecha, local de custodia y nivel de coincidencia.
- **Reglas relevantes:** alcance nacional; Redes Sociales no tiene permiso de búsqueda; no exponer en resultados datos sensibles ni tratar una coincidencia como prueba de propiedad.
- **Dependencias:** registros perdidos/encontrados, catálogos, establecimientos, matching, permisos y seguridad de datos.
- **Tratamiento arquitectónico:** **NUEVO** para consulta, criterios y algoritmo de búsqueda Lost & Found; **REUTILIZAR PEC** para autorización/contexto y patrones genéricos de consulta/listado si aplican; **PENDIENTE DE DEFINICIÓN** para algoritmo, prioridad, revisión humana y límites de visibilidad.
- **Capacidades PEC candidatas:** CASL/PoliciesGuard, datos de establecimientos y catálogos; reportes PEC no sustituyen la lógica de búsqueda del módulo.

### 6. Validación de propiedad

- **Objetivo funcional:** comprobar que quien reclama un artículo puede acreditar que le pertenece.
- **Datos principales:** preguntas y respuestas sobre color, marca, contenido, características y detalles particulares; evidencias; resultado de coincidencia alta, media o baja.
- **Reglas relevantes:** nunca mostrar previamente al cliente las respuestas registradas; una validación correcta habilita continuar con la entrega; la validación y el matching son dimensiones distintas.
- **Dependencias:** artículo/candidato, preguntas administradas, permisos, protección de respuestas, contacto y entrega.
- **Tratamiento arquitectónico:** **NUEVO** para flujo, criterios y protección específica; **ADAPTAR / EXTENDER PEC** para autorización y almacenamiento protegido de evidencias; **PENDIENTE DE DEFINICIÓN** para intentos, configuración de preguntas, umbral satisfactorio y manejo de resultado inconcluso.
- **Capacidades PEC candidatas:** CASL/PoliciesGuard, catálogos/configuración solo si se confirma su adecuación, GCS/`sspectfile` para evidencias. No se encontró lógica equivalente reutilizable en Quejas.

### 7. Entrega del artículo

- **Objetivo funcional:** registrar la entrega del artículo al cliente que superó la validación.
- **Datos principales:** cliente, artículo, local, responsable, fecha y hora, firma del cliente, firma del colaborador, observaciones y acta.
- **Reglas relevantes:** generar el acta; cargar obligatoriamente el acta firmada; no cerrar la entrega sin ella; conservar trazabilidad de entrega y cambio de custodia.
- **Dependencias:** matching, validación satisfactoria, artículo/local, usuario responsable, firma/acta, archivos y auditoría.
- **Tratamiento arquitectónico:** **NUEVO** para proceso y reglas de entrega; **ADAPTAR / EXTENDER PEC** para almacenar/servir el acta y autorización; **PENDIENTE DE DEFINICIÓN** para firma válida, representación, cancelación y envío físico del acta a Experiencia del Cliente (el destinatario aparece como “xxx”).
- **Capacidades PEC candidatas:** GCS/`sspectfile`, CASL/PoliciesGuard, `sspectuser`, `sspectestablishment`; no se encontró un flujo de entrega Lost & Found existente.

### 8. Gestión de custodia

- **Objetivo funcional:** seguir quién tiene el artículo, dónde permanece y cuándo se transfiere o dispone.
- **Datos principales:** local poseedor, estado funcional, responsable, eventos de custodia, destino de transferencia y evidencia asociada.
- **Reglas relevantes:** solo el local con posesión física puede gestionar custodia; conservar auditoría; la transferencia a GCSS y correspondencia forma parte del proceso funcional.
- **Dependencias:** artículo encontrado, establecimientos, vencimientos, transferencias, disposición, permisos y auditoría.
- **Tratamiento arquitectónico:** **NUEVO** para reglas y ciclo de custodia; **REUTILIZAR PEC** para referencias de locales/usuarios y scheduler; **ADAPTAR / EXTENDER PEC** para archivos/notificaciones; **PENDIENTE DE DEFINICIÓN** para acuses, recepción GCSS y acreditación operativa de posesión.
- **Capacidades PEC candidatas:** `sspectestablishment`, `UserEstablishment`, CASL/PoliciesGuard, GCS/`sspectfile`, notificaciones y scheduler. Los estados de Quejas no se reutilizan.

### 9. Gestión automática de contacto

- **Objetivo funcional:** activar seguimiento del propietario y registrar el resultado de cada contacto.
- **Datos principales:** canal correo o gestión telefónica; destinatario; fecha/intento; responsable; resultado: Contactado, No contesta, Número incorrecto o Se acercará al local.
- **Reglas relevantes:** activar correo y/o llamada cuando existan datos del propietario; limitar la información personal expuesta; guardar el resultado para el seguimiento operativo.
- **Dependencias:** datos de identificación/contacto, coincidencia, validación, artículo/local, eventos y plantillas.
- **Tratamiento arquitectónico:** **ADAPTAR / EXTENDER PEC** para canal in-app/email y procesadores por evento; **NUEVO** para eventos, reglas, registro de intentos y resultados Lost & Found; **PENDIENTE DE DEFINICIÓN** para consentimiento, destinatarios, plantillas, frecuencia, reintentos y manejo de fallos.
- **Capacidades PEC candidatas:** notificaciones in-app, email y procesadores por evento; no se reutiliza la lógica de notificación de Quejas como comportamiento funcional.

### 10. Gestión de vencimientos

- **Objetivo funcional:** ejecutar los plazos, alertas, transferencias y disposiciones diferenciados por tipo de artículo.
- **Datos principales:** clase de artículo/documento, fecha de referencia, vencimiento, local/destino, responsable, alerta y evidencia de transferencia o disposición.
- **Reglas relevantes — CONFIRMADAS:** documentos: 1 mes en el local → pendiente de transferencia → GCSS → permanencia de 3 meses → destrucción; artículos generales: 3 meses → transferencia/correspondencia interna; alimentos/comida rápida: hasta cierre del local → desecho con evidencia; emitir alertas 3 días antes del vencimiento.
- **Dependencias:** artículo, local, parámetros, responsables, notificaciones, Excel/reportes, scheduler, transferencia y auditoría.
- **Tratamiento arquitectónico:** **ADAPTAR / REUTILIZAR PEC** para infraestructura NestJS Schedule y canales; **NUEVO** para jobs, reglas y acciones Lost & Found; **PENDIENTE DE DEFINICIÓN** solo para evento técnico de inicio, zona horaria/calendario, excepciones, responsables, parametrización y manejo de fallos (incluida idempotencia y concurrencia).
- **Capacidades PEC candidatas:** scheduler NestJS, notificaciones/email, reportes/Excel y logging técnico como diagnóstico complementario; los jobs de vencimiento del módulo aún no existen.

### 11. Reportes

- **Objetivo funcional:** consultar seguimiento e indicadores operativos de los artículos y procesos.
- **Datos principales / reportes requeridos:** artículos encontrados; artículos perdidos; pendientes de entrega; próximos a vencer; transferidos; destruidos; documentos; tarjetas. Filtros/agrupación por local, zona, fechas, tema y estado; datos generales incluyen ID Encuentra, fecha de registro, comentario, tema/subtema, local, zona, cantidad, estado, responsable y vencimiento.
- **Reglas relevantes:** respetar permisos y ocultar datos sensibles; exportar a Excel; la lógica/definición de estos reportes es propia de Lost & Found.
- **Dependencias:** registros, estados, catálogo y zona, vencimientos, permisos, privacidad y exportación.
- **Tratamiento arquitectónico:** **REUTILIZAR PEC / ADAPTAR / EXTENDER PEC** para infraestructura genérica de reportes/exportación (`ReportsService`, `ExcelExportService`/ExcelJS y exportación frontend `xlsx`); **NUEVO** para consultas, métricas y reglas funcionales Lost & Found; **PENDIENTE DE DEFINICIÓN** para backend vs. frontend, permisos, volumen y campos exportables.
- **Capacidades PEC candidatas:** `ReportsService`, ExcelJS, `xlsx`, catálogos y locales. **NO REUTILIZAR** lógica de reportes de Quejas.

### 12. Administración

- **Objetivo funcional:** administrar valores y parámetros operativos que controlan la captura, clasificación, contacto, validación y vencimientos.
- **Datos principales:** elementos de configuración indicados en la tabla de administración que sigue.
- **Reglas relevantes:** cada catálogo debe corresponder a la semántica Lost & Found; su existencia en PEC no confirma que sea compartible; restringir cambios de configuración por permisos.
- **Dependencias:** empresa/local/zona, roles, notificaciones, validación, vencimientos y reportes.
- **Tratamiento arquitectónico:** **ADAPTAR / EXTENDER PEC** donde el mecanismo genérico de catálogos/configuración sea adecuado; **NUEVO** para valores y reglas propios; **PENDIENTE DE DEFINICIÓN** para autorización administrativa, propiedad de catálogo y alcance por empresa/local.
- **Capacidades PEC candidatas:** infraestructura de catálogos, usuarios/empresas/locales, notificaciones/email y configuración. No reutilizar catálogos de Quejas por equivalencia nominal solamente.

| Elemento administrable | Situación PEC confirmada | Tratamiento para Lost & Found |
|---|---|---|
| Temas | Existe infraestructura genérica de catálogos; equivalencia funcional no confirmada | **ADAPTAR / EXTENDER PEC** si los conceptos coinciden; de lo contrario **NUEVO**. Reutilización pendiente de confirmar. |
| Subtemas | Igual que temas; no se confirma jerarquía adecuada para el módulo | **ADAPTAR / EXTENDER PEC** o **NUEVO**; relación con tema pendiente. |
| Locales | Existen `sspectestablishment` y relaciones con usuarios/empresa | **REUTILIZAR PEC / ADAPTAR** al alcance del módulo. |
| Zonas | Existe atributo `id_zona`; no se encontró dominio `Zone` completo | **PENDIENTE DE DEFINICIÓN** de autoridad y catálogo; después decidir **ADAPTAR / EXTENDER PEC** o **NUEVO**. |
| Tipos de identificación | No se confirma catálogo funcional adecuado para este módulo | **PENDIENTE DE DEFINICIÓN**; usar infraestructura de catálogos solo tras validar contenido y dueño. |
| Canales | Existen canales técnicos in-app/email; catálogo funcional no confirmado | **ADAPTAR / EXTENDER PEC** para mecanismos; valores configurables del módulo **NUEVOS** o pendientes. |
| Estados | Existen estados específicos de Quejas; no máquina genérica reutilizable | **NUEVO** para semántica Lost & Found; no reutilizar estados de Quejas. |
| Roles | Existe Keycloak y CASL/PoliciesGuard | **ADAPTAR / EXTENDER PEC** para autorización; mapeo de roles del módulo **PENDIENTE DE DEFINICIÓN**. |
| Plantillas de correo | Existen email y procesadores por evento; administración de plantillas reusable no confirmada | **ADAPTAR / EXTENDER PEC** para canal/procesador; gestión de plantillas **PENDIENTE DE DEFINICIÓN** o **NUEVA**. |
| Preguntas de validación | No se confirma capacidad equivalente de dominio | **NUEVO**; permisos y protección de respuestas pendientes de diseño. |
| Tiempo de custodia | Scheduler disponible; parámetro de Lost & Found no confirmado | **NUEVO** como parámetro funcional; ejecución **ADAPTAR / EXTENDER PEC**. |
| Tiempo de alertas | Notificaciones y scheduler disponibles; parámetro específico no confirmado | **NUEVO** como parámetro funcional; canales **ADAPTAR / EXTENDER PEC**. |
| Destinos | No se confirma catálogo PEC para GCSS/correspondencia | **NUEVO** o **PENDIENTE DE DEFINICIÓN** según validación de destinos y contratos. |
| Responsables | Existen usuarios, empresas y establecimientos; asignación funcional no confirmada | **REUTILIZAR PEC / ADAPTAR** para referenciar usuarios/locales; reglas de asignación pendientes. |
| Tipos de documentos | No se confirma catálogo compatible con el módulo | **NUEVO** o **ADAPTAR / EXTENDER PEC** tras revisar catálogo existente. |
| Tipos de tarjetas | No se confirma catálogo compatible con el módulo | **NUEVO** o **ADAPTAR / EXTENDER PEC** tras revisar catálogo existente. |

### 13. Roles y permisos

- **Objetivo funcional:** permitir las acciones definidas por la matriz funcional y aplicar el límite de custodia por posesión física.
- **Datos principales:** roles Admin Local, Redes Sociales, Gestor Local y Gestión de Quejas; permisos de consulta, registro, búsqueda, validación, contacto, entrega, actas, custodia, vencimientos y correspondencia.
- **Reglas relevantes:** la matriz de permisos de la sección 13 está confirmada; Redes Sociales solo puede registrar artículos perdidos/encontrados entre las funciones enumeradas; el local poseedor es el único que modifica custodia.
- **Dependencias:** identidad Keycloak, contexto de usuario/empresa/local, `UserEnterpriseRole`, `UserEstablishment`, CASL/PoliciesGuard y contexto de posesión.
- **Tratamiento arquitectónico:** **REUTILIZAR PEC / ADAPTAR / EXTENDER PEC** para identidad y mecanismo de políticas; permisos del módulo son configuración/reglas **NUEVAS**; mapeo de Gestor Local, ámbito territorial y autorización contextual permanecen **PENDIENTES DE DEFINICIÓN**.
- **Capacidades PEC candidatas:** Keycloak, `AuthGuard`, CASL/PoliciesGuard, `sspectuser`, `sspectenterprise`, `sspectestablishment`, `UserEnterpriseRole` y `UserEstablishment`. No reutilizar reglas de Quejas como permisos del nuevo módulo.

### Componentes transversales recomendados

El documento funcional recomienda responsabilidades arquitectónicas para:

- **Reglas de negocio:** evaluar condiciones funcionales, permisos, plazos y restricciones antes de aceptar transiciones.
- **Automatizaciones:** ejecutar alertas, contacto y acciones programadas de vencimiento con reintentos, control de concurrencia e idempotencia por definir.
- **Búsqueda / identificación:** encontrar candidatos en ámbito nacional y soportar revisión/validación sin revelar datos restringidos.
- **Auditoría / trazabilidad:** conservar quién actuó, cuándo, desde qué local y qué cambió, especialmente en custodia, entrega, transferencia y destrucción.

**PROPUESTA / INFERIDO:** son responsabilidades lógicas, no una decisión de cuatro servicios físicos separados. Podrían distribuirse entre módulos o componentes según límites y contratos que se definan posteriormente.

### Datos personales y seguridad

El proceso contempla como **CONFIRMADOS** tipo/número de identificación, nombres y apellidos, celular/teléfono, email, fotografías, archivos/documentos, datos de tarjetas, firmas y respuestas de validación. Deben mantenerse estas condiciones arquitectónicas:

- **CONFIRMADO:** no mostrar la numeración completa de tarjetas; mostrar únicamente primeros 6 y últimos 4 dígitos.
- **CONFIRMADO:** no mostrar al cliente respuestas de validación previamente registradas.
- **CONFIRMADO:** aplicar la matriz por rol y limitar cambios de custodia al local poseedor.
- **PROPUESTA / INFERIDO:** restringir datos sensibles en logs técnicos y usar datos sintéticos/anonimizados en QA.
- **PENDIENTE DE DEFINICIÓN:** visibilidad por campo/canal, base legal, retención, manejo de datos completos, controles específicos de QA y gestión de incidentes.

### Cobertura visual

**CONFIRMADO:** existe un prototipo de Figma como referencia de UX/UI. La navegación/pantalla de **Búsqueda Nacional** está incompleta en ese prototipo; se registra solo como observación, no como requerimiento ni comportamiento esperado. Los filtros, resultados y permisos funcionales provienen del documento funcional.

## 17. Decisiones técnicas (cuando aplique)

| Decisión / principio | Motivo | Estado / consecuencia |
|---|---|---|
| Separar conceptualmente reporte de pérdida y artículo encontrado | **CONFIRMADO:** un reporte no implica necesariamente un objeto encontrado bajo custodia | Principio arquitectónico confirmado; estructura física y cardinalidades pendientes |
| Analizar dimensiones independientes para matching, validación, contacto y custodia | Son procesos distintos con reglas y responsables potencialmente distintos | **PROPUESTA / INFERIDO:** validar coordinación y estados antes de diseño técnico |
| Mantener ARCH-001 en el nivel de arquitectura y dominio | Entidades/tablas físicas, columnas, claves foráneas y modelos Prisma pertenecen a DATA-001; contratos y comportamiento detallado pertenecen a sus especificaciones | No incluir aquí esquema físico ni contratos implementables; mantener conceptos y responsabilidades funcionales |
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
20. **Reportes y administración:** los campos, reportes, catálogos y parámetros funcionales están enumerados en las secciones 14 y “Cobertura funcional completa del requerimiento”. ¿Qué roles pueden administrarlos, consultarlos y exportarlos, y qué restricciones de datos aplican?
21. **Aprobación y backlog:** ¿Qué work item, responsables de negocio/operación, aprobadores y fecha de aprobación deben asociarse a esta especificación?

## 19. Estado de aprobación

- **Estado:** BORRADOR
- **Aprobadores:** Pendiente de identificar
- **Fecha y observaciones:** 2026-09-30 — documento de arquitectura y dominio para revisión; propuestas e inferencias no aprobadas y preguntas abiertas.

## 20. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.1.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Primera especificación transversal de arquitectura y dominio; distingue confirmado, propuesto y pendiente. |
| 0.2.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Reclasifica vencimientos, alertas, permisos, datos personales, reportes/administración y observación de Figma según el documento funcional oficial; mantiene pendientes los aspectos técnicos y operativos no definidos. |
| 0.3.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Incorpora hallazgos del descubrimiento técnico de pec-api/pec-ui; clasifica capacidades PEC existentes, límites de reutilización, ausencia de implementación Lost & Found y preservación del estado de trabajo. |
| 0.4.0 | 2026-09-30 | Mbrion / equipo PEC — por confirmar | Completa la cobertura de los trece módulos funcionales, estados requeridos, administración, reportes, reglas y componentes transversales; añade matriz final de trazabilidad y refuerza el límite entre ARCH-001 y DATA-001. |

## Matriz de cobertura del requerimiento

| Requerimiento | Sección ARCH-001 | Tratamiento | Estado |
|---|---|---|---|
| Registro inicial e ID Encuentra | Cobertura funcional completa §1; Estados §12 | Adaptar/extender PEC para contexto y archivos; capacidad de registro nueva | Cubierto; formato del ID y datos obligatorios pendientes |
| Registro de artículo perdido | Cobertura funcional completa §2; Modelo conceptual §11 | Nuevo | Cubierto; reglas de edición/cierre y campos obligatorios pendientes |
| Registro de artículo encontrado | Cobertura funcional completa §3; Estados §12 | Nuevo; reutilizar contexto de locales/usuarios PEC | Cubierto; duplicados e identificación física pendientes |
| Registro de documentos y tarjetas | Cobertura funcional completa §4; Reglas RB-01; Datos personales §7 | Nuevo; adaptar archivos/contacto PEC | Cubierto; autocompletado e integración externa pendientes |
| Cédula, licencia, pasaporte y carné | Cobertura funcional completa §4 | Catálogos nuevos o adaptados tras validar equivalencia | Cubierto |
| Tarjeta bancaria, de afiliación y regalo | Cobertura funcional completa §4 | Catálogos nuevos o adaptados tras validar equivalencia | Cubierto |
| Datos adicionales de documentos/tarjetas y posible autocompletado | Cobertura funcional completa §4 | Nuevo; integración pendiente de confirmar | Cubierto; fuente y contrato pendientes |
| Enmascarar tarjeta: primeros 6 y últimos 4; nunca mostrar número completo | Reglas RB-01; Cobertura funcional completa §4; Seguridad | Regla funcional nueva, aplicable a todos los canales | Cubierto; tratamiento de números cortos y valor completo pendiente |
| Búsqueda nacional entre locales | Cobertura funcional completa §5; Pantallas §16 | Búsqueda/identificación nueva; reutilizar autorización y referencias PEC | Cubierto; algoritmo, prioridad y límites de datos pendientes |
| Validación con color, marca, contenido, características y detalles particulares | Cobertura funcional completa §6 | Nuevo; configurar preguntas propias | Cubierto; preguntas finales e intentos pendientes |
| Resultados de coincidencia alta/media/baja | Cobertura funcional completa §6; Estados §12 | Nuevo | Cubierto; umbrales pendientes |
| No mostrar respuestas previas; validación correcta permite entrega | Reglas RB-02; Cobertura funcional completa §6; Seguridad | Regla funcional nueva; aplicar autorización/protección PEC | Cubierto; controles técnicos específicos pendientes |
| Entrega con cliente, artículo, local, responsable, fecha, hora y observaciones | Cobertura funcional completa §7 | Nuevo; reutilizar contexto de usuario/local | Cubierto |
| Firmas de cliente y colaborador; generar acta y cargar acta firmada | Reglas RB-04; Cobertura funcional completa §7 | Flujo nuevo; adaptar infraestructura GCS/`sspectfile` | Cubierto; tipos de firma y flujo físico pendientes |
| No cerrar entrega sin acta firmada y conservar trazabilidad | Reglas RB-04/RB-05; Cobertura funcional completa §7; Estados §12 | Regla nueva; requiere auditoría de negocio durable | Cubierto; mecanismo de auditoría pendiente |
| Gestión de custodia y límite por posesión física | Reglas RB-03; Cobertura funcional completa §8; Roles §13 | Nuevo; reutilizar identidad/locales y extender autorización | Cubierto; acreditación de posesión y acuses pendientes |
| Estados de custodia local, transferencia, GCSS y destrucción | Estados §12; Cobertura funcional completa §8 | Nuevo; no reutilizar estados de Quejas | Cubierto; secuencia/acuse y destino operativo pendientes |
| Contacto automático por correo y gestión telefónica cuando existen datos | Cobertura funcional completa §9; Integraciones §14 | Adaptar/extender canales PEC; eventos/reglas nuevos | Cubierto; consentimiento, plantillas, frecuencia y reintentos pendientes |
| Resultados de contacto: Contactado, No contesta, Número incorrecto, Se acercará al local | Estados §12; Cobertura funcional completa §9 | Nuevo | Cubierto |
| Vencimiento de documentos: 1 mes local, transferencia, GCSS 3 meses y destrucción | Reglas RB-06; Estados §12; Cobertura funcional completa §10 | Jobs/reglas nuevos; scheduler adaptable | Cubierto; parámetros técnicos pendientes según RB-10 |
| Vencimiento de artículos generales: 3 meses y correspondencia interna | Reglas RB-07; Estados §12; Cobertura funcional completa §10 | Jobs/reglas nuevos; scheduler adaptable | Cubierto; responsables y acuses pendientes |
| Alimentos/comida rápida: hasta cierre, desecho con evidencia | Reglas RB-08; Cobertura funcional completa §10 | Regla/job nuevo; archivos PEC adaptables | Cubierto; evidencia operativa pendiente |
| Alertas 3 días antes del vencimiento | Reglas RB-09; Cobertura funcional completa §10 | Scheduler y canales PEC adaptables; evento nuevo | Cubierto; inicio, zona horaria, fallos y parámetros pendientes |
| Reportes de encontrados, perdidos, pendientes de entrega, próximos a vencer, transferidos, destruidos, documentos y tarjetas | Cobertura funcional completa §11; Integraciones §14 | Lógica Lost & Found nueva; exportación PEC reutilizable/adaptable | Cubierto; permisos/detalles técnicos pendientes |
| Filtros por local, zona, fechas, tema y estado | Cobertura funcional completa §11 | Lógica de consulta nueva; adaptar datos/catálogos PEC cuando sean adecuados | Cubierto; fuente de zonas pendiente |
| Exportación a Excel | Cobertura funcional completa §11; Integraciones §14 | Reutilizar/adaptar infraestructura ExcelJS/`xlsx`; lógica funcional nueva | Cubierto; backend/frontend y permisos pendientes |
| Administración de temas y subtemas | Cobertura funcional completa §12 | Catálogos PEC adaptables, equivalencia semántica pendiente | Cubierto; reutilización por confirmar |
| Administración de locales y zonas | Cobertura funcional completa §12; Integraciones §14 | Reutilizar establecimientos; zona pendiente de confirmar | Cubierto; fuente de zona pendiente |
| Administración de tipos de identificación, canales, estados y roles | Cobertura funcional completa §12 | Canales/mecanismos PEC adaptables; tipos y estados nuevos; roles/extensión CASL | Cubierto; mapeos pendientes |
| Administración de plantillas de correo y preguntas de validación | Cobertura funcional completa §12 | Canal email adaptable; administración de plantillas por confirmar; preguntas nuevas | Cubierto; ownership/configuración pendiente |
| Administración de tiempo de custodia, tiempo de alertas, destinos y responsables | Cobertura funcional completa §12 | Parámetros Lost & Found nuevos; reutilizar referencias PEC cuando aplique | Cubierto; responsables, parametrización y destinos pendientes |
| Administración de tipos de documentos y tarjetas | Cobertura funcional completa §12 | Catálogos propios o adaptados tras confirmar equivalencia | Cubierto; fuente/dueño pendiente |
| Roles y matriz de permisos | Roles §13; Cobertura funcional completa §13 | Adaptar Keycloak y CASL/PoliciesGuard; permisos de negocio nuevos | Cubierto; mapeo y alcance contextual pendientes |
| Estados Registrado, En custodia del local, Gestión de contacto, Cliente contactado, Entregado al cliente, Pendiente de transferencia, Transferido al GCSS, En custodia GCSS, Pendiente de destrucción, Destruido y Posible coincidencia | Estados §12; Cobertura funcional completa | Separar dimensiones; no reutilizar estado de Quejas | Cubierto; transiciones técnicas pendientes |
| Dimensiones separadas de custodia, matching, validación, contacto, entrega y vencimiento | Estados §12 | Propuesta de arquitectura | Cubierto como propuesta; coordinación final pendiente |
| Responsabilidades transversales de reglas, automatizaciones, búsqueda/identificación y auditoría/trazabilidad | Cobertura funcional completa — Componentes transversales; Decisiones §17 | Capacidades lógicas nuevas; no implica cuatro servicios físicos | Cubierto como recomendación arquitectónica |
| Datos personales, logs, QA y permisos por rol/local | Datos personales §7; Cobertura funcional completa — Datos personales y seguridad; Roles §13 | Reutilizar autorización PEC y aplicar controles nuevos del módulo | Cobertura funcional presente; base legal, retención y controles por campo pendientes |
| Figma y observación de Búsqueda Nacional incompleta | Pantallas §16; Cobertura funcional completa — Cobertura visual | Prototipo como referencia; defecto no es requisito | Cubierto; pantalla no define comportamiento esperado |
