# PEC-LF-DATA-001
## Modelo lógico de datos — Artículos encontrados y declarados perdidos

> Esta especificación define el modelo lógico de datos del módulo Lost & Found de PEC.
> Deriva de `PEC-LF-ARCH-001` y debe mantenerse alineada con esa arquitectura.
>
> Este documento NO autoriza todavía la creación de tablas, migraciones, modelos Prisma o cambios de código.
>
> Clasificación utilizada:
> - **CONFIRMADO**
> - **PROPUESTA / INFERIDO**
> - **PENDIENTE DE DEFINICIÓN**

---

## 1. Encabezado

| Campo | Valor |
|---|---|
| ID | `PEC-LF-DATA-001` |
| Título | Modelo lógico de datos — Artículos encontrados y declarados perdidos |
| Versión | `1.2.0` |
| Estado | `APPROVED` |
| Autor | Mbrion / equipo PEC |
| Fecha | `2026-10-05` |
| Arquitectura relacionada | `PEC-LF-ARCH-001` |
| Work item / backlog | Pendiente de referencia |

---

## 2. Objetivo

Definir el modelo lógico completo de datos requerido para soportar el módulo de Artículos encontrados y declarados perdidos de PEC.

El modelo debe permitir implementar progresivamente, mediante distintos sprints, los siguientes procesos:

- registro de artículo perdido;
- registro de artículo encontrado;
- documentos y tarjetas;
- búsqueda nacional;
- coincidencias entre reportes y artículos;
- validación de propiedad;
- contacto con cliente;
- custodia;
- transferencia;
- entrega;
- destrucción o desecho;
- vencimientos;
- auditoría;
- reportes.

El objetivo es contar desde el inicio con una arquitectura de datos coherente, aunque las entidades se implementen de manera incremental por Sprint.

---

## 3. Alcance

### 3.1 Incluye

- entidades de dominio candidatas;
- relaciones entre entidades;
- cardinalidades;
- entidades PEC reutilizadas;
- nuevas entidades Lost & Found;
- claves primarias y foráneas propuestas;
- relación con archivos;
- relación con usuarios;
- relación con establecimientos;
- relación con empresa;
- auditoría;
- catálogos;
- estado actual de cada decisión;
- preguntas pendientes.

### 3.2 Fuera de alcance

- crear modelos Prisma;
- crear migraciones;
- crear tablas físicas;
- definir endpoints;
- implementar servicios;
- implementar frontend;
- definir algoritmos de matching;
- definir todavía todos los índices;
- aprobar políticas físicas de borrado/cascade;
- definir optimizaciones de rendimiento.

---

## 4. Principios heredados de `PEC-LF-ARCH-001`

### DATA-P01 — Separación de conceptos

**CONFIRMADO**

Un reporte de pérdida no representa necesariamente un objeto físico encontrado.

Por lo tanto:

`LostItemReport != FoundItem`

---

### DATA-P02 — Coincidencias

**CONFIRMADO**

La relación entre un reporte de pérdida y un artículo encontrado se representa mediante una entidad de asociación:

`ItemMatch`

No se recomienda almacenar directamente:

`FoundItem.lostItemReportId`

porque un artículo podría relacionarse con múltiples reportes candidatos y un reporte podría tener múltiples artículos candidatos.

---

### DATA-P03 — Custodia

**CONFIRMADO**

El artículo encontrado mantiene una referencia al custodio/local actual para consulta rápida.

El historial de movimientos debe mantenerse separadamente.

Ejemplo:

`FoundItem.currentEstablishmentId`

+

`CustodyMovement`

---

### DATA-P04 — Auditoría

**CONFIRMADO**

Los logs técnicos de PEC no sustituyen la auditoría funcional del módulo.

Las transiciones relevantes deben registrar:

- quién;
- cuándo;
- desde qué local;
- qué cambió.

---

### DATA-P05 — Infraestructura existente

**CONFIRMADO**

PEC ya dispone de:

- usuarios;
- empresas;
- establecimientos;
- autenticación Keycloak;
- autorización CASL;
- archivos/GCS;
- notificaciones;
- reportes/exportación;
- scheduler.

Estas capacidades deben reutilizarse o adaptarse cuando corresponda.

---

# 5. Entidades PEC a reutilizar

## 5.1 Empresa

Modelo existente: `sspectenterprise`; PK lógica actual `sspecrpkent`.

**CONFIRMADO:** guardar `enterprise_id` en entidades raíz/transaccionales principales: `LostItemReport`, `FoundItem`, `RecoveryClaim`, `ItemMatch`, `ItemTransfer`, `ItemDelivery`, `ItemDisposal` y `LostFoundAudit`, con referencia a `sspectenterprise.sspecrpkent`. Debe concordar con el establecimiento relacionado. No duplicar `enterprise_id` en tablas hijas si se deriva de su padre.

## 5.2 Establecimiento

