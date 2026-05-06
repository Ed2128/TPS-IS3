# Spec: Módulo de Comentarios Públicos y Encuestas de Satisfacción

## 1. Objetivo y Contexto
Como parte del Sistema de Gestión de Eventos Académicos (SGEA), el objetivo de este módulo es habilitar canales de *feedback* una vez que un evento ha concluido. Proporciona un espacio público para comentarios y un mecanismo estructurado (encuesta de satisfacción) de uso interno. El sistema debe garantizar la participación exclusiva de los asistentes reales, integrándose con el stack tecnológico de Next.js y el backend en Spring Boot bajo las normativas arquitectónicas del proyecto.

---

## 2. Historias de Usuario y Criterios de Aceptación

### **HU1: Publicación de Comentarios**
Como asistente acreditado (participante, disertante u organizador), quiero dejar un comentario público en la página del evento una vez finalizado, para compartir mi experiencia con la comunidad.
* **Criterios de Aceptación:**
  * El botón de comentar en el frontend (Next.js) solo debe estar visible si la fecha y hora actual es posterior al fin del evento.
  * El backend debe verificar estrictamente el estado "Acreditado" del usuario para el evento en cuestión.
  * El comentario publicado debe mostrar el nombre del autor y su rol asociado.

### **HU2: Lectura de Comentarios**
Como usuario público o visitante de la plataforma, quiero leer los comentarios en la página de un evento finalizado, para conocer cómo fue su desarrollo.
* **Criterios de Aceptación:**
  * El acceso a la lectura debe ser público (no requiere token de autenticación).
  * Los comentarios deben estar ordenados del más reciente al más antiguo.
  * El endpoint de lectura debe estar correctamente documentado vía Swagger/OpenAPI.

### **HU3: Encuesta de Satisfacción**
Como asistente acreditado, quiero completar una encuesta de satisfacción al finalizar el evento, para enviar mis sugerencias de mejora de forma privada a la organización.
* **Criterios de Aceptación:**
  * Habilitación estricta post-evento y validación de estado "Acreditado" en el backend.
  * El sistema solo debe permitir un (1) envío por usuario para garantizar la integridad de las métricas.

---

## 3. Requisitos Funcionales y Reglas de Negocio

* **RF1:** El sistema debe evaluar dinámicamente el estado del evento (fecha de fin superada) antes de procesar peticiones `POST`. Si la validación falla, se debe lanzar una excepción manejada globalmente.
* **RF2:** Los comentarios ingresados no pueden superar la longitud máxima de 500 caracteres. Esta restricción debe validarse obligatoriamente mediante Bean Validation en la capa de transporte.
* **RF3:** La listación de comentarios debe implementar paginación mediante `page` y `size` como parámetros de query. Respuesta: `{"content": [...], "totalElements": N, "totalPages": M, "currentPage": P}`
* **RF4:** Las encuestas deben estar compuestas por preguntas con opciones de respuesta. Tipos permitidos: `TEXTO_LIBRE` (VARCHAR), `ESCALA_NUMERICA` (1-5), `SELECCION_UNICA` (opciones predefinidas).
* **RN1 (Control de Acceso):** Un usuario no registrado, o con estado de inscripción "Ausente" o "Inscripto" (no acreditado), no puede bajo ninguna circunstancia publicar comentarios o enviar encuestas.
* **RN2 (Privacidad):** Las respuestas detalladas de la encuesta son estrictamente privadas y exclusivas para los usuarios con rol de "Organizador" vinculados a ese evento.
* **RN3 (Validación de Acreditación):** La verificación del estado "Acreditado" se realiza mediante consulta a la tabla `inscriptions` con condiciones: `event_id = ? AND user_id = ? AND asistio = TRUE`.

---

## 4. Restricciones Técnicas Específicas

