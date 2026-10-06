# PEC-LF-FOUND-001

## Registro de artículo encontrado

### 1. Encabezado

| Campo | Valor |
|---|---|
| ID | PEC-LF-FOUND-001 |
| Título | Registro de artículo encontrado |
| Versión | 1.2.0 |
| Estado | APPROVED |
| Fecha | 2026-09-30 |
| Arquitectura relacionada | PEC-LF-ARCH-001 |
| Modelo de datos relacionado | PEC-LF-DATA-001 |
| Responsable | Walter Molina |

---

## 2. Objetivo

Permitir registrar un artículo encontrado dentro de PEC, conservando:
- identificación funcional del artículo;
- empresa;
- establecimiento custodio;
- zona interna;
- clasificación;
- datos de la persona que lo encontró;
- fecha y lugar del hallazgo, y hora cuando sea conocida;
- archivos/fotografías;
- usuario que registra;
- política de custodia aplicada;
- auditoría de creación.

---

## 3. Alcance funcional

Incluye:
- creación de FoundItem;
- generación automática de itemCode;
- selección de clasificación mediante una jerarquía configurable de hasta tres niveles: categoría, tema y subtema;
- selección de establecimiento;
- selección de zona interna habilitada para el establecimiento;
- registro del lugar donde fue encontrado;
- fecha de hallazgo obligatoria y hora opcional;
- datos del finder;
- carga de fotografías/archivos;
- estado inicial REGISTERED;
- asignación de custodio actual;
- aplicación inicial de CustodyPolicy;
- generación de expirationDate cuando aplique;
- auditoría de creación.

No incluye:
- matching;
- búsqueda nacional;
- RecoveryClaim;
- validación de propiedad;
- contacto;
- entrega;
- transferencia;
- disposición final.

---

## 4. Actor y permisos

Roles habilitados para registrar artículo encontrado:
- Admin Local
- Redes Sociales
- Gestor Local
- Gestión de Quejas

La autorización debe validarse en backend mediante Keycloak + CASL/PoliciesGuard.

El usuario que registra se obtiene de la sesión PEC.

El alcance del selector de empresa y establecimiento depende de las asociaciones del usuario:
- Admin, Superusuario, Redes Sociales y Gestión de Quejas pueden tener una o varias empresas asociadas. Solo pueden seleccionar una empresa asociada a su usuario; dentro de ella consultan establecimientos activos de esa empresa.
- Admin Local y Gestor Local usan la empresa asociada a su usuario y consultan únicamente establecimientos activos asignados a ese usuario. No pueden ampliar ese alcance enviando otro `establishmentId` o una empresa distinta.
- El cliente envía la empresa seleccionada tanto para establecimientos como para catálogos y zonas Lost & Found. El backend valida que la asociación usuario-empresa exista y que el establecimiento pertenezca a esa empresa; para roles locales también valida la asignación del establecimiento.
- El código funcional canónico `LOST_FOUND_CLASSIFICATION` se conserva en datos; la API acepta el código sin distinción de mayúsculas/minúsculas.

---

## 5. Flujo principal

1. Usuario ingresa a “Registrar artículo encontrado”.
2. Sistema obtiene una empresa asociada al usuario y consulta establecimientos según el alcance definido para el rol; el backend valida la asociación usuario-empresa y, para roles locales, la asignación del establecimiento.
3. Usuario selecciona establecimiento.
4. Sistema carga zonas internas habilitadas.
5. Usuario ingresa datos del artículo.
6. Usuario selecciona la clasificación desde las opciones jerárquicas de `LOST_FOUND_CLASSIFICATION`. El formulario muestra Categoría, Tema y Subtema según los niveles configurados para la rama, sin exigir niveles inexistentes ni crear opciones artificiales. El sistema conserva `classificationOptionId` del nodo seleccionable más específico disponible.
7. Usuario registra datos del finder.
8. Usuario adjunta fotografías/archivos.
9. Sistema valida campos obligatorios.
10. Sistema valida `classificationOptionId` y resuelve la CustodyPolicy activa aplicable según la prioridad definida.
11. Si no existe una política aplicable, sistema bloquea el registro.
12. Sistema calcula `expirationDate` cuando aplique.
13. Sistema genera `itemCode`.
14. Sistema crea FoundItem con `custodyPolicyId`, `expirationDate` y `currentEstablishmentId`, conservando la referencia a la versión inmutable aplicada.
15. Sistema registra archivos.
16. Sistema genera evento LostFoundAudit.
17. Sistema devuelve confirmación del registro.

