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
| Versión | `1.0.0` |
| Estado | `APPROVED` |
| Autor | Mbrion / equipo PEC |
| Fecha | `2026-09-30` |
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

**CONFIRMADO:** PEC cuenta con StorageService y Google Cloud Storage. Lost & Found tendrá metadata conceptual propia `sspectlffile`, independiente de `sspectfile`, y asociaciones específicas del dominio: `FoundItemFile`, `LostItemReportFile`, `DeliveryFile`, `TransferFile`, `DisposalFile` y opcionalmente `OwnershipValidationFile`. No añadir FKs Lost & Found a `sspectfile`. Permisos por recurso, retención y acceso quedan para definición técnica posterior.

---

# 6. Entidades del modelo lógico

Los nombres físicos siguientes son referencias candidatas al patrón PEC, no aprobación de tablas Prisma ni migraciones.

## 6.1 `LostItemReport` — `sspectlostitemreport` (candidato)

Representa la declaración de pérdida, no el artículo físico. Tiene identificador funcional generado `idEncuentra` separado de la PK (`LF-L-AAAA-XXXXXX`, formato ajustable antes de implementar), empresa, establecimiento que registra/consulta cuando aplica, tema/subtema, descripción, lugar/fecha aproximados, canal, fecha/hora y usuario registrador. Conserva snapshot del reportante (identificación, nombres, apellidos y contacto) y puede asociar archivos propios, incluyendo audio si la declaración es por voz.

Relaciones: empresa N:1; establecimiento de registro N:1 cuando aplique; `LostItemReport 1:N ItemMatch`; `LostItemReport 1:N LostItemReportFile`.

## 6.2 `FoundItem` — `sspectfounditem` (candidato)

Representa el artículo físico encontrado. Incluye `itemCode` generado independientemente de PK física (`LF-F-AAAA-XXXXXX`, formato ajustable), `enterprise_id`, tema/subtema, descripción/categoría, lugar/fecha de hallazgo, zona interna, estado/custodia por dimensiones, usuario registrador, `currentEstablishmentId` como custodio físico actual, `expirationDate` y snapshot/version de CustodyPolicy aplicada.

Relaciones: empresa N:1; establecimiento custodio actual N:1; usuario creador N:1; zona interna N:1; `FoundItem 1:N ItemMatch`, `RecoveryClaim`, `CustodyMovement`, `ItemTransfer`, `FoundItemValidationReference` y archivos; `FoundItem 0..1 DocumentDetail`, `0..1 CardDetail` y `0..1 ItemDisposal`.

**CONFIRMADO:** temas/subtemas usan la infraestructura de catálogos PEC con valores propios del módulo; `currentEstablishmentId` es custodio actual; Finder no es entidad inicial y sus datos históricos se guardan como snapshot; los estados se separan por dimensión.

## 6.3 `ItemMatch` — `sspectitemmatch` (candidato)

Asociación persistente de posible coincidencia entre un `LostItemReport` y un `FoundItem`; soporta la relación N:M. Estados controlados `POSSIBLE`, `CONFIRMED`, `DISCARDED`; distingue generación manual/automática y conserva conceptualmente score, nivel y método. Un match confirmado no prueba propiedad ni entrega.

Relaciones: N:1 `LostItemReport`, N:1 `FoundItem` y actor revisor PEC cuando corresponda. El algoritmo, pesos y umbrales exactos permanecen pendientes para la Spec de matching.

## 6.4 `RecoveryClaim` (entidad principal; `sspectlfrecoveryclaim`, candidato)

Representa que una persona reclama un `FoundItem`. Puede originarse por `LostItemReport` + `ItemMatch` o directamente sobre `FoundItem` sin declaración previa. `FoundItem 1:N RecoveryClaim` permite reclamaciones de varias personas. Conserva snapshot del reclamante y referencia a la empresa, establecimiento y usuario responsables que apliquen.

Relaciones: N:1 `FoundItem`; asociación opcional al reporte/match de origen; `RecoveryClaim 1:N OwnershipValidation`; `RecoveryClaim 1:N ContactAttempt`; `RecoveryClaim 0..1 ItemDelivery` final válida.

## 6.5 `OwnershipValidation` y sus entidades relacionadas

La validación pertenece a un reclamo: `RecoveryClaim 1:N OwnershipValidation`. No depende de `ItemMatch`.

- `ValidationQuestion`: configuración administrable asociable a tema/subtema, con active, order, required y questionType cuando aplique.
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

`DocumentDetail` conserva tipo de documento físico, número y datos del titular cuando apliquen. `CardDetail` conserva banco, tipo, primeros seis y últimos cuatro dígitos; nunca se muestra el número completo. No confundir tipo de identificación personal `public."documentType"` (DNI, PASSPORT, RUC, CI, ANONIMO) con tipo de documento físico Lost & Found.

