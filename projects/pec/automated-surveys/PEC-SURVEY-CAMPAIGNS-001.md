# Especificación: campañas automatizadas de encuestas

> Estado: borrador inicial. Mantener esta especificación como fuente de alcance durante Idea & Spec → Plan → Tasks → Execute → Validate.

## 1. Encabezado

| Campo | Valor |
|---|---|
| ID | PEC-SURVEY-CAMPAIGNS-001 |
| Título | Campañas automatizadas de encuestas |
| Versión | 0.4 |
| Autor | Equipo PEC |
| Fecha | 2026-10-15 |
| Work item / backlog | Sin referencia |

## 2. Problema

PEC permite crear encuestas y recibir respuestas, pero no automatiza la selección de clientes ni el envío periódico de invitaciones. El equipo necesita definir campañas que reutilicen encuestas existentes, seleccionen destinatarios mediante vistas corporativas y envíen invitaciones a través de Emblue.

## 3. Resultado esperado

Personal autorizado puede configurar, revisar, activar, pausar y consultar campañas asociadas a encuestas existentes. El sistema obtiene la audiencia desde vistas de divisiones, departamentos y clientes con correo, y ejecuta los envíos según fecha, hora y recurrencia configuradas.

## 4. Alcance

### Incluye

- Reutilizar encuestas existentes administradas con SurveyJS.
- Asociar una campaña con una encuesta y criterios de división/departamento.
- Resolver destinatarios a partir de vistas corporativas que serán entregadas.
- Programar ejecución única o recurrente.
- Enviar invitaciones mediante Emblue.
- Consultar estado y resultados operativos de ejecuciones y envíos.

### Fuera de alcance

- Crear un segundo editor de encuestas distinto de SurveyJS.
- Definir o modificar las vistas de clientes, divisiones y departamentos, responsabilidad del equipo que las proveerá.
- Confirmar métricas de apertura, clic o rebote hasta conocer los eventos/webhooks expuestos por Emblue.
- Envíos por canales distintos de Emblue.

### Supuestos

- Se entregarán vistas corporativas para divisiones, departamentos y clientes con correo. Sus nombres, columnas, motor y conexión aún no están disponibles.
- El equipo debe revisar y obtener la información técnica de Emblue; por ahora no se cuenta con el contrato/API para diseñar el adaptador de envío.
- Las ejecuciones serán procesadas por un planificador compartido en el API, no por un cron individual configurado para cada campaña.

## 5. Reglas de negocio

- **RB-01:** Una campaña debe referenciar una encuesta PEC existente y elegible para envío.
- **RB-02:** La audiencia se filtra por los criterios configurados de división y departamento y se resuelve desde las vistas acordadas.
- **RB-03:** Una ejecución no debe enviar dos veces al mismo destinatario por la misma campaña y período; la clave de idempotencia queda por definir con el contrato de Emblue.
- **RB-04:** Pausar o desactivar una campaña debe impedir nuevas ejecuciones, sin borrar su historial.
- **RB-05:** La invitación debe enlazar a la encuesta pública y permitir atribución de respuesta a la campaña sin exponer datos personales en la URL; el mecanismo está pendiente.
- **RB-06:** Los horarios y recurrencias deben evaluarse con una zona horaria explícita; queda pendiente confirmar si se fija America/Guayaquil.
- **RB-07:** La recurrencia debe admitir las frecuencias diaria, semanal, quincenal, mensual y trimestral.

## 6. Criterios de aceptación

### Positivos

- **CA-P01:** Dada una encuesta elegible, cuando un usuario autorizado crea una campaña con audiencia y calendario válidos desde la pestaña Campañas, entonces la campaña queda guardada y puede revisarse antes de activarla.
- **CA-P02:** Dada una campaña activa cuyo horario vence, cuando el planificador compartido la procesa, entonces se obtiene la audiencia y se solicita a Emblue el envío de las invitaciones.
- **CA-P03:** Dada una campaña recurrente diaria, semanal, quincenal, mensual o trimestral, cuando concluye una ejecución, entonces queda registrada y se calcula la siguiente fecha según su regla de recurrencia.
- **CA-P04:** Dada una ejecución con destinatarios válidos, cuando Emblue responde, entonces el estado de la ejecución permite distinguir éxito y fallo según el contrato acordado.
- **CA-P05:** Dada la pantalla de campañas automatizadas, cuando se abre la pestaña Inicio, entonces se muestran los conteos de Total, Activas, Programadas, Borradores, Suspendidas y Finalizadas, y debajo una tabla con campañas.
- **CA-P06:** Dada la pantalla de campañas automatizadas, cuando se abre la pestaña Campañas, entonces se presenta el apartado para crear una campaña.