---

## 6. Datos del formulario

### 6.1 Artículo

- nombre
- descripción
- Categoría
- Tema, cuando la categoría tenga ese nivel configurado
- Subtema, cuando el tema tenga ese nivel configurado
- lugar encontrado
- fecha encontrada: obligatoria
- hora encontrada: opcional
- establecimiento
- zona
- comentario

La clasificación final corresponde al nodo seleccionable más específico de la rama. Si una categoría no tiene hijos seleccionables, puede ser la clasificación final; si tiene tema sin subtema, puede serlo el tema. Un padre no puede seleccionarse como clasificación final cuando exista un hijo obligatorio/configurado que deba seleccionarse.

### 6.2 Persona que encontró

finderType:
- COLLABORATOR
- CUSTOMER
- OTHER

Si COLLABORATOR:
- finderEmployeeCode
- finderName
- finderLastName
- finderPosition

El usuario busca y selecciona al colaborador mediante la capacidad de consulta existente en PEC. El sistema autocompleta estos datos desde esa fuente y, al guardar FoundItem, conserva código, nombre, apellido y cargo como snapshot histórico. No se asume que el usuario pueda editar libremente los datos autocompletados. Si la consulta falla, no se inventan datos; el comportamiento de fallback manual queda pendiente de definición.

Si CUSTOMER:
- finderName obligatorio
- finderLastName obligatorio

Si OTHER:
- finderName opcional
- finderLastName opcional

`finderNote` es opcional para conservar observaciones sobre la persona que encontró el artículo.

Los datos del colaborador se guardan como snapshot histórico.

### 6.3 Archivos

Permitir:
- fotografías
- documentos/evidencias

Metadata en sspectlffile.
Storage mediante StorageService/GCS.

---

## 7. Reglas de negocio

RB-FND-01
FoundItem representa un objeto físico encontrado.

RB-FND-02
`itemCode` es el identificador funcional visible de FoundItem. Se genera automáticamente en backend, no lo ingresa ni edita el usuario y es independiente de la PK física.

Formato lógico inicial: `LF-F-AAAA-XXXXXX`, donde `LF-F` identifica Lost & Found / FoundItem, `AAAA` representa el año y `XXXXXX` un correlativo. El código debe ser único por combinación de enterprise y año, y no debe reutilizarse una vez asignado.

RB-FND-03
currentEstablishmentId representa al establecimiento que posee físicamente el artículo.

RB-FND-04
FoundItem debe registrar obligatoriamente una zona interna. La zona seleccionada debe estar activa y habilitada para el `currentEstablishmentId` mediante `LostFoundEstablishmentZone`. No se permite registrar FoundItem con `zoneId` nulo.

RB-FND-05
El estado inicial del artículo será REGISTERED.

RB-FND-06
El finder no se modela como entidad independiente.

RB-FND-07
Si finderType = COLLABORATOR, se debe conservar snapshot de nombre, apellido y cargo.

RB-FND-08
Los archivos del módulo no se almacenan en sspectfile.

RB-FND-09
El registro debe generar auditoría funcional.

RB-FND-10
La política de custodia vigente debe aplicarse al momento del registro.

RB-FND-11
Los cambios futuros de CustodyPolicy no deben modificar automáticamente la fecha de vencimiento ya aplicada. Cada registro de CustodyPolicy representa una versión inmutable; una nueva configuración genera un nuevo registro y no modifica el registro histórico referenciado por FoundItem.