Modelo PEC: `sspectestablishment`; PK `sspecrpkest`.

**CONFIRMADO:** reutilizarlo para establecimiento de registro, custodio actual, origen/destino de transferencia y local de entrega/disposición. GCSS se representa como establecimiento PEC. La zona Lost & Found es interna al establecimiento y no corresponde al `id_zona` territorial PEC.

## 5.3 Usuario PEC

Modelo PEC: `sspectuser`; PK `sspecrpkuse`.

**CONFIRMADO:** responsables operativos y actores de auditoría referencian usuarios PEC. No crear entidad `Responsible`, no codificar nombres de personas ni usar responsables en texto libre. Los eventos conservan el usuario que realmente ejecutó la operación; usar referencias como createdBy, reviewedBy, contactedBy, transferredBy, receivedBy, deliveredBy y disposedBy según aplique.

## 5.4 Archivos

**CONFIRMADO:** PEC cuenta con StorageService y Google Cloud Storage. Lost & Found reutiliza las políticas técnicas vigentes de PEC para extensiones, MIME, tamaños, límites y seguridad de la carga. `PHOTO` y `EVIDENCE` son clasificaciones funcionales del archivo, no reglas técnicas de formato. Lost & Found tendrá metadata conceptual propia `sspectlffile`, independiente de `sspectfile`, y asociaciones específicas del dominio: `FoundItemFile`, `LostItemReportFile`, `DeliveryFile`, `TransferFile`, `DisposalFile` y opcionalmente `OwnershipValidationFile`. No añadir FKs Lost & Found a `sspectfile`. Permisos por recurso, retención y acceso quedan para definición técnica posterior.

---

# 6. Entidades del modelo lógico

Los nombres físicos siguientes son referencias candidatas al patrón PEC, no aprobación de tablas Prisma ni migraciones.

## 6.1 `LostItemReport` — `sspectlostitemreport` (candidato)

Representa la declaración de pérdida, no el artículo físico. Tiene identificador funcional generado `idEncuentra` separado de la PK (`LF-L-AAAA-XXXXXX`, formato ajustable antes de implementar), empresa, establecimiento que registra/consulta cuando aplica, opción de clasificación Lost & Found, descripción, lugar/fecha aproximados, canal, fecha/hora y usuario registrador. Los niveles categoría/tema/subtema se obtienen recorriendo la jerarquía de opciones. Conserva snapshot del reportante (identificación, nombres, apellidos y contacto) y puede asociar archivos propios, incluyendo audio si la declaración es por voz.

Relaciones: empresa N:1; establecimiento de registro N:1 cuando aplique; opción de clasificación Lost & Found según corresponda; opción `LOST_FOUND_CHANNEL`; `LostItemReport 1:N ItemMatch`; `LostItemReport 1:N LostItemReportFile`.

## 6.2 `FoundItem` — `sspectfounditem` (candidato)

Representa el artículo físico encontrado. Incluye `itemCode` generado en backend e independiente de la PK física (formato lógico inicial `LF-F-AAAA-XXXXXX`), `enterprise_id`, `classificationOptionId`, descripción, lugar del hallazgo, `foundAt`, `foundTimeKnown`, `zoneId` obligatorio, estado/custodia por dimensiones, usuario registrador, `currentEstablishmentId` como custodio físico actual, `custodyPolicyId` y `expirationDate` cuando aplique. `classificationOptionId` referencia una opción activa de `LOST_FOUND_CLASSIFICATION` del mismo enterprise; los niveles presentes de categoría/tema/subtema se derivan recorriendo `parentId`.

Relaciones: empresa N:1; establecimiento custodio actual N:1; usuario creador N:1; zona interna N:1 mediante `zoneId`; `CustodyPolicy` N:1 mediante `custodyPolicyId`; `classificationOptionId` N:1 a una opción del catálogo `LOST_FOUND_CLASSIFICATION`; `FoundItem 1:N ItemMatch`, `RecoveryClaim`, `CustodyMovement`, `ItemTransfer`, `FoundItemValidationReference` y `FoundItemFile`; `FoundItem 0..1 DocumentDetail`, `0..1 CardDetail` y `0..1 ItemDisposal`.

**CONFIRMADO:** `classificationOptionId` apunta al nodo final válido de una opción activa de `LOST_FOUND_CLASSIFICATION` del mismo enterprise y debe cumplir su jerarquía flexible de hasta tres niveles; categoría/tema/subtema son niveles derivados cuando existan. `currentEstablishmentId` es custodio actual; Finder no es entidad inicial y sus datos históricos se guardan como snapshot; los estados se separan por dimensión y el inicial del artículo es `REGISTERED`.

## 6.3 `ItemMatch` — `sspectitemmatch` (candidato)

