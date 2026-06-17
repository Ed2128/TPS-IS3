# Título: ADR-003 Elección de Java y Spring Boot para el desarrollo del Backend

**Estado:** Aceptado  
**Fecha:** 2026-06-17  
**Decisores:** Paola Samudio  
**Relacionado:** Project.md (Stack Tecnológico)

## Contexto
- **Qué problema se está resolviendo:** El Sistema de Gestión de Eventos Académicos (SGEA) necesita un entorno de backend robusto, seguro y capaz de gestionar la lógica de negocio de inscripciones y generación de actas.
- **Qué restricciones aplican:** Debe integrarse de manera limpia con arquitectura de contenedores Docker y un motor de base de datos relacional.
- **Qué datos de proyecto sustentan la decisión:** El equipo cuenta con conocimiento previo en el lenguaje Java y el framework Spring Boot obtenido en las materias anteriores de la carrera.

## Decisión
- **Qué se decide exactamente:** Utilizar Java 21 junto con el framework Spring Boot 3.x como la tecnología principal para el desarrollo del backend del sistema.
- **Alcance:** Cubre toda la API REST, la seguridad, los servicios de negocio y la persistencia de datos mediante JPA/Hibernate.

## Alternativas consideradas
- **Opción A: Node.js (Express)**
  - Pros: Desarrollo rápido, manejo nativo de JSON.
  - Contras: Menor tipado estricto por defecto, requiere librerías externas adicionales para igualar el ecosistema de seguridad de Spring.

- **Opción B: Python (Django)**
  - Pros: Panel de administración integrado muy rápido de configurar.
  - Contras: Curva de aprendizaje del framework para el equipo actual y menor rendimiento en operaciones concurrentes masivas en comparación con la JVM.

## Consecuencias
- **Beneficios esperados:** Escalabilidad horizontal nativa, tipado seguro que evita errores en producción y excelente ecosistema para inyección de dependencias.
- **Costos o riesgos que se aceptan:** Mayor consumo inicial de memoria RAM en los entornos de desarrollo local.
- **Impacto en operación y equipo:** El equipo de desarrollo utilizará entornos basados en el JDK de Java para compilar el código.

## Plan de implementación
- **Pasos mínimos para ejecutarla:** Configurar el archivo `pom.xml` inicial con las dependencias de `spring-boot-starter-web`. Crear la estructura básica de paquetes (`controllers`, `services`, `repositories`).
- **Dependencias:** JDK 21 instalado en el entorno.
- **Métrica de éxito:** Tiempo de respuesta de endpoints de lógica intermedia por debajo de los 300ms.

## Triggers de revisión
- **Qué condiciones obligan a reabrir esta ADR:** Si las limitaciones de hardware de los hosts locales impiden el testeo del empaquetado final de forma permanente.
- **Fecha sugerida de revisión:** 2026-11-15.