RB-FND-12
FoundItem utiliza únicamente `classificationOptionId` para la clasificación. Los niveles presentes de categoría, tema y subtema se obtienen recorriendo `parentId` en `LostFoundCatalogOption`; no se guardan `categoryId`, `topicId` ni `subtopicId` en FoundItem.

RB-FND-13
La opción de clasificación debe estar activa, pertenecer al catálogo `LOST_FOUND_CLASSIFICATION` y al enterprise correspondiente. La jerarquía soportada tiene como máximo CATEGORY → THEME → SUBTHEME; las ramas pueden terminar en CATEGORY, THEME o SUBTHEME. FoundItem almacena `classificationOptionId` del nodo seleccionable más específico disponible. Si una opción posee un nivel hijo obligatorio/configurado que deba seleccionarse, no puede utilizarse su padre como clasificación final.

RB-FND-14
FoundItem debe conservar la fecha del hallazgo y, cuando sea conocida, también la hora. `foundAt` representa la fecha y hora informadas del hallazgo; `foundTimeKnown` indica si el usuario proporcionó realmente la hora. Si solo se conoce la fecha, `foundTimeKnown = false`; el sistema no debe presentar una hora inferida o generada por defecto como si la hubiera informado el usuario.

RB-FND-15
Si un establecimiento no tiene ninguna zona Lost & Found activa y configurada, el sistema debe bloquear el registro e informar que requiere configuración de zonas antes de registrar el artículo. Una zona genérica como `OTRA` o `NO_ESPECIFICADA` puede existir como opción administrable para ubicaciones desconocidas, pero no sustituye la obligatoriedad de un `zoneId` válido.

RB-FND-17
Los archivos asociados a FoundItem deben cumplir las mismas políticas técnicas de carga y validación vigentes en PEC. Lost & Found no define límites ni formatos diferentes salvo que una Spec futura lo requiera explícitamente. La metadata funcional se almacena en `sspectlffile` y la asociación con FoundItem mediante `FoundItemFile`.

RB-FND-18
Si `finderType = COLLABORATOR`, los datos del finder se obtienen mediante la capacidad existente de consulta de colaboradores de PEC. Al confirmar el registro, FoundItem persiste como snapshot `finderEmployeeCode`, `finderName`, `finderLastName` y `finderPosition`, sin depender únicamente de una referencia dinámica al colaborador. El snapshot no se recalcula automáticamente si los datos del colaborador cambian posteriormente. No se crea una fuente nueva de datos cuando la capacidad PEC existente resuelve la consulta.

RB-FND-19
Al registrar FoundItem, el sistema debe resolver una CustodyPolicy activa en este orden de prioridad: (1) política específica del `classificationOptionId` validado; (2) política aplicable por tipo funcional del artículo; (3) política `GENERAL`. Se utiliza la primera política aplicable según ese orden. Si no existe ninguna política activa aplicable, se bloquea el registro.

RB-FND-20
Los plazos de custodia no deben codificarse directamente en el servicio de creación de FoundItem. El artículo conserva `custodyPolicyId` de la versión inmutable aplicada y la `expirationDate` resultante cuando corresponda. Una nueva versión de CustodyPolicy no cambia el `custodyPolicyId` histórico ni recalcula automáticamente `expirationDate`.

RB-FND-21
La generación de `itemCode` debe ser segura ante concurrencia. Dos registros simultáneos del mismo enterprise y año no pueden recibir el mismo código. La estrategia física que garantiza la unicidad se define en el Plan técnico.

RB-FND-22
El selector de empresa y establecimiento Lost & Found respeta las asociaciones del usuario: Admin, Superusuario, Redes Sociales y Gestión de Quejas pueden seleccionar una empresa asociada y cualquier establecimiento activo de esa empresa; Admin Local y Gestor Local solo pueden consultar establecimientos activos asignados a su usuario dentro de su empresa asociada. El cliente envía el contexto de empresa para cargar establecimientos, catálogos y zonas; el backend valida la asociación usuario-empresa y la pertenencia/asignación del establecimiento. Los parámetros del cliente no pueden ampliar el alcance. Los códigos de catálogo se comparan sin distinguir mayúsculas/minúsculas y se almacenan con su código canónico aprobado.