Asociación persistente de posible coincidencia entre un `LostItemReport` y un `FoundItem`; soporta la relación N:M. Estados controlados `POSSIBLE`, `CONFIRMED`, `DISCARDED`; distingue generación manual/automática y conserva conceptualmente score, nivel y método. Un match confirmado no prueba propiedad ni entrega.

Relaciones: N:1 `LostItemReport`, N:1 `FoundItem` y actor revisor PEC cuando corresponda. El algoritmo, pesos y umbrales exactos permanecen pendientes para la Spec de matching.

## 6.4 `RecoveryClaim` (entidad principal; `sspectlfrecoveryclaim`, candidato)

Representa que una persona reclama un `FoundItem`. Puede originarse por `LostItemReport` + `ItemMatch` o directamente sobre `FoundItem` sin declaración previa. `FoundItem 1:N RecoveryClaim` permite reclamaciones de varias personas. Conserva snapshot del reclamante y referencia a la empresa, establecimiento y usuario responsables que apliquen.

Relaciones: N:1 `FoundItem`; asociación opcional al reporte/match de origen; `RecoveryClaim 1:N OwnershipValidation`; `RecoveryClaim 1:N ContactAttempt`; `RecoveryClaim 0..1 ItemDelivery` final válida.

## 6.5 `OwnershipValidation` y sus entidades relacionadas

La validación pertenece a un reclamo: `RecoveryClaim 1:N OwnershipValidation`. No depende de `ItemMatch`.

- `ValidationQuestion`: configuración administrable asociable a opciones de clasificación (categoría/tema/subtema), con active, order, required y questionType cuando aplique.
- `FoundItemValidationReference`: valores privados de referencia asociados al artículo y a una pregunta; no mostrarlos al reclamante antes de que responda.
- `OwnershipValidation`: cada intento/resultados `HIGH_MATCH`, `MEDIUM_MATCH`, `LOW_MATCH`.
- `OwnershipValidationAnswer`: respuesta del reclamante, FK lógica a pregunta y snapshot del texto exacto presentado.

Cardinalidades de referencia y respuesta se detallan en §10. La lógica exacta de evaluación pertenece a la Spec de matching/validación posterior; no revelar `referenceValue` previamente.

## 6.6 `ContactAttempt` — `sspectlfcontactattempt` (candidato)

Registra una comunicación asociada a un `RecoveryClaim`, nunca directamente a `ItemMatch` o `LostItemReport`. Relación `RecoveryClaim 1:N ContactAttempt`. Cada intento indica modalidad `MANUAL` o `AUTOMATIC`, usuario cuando corresponda, canal controlado, fecha/hora y resultado: `CONTACTED`, `NO_ANSWER`, `WRONG_NUMBER` o `WILL_VISIT_STORE`.

Regla aprobada: N intentos manuales y máximo 3 intentos automáticos. No se asigna valor a N en este modelo.

## 6.7 `CustodyMovement` — `sspectcustodymovement` (candidato)

Historial estructurado de cambio físico de custodia: `FoundItem 1:N CustodyMovement`; registra establecimientos origen/destino y usuario responsable cuando aplique. `FoundItem.currentEstablishmentId` refleja custodio actual y se mantiene consistente/transaccionalmente con el movimiento. Solo el establecimiento que posee físicamente el artículo modifica custodia.

## 6.8 `ItemTransfer` — `sspectlfitemtransfer` (candidato)

Entidad de proceso propia y distinta de `CustodyMovement`; `FoundItem 1:N ItemTransfer`. Describe origen/destino, despacho, responsable, recepción, evidencias, fechas y estado. No se completa como recibida sin registrar la recepción. Al completarse, genera/refleja movimiento de custodia y actualiza el custodio actual según corresponda.

## 6.9 `ItemDelivery` — `sspectitemdelivery` (candidato)

Proceso de entrega asociado a reclamación: `RecoveryClaim 0..1 ItemDelivery` final válida. No mantiene `ItemMatch` como dependencia de dominio. Registra artículo, cliente/reclamante, establecimiento, usuario responsable, fecha/hora, firmas de cliente y colaborador, observaciones y archivo del acta firmada. Solo puede completarse cuando la reclamación supera la validación requerida y existe el acta firmada. El mecanismo técnico de firma queda pendiente para DELIVERY-001.

## 6.10 `ItemDisposal` — `sspectitemdisposal` (candidato)

Cierre definitivo sin entrega; `FoundItem 0..1 ItemDisposal`. Tipos controlados `DESTROYED` / `DISCARDED`; relaciona artículo, establecimiento y usuario responsable, fecha, observaciones y evidencias. Es final y auditable, no elimina artículo/historial y no puede completarse si ya se entregó.

## 6.11 `DocumentDetail` y `CardDetail`

Detalles opcionales del artículo base: `FoundItem 0..1 DocumentDetail`; `FoundItem 0..1 CardDetail`.

