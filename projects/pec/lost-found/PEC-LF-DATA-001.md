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
| Versión | `0.1.0` |
| Estado | `BORRADOR` |
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

**PROPUESTA / INFERIDO**

La relación entre un reporte de pérdida y un artículo encontrado se representa mediante una entidad de asociación:

`ItemMatch`

No se recomienda almacenar directamente:

`FoundItem.lostItemReportId`

porque un artículo podría relacionarse con múltiples reportes candidatos y un reporte podría tener múltiples artículos candidatos.

---

### DATA-P03 — Custodia

**PROPUESTA / INFERIDO**

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

# 5. Entidades existentes PEC a reutilizar

## 5.1 Empresa

Modelo existente:

`sspectenterprise`

PK actual:

`sspecrpkent`

Uso propuesto:

- aislamiento de datos;
- contexto empresarial;
- catálogos;
- registros Lost & Found.

### Estado

**CONFIRMADO** como entidad existente.

### Pendiente

Determinar si todas las entidades Lost & Found necesitan `enterprise_id` explícito o si algunas pueden derivarlo mediante establecimiento.

---

## 5.2 Establecimiento

Modelo existente:

`sspectestablishment`

PK actual:

`sspecrpkest`

Uso propuesto:

- local donde se registra;
- local que posee físicamente el artículo;
- origen/destino de movimientos;
- local de entrega.

### Estado

**CONFIRMADO** como entidad existente.

---

## 5.3 Usuario PEC

Modelo existente:

`sspectuser`

PK actual:

`sspecrpkuse`

Uso propuesto:

- usuario que registra;
- usuario que valida;
- usuario que contacta;
- responsable de custodia;
- responsable de entrega;
- responsable de destrucción;
- actor de auditoría.

### Estado

**CONFIRMADO** como entidad existente.

---

## 5.4 Archivo

Modelo existente:

`sspectfile`

Uso propuesto:

- fotografías;
- documentos;
- evidencias;
- actas;
- evidencia de destrucción;
- evidencia de desecho.

### Estado

**CONFIRMADO** como infraestructura existente.

### Restricción

No reutilizar asociaciones específicas de Quejas.

Lost & Found requiere asociaciones propias.

---

# 6. Entidades nuevas propuestas

## 6.1 `LostItemReport`

Tabla física candidata:

`sspectlostitemreport`

PK candidata:

`sspecrpklir`

### Responsabilidad

Representar una declaración de pérdida realizada por una persona.

### Relaciones propuestas

- N:1 `sspectenterprise`
- N:1 `sspectestablishment` como local de registro/consulta, si aplica.
- 1:N `ItemMatch`
- 1:N archivos, si el reporte permite adjuntos.

### Datos conceptuales esperados

- ID Encuentra;
- descripción;
- tema;
- subtema;
- lugar aproximado;
- fecha aproximada;
- local donde consulta;
- tipo de identificación;
- número de identificación;
- nombres;
- apellidos;
- celular;
- teléfono;
- email;
- canal;
- fecha/hora;
- usuario que registra.

### Estado

**PROPUESTA / INFERIDO**

---

## 6.2 `FoundItem`

Tabla física candidata:

`sspectfounditem`

PK candidata:

`sspecrpkfit`

### Responsabilidad

Representar el artículo físico encontrado y sujeto a custodia.

### Relaciones propuestas

- N:1 `sspectenterprise`
- N:1 `sspectestablishment`
- N:1 `sspectuser` como creador
- 1:N `ItemMatch`
- 1:N `CustodyMovement`
- 1:N archivos
- 0..1 / 1:N `ItemDelivery`
- 0..1 / 1:N `ItemDisposal`

### Campos conceptuales candidatos

- id;
- código de artículo;
- nombre;
- tema;
- subtema;
- lugar encontrado;
- fecha encontrada;
- finderType;
- código de colaborador;
- nombres de quien encontró;
- apellidos;
- cargo;
- establecimiento actual;
- zona;
- usuario creador;
- estado actual;
- fecha de creación;
- fecha de modificación.

### Estado inicial confirmado

`REGISTRADO`

### Estado

**PROPUESTA / INFERIDO**

---

## 6.3 `ItemMatch`

Tabla física candidata:

`sspectitemmatch`

PK candidata:

`sspecrpkmat`

### Responsabilidad

Representar una posible coincidencia entre:

`LostItemReport`

y

`FoundItem`

### Relaciones

- N:1 `LostItemReport`
- N:1 `FoundItem`
- N:1 usuario revisor, si aplica.