## 6.12 Zonas internas

`LostFoundZone` es catálogo lógico global de ubicaciones internas del establecimiento (Servicio al cliente, cajas, pasillos, estacionamiento, zona de comidas, entrada principal, etc.). `LostFoundEstablishmentZone` habilita la relación entre establecimiento y zonas disponibles. Cada establecimiento utiliza solo zonas asociadas y `FoundItem` referencia la zona correspondiente. No derivar de Region/`id_zona`, no usar snapshot territorial ni modelar zona geográfica para esta función.

## 6.13 `CustodyPolicy`

Configuración aprobada, no hardcodeada y versionada: categoría/tipo, días de custodia local/central, días de alerta, establecimiento destino, usuario responsable cuando aplique, acción final, requiere transferencia/evidencia, activo y versión. Al crear/aplicar una política, conservar en `FoundItem`/caso la versión aplicada y `expirationDate`. Cambios futuros de política no recalculan automáticamente registros históricos.

Valores: documentos 30 días en local, transferencia a GCSS, 90 días en GCSS y destrucción; artículos generales 90 días antes de correspondencia/transferencia configurada; alimentos hasta cierre del local y desecho con evidencia; alerta tres días antes. Solo detalles de cómputo y operación expresamente pendientes quedan abiertos en ARCH-001.

## 6.14 `LostFoundAudit` — `sspectlfaudit` (candidato)

Auditoría genérica durable de negocio para Lost & Found. Registra como mínimo entityType/entityId/eventType, userId, username/contexto histórico cuando aplique, establishmentId, timestamp, previousValue/newValue y observations. Complementa, no sustituye, `CustodyMovement`, `ItemTransfer`, `ItemDelivery` e `ItemDisposal`. Los logs técnicos no son auditoría de negocio.

---

# 7. Asociaciones con archivos

**CONFIRMADO:** metadata independiente `sspectlffile`, almacenamiento compartido con PEC StorageService/Google Cloud Storage y asociaciones específicas Lost & Found: `FoundItemFile`, `LostItemReportFile`, `DeliveryFile`, `TransferFile`, `DisposalFile` y opcional `OwnershipValidationFile`. No agregar FKs de Lost & Found a `sspectfile` ni reutilizar asociaciones de Quejas. La política de retención/borrado físico de archivos se define separadamente.

---

# 8. Catálogos y configuración

**CONFIRMADO:** reutilizar infraestructura de catálogo PEC para temas/subtemas, con valores semánticos propios de Lost & Found. No asumir valores de Quejas. Reutilizar `public."documentType"` solo para identificación de cliente/reportante. Mantener propios los tipos de documento físico y tarjeta, preguntas de validación, `LostFoundZone` y `CustodyPolicy`.

Canales controlados: Presencial, Teléfono, Chat, Correo electrónico, Redes sociales y App móvil. Estados son valores de dominio controlados y no se crean libremente. Destinos físicos usan `sspectestablishment`; destinos lógicos se expresan mediante reglas/proceso. Responsables se referencian por `sspectuser`; no texto libre ni entidad `Responsible`.

---

# 9. Modelo lógico general

```text
sspectenterprise
  ├── LostItemReport ──1:N── ItemMatch ──N:1── FoundItem
  └── FoundItem ──1:N── RecoveryClaim
                         ├──1:N── OwnershipValidation ──1:N── OwnershipValidationAnswer
                         ├──1:N── ContactAttempt
                         └──0..1── ItemDelivery

FoundItem ──1:N── CustodyMovement
FoundItem ──1:N── ItemTransfer
FoundItem ──0..1── ItemDisposal
FoundItem ──0..1── DocumentDetail
FoundItem ──0..1── CardDetail
FoundItem ──1:N── FoundItemValidationReference
ValidationQuestion ──1:N── FoundItemValidationReference
ValidationQuestion ──1:N── OwnershipValidationAnswer

sspectestablishment ── custodio/destinos/locales de proceso
sspectuser ── actores responsables de operaciones y auditoría
LostFoundZone ── LostFoundEstablishmentZone ── sspectestablishment
FoundItem/LostItemReport/Delivery/Transfer/Disposal ── asociaciones propias ── sspectlffile
```

---

# 10. Cardinalidades aprobadas

| Origen | Relación | Destino |
|---|---|---|
| LostItemReport | 1:N | ItemMatch |
| FoundItem | 1:N | ItemMatch |
| FoundItem | 1:N | RecoveryClaim |
| RecoveryClaim | 1:N | OwnershipValidation |
| RecoveryClaim | 1:N | ContactAttempt |
| RecoveryClaim | 0..1 | ItemDelivery |
| FoundItem | 1:N | CustodyMovement |
| FoundItem | 1:N | ItemTransfer |
| FoundItem | 0..1 | ItemDisposal |
| FoundItem | 0..1 | DocumentDetail |
| FoundItem | 0..1 | CardDetail |
| FoundItem | 1:N | FoundItemValidationReference |
| ValidationQuestion | 1:N | FoundItemValidationReference |
| OwnershipValidation | 1:N | OwnershipValidationAnswer |
| ValidationQuestion | 1:N | OwnershipValidationAnswer |

