# Título: ADR-002 Implementación de Caché con Redis para el listado público de eventos

**Estado:** Propuesto
**Fecha:** 2026-06-16 
**Decisores:** Eduardo Navarro 
**Relacionado:** Spec Gestión de Eventos

## Contexto
- **Qué problema se está resolviendo:** Se anticipa una latencia alta en las consultas de la vista pública de próximos eventos ("Filtro de eventos a futuro"), debido a los picos de tráfico durante las fechas de apertura de inscripciones.
- **Qué restricciones aplican:** La información debe servirse rápido para la UI en Next.js, pero no requiere estar actualizada al milisegundo absoluto.
- **Qué datos de proyecto sustentan la decisión:** La consulta a la tabla `eventos` con filtros de fechas consume recursos de CPU en PostgreSQL de forma innecesaria cuando los datos cambian poco frecuentemente.

## Decisión
**Qué se decide exactamente:** Introducir un sistema de caché en memoria utilizando Redis para almacenar los resultados del listado público de eventos.
- **Alcance:** Aplica únicamente al endpoint público de visualización de eventos. No aplica a módulos transaccionales (acreditaciones, asignación de roles o inscripciones).

## Alternativas consideradas
- **Opción A: Caché local en memoria de Spring Boot (ConcurrentHashMap)** 
  - Pros: Fácil implementación sin infraestructura adicional.
  - Contras: Si la aplicación escala a múltiples instancias, la caché se desincroniza.
-**Opción B: Escalar verticalmente PostgreSQL** 
  - Pros: No requiere cambiar el código del backend.
  - Contras: Altamente costoso y no resuelve el problema de raíz de procesar peticiones idénticas repetidamente.

## Consecuencias
-**Beneficios esperados:** Reducción del tiempo de respuesta del listado de eventos a < 50ms, aliviando la carga de PostgreSQL.
- **Costos o riesgos que se aceptan:** Se introduce una nueva pieza de infraestructura que administrar. 
Riesgo de "Stale Data" (mostrar eventos desactualizados por unos minutos).
- **Impacto en operación y equipo:** Los desarrolladores deberán aprender el uso de las anotaciones `@Cacheable` y `@CacheEvict` en Spring Boot.

## Plan de implementación
- **Pasos mínimos para ejecutarla:** Añadir el contenedor de Redis al `docker-compose.yml`. Agregar la dependencia `spring-boot-starter-data-redis` en el proyecto.Intervenir el `EventoService` con las anotaciones correspondientes.
- **Dependencias:** Servicio de Redis operativo.
- **Métrica de éxito:** El 90% de las peticiones públicas de eventos no impactan directamente contra la base de datos.

## Triggers de revisión
- **Qué condiciones obligan a reabrir esta ADR:** Si el consumo de memoria RAM de Redis supera los límites asignados del servidor o si la invalidación de la caché genera demasiados errores en la UI[cite: 372].
- **Fecha sugerida de revisión:** 2026-11-15.