### Cardinalidad conceptual

`LostItemReport 1:N ItemMatch`

`FoundItem 1:N ItemMatch`

Esto permite conceptualmente una relación N:M entre reportes y artículos.

### Datos candidatos

- score / nivel de coincidencia;
- origen del match;
- estado;
- creado por;
- revisado por;
- fecha creación;
- fecha revisión;
- observaciones.

### Estado

**PROPUESTA / INFERIDO**

---

## 6.4 `OwnershipValidation`

Tabla física candidata:

`sspectownershipvalidation`

PK candidata:

`sspecrpkval`

### Responsabilidad

Registrar el proceso mediante el cual se intenta confirmar que una persona es propietaria de un artículo.

### Relación propuesta

Preferencia actual:

N:1 `ItemMatch`

### Pendiente

Determinar si debe relacionarse:

- con `ItemMatch`;
- con `FoundItem`;
- con una futura entidad `Claim`.

### Restricción confirmada

Las respuestas correctas no deben mostrarse previamente al cliente.

### Estado

**PROPUESTA / INFERIDO**

---

## 6.5 `ContactAttempt`

Tabla física candidata:

`sspectlfcontactattempt`

PK candidata:

`sspecrpkcta`

### Responsabilidad

Registrar intentos y resultados de contacto con el propietario/reportante.

### Relaciones candidatas

- N:1 `ItemMatch`
- N:1 `LostItemReport`
- N:1 `sspectuser`

### Pendiente

Definir si el contacto pertenece:

- a una coincidencia;
- al reporte;
- al artículo;
- o a una futura entidad de reclamación.

### Resultados funcionales confirmados

- Contactado
- No contesta
- Número incorrecto
- Se acercará al local

---

## 6.6 `CustodyMovement`

Tabla física candidata:

`sspectcustodymovement`

PK candidata:

`sspecrpkcsm`

### Responsabilidad

Registrar cualquier cambio relevante en la custodia física del artículo.

### Relaciones

- N:1 `FoundItem`
- N:1 establecimiento origen
- N:1 establecimiento destino
- N:1 usuario responsable

### Tipos conceptuales candidatos

- RECEIVED
- TRANSFER_REQUESTED
- TRANSFERRED
- RECEIVED_AT_GCSS
- DELIVERED
- DISPOSED

### Principio

`CustodyMovement` representa el historial.

`FoundItem.currentEstablishmentId` representa el custodio actual.

### Estado

**PROPUESTA / INFERIDO**

---

## 6.7 `ItemDelivery`

Tabla física candidata:

`sspectitemdelivery`

PK candidata:

`sspecrpkdel`

### Responsabilidad

Registrar el proceso de entrega del artículo.

### Relaciones propuestas

- N:1 `FoundItem`
- N:1 `ItemMatch`, cuando corresponda
- N:1 establecimiento
- N:1 usuario responsable
- 1:N archivos/evidencias

### Datos esperados

- cliente;
- responsable;
- fecha;
- hora;
- observaciones;
- firma cliente;
- firma colaborador;
- acta.

### Restricción confirmada

No puede cerrarse la entrega sin acta firmada.

---

## 6.8 `ItemDisposal`

Tabla física candidata:

`sspectitemdisposal`

PK candidata:

`sspecrpkdsp`

### Responsabilidad

Registrar destrucción o desecho final del artículo.

### Relaciones

- N:1 `FoundItem`
- N:1 establecimiento
- N:1 usuario responsable
- 1:N evidencias

### Casos

- destrucción de documentos;
- desecho de alimentos;
- otras disposiciones futuras.

### Estado

**PROPUESTA / INFERIDO**

---

## 6.9 `LostFoundAudit`

Tabla física candidata:

`sspectlfaudit`

PK candidata:

`sspecrpkaud`

### Responsabilidad

Mantener auditoría funcional durable de eventos relevantes.

### Debe registrar como mínimo

- actor;
- fecha/hora;
- establecimiento;
- acción;
- entidad afectada;
- valor anterior;
- valor posterior.

### Diseño pendiente

Determinar si se utiliza:

#### Opción A

Tabla genérica:

- entityType
- entityId

Ventaja:
flexible.

Riesgo:
no existe FK fuerte hacia todas las entidades.

#### Opción B

Auditoría por dominio/eventos con FKs específicas.

Ventaja:
integridad relacional.

Riesgo:
más tablas/estructura.

### Estado

**PENDIENTE DE DEFINICIÓN**

---

# 7. Asociaciones con archivos

