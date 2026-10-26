# PEC — Plan maestro de implementación autónoma de Campaign

## Estado de avance (2026-10-26)

- Fases A–C: catálogos, segmentación, vista previa y borradores están implementados; la audiencia todavía usa el lote PostgreSQL provisional.
- Fase D: simulación sin efectos y despacho manual están implementados. El usuario confirmó que recibió la invitación, abrió la encuesta pública y pudo responderla usando la ruta `/campaign-invitation/:token` de `pec-public-surveys-web`. El código guarda la respuesta en `sspectresult` y consume el token/correlación en una transacción; falta comprobar por lectura de base de datos que el resultado de esta ejecución quedó persistido con la correlación esperada.
- El envío manual usa el transporte emBlue configurado y un cuerpo de invitación construido en código. La entrega de prueba fue confirmada por el usuario; falta contrato aprobado de plantilla Campaign, variables y enlace para considerarlo listo para operación formal.
- Trazabilidad preparada: claves de ejecución y unicidad del recipient por campaña/origen/factura; la concurrencia real en PostgreSQL y recuperación tras fallos externos aún necesitan validación operativa.
- Programación: `CampaignSchedulerService` materializa ejecuciones mediante `runOnce`, pero no hay cron registrado ni despacho automático. Schedule/Resume siguen bloqueados por la fuente de audiencia BI definitiva, su contrato, y el contrato aprobado de plantilla.
- Interfaz final pendiente: el detalle aún no presenta acciones de Programar/Suspender/Reanudar aunque existen endpoints de lifecycle. Antes de mostrarlas, el backend debe incluir una condición verificable de worker activo además de audiencia BI/plantilla; no basta con que el preflight de datos esté listo.
- El usuario indicó que las tablas BI definitivas no estarán disponibles durante al menos un mes. Hasta recibirlas, usar el lote provisional solo en entorno de desarrollo con destinatarios controlados; no presentarlo como fuente real BI ni habilitar cron.
- UX de Campaign: fechas visibles en día/mes/año con calendario, limpieza de fechas y avisos/confirmaciones con los popups compartidos de PEC. El detalle distingue ahora simulación sin efectos, envío manual inmediato y programación automática aún deshabilitada; los estados de ejecución se muestran en español y la fuente provisional se identifica sin exponer su código interno.
- La confirmación de envío manual queda ligada a la fecha de evaluación, fuente, snapshot/lote y total de facturas contactables. El backend recalcula y compara esos cuatro valores antes de crear la ejecución; además rechaza una fecha distinta al día actual de la zona horaria de la campaña, ya que el envío inmediato evalúa la audiencia para hoy.
- El envío manual de respaldo está disponible para campañas en borrador, programadas y activas, previa confirmación y revalidación de la versión guardada. En el detalle, la vista previa y el respaldo son las acciones operativas visibles; la proyección sin envío quedó como herramienta secundaria opcional.
- Al abrir el detalle de una campaña con División y Departamento, la vista previa se consulta automáticamente para la fecha actual de la zona horaria configurada. La interfaz deja la proyección sin envío plegada como análisis opcional y diferencia el calendario guardado del proceso automático aún no habilitado.
- Simplificación del detalle: se quitaron del recorrido principal la fecha arbitraria de evaluación, la proyección de invitaciones y la tabla de identificadores de cliente/factura. Se muestran métricas agregadas, solo filtros seleccionados con etiquetas legibles, fecha/frecuencia aplicable y el historial plegado.
- El usuario confirmó que recibió la invitación, abrió `/campaign-invitation/:token` en el frontend público y completó la encuesta. Esto verifica el flujo funcional observado, pero la persistencia y auditoría del token/respuesta todavía requieren lectura autorizada de base de datos.
- El usuario ejecutó manualmente en Neon el SQL del rango de compras. La salida compartida confirmó columnas y constraint, pero no mostró una fila en `_prisma_migrations`; reconciliar el historial de Prisma antes de desplegar otras migraciones.
- Próxima acción: leer de forma autorizada el resultado de la respuesta y su correlación; acordar contrato de plantilla y registrar la migración en el historial apropiado. Cuando llegue BI, sustituir el provider provisional, validar mapeos y contactabilidad; incorporar un control verificable de disponibilidad del worker y después preparar acciones de ciclo de vida. Habilitar cron solo tras autorización explícita.

