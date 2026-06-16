# Título: ADR-001 Elección de PostgreSQL como motor de base de datos relacional 

**Estado:** Aceptado 
**Fecha:** 2026-06-16 
**Decisores:** Eduardo Navarro 
**Relacionado:** Project.md (Stack Tecnológico Forzado) 

## Contexto
- **Qué problema se está resolviendo:** El Sistema de Gestión de Eventos Académicos (SGEA) requiere un sistema de almacenamiento persistente, robusto y transaccional para manejar inscripciones, acreditaciones y certificados.
- **Qué restricciones aplican:** Se debe integrar nativamente con Java 21, Spring Boot 3.x y JPA/Hibernate.
-**Qué datos de proyecto sustentan la decisión:** El modelo de datos presenta alta relacionalidad entre Usuarios, Eventos e Inscripciones.

## Decisión
**Qué se decide exactamente:** Utilizar PostgreSQL como el motor de base de datos relacional principal.
- **Alcance:** Cubre el almacenamiento de todas las entidades de negocio. No cubre el almacenamiento de archivos binarios (los PDF de certificados se generan al vuelo o se guardan en un Object Storage).

## Alternativas consideradas
- **Opción A: MySQL / MariaDB** 
  - Pros: Amplia adopción, curva de aprendizaje baja.
  - Contras: Menor cumplimiento estricto del estándar SQL en comparación con PostgreSQL, menor eficiencia en concurrencia masiva.
- **Opción B: MongoDB (NoSQL)** 
  - Pros: Esquema flexible, rápido desarrollo inicial.
  - Contras: Ausencia de transacciones ACID nativas simples para relaciones complejas, lo que dificulta la lógica de acreditaciones e inscripciones limitadas por cupo.

## Consecuencias
- **Beneficios esperados:** Consistencia transaccional fuerte, soporte avanzado para tipos de datos (como JSONB si a futuro se necesitan formularios de encuestas dinámicos), y excelente compatibilidad con JPA.
- **Costos o riesgos que se aceptan:** La configuración de respaldos y monitoreo es ligeramente más compleja que en otras alternativas simples.
- **Impacto en operación y equipo:** El equipo ya configuró el entorno local con Docker en el TP5, por lo que el impacto operativo actual es nulo.

## Plan de implementación
- **Pasos mínimos para ejecutarla:** Incluir la imagen `postgres:15-alpine` en el `docker-compose.yml` local. Configurar las variables `SPRING_DATASOURCE_URL` en el backend.
- **Dependencias:** Driver de PostgreSQL en el `pom.xml`.
- **Métrica de éxito:** Tiempos de respuesta de consultas complejas (ej. reporte de informes) por debajo de 500ms en desarrollo.

## Triggers de revisión
- **Qué condiciones obligan a reabrir esta ADR:** Si el volumen de lecturas escala drásticamente y la base de datos comienza a presentar bloqueos o latencia superior a 2 segundos.
- **Fecha sugerida de revisión:** 2026-12-01.