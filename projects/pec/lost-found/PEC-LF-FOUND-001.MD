# PEC-LF-FOUND-001

## Registro de artículo encontrado

### 1. Encabezado

| Campo | Valor |
|---|---|
| ID | PEC-LF-FOUND-001 |
| Título | Registro de artículo encontrado |
| Versión | 0.1.0 |
| Estado | BORRADOR |
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
- fecha y lugar del hallazgo;
- archivos/fotografías;
- usuario que registra;
- política de custodia aplicada;
- auditoría de creación.

---

## 3. Alcance funcional

Incluye:
- creación de FoundItem;
- generación automática de itemCode;
- selección de tema y subtema;
- selección de establecimiento;
- selección de zona interna habilitada para el establecimiento;
- registro del lugar donde fue encontrado;
- fecha de hallazgo;
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

---

## 5. Flujo principal

1. Usuario ingresa a “Registrar artículo encontrado”.
2. Sistema obtiene empresa, usuario y establecimientos permitidos.
3. Usuario selecciona establecimiento.
4. Sistema carga zonas internas habilitadas.
5. Usuario ingresa datos del artículo.
6. Usuario selecciona tema/subtema.
7. Usuario registra datos del finder.
8. Usuario adjunta fotografías/archivos.
9. Sistema valida campos obligatorios.
10. Sistema genera itemCode.
11. Sistema crea FoundItem.
12. Sistema asigna currentEstablishmentId.
13. Sistema aplica CustodyPolicy vigente.
14. Sistema calcula expirationDate cuando aplique.
15. Sistema registra archivos.
16. Sistema genera evento LostFoundAudit.
17. Sistema devuelve confirmación del registro.

---

## 6. Datos del formulario

### 6.1 Artículo

- nombre
- descripción
- tema
- subtema
- lugar encontrado
- fecha encontrada
- establecimiento
- zona
- comentario

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

Si CUSTOMER / OTHER:
- finderName
- finderLastName
- información adicional solo si negocio la solicita

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
itemCode se genera automáticamente y no utiliza la PK física.

Formato inicial:
LF-F-AAAA-XXXXXX

RB-FND-03
currentEstablishmentId representa al establecimiento que posee físicamente el artículo.

RB-FND-04
La zona seleccionada debe estar habilitada para el establecimiento seleccionado.

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
Los cambios futuros de CustodyPolicy no deben modificar automáticamente la fecha de vencimiento ya aplicada.

---

## 8. Validaciones

- nombre obligatorio
- tema obligatorio
- subtema obligatorio
- establecimiento obligatorio
- zona obligatoria
- lugar encontrado obligatorio
- fecha encontrada obligatoria
- finderType obligatorio
- si finderType = COLLABORATOR:
  - código colaborador obligatorio
  - nombres obligatorios
  - apellidos obligatorios
  - cargo obligatorio
- zona debe pertenecer al establecimiento
- usuario debe estar autorizado
- enterprise_id debe coincidir con el establecimiento
- fecha encontrada no puede ser futura, salvo decisión funcional posterior

---

## 9. Modelo lógico involucrado

FoundItem

Relaciones:
- N:1 sspectenterprise
- N:1 sspectestablishment
- N:1 sspectuser
- N:1 LostFoundZone
- 1:N FoundItemFile
- referencia a CustodyPolicyVersion
- auditoría mediante LostFoundAudit

Campos conceptuales:
- id
- itemCode
- enterprise_id
- currentEstablishmentId
- name
- description
- topicId
- subtopicId
- foundPlace
- foundDate
- zoneId
- finderType
- finderEmployeeCode
- finderName
- finderLastName
- finderPosition
- registeredByUserId
- status
- expirationDate
- policyVersionId
- comment
- createdAt
- updatedAt

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

Tipos iniciales sugeridos:
- PHOTO
- EVIDENCE

Cada archivo debe:
- pertenecer al dominio Lost & Found;
- asociarse al FoundItem;
- almacenarse mediante GCS;
- conservar metadata en sspectlffile.

---

## 13. Criterios de aceptación

### Positivos

CA-FND-01
Dado un usuario autorizado, cuando registra todos los datos válidos, entonces se crea FoundItem con itemCode automático y estado REGISTERED.

CA-FND-02
Dado un establecimiento, cuando se carga el formulario, entonces solo se muestran zonas habilitadas para ese establecimiento.

CA-FND-03
Dado finderType = COLLABORATOR, cuando se selecciona un colaborador, entonces se guardan código, nombre, apellido y cargo como snapshot.

CA-FND-04
Dado un archivo válido, cuando se adjunta al registro, entonces se almacena mediante StorageService/GCS y se asocia mediante metadata Lost & Found.

CA-FND-05
Dado un registro exitoso, entonces se crea auditoría FOUND_ITEM_CREATED.

CA-FND-06
Dado que existe CustodyPolicy aplicable, entonces se conserva la versión aplicada y la expirationDate correspondiente.

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

---

## 14. Transacción

La operación de creación debería mantener consistencia entre:
- FoundItem
- archivos/asociaciones
- aplicación de política
- auditoría

Si una parte crítica falla, no debe quedar un registro parcialmente válido.

---

## 15. Pendientes de esta Spec

- definir campos exactos obligatorios/opcionales de CUSTOMER / OTHER;
- confirmar límites y tipos de archivos;
- confirmar si foundDate incluye hora;
- confirmar si la zona será obligatoria en todos los establecimientos;
- definir contrato exacto del autocompletado de colaborador;
- definir DTOs/endpoints en Plan técnico.

---

## 16. Estado de aprobación

Estado: BORRADOR
Versión: 0.1.0
Aprobador: Walter Molina