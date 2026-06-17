# Título: ADR-004 Manejo Global de Excepciones y Control de Errores para APIs Externas

**Estado:** Propuesto  
**Fecha:** 2026-06-17  
**Decisores:** Paola Samudio  
**Relacionado:** Spec de Gestión de Eventos

## Contexto
- **Qué problema se está resolviendo:** Al conectar el sistema con servicios externos (como pasarelas de pago para eventos arancelados o validadores de identidad), las caídas de estas APIs de terceros provocan errores internos (500) no controlados que afectan la experiencia del usuario.
- **Qué restricciones aplican:** Las respuestas de error hacia el frontend deben ser amigables, uniformes y en formato JSON.
- **Qué datos de proyecto sustentan la decisión:** Las pruebas de integración iniciales muestran que las respuestas inesperadas de red bloquean hilos de ejecución en el servidor de aplicaciones.

## Decisión
- **Qué se decide exactamente:** Implementar un componente centralizado de manejo de excepciones utilizando la anotación `@ControllerAdvice` de Spring junto con la estructura de problemas estandarizada (Problem Details - RFC 7807).
- **Alcance:** Aplica a todas las excepciones lanzadas por los controladores y los fallos de comunicación con servicios externos integrados.

## Alternativas consideradas
- **Opción A: Bloques Try-Catch individuales en cada controlador**
  - Pros: Fácil de programar para casos aislados en el momento.
  - Contras: Código repetitivo (boilerplate), difícil de mantener y propenso a omitir excepciones imprevistas.

- **Opción B: Delegar el error directamente al servidor de aplicaciones (Tomcat)**
  - Pros: No requiere desarrollo de código adicional.
  - Contras: Muestra pantallas de error genéricas de la infraestructura que exponen datos sensibles del sistema (stacktraces).

## Consecuencias
- **Beneficios esperados:** Respuestas homogéneas en toda la API, ocultamiento de datos internos de fallos por seguridad y facilidad para diagnosticar errores mediante logs unificados.
- **Costos o riesgos que se aceptan:** Sobrecarga mínima al interceptar cada petición HTTP que resulta en fallo.
- **Impacto en operación y equipo:** Los desarrolladores no deben capturar errores manualmente en la capa web; deben lanzar excepciones personalizadas y delegarlas al manejador global.

## Plan de implementación
- **Pasos mínimos para ejecutarla:** Crear la clase `GlobalExceptionHandler` con la anotación `@ControllerAdvice`. Definir los métodos de captura para `WebClientResponseException` y excepciones genéricas del sistema.
- **Dependencias:** Módulo Spring Web.
- **Métrica de éxito:** El 100% de los errores devueltos por la API deben tener la estructura JSON estandarizada del proyecto.

## Triggers de revisión
- **Qué condiciones obligan a reabrir esta ADR:** Cambios mayores en la especificación de manejo de errores de los frameworks del frontend (como Next.js) que requieran otro tipo de payloads.
- **Fecha sugerida de revisión:** 2026-12-20.