No se recomienda agregar referencias Lost & Found dentro del campo actual de Quejas.

Se plantean dos alternativas.

## Alternativa A — tablas de asociación específicas

Ejemplo:

`sspectfounditemfile`

`sspectdeliveryfile`

`sspectdisposalfile`

### Ventajas

- integridad referencial;
- semántica clara;
- permisos específicos.

---

## Alternativa B — asociación genérica

Ejemplo conceptual:

`LostFoundFileLink`

- fileId;
- entityType;
- entityId;
- fileType.

### Ventaja

menos tablas.

### Riesgo

relación polimórfica sin FK fuerte.

### Estado

**PENDIENTE DE DEFINICIÓN**

---

# 8. Catálogos candidatos

PEC tiene infraestructura de catálogos.

No está aprobado todavía reutilizar los valores actuales.

Los siguientes catálogos son funcionalmente requeridos:

- temas;
- subtemas;
- tipos de documento;
- tipos de tarjeta;
- canales;
- preguntas de validación;
- estados;
- destinos;
- responsables.

---

## Opción A — reutilizar infraestructura existente

Usar `sspectcatalog` o estructuras actuales.

### Ventaja

menos tablas nuevas.

### Riesgo

mezclar semánticas de distintos módulos.

---

## Opción B — catálogos propios

Tablas candidatas:

`sspectlftheme`

`sspectlfsubtheme`

`sspectlfdocumenttype`

`sspectlfcardtype`

`sspectlfvalidationquestion`

### Estado

**PENDIENTE DE DEFINICIÓN**

---

# 9. Modelo lógico general