`DocumentDetail` conserva referencia a opción de `LOST_FOUND_DOCUMENT_TYPE`, número y datos del titular cuando apliquen. `CardDetail` conserva banco, referencia a opción de `LOST_FOUND_CARD_TYPE` y primeros seis/últimos cuatro dígitos; nunca se muestra el número completo. No confundir tipo de identificación personal `public."documentType"` (DNI, PASSPORT, RUC, CI, ANONIMO) con tipo de documento físico Lost & Found.

## 6.12 Zonas internas

`LostFoundZone` es catálogo lógico global de ubicaciones internas del establecimiento (Servicio al cliente, cajas, pasillos, estacionamiento, zona de comidas, entrada principal, etc.). `LostFoundEstablishmentZone` habilita la relación entre establecimiento y zonas disponibles. Al registrar, `FoundItem.zoneId` es obligatorio y referencia una zona activa, habilitada para `currentEstablishmentId` mediante una relación activa `LostFoundEstablishmentZone`. No se permite `FoundItem` sin `zoneId`; si el establecimiento no tiene zonas configuradas y activas, la capa funcional bloquea el registro. No derivar de Region/`id_zona`, no usar snapshot territorial ni modelar zona geográfica para esta función.

## 6.13 `CustodyPolicy`

Configuración aprobada, no hardcodeada y versionada mediante registros inmutables. Cada registro de `CustodyPolicy` representa una versión concreta; una nueva configuración crea otro registro y no modifica el histórico referenciado por `FoundItem`. Conceptualmente puede contemplar `policyCode`, `classificationOptionId` opcional, tipo funcional aplicable, parámetros de custodia, `destinationEstablishmentId`, `responsibleUserId` cuando aplique, `finalAction`, `requiresTransfer`, `requiresEvidence`, `active` y `version`. No se definen aquí columnas físicas finales.

Al registrar `FoundItem`, se selecciona una política activa aplicable en este orden: (1) específica del `classificationOptionId` validado; (2) por tipo funcional; (3) `GENERAL`. Si ninguna aplica, se bloquea el registro. `FoundItem` conserva `custodyPolicyId` del registro inmutable aplicado y `expirationDate` cuando corresponda. Nuevas versiones no cambian `custodyPolicyId` ni recalculan `expirationDate` de artículos históricos.

Referencias funcionales iniciales: `DOCUMENT` — 30 días en local, transferencia a GCSS, 90 días adicionales en GCSS y destrucción según el flujo aprobado; `FOOD` — hasta el cierre del local y desecho con evidencia; `GENERAL` — 90 días y luego correspondencia/transferencia configurada. La alerta tres días antes se conserva como referencia del modelo. Los detalles técnicos de cómputo y operación permanecen para el Plan técnico.

## 6.14 `LostFoundAudit` — `sspectlfaudit` (candidato)

Auditoría genérica durable de negocio para Lost & Found. Registra como mínimo entityType/entityId/eventType, userId, username/contexto histórico cuando aplique, establishmentId, timestamp, previousValue/newValue y observations. Complementa, no sustituye, `CustodyMovement`, `ItemTransfer`, `ItemDelivery` e `ItemDisposal`. Los logs técnicos no son auditoría de negocio.

---

# 7. Asociaciones con archivos

**CONFIRMADO:** metadata independiente `sspectlffile`, almacenamiento compartido con PEC StorageService/Google Cloud Storage y asociaciones específicas Lost & Found: `FoundItemFile`, `LostItemReportFile`, `DeliveryFile`, `TransferFile`, `DisposalFile` y opcional `OwnershipValidationFile`. Se reutilizan las reglas técnicas PEC de extensiones, MIME, tamaños, límites y seguridad; `PHOTO` / `EVIDENCE` solo indican propósito funcional. No agregar FKs de Lost & Found a `sspectfile` ni reutilizar asociaciones de Quejas. La política de retención/borrado físico de archivos se define separadamente.

---

# 8. Catálogos y configuración

**CONFIRMADO:** Lost & Found tendrá infraestructura maestra de catálogos independiente del catálogo funcional actual de otros módulos PEC. No reutilizar `sspectcatalog` como catálogo funcional Lost & Found. Modelo conceptual: `LostFoundCatalog 1:N LostFoundCatalogOption`; nombres físicos candidatos `sspectlfcatalog` y `sspectlfcatalogoption`, sin crear tablas, modelos Prisma ni migraciones en esta especificación.

`LostFoundCatalog` contiene conceptualmente `id`, `code`, `name`, `description`, `enterpriseId`, `active` y campos estándar PEC de auditoría/versionado. `code` identifica el catálogo funcional. Catálogos iniciales: `LOST_FOUND_CLASSIFICATION`, `LOST_FOUND_DOCUMENT_TYPE`, `LOST_FOUND_CARD_TYPE` y `LOST_FOUND_CHANNEL`.