Como referencia funcional inicial, las políticas contemplan:
- `DOCUMENT`: 30 días en el local; luego transferencia a GCSS; 90 días adicionales en GCSS; destrucción posterior según el flujo aprobado.
- `FOOD`: custodia hasta el cierre del establecimiento/local; disposición o desecho con evidencia.
- `GENERAL`: 90 días; luego correspondencia o transferencia configurada.

Una CustodyPolicy específica puede asociarse opcionalmente a `classificationOptionId` y prevalece sobre las políticas por tipo funcional y `GENERAL`. Por ejemplo, una política activa para la clasificación `LAPTOP` prevalece sobre `GENERAL`.

---

## 8. Validaciones

- nombre obligatorio
- `classificationOptionId` obligatorio, correspondiente al nodo seleccionable más específico de la rama configurada; no exigir tres niveles si la rama tiene menor profundidad
- establecimiento obligatorio
- zona obligatoria
- `zoneId` no puede ser nulo
- lugar encontrado obligatorio
- fecha encontrada obligatoria
- hora encontrada opcional
- finderType obligatorio
- si finderType = COLLABORATOR:
  - `finderEmployeeCode` obligatorio
  - `finderName` obligatorio
  - `finderLastName` obligatorio
  - `finderPosition` obligatorio
  - los datos guardados deben corresponder al colaborador seleccionado/consultado; no permitir un snapshot incompleto ni inventar datos si falla la consulta
  - el snapshot histórico no se recalcula en consultas futuras
- si finderType = CUSTOMER: `finderName` y `finderLastName` obligatorios
- si finderType = OTHER: `finderName` y `finderLastName` opcionales
- `finderNote` opcional para todos los tipos de finder
- la zona debe estar activa y pertenecer al dominio Lost & Found
- debe existir una relación activa `LostFoundEstablishmentZone` entre `establishmentId` y `zoneId`
- si el establecimiento no tiene zonas Lost & Found activas y configuradas, bloquear el registro e informar que requiere configuración previa
- la opción de clasificación debe estar activa y pertenecer al catálogo `LOST_FOUND_CLASSIFICATION` y al enterprise del registro
- la cadena de `parentId` debe respetar los niveles configurados de CATEGORY → THEME → SUBTHEME; rechazar opciones con padre de otro catálogo o jerarquía inválida
- no permitir como clasificación final un padre con un hijo obligatorio/configurado que deba seleccionarse
- la CustodyPolicy seleccionada debe estar activa y pertenecer al enterprise correspondiente cuando aplique
- una política específica debe corresponder al `classificationOptionId` validado
- no permitir registrar FoundItem sin CustodyPolicy activa aplicable ni utilizar una política inactiva
- los cambios posteriores de CustodyPolicy no recalculan automáticamente `custodyPolicyId` ni `expirationDate` históricos
- `itemCode` no se recibe como dato editable del cliente; se genera en backend
- `itemCode` debe ser único por enterprise y año, sin duplicados ni reutilización de códigos previamente asignados
- la creación debe fallar de forma controlada si no puede garantizarse la unicidad de `itemCode`, incluso bajo concurrencia
- usuario debe estar autorizado
- enterprise_id debe coincidir con el establecimiento
- fecha/hora de hallazgo no puede ser futura; si se informa una hora, se valida junto con la fecha
- si no se informa hora, `foundTimeKnown = false` y ninguna hora inferida se presenta como dato informado
- cada archivo debe cumplir las validaciones técnicas de carga vigentes en PEC; reutilizar las validaciones centralizadas cuando corresponda, sin duplicar sus reglas en Lost & Found
- un archivo rechazado por la infraestructura PEC no puede asociarse a FoundItem; la metadata Lost & Found solo se registra cuando la carga haya sido aceptada