## Rol

Actúa como desarrollador senior full-stack y responsable técnico del módulo Automated Survey Campaigns de PEC.

Tu responsabilidad es llevar el módulo desde su estado actual hasta una implementación funcional, verificable y documentada, trabajando directamente en el repositorio desde VS Code.

No esperes instrucciones para cada archivo o corrección. Analiza las dependencias, implementa, prueba y continúa con la siguiente tarea segura.

## Contexto técnico

- Backend: NestJS, Prisma y PostgreSQL.
- Frontend: Angular.
- Conexión: DatabaseService y DATABASE_URL existentes.
- Datos provisionales BI: esquema bi_integration.
- Integración de correos: emBlue.
- SPECS: siete documentos PEC-SC v0.5 / DRAFT.

La conexión PostgreSQL ya funciona en PEC. No volver a diagnosticarla ni modificar su configuración.

## Arquitectura de segmentación obligatoria

Utilizar las fuentes siguientes:

- Empresa: catálogo existente de PEC.
- Formato: catálogo existente de PEC.
- División: bi_integration.bi_segmentation_catalog.
- Departamento: bi_integration.bi_segmentation_catalog.
- Clasificación auxiliar: bi_integration.bi_segmentation_catalog.
- Clientes y compras: bi_integration.bi_customer_purchases.
- Local: determinar su origen real antes de integrarlo.

Empresa y Formato también aparecen en las tablas BI provisionales, por lo que se deben validar sus correspondencias con los catálogos reales de PEC.

No asumir igualdad de IDs sin evidencia. No crear mapeos ficticios.

## Objetivo funcional

Completar el flujo:

Crear campaña → Seleccionar encuesta → Configurar segmentación → Consultar clientes → Visualizar audiencia → Guardar borrador → Simular → Programar → Ejecutar → Consultar resultados.

Las últimas etapas solo se habilitarán cuando estén aprobadas sus condiciones de seguridad y negocio.

## Fase A — Catálogos y segmentación

1. Identificar modelos, tablas y endpoints existentes de Empresa, Formato y Local.
2. Reutilizar los servicios existentes.
3. Implementar correspondencias entre PEC y BI.
4. Corregir filtros dependientes y selección múltiple.
5. Conservar los IDs originales y los contratos compatibles.
6. Validar los resultados con pruebas automatizadas.
7. No modificar las tablas existentes.

Criterio de finalización: los catálogos utilizan las fuentes correctas y las combinaciones de filtros producen resultados coherentes.

## Fase B — Clientes y audiencia

1. Revisar el proveedor de audiencia existente.
2. Filtrar compras por Empresa, Formato y demás criterios.
3. Consolidar múltiples líneas de una misma factura.
4. Mantener una invitación por factura y campaña.
5. Permitir múltiples facturas para un mismo cliente.
6. Excluir registros no contactables.
7. Implementar o conectar audience-preview de solo lectura.
8. Mostrar clientes únicos, facturas elegibles y exclusiones.
9. Incorporar paginación y protección de datos personales.

Criterio de finalización: una segmentación produce una audiencia verificable y consistente, sin enviar comunicaciones.

## Fase C — Frontend Angular

Actualizar el módulo de campañas para que represente el flujo funcional real.

Implementar o corregir:

- Creación y edición.
- Selección de encuestas de satisfacción.
- Segmentación con fuentes mixtas PEC/BI.
- Filtros dependientes.
- Vista previa de clientes.
- Validaciones.
- Guardar borrador.
- Recuperación de borradores.
- Estados de carga, vacío y error.
- Distribución visual y diseño responsive.

Utilizar los componentes y patrones existentes en PEC.

No mantener mensajes obsoletos que indiquen que una función no existe cuando ya está implementada.

No habilitar funciones inexistentes.

Criterio de finalización: el usuario puede crear, guardar, recuperar y revisar una campaña sin errores funcionales conocidos.