`LostFoundCatalogOption` contiene conceptualmente `id`, `catalogId`, `parentId` nullable, `optionType`, `code`, `label`, `value` nullable, `sortOrder`, `active`, `enterpriseId` y campos estándar PEC. Cada opción puede tener cero/uno parent y varios children. `parentId` debe apuntar a una opción del mismo catálogo; no se permiten ciclos jerárquicos. Las opciones utilizadas históricamente no se eliminan físicamente y se desactivan mediante `active`. No usar `scope_type` / `scope_ref_id` en esta versión.

**Clasificación:** `LOST_FOUND_CLASSIFICATION` admite como máximo CATEGORY (parent null) → THEME (parent CATEGORY) → SUBTHEME (parent THEME). Una rama puede terminar en cualquiera de los tres niveles. `FoundItem` guarda solo `classificationOptionId` del nodo seleccionable más específico de la rama; categoría/tema/subtema se derivan recorriendo `parentId` cuando existan. La opción debe estar activa, pertenecer al enterprise correspondiente y al catálogo correcto, y cumplir la jerarquía configurada. No se exigen niveles inexistentes ni se selecciona un padre si un hijo configurado es obligatorio.

**Otros catálogos administrables:** `LOST_FOUND_DOCUMENT_TYPE` (por ejemplo Cédula, Licencia, Pasaporte, Carné), `LOST_FOUND_CARD_TYPE` (Crédito, Débito, Regalo, Afiliación) y `LOST_FOUND_CHANNEL` (Presencial, Teléfono, Chat, Correo electrónico, Redes sociales, App móvil). Son datos configurables.

**Fuera del catálogo maestro:** `LostFoundZone` / `LostFoundEstablishmentZone`; `public."documentType"` para identificación personal; estados controlados del dominio; `ValidationQuestion`; `CustodyPolicy`; responsables `sspectuser`; destinos físicos `sspectestablishment`; y `finderType` como valor controlado (`COLLABORATOR`, `CUSTOMER`, `OTHER`). Mantener temas/subtemas de búsqueda y reportes como niveles derivados de la opción de clasificación, no como columnas separadas de `FoundItem`.

---

# 9. Modelo lógico general

```text
sspectenterprise
  ├── LostItemReport ──1:N── ItemMatch ──N:1── FoundItem
  ├── LostFoundCatalog ──1:N── LostFoundCatalogOption ──parentId── LostFoundCatalogOption
  └── FoundItem ──1:N── RecoveryClaim
                         ├──1:N── OwnershipValidation ──1:N── OwnershipValidationAnswer
                         ├──1:N── ContactAttempt
                         └──0..1── ItemDelivery

FoundItem ──1:N── CustodyMovement
FoundItem ──1:N── ItemTransfer
FoundItem ──N:1── CustodyPolicy (custodyPolicyId)
FoundItem ──N:1── LostFoundZone (zoneId obligatorio)
FoundItem ──1:N── FoundItemFile ──N:1── sspectlffile
FoundItem ──0..1── ItemDisposal
FoundItem ──0..1── DocumentDetail
FoundItem ──0..1── CardDetail
FoundItem ──1:N── FoundItemValidationReference
ValidationQuestion ──1:N── FoundItemValidationReference
ValidationQuestion ──1:N── OwnershipValidationAnswer

sspectestablishment ── custodio/destinos/locales de proceso
sspectuser ── actores responsables de operaciones y auditoría
LostFoundZone ── LostFoundEstablishmentZone ── sspectestablishment
LostItemReport/ContactAttempt ── N:1 ── LOST_FOUND_CHANNEL option
FoundItem ── N:1 ── classificationOptionId (LOST_FOUND_CLASSIFICATION)
DocumentDetail ── N:1 ── LOST_FOUND_DOCUMENT_TYPE option
CardDetail ── N:1 ── LOST_FOUND_CARD_TYPE option
FoundItem/LostItemReport/Delivery/Transfer/Disposal ── asociaciones propias ── sspectlffile
```

---

# 10. Cardinalidades aprobadas