`ItemMatch` en ambos sentidos implementa la relación N:M entre LostItemReport y FoundItem. Las entidades raíz/transaccionales listadas en §5 guardan `enterprise_id`; hijos lo derivan cuando corresponda.

---

# 11. Datos conceptuales de FoundItem

| Dato conceptual | Tratamiento aprobado |
|---|---|
| id / PK física | Interna; no es identificador visible funcional |
| itemCode | Generado, separado de PK; formato inicial `LF-F-AAAA-XXXXXX`, independiente por empresa/año |
| enterprise_id | FK lógica a empresa PEC; consistente con establecimiento |
| currentEstablishmentId | Custodio físico actual, distinto de ubicación histórica |
| tema/subtema | Catálogos PEC con valores propios Lost & Found |
| zona | FK lógica a LostFoundZone habilitada para el establecimiento |
| finderType | COLLABORATOR, CUSTOMER u OTHER |
| finderEmployeeCode | Código colaborador cuando aplique |
| finderName / finderLastName / finderPosition | Snapshot histórico de nombre, apellido y cargo |
| registeredByUserId | Usuario PEC que registró |
| estado | Dimensiones de dominio separadas y controladas |
| expirationDate / policyVersion | Valor calculado y versión CustodyPolicy aplicada |
| foundDate / createdAt / updatedAt | Fechas de negocio y sistema, según corresponda |

---

# 12. Datos conceptuales del reportante y Finder

`LostItemReport` conserva snapshot del reportante: tipo/número de identificación, nombres, apellidos, celular/teléfono, email y demás datos registrados para el proceso. No se crea tabla genérica de snapshots.

Finder permanece embebido conceptualmente en `FoundItem`, sin entidad Finder. Para colaborador se registra código, nombres, apellidos y cargo; nombre, apellido y cargo se conservan como snapshot del momento del hallazgo.

---

# 13. Zonas

`LostFoundZone` + `LostFoundEstablishmentZone` representan ubicaciones internas de cada local. Ejemplos: servicio al cliente, cajas, pasillos, estacionamiento, zona de comidas y entrada principal. El establecimiento solo habilita/usa sus zonas vinculadas. `FoundItem` guarda referencia a la zona interna elegida.

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

Entidades/capacidades candidatas:

- `FoundItem`;
- relación con empresa;
- relación con establecimiento;
- relación con usuario;
- asociación con archivos;
- catálogos mínimos;
- estado inicial;
- auditoría mínima de creación.

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

La aprobación `v1.0 APPROVED` de `PEC-LF-DATA-001` queda registrada con el cumplimiento de estos criterios:

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

**Versión:** `1.0.0`

**Aprobador:**

- Walter Molina

**Fecha:** `2026-09-30`

**Observación:**

Documento creado para revisión técnica del modelo lógico completo del módulo Lost & Found antes de iniciar implementación física.

La aprobación de este documento no implica crear todas las tablas inmediatamente. La implementación será incremental conforme al backlog y a las Specs funcionales aprobadas.

---

# 21. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.1.0 | 2026-09-30 | Mbrion / equipo PEC | Primer borrador del modelo lógico completo de Lost & Found. |
| 0.2.0 | 2026-09-30 | Mbrion / equipo PEC | Consolidación de decisiones arquitectónicas y de modelo aprobadas durante revisión funcional/técnica. El resumen §22 remite a las decisiones documentadas en las secciones del modelo. |
| 1.0.0 | 2026-09-30 | Walter Molina | Aprobación del modelo lógico Lost & Found por Walter Molina. |

---

# 22. Resumen de decisiones del modelo

Este resumen orienta la consulta del modelo lógico desarrollado en §§5–16. `LostItemReport` y `FoundItem` son conceptos independientes; las relaciones, entidades y cardinalidades se describen en §§6 y 10. Las referencias y reutilización PEC, archivos, zonas, catálogo, roles y configuración se describen en §§5, 7–8 y 13. Estados, auditoría, integridad y snapshots se describen en §§14–16.

Las políticas funcionales confirmadas de custodia y reportes se describen en §6.13 y §17. Los nombres físicos siguen siendo referencias conceptuales y no autorizan implementar tablas, Prisma ni migraciones. El documento está en estado **APPROVED**.

Pendientes reales detallados en §18: mecanismo técnico de firma; algoritmo/pesos/umbrales de matching; proveedor/detalles de transcripción; contrato técnico del servicio externo de datos del cliente; y detalle implementable de Specs funcionales posteriores.