---

## 9. Modelo lógico involucrado

FoundItem

Relaciones:
- N:1 sspectenterprise
- N:1 sspectestablishment
- N:1 sspectuser
- N:1 LostFoundZone mediante `zoneId` obligatorio
- N:1 LostFoundCatalogOption mediante `classificationOptionId` del catálogo `LOST_FOUND_CLASSIFICATION`
- 1:N FoundItemFile
- N:1 CustodyPolicy mediante `custodyPolicyId`
- auditoría mediante LostFoundAudit

Campos conceptuales:
- id
- itemCode
- enterprise_id
- currentEstablishmentId
- name
- description
- classificationOptionId
- foundPlace
- foundAt
- foundTimeKnown
- zoneId
- finderType
- finderEmployeeCode
- finderName
- finderLastName
- finderPosition
- finderNote
- registeredByUserId
- status
- expirationDate
- custodyPolicyId
- comment
- createdAt
- updatedAt

`itemCode` es el identificador funcional generado en backend, independiente de `id` como PK física y único por enterprise y año. El mecanismo físico del correlativo y la restricción de unicidad se definirán en el Plan técnico.

CustodyPolicy se versiona mediante registros inmutables. Conceptualmente puede contemplar `policyCode`, `classificationOptionId` opcional, tipo funcional aplicable, parámetros de custodia, `active` y `version`. Las columnas físicas se definirán en el Plan técnico.

---

## 10. Estado inicial

REGISTERED

El artículo queda asociado al establecimiento custodio actual.

---

## 11. Auditoría mínima

Evento:
FOUND_ITEM_CREATED

Registrar:
- entityType = FoundItem
- entityId
- userId
- username
- establishmentId
- timestamp
- newValue
- observations

---

## 12. Archivos

Tipos funcionales conceptuales:
- PHOTO
- EVIDENCE

`PHOTO` y `EVIDENCE` clasifican el propósito funcional del archivo dentro de Lost & Found. No definen extensiones ni tipos MIME diferentes. Las extensiones, tipos MIME, tamaños máximos, cantidades cuando existan, validaciones de seguridad, manejo de errores y mecanismo de carga son los vigentes en PEC; esta Spec no duplica valores técnicos centralizados.

Cada archivo debe:
- pertenecer al dominio Lost & Found;
- cumplir las reglas técnicas de carga y validación vigentes en PEC;
- almacenarse mediante `StorageService`/GCS existente;
- conservar metadata propia en `sspectlffile` y asociarse al FoundItem mediante `FoundItemFile` una vez aceptada la carga.

No se reutiliza `sspectfile` como metadata del dominio Lost & Found.

---

## 13. Criterios de aceptación

### Positivos

CA-FND-01
Dado un usuario autorizado, cuando registra todos los datos válidos, entonces se crea FoundItem con `itemCode` generado automáticamente en backend y estado REGISTERED.

CA-FND-02
Dado un establecimiento, cuando se carga el formulario, entonces solo se muestran zonas Lost & Found activas y habilitadas para ese establecimiento.

CA-FND-03
Dado `finderType = COLLABORATOR`, cuando el usuario selecciona un colaborador válido, entonces el sistema autocompleta código, nombre, apellido y cargo y, al guardar FoundItem, conserva esos valores como snapshot histórico.

CA-FND-04
Dado un archivo que cumple las reglas vigentes de PEC, cuando se adjunta a FoundItem, entonces se almacena mediante StorageService/GCS y se registra su metadata en `sspectlffile` y su asociación `FoundItemFile`.

CA-FND-05
Dado un registro exitoso, entonces se crea auditoría FOUND_ITEM_CREATED.

CA-FND-06
Dado que existe una CustodyPolicy activa aplicable, cuando se registra FoundItem, entonces el sistema selecciona la primera política según la prioridad clasificación → tipo funcional → `GENERAL` y guarda `custodyPolicyId` de la versión aplicada y la `expirationDate` correspondiente cuando aplique.