| Origen | Relación | Destino |
|---|---|---|
| LostFoundCatalog | 1:N | LostFoundCatalogOption |
| LostFoundCatalogOption | 0..1 parent / 1:N children | LostFoundCatalogOption (mismo catálogo, sin ciclos) |
| LostItemReport | 1:N | ItemMatch |
| FoundItem | 1:N | ItemMatch |
| FoundItem | 1:N | RecoveryClaim |
| RecoveryClaim | 1:N | OwnershipValidation |
| RecoveryClaim | 1:N | ContactAttempt |
| RecoveryClaim | 0..1 | ItemDelivery |
| FoundItem | 1:N | CustodyMovement |
| FoundItem | 1:N | ItemTransfer |
| FoundItem | N:1 | CustodyPolicy (registro inmutable mediante custodyPolicyId) |
| FoundItem | N:1 | LostFoundZone activa (zoneId obligatorio y habilitada para currentEstablishmentId) |
| FoundItem | 1:N | FoundItemFile |
| FoundItem | 0..1 | ItemDisposal |
| FoundItem | 0..1 | DocumentDetail |
| FoundItem | 0..1 | CardDetail |
| FoundItem | 1:N | FoundItemValidationReference |
| ValidationQuestion | 1:N | FoundItemValidationReference |
| OwnershipValidation | 1:N | OwnershipValidationAnswer |
| ValidationQuestion | 1:N | OwnershipValidationAnswer |
| LostFoundCatalogOption (LOST_FOUND_CLASSIFICATION) | 1:N | FoundItem (referencia mediante classificationOptionId) |
| LostFoundCatalogOption (LOST_FOUND_CLASSIFICATION) | 1:N | LostItemReport (opción de clasificación cuando corresponda) |
| LostFoundCatalogOption (LOST_FOUND_CHANNEL) | 1:N | LostItemReport / ContactAttempt |
| LostFoundCatalogOption (LOST_FOUND_DOCUMENT_TYPE) | 1:N | DocumentDetail |
| LostFoundCatalogOption (LOST_FOUND_CARD_TYPE) | 1:N | CardDetail |

`ItemMatch` en ambos sentidos implementa la relación N:M entre LostItemReport y FoundItem. Las entidades raíz/transaccionales listadas en §5 guardan `enterprise_id`; hijos lo derivan cuando corresponda.

---

# 11. Datos conceptuales de FoundItem

| Dato conceptual | Tratamiento aprobado |
|---|---|
| id / PK física | Interna; no es identificador visible funcional |
| itemCode | Identificador funcional visible, generado en backend, independiente de PK, no editable ni reutilizable; formato lógico inicial `LF-F-AAAA-XXXXXX`, único por enterprise/año y seguro ante concurrencia. Mecanismo físico de correlativo/sequence en Plan técnico |
| enterprise_id | FK lógica a empresa PEC; consistente con establecimiento |
| currentEstablishmentId | Custodio físico actual, distinto de ubicación histórica |
| classificationOptionId | Nodo final válido de `LOST_FOUND_CLASSIFICATION`; categoría/tema/subtema presentes se derivan de su jerarquía `parentId` |
| zoneId | Obligatorio; FK lógica a LostFoundZone activa y habilitada para currentEstablishmentId mediante LostFoundEstablishmentZone |
| finderType | COLLABORATOR, CUSTOMER u OTHER |
| finderEmployeeCode | Obligatorio para COLLABORATOR; proviene de la capacidad PEC existente |
| finderName / finderLastName / finderPosition | Snapshot histórico: nombre, apellido y cargo obligatorios para COLLABORATOR; nombre y apellido obligatorios para CUSTOMER y opcionales para OTHER; cargo aplica a COLLABORATOR |
| finderNote | Opcional para observaciones del finder |
| registeredByUserId | Usuario PEC que registró |
| estado | Dimensiones de dominio separadas y controladas; estado inicial REGISTERED |
| custodyPolicyId / expirationDate | Referencia al registro inmutable de CustodyPolicy aplicado y fecha de vencimiento calculada cuando corresponda; cambios posteriores no modifican históricos |
| foundAt / foundTimeKnown | Fecha de hallazgo obligatoria y hora opcional; foundTimeKnown indica si la hora fue informada realmente, sin presentarla como conocida cuando fue inferida |
| createdAt / updatedAt | Fechas de sistema |

---

# 12. Datos conceptuales del reportante y Finder

`LostItemReport` conserva snapshot del reportante: tipo/número de identificación, nombres, apellidos, celular/teléfono, email y demás datos registrados para el proceso. No se crea tabla genérica de snapshots.

Finder permanece embebido conceptualmente en `FoundItem`, sin entidad Finder. `finderType` toma `COLLABORATOR`, `CUSTOMER` u `OTHER`. Para `COLLABORATOR`, código, nombre, apellido y cargo son obligatorios; se obtienen mediante la capacidad PEC existente y se conservan como snapshot histórico. Para `CUSTOMER`, nombre y apellido son obligatorios. Para `OTHER`, nombre y apellido son opcionales. `finderNote` es opcional. Sprint 1 no incorpora identificación, teléfono ni email del finder.

---

# 13. Zonas

`LostFoundZone` + `LostFoundEstablishmentZone` representan ubicaciones internas de cada local. Ejemplos: servicio al cliente, cajas, pasillos, estacionamiento, zona de comidas y entrada principal. Al registrar, `FoundItem.zoneId` es obligatorio y solo puede referenciar una zona activa y vinculada mediante una relación activa con `currentEstablishmentId`. Si el establecimiento no tiene zonas Lost & Found activas configuradas, el registro se bloquea en la capa funcional.

No utilizar `Region → id_zona`, snapshots de zona territorial ni `id_zona` geográfico para este dominio.