### Negativos

- **CA-N01:** Dada una encuesta inexistente o no elegible, cuando se crea o activa una campaña, entonces la operación se rechaza con un error comprensible.
- **CA-N02:** Dada una campaña pausada, cuando llega la fecha prevista, entonces no se inicia un envío.
- **CA-N03:** Dada una falla de vistas o Emblue, cuando se procesa la campaña, entonces se registra el fallo sin perder el historial y se aplica la política de reintentos acordada.

### Casos de borde

- **CA-B01:** Dado un cliente sin correo válido o duplicado en las vistas, cuando se prepara la audiencia, entonces se excluye o deduplica según reglas pendientes y se deja un conteo auditable.
- **CA-B02:** Dada una campaña que vence durante una ejecución o se pausa mientras está en curso, cuando se procesa, entonces se respeta una política de concurrencia y cancelación aún por definir.

## 7. Datos personales

- **¿Aplica?** Sí.
- **Datos tratados:** Correo del cliente; identificador corporativo mínimo para deduplicación y atribución, por definir.
- **Finalidad y uso:** Seleccionar destinatarios, solicitar el envío y asociar una respuesta a la campaña.
- **Restricciones de acceso:** Pendiente definir roles autorizados para administrar campañas y consultar destinatarios.
- **Restricciones de logs:** No registrar correos, tokens ni payloads personales completos; usar identificadores técnicos y sanitización.
- **Restricciones de QA:** Utilizar datos sintéticos y credenciales de Emblue no productivas.

## 8. Riesgos técnicos

| Riesgo | Impacto | Mitigación / respuesta |
|---|---|---|
| Las vistas aún no están disponibles o su esquema cambia | No se puede construir la audiencia | Confirmar propietario, conexión, columnas, actualización y contrato antes de implementar el adaptador |
| Contrato de Emblue incompleto | Envíos no verificables o duplicados | Revisar autenticación, lote, límites, respuesta, idempotencia, reintentos y webhooks |
| Ejecución en varias réplicas del API | Envíos duplicados | Diseñar bloqueo/claim transaccional e idempotencia antes de habilitar campañas en producción |
| Correos y respuestas permiten identificar clientes | Exposición de datos personales | Minimizar datos, proteger accesos, evitar PII en logs y definir retención |

## 9. Evidencia esperada

- Pruebas unitarias de reglas, recurrencia, deduplicación, idempotencia y fallos de proveedores.
- Validación de migración y consultas con las vistas acordadas.
- Ejecución integrada con ambiente de pruebas de Emblue y audiencia sintética.
- Verificación autenticada de las pantallas y permisos de administración.
- Evidencia de que los logs sanitizan destinatarios y credenciales.

## 10. Dependencias

- Equipo de datos/corporativo: entregar nombres, columnas, conexión, permisos, actualización y semántica de vistas de divisiones, departamentos y clientes con correos. Aún no están disponibles.
- Equipo PEC responsable de Emblue: revisar y compartir contrato/API, credenciales por ambiente, límites, idempotencia, estados, reintentos, webhooks y mecanismo de bajas/supresión. Información pendiente de recopilar.
- Negocio: reglas exactas de audiencia, zona horaria, frecuencias y tratamiento de respuestas por cliente.

## 11. Modelo conceptual / datos

- **Campaña:** encuesta asociada, nombre, estado, filtros de audiencia, fecha de inicio, zona horaria, regla de recurrencia, próxima ejecución y auditoría.
- **Ejecución:** campaña, período/fecha programada, estado, conteos, referencia externa de Emblue, timestamps y error sanitizado.
- **Destinatario de ejecución:** identificador mínimo del cliente, estado de envío y referencia de correlación no sensible.
- Los nombres y tipos finales dependen de las vistas corporativas y del contrato de Emblue. No duplicar datos de clientes más allá de lo necesario para ejecutar y auditar.

## 12. Estados y transiciones

| Estado inicial | Evento / condición | Estado final | Restricciones |
|---|---|---|---|
| Borrador | Activación válida | Activa | Encuesta y audiencia válidas |
| Activa | Pausa solicitada | Pausada | Usuario autorizado |
| Pausada | Reanudación | Activa | Usuario autorizado |
| Activa | Ejecución programada | Procesando | Claim único por campaña/período |
| Procesando | Resultado final | Completada / Fallida | Según respuesta de Emblue |
| Activa / Pausada | Desactivación | Inactiva | Conservar historial |

## 13. Roles y permisos