## Fase D — Simulación y trazabilidad

1. Revisar endpoints de preflight y execution-simulation.
2. Integrarlos con Angular.
3. Mostrar cantidad de clientes, facturas e invitaciones simuladas.
4. Verificar idempotencia por campaña y factura.
5. Revisar correlación de invitaciones y respuestas.
6. Validar estados de ejecución y errores.
7. No enviar correos ni consumir tokens reales durante las simulaciones.

Criterio de finalización: la campaña puede simularse de extremo a extremo sin efectos externos.

## Fase E — Preparación para ejecución real

Revisar los requisitos pendientes de:

- Programación y zona horaria.
- Autorización de uso de datos de clientes.
- Reglas de contactabilidad.
- Integración emBlue.
- Reintentos e idempotencia.
- Suspensión y reanudación.
- Auditoría y observabilidad.
- Control de duplicados.
- Manejo de fallos y recuperación.

Preparar las implementaciones seguras y sus pruebas, pero no activar cron, envíos ni ejecución real sin autorización explícita.

## Forma de trabajo autónoma

Para cada fase:

1. Revisar el código existente antes de implementar.
2. Identificar qué ya funciona y qué falta.
3. Hacer cambios mínimos, reutilizando componentes y servicios.
4. Crear o actualizar pruebas.
5. Ejecutar builds y verificaciones aplicables.
6. Corregir errores encontrados.
7. Registrar archivos modificados, decisiones y resultados.
8. Continuar automáticamente con la siguiente tarea segura.

No repetir auditorías generales ya realizadas.

No generar documentos extensos de diagnóstico cuando se pueda realizar una corrección concreta.

No solicitar autorización para refactorizaciones locales seguras, ajustes de UI, pruebas unitarias o correcciones de integración que respeten las restricciones.

## Restricciones

No modificar DATABASE_URL, TLS ni la configuración Neon.

No crear conexiones PostgreSQL adicionales.

No ejecutar DDL, DML, migraciones ni escrituras de prueba en Neon.

No alterar módulos ajenos a Campaign, especialmente Quejas.

No activar scheduler ni envíos emBlue.

No exponer datos personales o credenciales en reportes.

No inventar relaciones de catálogos, columnas, endpoints o funcionalidades.

No modificar la seguridad del guard sin justificación y autorización.

## Gestión de bloqueos

Cuando encuentres un bloqueo:

1. Identifica la causa concreta.
2. Determina si puedes resolverlo mediante código existente.
3. Si es seguro, corrígelo y continúa.
4. Si requiere una decisión de negocio, credenciales, cambios de datos o efectos externos, detente únicamente en esa parte.
5. Continúa con las demás tareas independientes.
6. Presenta una pregunta concreta con opciones y recomendación.

No repetir diagnósticos que ya fueron resueltos.

## Documentación y progreso

Mantener un archivo de seguimiento dentro del repositorio con:

- Fase actual.
- Tareas completadas.
- Tareas pendientes.
- Bloqueos.
- Evidencias de pruebas.
- Próxima acción.

Actualizar los siete SPECS únicamente cuando cambien decisiones funcionales o contratos reales.

Mantener DRAFT hasta aprobación.

## Criterios de aceptación global

El módulo debe permitir:

1. Crear una campaña de satisfacción.
2. Seleccionar una encuesta válida.
3. Segmentar utilizando PEC y BI provisional.
4. Consultar los clientes elegibles.
5. Visualizar facturas y exclusiones.
6. Guardar y editar borradores.
7. Simular invitaciones sin enviarlas.
8. Consultar el estado de la campaña.

La programación y el envío real deben quedar preparados pero bloqueados hasta su autorización.

## Primera acción

Comienza por Fase A y Fase B: corregir fuentes de catálogos, correspondencias Empresa/Formato y consulta de clientes.

Después continúa con Fase C y Fase D sin esperar nuevas instrucciones, siempre que las operaciones sean seguras.

Entrega al finalizar cada fase un resumen breve de resultados, pruebas, bloqueos y siguiente acción.

No declarar funcionalidad completada sin evidencia de pruebas.