* **Arquitectura:** Se debe respetar rigurosamente la arquitectura por capas (`Controller`, `Service`, `Repository`) utilizando Spring Boot 3.x y Java 21.
* **Inmutabilidad:** Uso estricto de objetos de transferencia de datos (DTOs como `CommentRequestDTO`, `CommentResponseDTO`) para la comunicación con Next.js. Las entidades JPA no deben exponerse al cliente.
* **Validación de Datos:** Implementación de Bean Validation (JSR 303) en los DTOs, utilizando anotaciones como `@NotNull`, `@NotBlank` y `@Size(max=500)`.
* **Manejo de Errores:** Cualquier violación a las Reglas de Negocio debe lanzar una excepción personalizada (ej. `EventNotFinishedException`, `UserNotAccreditedException`). Estas deben ser capturadas por el `GlobalExceptionHandler` para retornar respuestas consistentes (RFC 7807 o JSON estándar del proyecto).
* **Código Limpio:** Uso obligatorio de la librería Lombok (`@Data`, `@RequiredArgsConstructor`, `@Builder`) en DTOs y Entidades para reducir el *boilerplate*.
* **Documentación:** Los endpoints expuestos deben incluir anotaciones de Swagger/OpenAPI (`@Operation`, `@ApiResponses`) detallando códigos HTTP esperados (200, 201, 400, 403, 404).
* **Excepciones Personalizadas:** Crear las siguientes excepciones en paquete `com.sgea.exception`: `EventNotFinishedException`, `UserNotAccreditedException`, `DuplicateSurveySubmissionException`, `SurveyQuestionNotFoundException`.

---

## 5. Modelo de Datos

Implementación mediante JPA/Hibernate en PostgreSQL. Tablas en **plural** y formato **snake_case**:

* `event_comments`
  * `id` (PK, UUID)
  * `event_id` (FK)
  * `user_id` (FK)
  * `content` (VARCHAR(500))
  * `created_at` (TIMESTAMP)

* `survey_submissions`
  * `id` (PK, UUID)
  * `event_id` (FK)
  * `user_id` (FK)
  * `submitted_at` (TIMESTAMP)

* `survey_questions`
  * `id` (PK, UUID)
  * `event_id` (FK)
  * `question_text` (TEXT, no nulo)
  * `response_type` (VARCHAR(20): TEXTO_LIBRE, ESCALA_NUMERICA, SELECCION_UNICA)
  * `created_at` (TIMESTAMP)
  * Índice compuesto: (`event_id`, `id`)

* `survey_answers`
  * `id` (PK, UUID)
  * `submission_id` (FK)
  * `question_id` (FK)
  * `value` (VARCHAR(500), puede ser NULL para respuestas pendientes)
  * `answered_at` (TIMESTAMP)

---

## 5.1 Endpoints REST

### Comentarios
* **POST** `/api/v1/events/{eventId}/comments`  
  * Requiere: Token JWT, body `CommentRequestDTO`
  * Valida: Evento finalizado, usuario acreditado
  * Respuesta: `201 Created` con `CommentResponseDTO`
  * Excepciones: `EventNotFinishedException`, `UserNotAccreditedException`

* **GET** `/api/v1/events/{eventId}/comments?page=0&size=10`  
  * Público, sin autenticación requerida
  * Respuesta: `200 OK` con `Page<CommentResponseDTO>` (ordenados DESC por `created_at`)
  * Documentación Swagger: Incluir ejemplos de paginación

### Encuestas
* **GET** `/api/v1/events/{eventId}/survey`  
  * Requiere: Token JWT, usuario acreditado para el evento
  * Respuesta: `200 OK` con lista de `SurveyQuestionDTO` y estado de completitud
  * Retorna: `404 Not Found` si no hay encuesta para el evento

* **POST** `/api/v1/events/{eventId}/survey/submit`  
  * Requiere: Token JWT, body `SurveySubmissionRequestDTO` (lista de respuestas)
  * Valida: Evento finalizado, usuario acreditado, primer envío únicamente
  * Respuesta: `201 Created` con ID de `submission`
  * Excepciones: `EventNotFinishedException`, `UserNotAccreditedException`, `DuplicateSurveySubmissionException`

* **GET** `/api/v1/events/{eventId}/survey/results` (Solo para Organizadores)  
  * Requiere: Token JWT, rol ORGANIZADOR vinculado al evento
  * Respuesta: `200 OK` con estadísticas agregadas y todas las respuestas (privadas)
  * Respuesta: `403 Forbidden` si no tiene rol autorizado

---

## 6. Plan de Tareas