| Rol | Puede | No puede / restricciones |
|---|---|---|
| Administrador de campañas | Crear, editar, activar, pausar y consultar campañas | Rol concreto pendiente de mapear a permisos PEC |
| Consulta | Ver campañas y ejecuciones permitidas | No cambiar configuración ni lanzar envíos |

## 14. Integraciones

| Sistema | Propósito | Dirección / mecanismo | Manejo de fallos |
|---|---|---|---|
| Vistas corporativas | Obtener divisiones, departamentos y clientes con correo | Lectura desde conexión por confirmar | Registrar fallo sanitizado; no iniciar envío con audiencia incompleta |
| Emblue | Entregar las invitaciones | API/contrato por entregar | Reintento, límite de lote, idempotencia y clasificación por definir |
| PEC Surveys | Obtener metadatos y URL pública de encuestas existentes | API interna `surveys` | Validar vigencia y disponibilidad antes de activar |

## 15. Contratos / API

Pendiente definir recursos y rutas después de confirmar roles, esquema de audiencia y estados de Emblue. Operaciones previstas: administrar campañas, validar/contar audiencia, activar/pausar, consultar ejecuciones y solicitar reintento autorizado.

## 16. Pantallas afectadas

- Nuevo apartado de campañas automatizadas dentro de Encuestas, en `pec-ui`.
- El apartado tendrá dos pestañas internas:
  - **Inicio:** dashboard inicial con un bloque **Resumen de Campañas** y tarjetas para **Total**, **Activas**, **Programadas**, **Borradores**, **Suspendidas** y **Finalizadas**. Debajo se mostrará una tabla para enlistar las campañas.
  - **Campañas:** apartado para crear campañas. El formulario deberá asociar una encuesta existente y configurar audiencia y programación, de acuerdo con las reglas confirmadas y los contratos pendientes.
- La referencia visual entregada muestra seis tarjetas de resumen en una fila, cada una con conteo, icono y color diferenciado. La distribución deberá adaptarse a pantallas angostas.
- Los campos/columnas de la tabla, filtros, acciones por fila y detalles del formulario quedan pendientes de definición.
- Se reutiliza el módulo existente de encuestas y SurveyJS; no se modifica el flujo público hasta definir la atribución segura de respuestas.

## 17. Decisiones técnicas

| Decisión | Motivo | Consecuencias |
|---|---|---|
| Usar el planificador global existente de NestJS para buscar ejecuciones vencidas | El API ya inicializa `@nestjs/schedule`; se evita un cron por campaña | La ejecución debe ser idempotente y segura en múltiples réplicas |
| Mantener campañas como entidad separada de encuestas y respuestas | Una encuesta puede reutilizarse sin mezclar su definición con calendario y envíos | Requiere persistir la relación campaña-ejecución y migrar el esquema |

## 18. Preguntas pendientes

- [ ] **Pendiente de entrega:** nombres/columnas, motor, conexión, permisos y frecuencia de actualización de las vistas — Responsable: equipo de datos.
- [ ] **Pendiente de revisión por el equipo PEC:** contrato Emblue, autenticación, tamaños de lote, idempotencia, eventos de entrega, reintentos y bajas — Responsable: integración Emblue.
- [ ] Zona horaria, fechas de fin, día/hora de ejecución para cada frecuencia y comportamiento ante ejecuciones omitidas — Responsable: negocio.
- [ ] Clave de deduplicación de clientes y consentimiento/supresión de comunicaciones — Responsable: negocio/datos.
- [ ] Mecanismo seguro para identificar campaña en respuestas, duración del enlace y restricciones de acceso — Responsable: negocio/seguridad.
- [ ] Roles PEC que administrarán campañas — Responsable: producto/seguridad.
- [ ] Volumen máximo de audiencia y ventana de envío — Responsable: negocio/Emblue.

## 19. Estado de aprobación

- **Estado:** Borrador.
- **Aprobadores:** Pendiente.
- **Fecha y observaciones:** 2026-10-15 — Alcance y frecuencias confirmados por negocio; vistas y contrato Emblue aún no disponibles; reglas operativas pendientes.

## 20. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| 0.2 | 2026-10-15 | Equipo PEC | Se confirmaron las frecuencias diaria, semanal, quincenal, mensual y trimestral. |
| 0.3 | 2026-10-15 | Equipo PEC | Se aclaró que los nombres/esquemas de vistas y la documentación de Emblue todavía deben recopilarse. |
| 0.4 | 2026-10-15 | Equipo PEC | Se definió el layout con pestañas Inicio y Campañas, resumen visual de seis estados y tabla de campañas en Inicio. |