CA-FND-07
Dada una clasificación válida, cuando se registra FoundItem, entonces se guarda `classificationOptionId` del nodo seleccionable más específico y los niveles superiores presentes se derivan mediante `parentId`.

CA-FND-08
Dada una rama de clasificación con menos de tres niveles, cuando el usuario selecciona el nodo final válido, entonces el sistema permite registrar el artículo sin exigir niveles inexistentes.

CA-FND-09
Dado que el usuario conoce fecha y hora del hallazgo, cuando registra FoundItem, entonces se conserva `foundAt` con fecha/hora y `foundTimeKnown = true`.

CA-FND-10
Dado que el usuario conoce únicamente la fecha del hallazgo, cuando registra FoundItem, entonces el sistema permite guardar el artículo, conserva la fecha en `foundAt` y `foundTimeKnown = false`, sin mostrar una hora no informada como real.

CA-FND-11
Dado un establecimiento con zonas Lost & Found activas, cuando el usuario registra un artículo, entonces debe seleccionar una zona habilitada y el sistema guarda `zoneId`.

CA-FND-15
Dada una clasificación con CustodyPolicy específica activa, cuando se registra FoundItem, entonces el sistema utiliza esa política antes que las políticas por tipo funcional o `GENERAL`.

CA-FND-16
Dada una clasificación sin política específica activa, cuando existe una CustodyPolicy activa aplicable por tipo funcional, entonces el sistema utiliza esa política.

CA-FND-17
Dado que no existe una política específica ni por tipo funcional aplicable, cuando existe una CustodyPolicy `GENERAL` activa, entonces el sistema utiliza la política `GENERAL`.

CA-FND-18
Dado un registro válido de FoundItem, cuando el backend genera `itemCode`, entonces el código sigue el formato lógico `LF-F-AAAA-XXXXXX` y es único dentro del enterprise y año correspondientes.

### Negativos

CA-FND-N01
No permitir guardar sin establecimiento.

CA-FND-N02
No permitir zona que no pertenezca al establecimiento.

CA-FND-N03
No permitir registrar si el usuario no tiene permiso.

CA-FND-N04
No permitir enterprise_id inconsistente con el establecimiento.

CA-FND-N05
No crear FoundItem si falla la transacción principal.

CA-FND-N06
No permitir guardar con una opción de clasificación inactiva, de otro enterprise/catálogo o con jerarquía configurada inválida.

CA-FND-N07
No permitir seleccionar como clasificación final un nodo padre cuando exista un hijo obligatorio/configurado que deba seleccionarse.

CA-FND-N08
No permitir registrar una fecha/hora de hallazgo futura.

CA-FND-N09
No permitir guardar FoundItem con `zoneId` vacío o nulo.

CA-FND-N10
No permitir guardar FoundItem con una zona inactiva o no habilitada para el establecimiento seleccionado.

CA-FND-N11
Dado un establecimiento sin zonas Lost & Found activas y configuradas, cuando el usuario intenta registrar un artículo, entonces el sistema bloquea el registro y solicita configurar zonas.

CA-FND-N14
Dado un archivo que no cumple las reglas técnicas vigentes de PEC, cuando el usuario intenta adjuntarlo, entonces el sistema rechaza la carga y no crea una asociación válida con FoundItem.

CA-FND-N15
Dado `finderType = COLLABORATOR`, cuando no se puede obtener o completar la información obligatoria del colaborador, entonces el sistema no permite guardar FoundItem como COLLABORATOR.

CA-FND-N16
Dado que no existe ninguna CustodyPolicy activa aplicable, cuando el usuario intenta registrar FoundItem, entonces el sistema bloquea el registro.

CA-FND-N17
Dado que el cliente intenta enviar o modificar `itemCode`, cuando se procesa el registro, entonces el backend ignora o rechaza ese valor según el contrato técnico y genera o controla el identificador funcional.

