# ADR-005: Elección de Next.js como Framework para el Frontend

* **Estado:** Aceptado
* **Fecha:** 2026-06-17
* **Decisores:** Karen Proscopio
* **Relacionado:** Project.md (Stack Tecnológico)

## Contexto
* **Qué problema se está resolviendo:** El SGEA requiere una interfaz web moderna, ágil y optimizada para dos perfiles de uso muy marcados: la administración interna de eventos/actas y la consulta e inscripción pública por parte de alumnos, docentes y externos.
* **Qué restricciones aplican:** Debe comunicarse eficientemente de forma asincrónica con la API REST en Spring Boot (ADR-003) y garantizar una excelente velocidad de carga en dispositivos móviles para las inscripciones rápidas.
* **Qué datos de proyecto sustentan la decisión:** Se anticipan picos masivos de tráfico público el día de apertura de inscripciones a congresos. El renderizado puramente en el cliente causaría lentitud y latencia alta en las consultas de eventos disponibles.

## Decisión
* **Qué se decide exactamente:** Adoptar **Next.js** (framework basado en React) como la tecnología principal para el desarrollo de la aplicación del frontend.
* **Alcance:** Cubre la totalidad de las vistas de usuario (públicas de visualización y privadas de gestión). No incluye el desarrollo de aplicaciones móviles nativas.

## Alternativas consideradas
* **Opción A: React SPA (Single Page Application) puro con Vite**
  * *Pros:* Configuración inicial sumamente veloz, menor curva de aprendizaje.
  * *Contras:* Todo el procesamiento ocurre en el navegador del usuario; la experiencia inicial de carga se degrada ante listas grandes de eventos y dificulta el SEO de congresos abiertos.
* **Opción B: Blade Templates / Thymeleaf (Renderizado integrado en Spring Boot)**
  * *Pros:* No requiere separar el proyecto en dos repositorios o servidores; acoplamiento directo con los modelos de Java.
  * *Contras:* Interfaz de usuario más rígida, recargas completas de página indeseadas y sobrecarga de procesamiento de vistas en el servidor de backend.

## Consecuencias
* **Beneficios esperados:** Carga inicial ultra rápida gracias a Server-Side Rendering (SSR) e hidratación eficiente, permitiendo un listado público fluido que reduce la percepción de latencia alta.
* **Costos o riesgos que se aceptan:** Requiere gestionar un entorno Node.js en producción para el servidor de frontend y adaptabilidad del equipo al App Router de Next.js.
* **Impacto en operación y equipo:** El equipo dividirá claramente las tareas de maquetado e integración de endpoints, consumiendo los JSON estandarizados definidos en el ADR-004.

## Plan de implementación
* **Pasos mínimos para ejecutarla:** Inicializar el proyecto con `npx create-next-app@latest`. Establecer la configuración de comunicación base hacia la IP/URL del Backend y definir la estructura de rutas dinámicas para `/eventos/{id}`.
* **Dependencias:** Node.js v20+ en las estaciones de desarrollo.
* **Métrica de éxito:** Puntuación de rendimiento en Lighthouse de Google superior a 80 puntos en la página principal de eventos.

## Triggers de revisión
* **Qué condiciones obligan a reabrir esta ADR:** Si las especificaciones del negocio viran drásticamente hacia una aplicación móvil obligatoria como único canal, forzando la migración a React Native.
* **Fecha sugerida de revisión:** 2026-11-20