1. **Persistencia:** Crear las entidades JPA (`EventComment`, `SurveySubmission`, `SurveyQuestion`, `SurveyAnswer`) mapeando a la base de datos según los estándares de nomenclatura. Incluir índices en `event_comments(event_id)` y `survey_questions(event_id)` para optimizar búsquedas.
2. **DTOs:** Definir los DTOs de entrada/salida:
   * `CommentRequestDTO`: `content (String)`
   * `CommentResponseDTO`: `id, author, role, content, createdAt`
   * `SurveyQuestionDTO`: `id, questionText, responseType, options (para SELECCION_UNICA)`
   * `SurveySubmissionRequestDTO`: Lista de pares `{questionId, value}`
   * `SurveyAnswerDTO`: `questionId, value, answeredAt`
   * Todos con JSR 303 y Lombok.
3. **Repositorios:** Crear interfaces extendiendo `JpaRepository` con métodos personalizados:
   * `EventCommentRepository`: `findByEventId(UUID, Pageable)`, `findByEventIdAndUserId(UUID, UUID)`
   * `SurveyQuestionRepository`: `findByEventId(UUID)`
   * `SurveySubmissionRepository`: `findByEventIdAndUserId(UUID, UUID)`, `existsByEventIdAndUserId(UUID, UUID)`
4. **Lógica de Negocio:** Implementar `CommentService`, `SurveyService` con validaciones:
   * Consultar `inscriptions` para verificar `asistio = TRUE`
   * Validar fechas del evento mediante consulta a tabla `events`
   * Prevenir envíos duplicados de encuestas
5. **Controladores y Documentación:** Desarrollar `CommentController` y `SurveyController` con anotaciones OpenAPI (`@Operation`, `@ApiResponses`, `@Parameter`).
6. **Manejo de Excepciones:** Crear excepciones personalizadas y actualizar `GlobalExceptionHandler` con mapeo a `ProblemDetail` (RFC 7807).
7. **Frontend Next.js:**
   * Página `/events/[eventId]/feedback` con tabs: "Comentarios" y "Encuesta"
   * Componente `CommentList`: Paginación, ordenamiento, sin requerir autenticación
   * Componente `CommentForm`: Textarea validado (máximo 500 caracteres), botón condicional (visible post-evento)
   * Componente `SurveyForm`: Renderizar campos según tipo (textarea, slider 1-5, radio buttons)
   * Componente `SurveyResults`: Dashboard solo para Organizadores con gráficas agregadas
   * Estados: `loading`, `success`, `error` con notificaciones toast

---

## 7. Estrategia de Verificación

### Pruebas de Validación (JSR 303)
* Enviar un *payload* JSON con un campo `content` de 501 caracteres al endpoint de comentarios. Verificar que el `GlobalExceptionHandler` intercepte la falla y retorne `400 Bad Request`.
* Enviar una respuesta de encuesta con formato inválido (ej: valor fuera de rango 1-5 para ESCALA_NUMERICA). Verificar `400 Bad Request` con descripción específica.

### Pruebas de Lógica de Negocio (Unit Tests, JUnit 5 + Mockito)
* **Test 1:** Usuario "Inscripto" (no acreditado) intenta comentar → Lanzar `UserNotAccreditedException`
* **Test 2:** Usuario acreditado intenta comentar antes de que finalice el evento → Lanzar `EventNotFinishedException`
* **Test 3:** Usuario intenta enviar dos encuestas del mismo evento → Lanzar `DuplicateSurveySubmissionException` en segundo intento
* **Test 4:** Verificar que `CommentRepository.findByEventId()` retorna comentarios ordenados DESC por `created_at`

### Pruebas de Integración (Spring Boot Test + TestContainers PostgreSQL)
* **Test 1:** POST comentario con usuario acreditado y evento finalizado → Verificar `201 Created` y persistencia en BD
* **Test 2:** GET comentarios sin autenticación → Verificar `200 OK` y paginación correcta
* **Test 3:** POST encuesta con respuestas válidas → Verificar `201 Created` y que `DuplicateSurveySubmissionException` bloquea segundo envío
* **Test 4:** GET resultados de encuesta sin rol Organizador → Verificar `403 Forbidden`

### Pruebas de Frontend (Playwright/Jest)
* Verificar que botón de comentarios está oculto antes de la fecha de fin del evento
* Verificar validación en tiempo real: campo limita a 500 caracteres
* Verificar paginación: cambiar página y verificar que comentarios se actualizan
* Verificar que encuesta desaparece después de primer envío