CA-FND-N18
Dadas dos creaciones concurrentes dentro del mismo enterprise y año, cuando ambas generan `itemCode`, entonces no pueden producir el mismo código.

---

## 14. Transacción

La operación de creación debería mantener consistencia entre:
- FoundItem
- archivos/asociaciones
- aplicación de política
- auditoría

Si una parte crítica falla, no debe quedar un registro parcialmente válido.

`StorageService`/GCS no participa en la misma transacción ACID de PostgreSQL. La implementación debe evitar asociaciones a archivos inexistentes, metadata definitiva de cargas fallidas y un FoundItem inconsistente por un fallo parcial de archivo. La estrategia técnica concreta de staging o compensación se define en el Plan técnico.

---

## 15. Pendientes de esta Spec

- verificar en el Plan técnico el contrato y los endpoints reales del servicio PEC de colaboradores, y definir el comportamiento de fallback manual si la consulta no está disponible;
- definir en el Plan técnico el algoritmo exacto de cálculo de `expirationDate` para políticas basadas en días, cierre de local, GCSS y otras condiciones operativas;
- definir en el Plan técnico el mecanismo físico de correlativo o sequence y la restricción de unicidad de `itemCode` por enterprise y año;
- definir DTOs, endpoints, services, transacciones y detalles de implementación en el Plan técnico.

Estos pendientes técnicos se trasladan al Plan técnico y no bloquean la aprobación funcional de FOUND-001.

---

## 16. Estado de aprobación

Estado: APPROVED
Versión: 1.2.0
Aprobador: Walter Molina
Fecha de aprobación: 2026-10-06

---

## 17. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.2.0 | 2026-10-03 | Walter Molina | Se actualiza el modelo conceptual a `classificationOptionId` y clasificación jerárquica Lost & Found; se mantiene en borrador. |
| 0.3.0 | 2026-10-04 | Walter Molina | Se flexibiliza la clasificación jerárquica para permitir ramas de uno, dos o tres niveles, conservando `classificationOptionId` como nodo final. |
| 0.4.0 | 2026-10-04 | Walter Molina | Se documenta el estado real del frontend existente y se redefine Sprint 1 como integración backend y adaptación de UI existente, no como construcción visual desde cero. |
| 0.5.0 | 2026-10-04 | Walter Molina | Se define CustodyPolicy versionada mediante registros inmutables y FoundItem referencia custodyPolicyId + expirationDate, sin entidad CustodyPolicyVersion separada. |
| 0.6.0 | 2026-10-04 | Walter Molina | Se define fecha de hallazgo obligatoria y hora opcional mediante `foundAt` + `foundTimeKnown`, evitando inferir horas no informadas. |
| 0.7.0 | 2026-10-05 | Walter Molina | Se define `zoneId` obligatorio para todo FoundItem y se bloquea el registro cuando el establecimiento no posee zonas Lost & Found configuradas. |
| 0.8.0 | 2026-10-05 | Walter Molina | Se definen las reglas de datos del finder para COLLABORATOR, CUSTOMER y OTHER, y se incorpora finderNote opcional. |
| 0.9.0 | 2026-10-05 | Walter Molina | Se define que Lost & Found reutiliza las reglas técnicas existentes de PEC para carga y validación de archivos, manteniendo metadata y asociaciones propias. |
| 0.10.0 | 2026-10-05 | Walter Molina | Se define el autocompletado de colaborador reutilizando la capacidad existente de PEC y la persistencia de snapshot histórico en FoundItem. |
| 0.11.0 | 2026-10-05 | Walter Molina | Se define la prioridad de selección de CustodyPolicy por clasificación, tipo funcional y política GENERAL, bloqueando el registro cuando no existe política aplicable. |
| 0.12.0 | 2026-10-05 | Walter Molina | Se define itemCode como identificador funcional generado en backend, único por enterprise/año, no editable y seguro ante concurrencia. |
| 1.0.0 | 2026-10-05 | Walter Molina | Aprobación funcional de PEC-LF-FOUND-001 para Sprint 1. Se cierran las decisiones funcionales del registro de artículo encontrado y los pendientes restantes se trasladan al Plan técnico. |
| 1.1.0 | 2026-10-06 | Walter Molina | Se precisa el alcance de empresa y establecimiento por rol para el registro y consulta de catálogos Lost & Found (RB-FND-22). |
| 1.2.0 | 2026-10-06 | Walter Molina | Se precisa que el acceso de estos perfiles depende de empresas asociadas al usuario y que las consultas de zonas reciben y validan el contexto de empresa seleccionado. |