---

# 14. Estados

No reutilizar estados de Quejas ni modelar todos los procesos con un status único. Mantener dimensiones controladas por dominio: artículo/custodia, matching, RecoveryClaim, validación, contacto, entrega, transferencia y disposición. No permitir alta libre de estados.

Representar los estados funcionales requeridos: Registrado; En custodia del local; Gestión de contacto; Cliente contactado; Entregado al cliente; Pendiente de transferencia; Transferido al GCSS; En custodia GCSS; Pendiente de destrucción; Destruido; Posible coincidencia. Su asignación exacta a dimensión respeta los ciclos independientes y no crea un único enum.

---

# 15. Auditoría y trazabilidad

`LostFoundAudit` es la auditoría genérica durable de negocio. Registra quién ejecutó la acción, cuándo, desde qué local y qué cambió, además de entityType/entityId/eventType, contexto histórico, previousValue/newValue y observaciones cuando corresponda.

Complementa las entidades de proceso `CustodyMovement`, `ItemTransfer`, `ItemDelivery` e `ItemDisposal`, que conservan su propia historia estructurada. Los logs técnicos y campos created/modified no sustituyen auditoría funcional.

---

# 16. Reglas de integridad

- `FoundItem.currentEstablishmentId` representa el custodio actual mientras el artículo esté bajo custodia PEC; todo cambio se coordina con `CustodyMovement`.
- Al registrar `FoundItem`, `zoneId` no puede ser nulo y referencia una LostFoundZone activa, habilitada para el establecimiento custodio mediante LostFoundEstablishmentZone.
- `FoundItem.custodyPolicyId` referencia el registro inmutable de CustodyPolicy aplicado; no se registra FoundItem sin política activa aplicable ni se recalcula automáticamente su `expirationDate` histórica.
- `FoundItem.itemCode` se genera en backend, es único por enterprise/año, no editable, no reutilizable y seguro ante concurrencia; el mecanismo físico se define en el Plan técnico.
- `CustodyMovement`, `ItemTransfer`, `RecoveryClaim`, detalles y procesos deben referir al padre de dominio correspondiente.
- No completar `ItemDelivery` sin validación requerida y acta firmada.
- No completar `ItemDisposal` después de entrega; entrega y disposición completadas son cierres finales excluyentes.
- No completar `ItemTransfer` como recibida sin recepción registrada.
- Solo el establecimiento custodio puede cambiar custodia.
- `ItemMatch.CONFIRMED` no equivale a `OwnershipValidation` aprobada; validación aprobada habilita entrega, pero no la completa.
- No exponer PAN completo ni referencias/respuestas de validación antes de que el reclamante responda.
- Conservar historia transaccional; desactivar catálogos/configuración referenciados en vez de borrarlos físicamente. FK con RESTRICT por defecto; CASCADE solo en dependencias puramente técnicas sin pérdida de historia. Retención/borrado de archivos es política separada.
- Snapshot dentro de su entidad/evento: reportante en LostItemReport, reclamante en RecoveryClaim, Finder en FoundItem, personas representadas en acta en ItemDelivery, texto de pregunta en OwnershipValidationAnswer y contexto de evento en LostFoundAudit. No crear tabla genérica de snapshots.

---
# 17. Estrategia de implementación por Sprint

El modelo lógico completo se define desde el inicio, pero las entidades se implementarán progresivamente.

## Sprint 1 — Registro de artículo encontrado

Alcance lógico aprobado para el registro, con implementación física pendiente del Plan técnico:

- `FoundItem`, estado inicial `REGISTERED` y generación de `itemCode`;
- `LostFoundCatalog` y `LostFoundCatalogOption` para la clasificación;
- `LostFoundZone` y `LostFoundEstablishmentZone`;
- `sspectlffile` y `FoundItemFile`, con reglas de carga PEC;
- `CustodyPolicy`, aplicación de la versión inmutable y `expirationDate` cuando corresponda;
- `LostFoundAudit` para la auditoría mínima de creación;
- referencias a `sspectenterprise`, `sspectestablishment` y `sspectuser`;
- seguridad backend y validaciones del registro.

Este alcance lógico no autoriza crear todavía modelos Prisma, tablas ni migraciones.

## Sprint 2 — Registro de artículo perdido

Entidades/capacidades candidatas:

- `LostItemReport`;
- datos de cliente/reportante;
- archivos;
- canal;
- local;
- generación de ID Encuentra.

## Sprint posterior — Búsqueda y matching

- `ItemMatch`;
- búsqueda nacional;
- score/nivel de coincidencia;
- revisión y descarte.

## Sprint posterior — Validación y contacto

- `OwnershipValidation`;
- `ContactAttempt`.

## Sprint posterior — Custodia

- `CustodyMovement`;
- transferencias;
- recepción;
- custodia GCSS.

## Sprint posterior — Entrega