```text
                            sspectenterprise
                                   |
                    +--------------+--------------+
                    |                             |
                    v                             v
          sspectlostitemreport             sspectfounditem
                    |                             |
                    |                             |
                    +----------+       +----------+
                               v       v
                           sspectitemmatch
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
      sspectownershipvalidation      sspectlfcontactattempt


sspectfounditem
      |
      +------ 1:N ------> sspectcustodymovement
      |
      +------ 1:N ------> archivos / evidencias
      |
      +------ 0..N -----> sspectitemdelivery
      |
      +------ 0..N -----> sspectitemdisposal


sspectestablishment
      |
      +------ custodio actual
      +------ origen/destino movimientos
      +------ local de entrega
      +------ local de disposición


sspectuser
      |
      +------ creador
      +------ revisor
      +------ responsable contacto
      +------ responsable custodia
      +------ responsable entrega
      +------ responsable disposición

# 10. Cardinalidades propuestas

| Origen | Relación | Destino |
|---|---|---|
| Enterprise | 1:N | LostItemReport |
| Enterprise | 1:N | FoundItem |
| LostItemReport | 1:N | ItemMatch |
| FoundItem | 1:N | ItemMatch |
| ItemMatch | 1:N | OwnershipValidation |
| ItemMatch / Report | 1:N | ContactAttempt |
| FoundItem | 1:N | CustodyMovement |
| FoundItem | 0..N | ItemDelivery |
| FoundItem | 0..N | ItemDisposal |
| FoundItem | 1:N | FileLink |
| Delivery | 1:N | FileLink |
| Disposal | 1:N | FileLink |
| Establishment | 1:N | FoundItem |
| User | 1:N | Operaciones de dominio |

> Todas las cardinalidades permanecen sujetas a revisión hasta aprobación de negocio y revisión técnica.

---

# 11. Modelo actual del artículo encontrado

Para Sprint 1, la entidad raíz esperada es:

`FoundItem`

## Campos candidatos

| Campo | Obligatorio | Origen |
|---|---:|---|
| id | Sí | Sistema |
| code | Sí | Sistema |
| enterpriseId | Pendiente | Contexto |
| currentEstablishmentId | Sí | Contexto / local |
| name | Sí | Usuario |
| topicId | Sí | Catálogo |
| subtopicId | Sí | Catálogo |
| foundPlace | Sí | Usuario |
| foundDate | Sí | Usuario |
| finderType | Sí | Usuario |
| finderEmployeeCode | Condicional | Usuario |
| finderName | Condicional | Sistema / usuario |
| finderLastName | Condicional | Sistema / usuario |
| finderPosition | Condicional | Sistema |
| registeredByUserId | Sí | Sesión |
| status | Sí | Sistema |
| createdAt | Sí | Sistema |
| updatedAt | Sí | Sistema |

## Estado inicial

**CONFIRMADO**

`REGISTERED`

## Decisiones pendientes

- formato y generación de `code`;
- si `enterpriseId` se persiste o se deriva;
- si `currentEstablishmentId` representa custodio actual o local de registro;
- si `topicId` y `subtopicId` reutilizan catálogos existentes;
- si los datos del colaborador se almacenan como snapshot;
- si `foundDate` incluye hora;
- estrategia física de `status`.

---

# 12. Modelo de persona que encontró el artículo

Se consideran dos alternativas.

## Alternativa A — campos en `FoundItem`

Campos candidatos:

- `finderType`;
- `finderEmployeeCode`;
- `finderName`;
- `finderLastName`;
- `finderPosition`.

### Ventajas

- implementación más simple;
- suficiente para el primer flujo;
- evita crear una entidad sin necesidad confirmada.

### Riesgos

- duplicación si el concepto crece;
- menor normalización;
- dificultad futura si una misma persona se relaciona con múltiples registros.

---

## Alternativa B — entidad `Finder`

Tabla nueva candidata:

`sspectlffinder`

### Posibles relaciones

- 1:N con `FoundItem`;
- posible vínculo con `sspectuser` cuando sea colaborador;
- datos propios cuando sea cliente u otro.

### Ventajas

- mayor normalización;
- facilita reutilización futura;
- separación clara entre persona y artículo.

### Riesgos

- mayor complejidad;
- posible sobrediseño si el concepto no necesita vida propia.

## Propuesta actual

**PROPUESTA / INFERIDO**

Usar inicialmente campos dentro de `FoundItem`, salvo que la revisión con Walter y Roberto determine que `Finder` debe ser una entidad reutilizable.

---

# 13. Zonas

En PEC se detectó `id_zona` como atributo, pero no un dominio completo `Zone`.

Se consideran las siguientes alternativas:

## Opción A — derivar zona

Derivar la zona desde:

`Establishment → Region → id_zona`

### Ventaja

Evita duplicar información.

### Riesgo

Depende de que la relación existente sea confiable y suficiente para el módulo.

---

## Opción B — almacenar `zoneId`

Persistir una referencia explícita a zona si existe una fuente oficial.

### Ventaja

Consulta directa y trazabilidad.

### Riesgo

Duplicación o inconsistencia si el establecimiento cambia.

---

## Opción C — guardar snapshot

Guardar el código o descripción de zona al momento del registro.

### Ventaja

Conserva contexto histórico.

### Riesgo

Duplica datos y requiere reglas de actualización.

## Estado

**PENDIENTE DE DEFINICIÓN**

---

# 14. Estados

No se recomienda reutilizar estados de Quejas.

El modelo debe contemplar dimensiones distintas para:

- artículo encontrado;
- reporte perdido;
- custodia;
- matching;
- validación;
- contacto;
- entrega;
- vencimiento.

## Estado inicial de artículo encontrado

**CONFIRMADO**

`REGISTERED`

## Implementación física de estados

**PENDIENTE DE DEFINICIÓN**

Opciones:

- enum Prisma;
- catálogo;
- tabla de estados;
- combinación según dimensión.

## Principio arquitectónico

**PROPUESTA / INFERIDO**

No utilizar un único campo de estado para representar simultáneamente:

- custodia;
- matching;
- contacto;
- validación;
- entrega.

---

# 15. Auditoría y trazabilidad

Las siguientes operaciones deben considerarse auditables:

- creación de reporte perdido;
- registro de artículo encontrado;
- edición;
- creación de coincidencia;
- confirmación o descarte de coincidencia;
- validación de propiedad;
- intento y resultado de contacto;
- cambio de custodia;
- transferencia;
- recepción en GCSS;
- entrega;
- destrucción;
- desecho;
- cambios sensibles de datos.

## Campos mínimos conceptuales

- `userId`;
- `establishmentId`;
- `timestamp`;
- `eventType`;
- `entityType`;
- `entityId`;
- `previousValue`;
- `newValue`;
- `observations`;
- `evidenceReference`, cuando aplique.

## Regla confirmada

La auditoría debe permitir conocer:

- quién realizó la acción;
- cuándo;
- desde qué local;
- qué cambió.

## Estado

**PENDIENTE DE DEFINICIÓN**

Falta decidir el mecanismo físico definitivo de auditoría.

---

# 16. Reglas de integridad propuestas

## RI-01

Un `FoundItem` debe estar asociado a un establecimiento custodio actual mientras permanezca bajo custodia de PEC.

## RI-02

Un `CustodyMovement` no puede existir sin un `FoundItem`.

## RI-03

Las operaciones históricas de custodia, entrega, disposición y auditoría no deberían eliminarse físicamente sin una política explícita de retención.

## RI-04

Una entrega no puede cerrarse sin acta firmada.

## RI-05

No se debe mostrar el número completo de una tarjeta.

Solo deben mostrarse:

- primeros 6 dígitos;
- últimos 4 dígitos.

## RI-06

Las respuestas correctas de validación de propiedad nunca deben mostrarse al cliente antes de que este responda.

## RI-07

Solo el local que posee físicamente el artículo puede modificar su custodia.

## RI-08

Una coincidencia no implica validación de propiedad.

## RI-09

Un `LostItemReport` puede existir sin que exista un `FoundItem` asociado.

## RI-10

Un `FoundItem` puede existir sin que exista un `LostItemReport` asociado.

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

# 18. Decisiones pendientes para revisión Walter + Roberto

1. ¿Todas las entidades principales deben llevar `enterprise_id`?
2. ¿`FoundItem` mantiene `currentEstablishmentId` además del historial `CustodyMovement`?
3. ¿`ItemTransfer` será entidad propia o un tipo de `CustodyMovement`?
4. ¿`ItemDelivery` admite múltiples intentos o registros por artículo?
5. ¿`ItemDisposal` representa un único cierre final?
6. ¿Cómo se representa GCSS dentro del modelo?
7. ¿`Finder` se mantiene dentro de `FoundItem` o se convierte en entidad?
8. ¿Cómo se relacionan archivos con las entidades Lost & Found?
9. ¿Tema/subtema reutilizan `sspectcatalog` o se crean catálogos propios?
10. ¿Estados serán enum, catálogo, tabla o combinación?
11. ¿Zona se deriva o se persiste?
12. ¿`LostFoundAudit` será genérica o específica por dominio?
13. ¿El reportante puede vincularse opcionalmente a `sspectuser`?
14. ¿`OwnershipValidation` pertenece a `ItemMatch` o a una futura entidad de reclamación?
15. ¿`ContactAttempt` pertenece al reporte, al match o a una reclamación?
16. ¿Documentos y tarjetas son subtipo de `FoundItem`, categoría o detalle especializado?
17. ¿Qué estrategia de borrado aplica a cada FK?
18. ¿Qué convención exacta de PK física debe seguir el módulo?
19. ¿Cómo se genera el código del artículo encontrado?
20. ¿Cómo se genera el `ID Encuentra`?
21. ¿Qué campos requieren snapshot histórico aunque exista una FK?
22. ¿El local donde se registra un reporte perdido es distinto del local donde se perdió el objeto?
23. ¿El local actual del artículo debe conservarse también en `FoundItem` para consulta rápida?
24. ¿Cómo se modelan destinos que no sean establecimientos PEC?
25. ¿Qué entidades requieren control de versión/concurrencia?

---

# 19. Criterios de aprobación

`PEC-LF-DATA-001` podrá pasar a `v1.0 APPROVED` cuando:

- [ ] `PEC-LF-ARCH-001` esté aprobado.
- [ ] Las entidades principales estén acordadas.
- [ ] Las relaciones principales estén acordadas.
- [ ] Las cardinalidades críticas estén acordadas.
- [ ] Las tablas PEC reutilizadas estén identificadas.
- [ ] La estrategia de archivos esté acordada.
- [ ] La estrategia de auditoría esté acordada.
- [ ] La estrategia de custodia esté acordada.
- [ ] La estrategia de catálogos esté acordada.
- [ ] La estrategia de estados esté acordada.
- [ ] La representación de GCSS esté acordada.
- [ ] Las decisiones bloqueantes de Sprint 1 estén resueltas.
- [ ] Walter y Roberto hayan revisado el modelo.
- [ ] Las discrepancias con `PEC-LF-ARCH-001` estén resueltas.

---

# 20. Estado de aprobación

**Estado:** `BORRADOR`

**Versión:** `0.1.0`

**Aprobadores propuestos:**

- Walter Molina
- Roberto — revisión técnica
- Responsable funcional / producto — si corresponde

**Fecha:** `2026-09-30`

**Observación:**

Documento creado para revisión técnica del modelo lógico completo del módulo Lost & Found antes de iniciar implementación física.

La aprobación de este documento no implica crear todas las tablas inmediatamente. La implementación será incremental conforme al backlog y a las Specs funcionales aprobadas.

---

# 21. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.1.0 | 2026-09-30 | Mbrion / equipo PEC | Primer borrador del modelo lógico completo de Lost & Found. |