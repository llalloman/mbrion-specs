# Especificación: [título]

> Esta especificación es la fuente única de verdad para el desarrollo. Mantenerla actualizada durante el flujo **Idea & Spec → Plan → Tasks → Execute → Validate**. Los planes, tareas, implementación y validación deben derivarse de esta especificación; registrar aquí los cambios de alcance o decisiones.

## 1. Encabezado

| Campo | Valor |
|---|---|
| ID | `[ID de la especificación]` |
| Título | `[Título breve]` |
| Versión | `[Versión]` |
| Autor | `[Nombre o equipo]` |
| Fecha | `[AAAA-MM-DD]` |
| Work item / backlog | `[ID y enlace; indicar “Sin referencia” si no existe]` |

## 2. Problema

[¿Qué necesidad, dificultad u oportunidad da origen a este trabajo? ¿A quién afecta y cuál es la situación actual?]

## 3. Resultado esperado

[Describir el resultado observable y el valor que se espera obtener.]

## 4. Alcance

### Incluye

- [Capacidad o comportamiento incluido]

### Fuera de alcance

- [Capacidad o comportamiento expresamente excluido]

### Supuestos

- [Supuesto que condiciona la solución]

## 5. Reglas de negocio

- **RB-01:** [Regla verificable]

## 6. Criterios de aceptación

Escribir criterios verificables. Usar Given / When / Then cuando ayude a dejar claro el contexto, la acción y el resultado.

### Positivos

- **CA-P01:** Dado [contexto], cuando [acción], entonces [resultado esperado].

### Negativos

- **CA-N01:** Dado [contexto no válido], cuando [acción], entonces [rechazo o respuesta esperada].

### Casos de borde

- **CA-B01:** Dado [límite o condición excepcional], cuando [acción], entonces [resultado esperado].

## 7. Datos personales

- **¿Aplica?** [ ] Sí  [ ] No
- **Datos tratados:** [Tipos de datos o “No aplica”]
- **Finalidad y uso:** [Para qué se usan o “No aplica”]
- **Restricciones de acceso:** [Quién puede acceder y bajo qué condición; o “No aplica”]
- **Restricciones de logs:** [Datos que no deben registrarse, enmascaramiento y retención; o “No aplica”]
- **Restricciones de QA:** [Datos sintéticos, anonimización u otras condiciones; o “No aplica”]

## 8. Riesgos técnicos

| Riesgo | Impacto | Mitigación / respuesta |
|---|---|---|
| [Riesgo identificado] | [Impacto posible] | [Mitigación o “Por definir”] |

## 9. Evidencia esperada

- [Evidencia que demostrará cada criterio: pruebas, capturas, registros sanitizados, métricas u otra]

## 10. Dependencias

- [Equipo, servicio, sistema, decisión o entrega requerida; o “Ninguna identificada”]

## 11. Modelo conceptual / datos (cuando aplique)

[Entidades, atributos relevantes, relaciones, origen y destino de datos. No incluir datos reales ni secretos.]

## 12. Estados y transiciones (cuando aplique)

| Estado inicial | Evento / condición | Estado final | Restricciones |
|---|---|---|---|
| [Estado] | [Evento] | [Estado] | [Regla o permiso requerido] |

## 13. Roles y permisos (cuando aplique)

| Rol | Puede | No puede / restricciones |
|---|---|---|
| [Rol] | [Acciones permitidas] | [Límites] |

## 14. Integraciones (cuando aplique)

| Sistema | Propósito | Dirección / mecanismo | Manejo de fallos |
|---|---|---|---|
| [Sistema] | [Propósito] | [Entrada, salida y mecanismo] | [Comportamiento esperado] |

## 15. Contratos / API (cuando aplique)

| Operación / contrato | Entrada | Salida | Errores y validaciones |
|---|---|---|---|
| [Método, ruta, evento o contrato] | [Datos y restricciones] | [Respuesta esperada] | [Errores relevantes] |

## 16. Pantallas afectadas (cuando aplique)

- [Pantalla, flujo o enlace a diseño; indicar cambios relevantes]

## 17. Decisiones técnicas (cuando aplique)

| Decisión | Motivo | Consecuencias |
|---|---|---|
| [Decisión tomada] | [Motivo] | [Impacto o trade-off] |

## 18. Preguntas pendientes

- [ ] **[Pregunta]** — Responsable: [persona/equipo] — Fecha objetivo: [AAAA-MM-DD]

## 19. Estado de aprobación

- **Estado:** [Borrador / En revisión / Aprobada / Rechazada]
- **Aprobadores:** [Nombres o roles]
- **Fecha y observaciones:** [AAAA-MM-DD — observaciones o “Sin observaciones”]

## 20. Historial de versiones

| Versión | Fecha | Autor | Cambios |
|---|---|---|---|
| [Versión] | [AAAA-MM-DD] | [Nombre o equipo] | [Resumen del cambio] |