- `ItemDelivery`;
- acta;
- firmas;
- evidencias.

## Sprint posterior — Disposición

- `ItemDisposal`;
- destrucción;
- desecho;
- evidencias.

> La secuencia final se definirá en el Product Backlog. Esta sección no implica que todas las entidades deban crearse desde el primer Sprint.

---

# 18. Preguntas pendientes

Los siguientes asuntos conservan su estado abierto y se detallan en las Specs funcionales o técnicas correspondientes.

Pendientes reales únicamente:

1. Mecanismo técnico exacto de firma del acta (DELIVERY-001).
2. Algoritmo, pesos y umbrales exactos de matching.
3. Proveedor y detalles técnicos de transcripción.
4. Contrato técnico del servicio externo de datos/autocompletado del cliente.
5. Detalles de decisión que correspondan a Specs funcionales posteriores.
# 19. Criterios de aprobación

La aprobación `v1.2.0 APPROVED` de `PEC-LF-DATA-001` queda registrada con el cumplimiento de estos criterios:

- [x] `PEC-LF-ARCH-001` esté aprobado.
- [x] Las entidades principales estén acordadas.
- [x] Las relaciones principales estén acordadas.
- [x] Las cardinalidades críticas estén acordadas.
- [x] Las referencias PEC reutilizadas estén identificadas.
- [x] La estrategia de archivos esté acordada.
- [x] La estrategia de auditoría esté acordada.
- [x] La estrategia de custodia esté acordada.
- [x] La estrategia de catálogos esté acordada.
- [x] La estrategia de estados esté acordada.
- [x] La representación de GCSS esté acordada.
- [x] Las decisiones bloqueantes de Sprint 1 estén resueltas.
- [x] Walter Molina haya revisado y aprobado el modelo.
- [x] Las discrepancias con `PEC-LF-ARCH-001` estén resueltas.

---

# 20. Estado de aprobación

**Estado:** `APPROVED`

**Versión:** `1.2.0`

**Aprobador:**

- Walter Molina

**Fecha:** `2026-10-05`

**Observación:**

Modelo lógico aprobado y alineado con `PEC-LF-FOUND-001` v1.0.0 antes de iniciar implementación física.

La aprobación de este documento no implica crear todas las tablas inmediatamente. La implementación será incremental conforme al backlog y a las Specs funcionales aprobadas.

---

# 21. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.1.0 | 2026-09-30 | Mbrion / equipo PEC | Primer borrador del modelo lógico completo de Lost & Found. |
| 0.2.0 | 2026-09-30 | Mbrion / equipo PEC | Consolidación de decisiones arquitectónicas y de modelo aprobadas durante revisión funcional/técnica. El resumen §22 remite a las decisiones documentadas en las secciones del modelo. |
| 1.0.0 | 2026-09-30 | Walter Molina | Aprobación del modelo lógico Lost & Found por Walter Molina. |
| 1.1.0 | 2026-10-03 | Walter Molina | Se incorpora LostFoundCatalog/LostFoundCatalogOption y se sustituye topicId/subtopicId de FoundItem por classificationOptionId. |
| 1.2.0 | 2026-10-05 | Walter Molina | Se alinea el modelo lógico con PEC-LF-FOUND-001 v1.0.0: custodyPolicyId, foundAt/foundTimeKnown, finderNote, zoneId obligatorio, prioridad de CustodyPolicy, itemCode y alcance técnico lógico de Sprint 1. |

---

# 22. Resumen de decisiones del modelo

Este resumen orienta la consulta del modelo lógico desarrollado en §§5–16. `LostItemReport` y `FoundItem` son conceptos independientes; las relaciones, entidades y cardinalidades se describen en §§6 y 10. Las referencias y reutilización PEC, archivos, zonas, catálogo, roles y configuración se describen en §§5, 7–8 y 13. Estados, auditoría, integridad y snapshots se describen en §§14–16.

Lost & Found usa `LostFoundCatalog` / `LostFoundCatalogOption` independientes del catálogo funcional PEC actual; la clasificación flexible categoría/tema/subtema se deriva por `parentId`, y `FoundItem` guarda solo `classificationOptionId`. El artículo conserva `foundAt`, `foundTimeKnown`, `zoneId` obligatorio y `custodyPolicyId` del registro inmutable aplicado; `itemCode` es único por enterprise/año. Finder permanece embebido con `finderNote` opcional. Las reglas de CustodyPolicy y el alcance lógico de Sprint 1 se describen en §6.13 y §17. Los nombres físicos siguen siendo referencias conceptuales y no autorizan implementar tablas, Prisma ni migraciones. El documento está en estado **APPROVED**.

Pendientes reales detallados en §18: mecanismo técnico de firma; algoritmo/pesos/umbrales de matching; proveedor/detalles de transcripción; contrato técnico del servicio externo de datos del cliente; y detalle implementable de Specs funcionales posteriores.
