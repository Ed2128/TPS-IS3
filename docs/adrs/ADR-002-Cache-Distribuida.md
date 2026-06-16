# ADR-002 Implementación de Caché Distribuida con Redis

**Estado:** Propuesto
**Fecha:** 2026-06-16
**Decisores:** Juan Diego
**Relacionado:** Spec de Acreditación, Spec de Eventos

## Contexto
- **Qué problema se está resolviendo:** Latencia alta en la consulta del listado público de eventos y en la verificación de cupos durante picos de tráfico masivo.
- **Qué restricciones aplican:** La solución debe soportar el escalado horizontal de la aplicación (múltiples nodos del backend).
- **Qué datos de proyecto sustentan la decisión:** Las lecturas a la base de datos PostgreSQL se están encolando en los momentos previos a un evento.

## Decisión
Se decide integrar Redis como mecanismo de caché distribuida en la capa de servicios.
- **Alcance (qué cubre y qué no cubre):** Cubre exclusivamente las consultas de lectura intensiva (GET). No reemplaza a PostgreSQL como fuente de verdad.

## Alternativas consideradas
- **Opción A (Memcached):** pros: Muy rápido y simple. contras: Soporte limitado de tipos de datos en comparación a Redis.
- **Opción B (Caché local en memoria - Caffeine):** pros: Cero latencia de red, sin dependencias extra. contras: Genera inconsistencia de datos si el backend escala a más de una instancia.

## Consecuencias
- **Beneficios esperados:** Reducción dramática de la latencia (de cientos de milisegundos a < 10ms) y liberación de carga de la base de datos principal.
- **Costos o riesgos que se aceptan:** Riesgo de mostrar datos obsoletos (Stale Data) y mayor complejidad en la arquitectura local.
- **Impacto en operación y equipo:** Los desarrolladores deberán gestionar correctamente la invalidación de caché en las operaciones POST/PUT.

## Plan de implementación
- **Pasos mínimos para ejecutarla:** Añadir la imagen de Redis al docker-compose.yml, incluir la dependencia spring-boot-starter-data-redis y anotar los endpoints críticos.

## Dependencias
Contenedor Docker de Redis, librería de Spring Data Redis.

## Métrica de éxito
Disminución del 80% en los tiempos de respuesta de los endpoints cacheados.

## Triggers de revisión
- **Qué condiciones obligan a reabrir esta ADR:** Si el porcentaje de aciertos de la caché (Cache Hit Ratio) es muy bajo o hay cuellos de botella en la RAM del contenedor.
- **Fecha sugerida de revisión:** 2026-10-01