---

## 18. Estado actual de implementación

**Interfaz existente:** `pec-ui` ya tiene una pantalla visual de registro de artículo encontrado y autocompletado de colaborador. La clasificación todavía usa datos locales de tema/subtema y las zonas provienen de una lista local fija. El local se muestra de forma informativa, pero el flujo de encontrado no selecciona ni envía `establishmentId`. El botón «Guardar» está deshabilitado y no hay llamada a una API Lost & Found.

**Integración pendiente:** no se persiste `FoundItem`, no se genera un `itemCode` real ni se aplica `CustodyPolicy`. El autocompletado todavía no persiste el snapshot de nombre, apellido y cargo. Los archivos del formulario no usan metadata `sspectlffile` ni asociación `FoundItemFile`; tampoco se genera `LostFoundAudit`. Los permisos funcionales deben implementarse y validarse en backend mediante Keycloak + CASL/PoliciesGuard.

**Backend específico:** `pec-api` aún no tiene modelos Prisma, migraciones, endpoints ni servicios de dominio Lost & Found. Para la persistencia se reutilizará `DatabaseModule`/`DatabaseService` existente, sin crear clientes `PrismaClient` adicionales por módulo. También se aprovecharán las capacidades PEC de establecimientos, colaboradores y StorageService/GCS según sus límites de dominio.

| Requisito FOUND-001 | Estado UI actual | Acción pendiente |
|---|---|---|
| `establishmentId` obligatorio | Local solo informativo; no se selecciona ni envía | Integrar selección y envío del establecimiento permitido. |
| Zonas por establecimiento | Lista local fija | Consumir zonas habilitadas desde API. |
| `classificationOptionId` | Listas locales de tema/subtema | Consumir catálogo jerárquico y enviar el nodo final válido. |
| Snapshot del finder | Autocompletado de colaborador en UI | Guardar código, nombre, apellido y cargo mediante backend. |
| Archivos Lost & Found | Selección visual de archivos | Integrar carga y metadata `sspectlffile`/`FoundItemFile`. |
| `itemCode` | Sin generación real | Generar en backend. |
| `CustodyPolicy` | Sin aplicación real | Aplicar versión vigente en backend. |
| `expirationDate` | Sin cálculo real | Calcular cuando corresponda según política aplicada. |
| `LostFoundAudit` | Sin auditoría funcional | Registrar `FOUND_ITEM_CREATED` en backend. |
| Permisos backend | Matriz parcial en frontend | Autorizar en backend con rol y contexto de empresa/establecimiento. |

### Enfoque de Sprint 1

Sprint 1 **no debe recrear el frontend desde cero**. Su objetivo es integrar el backend de Lost & Found y adaptar la pantalla existente a los contratos que se definan a partir de ARCH-001, DATA-001 y esta Spec.

- **Backend:** implementar catálogo maestro Lost & Found, zonas internas, `FoundItem`, archivos propios del dominio, `CustodyPolicy`, `LostFoundAudit`, APIs, validaciones y seguridad backend, de forma incremental conforme al alcance aprobado.
- **Frontend:** adaptar el formulario existente para consumir APIs reales, sustituir las listas locales aplicables, seleccionar y enviar `establishmentId`, enviar `classificationOptionId`, cargar zonas por establecimiento, habilitar el guardado e integrar archivos.

Los pendientes técnicos de §15 se resolverán en el Plan técnico; esta descripción del estado actual no define DTOs, endpoints ni reglas nuevas.
