# ADR-001 Elección del Motor de Base de Datos Principal

**Estado:** Aceptado
**Fecha:** 2026-06-16
**Decisores:** Juan Diego
**Relacionado:** project.md, Specs generales

## Contexto
- **Qué problema se está resolviendo:** Necesidad de un motor relacional sólido para estructurar la información del Sistema de Gestión de Eventos Académicos (SGEA).
- **Qué restricciones aplican:** Compatibilidad con el ecosistema de Spring Data JPA y soporte para transacciones ACID estrictas.
- **Qué datos de proyecto sustentan la decisión:** El control de cupos de inscripción exige un buen manejo de concurrencia y bloqueos a nivel de fila.

## Decisión
Se decide utilizar PostgreSQL como motor de base de datos relacional principal.
- **Alcance (qué cubre y qué no cubre):** Aplica para la persistencia de todos los datos transaccionales (usuarios, eventos, inscripciones). No cubre el almacenamiento físico de archivos (como los PDF generados).

## Alternativas consideradas
- **Opción A (MySQL):** pros: Fácil configuración inicial. contras: Menor rendimiento nativo para estructuras JSON u operaciones complejas frente a Postgres.
- **Opción B (MongoDB):** pros: Esquemas dinámicos. contras: No es relacional, lo que dificulta mantener la integridad referencial requerida por el negocio.

## Consecuencias
- **Beneficios esperados:** Alta fiabilidad, integridad referencial y excelente compatibilidad con Hibernate.
- **Costos o riesgos que se aceptan:** Curva de aprendizaje técnica para optimización del motor.
- **Impacto en operación y equipo:** Todo el equipo deberá levantar localmente el servicio mediante Docker Compose.

## Plan de implementación
- **Pasos mínimos para ejecutarla:** Configurar el docker-compose.yml con la imagen oficial de Postgres y agregar el driver en el pom.xml.

## Dependencias
Motor de Docker en entornos locales, driver org.postgresql.

## Métrica de éxito
Servicio de base de datos levantado sin errores de conexión desde la aplicación Spring Boot.

## Triggers de revisión
- **Qué condiciones obligan a reabrir esta ADR:** Si el volumen de lecturas supera ampliamente la capacidad de respuesta de una sola instancia relacional.
- **Fecha sugerida de revisión:** 2026-12-01