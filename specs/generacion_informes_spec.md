# Especificación: Módulo de Generación de Informes y Agenda

## 1. Objetivo y Contexto
En el marco del Sistema de Gestión de Eventos Académicos (SGEA), este módulo provee herramientas de visualización y exportación de información estructurada. Cumple un doble propósito: por un lado, generar y exponer la Agenda del evento de forma pública para asistentes y visitantes; por otro, consolidar un Informe de Rendimiento privado exclusivo para los organizadores, integrando métricas de asistencia (inscriptos vs. acreditados) y los resultados de las encuestas de satisfacción. El sistema debe soportar tanto eventos de estructura simple como eventos complejos (con múltiples actividades y disertantes).

---

## 2. Historias de Usuario y Criterios de Aceptación

### **HU1: Visualización Pública de la Agenda**
Como participante o visitante público, quiero visualizar y descargar la agenda de un evento, para conocer el cronograma, la duración y quiénes son los disertantes.
* **Criterios de Aceptación:**
  * Si el evento es "Simple", debe mostrar el nombre del evento, fecha de inicio/fin, duración total y disertante/s principal/es.
  * Si el evento es "Complejo", debe mostrar adicionalmente el cronograma desglosado por actividades (horarios y disertante por cada actividad).
  * Debe estar disponible para descarga en formato PDF sin requerir autenticación.

### **HU2: Panel de Métricas del Evento**
Como Organizador, quiero acceder a un *dashboard* con el informe del evento, para analizar su rendimiento general y la recepción del público.
* **Criterios de Aceptación:**
  * El sistema debe verificar el rol de "Organizador" del usuario solicitante.
  * El dashboard debe mostrar el nombre del evento, fecha, duración y una comparativa cuantitativa entre la cantidad de inscriptos y los efectivamente acreditados.
  * Debe incluir un resumen estadístico de las encuestas de satisfacción (ej. promedio de puntuaciones).

### **HU3: Exportación del Informe de Rendimiento**
Como Organizador, quiero descargar el informe consolidado en formato PDF, para tener un respaldo documental de las métricas del evento.
* **Criterios de Aceptación:**
  * El PDF generado debe contener toda la información de la HU2, maquetada de forma institucional.

---

## 3. Requisitos Funcionales y Reglas de Negocio

* **RF1:** El sistema debe calcular dinámicamente la tasa de asistencia real aplicando la fórmula: $Tasa = \frac{Acreditados}{Inscriptos} \times 100$.
* **RF2:** La generación del PDF de la agenda o del informe debe realizarse en el servidor mediante iText 7 y devolverse con `Content-Type: application/pdf`.
* **RF3:** La agenda pública (simple o compleja) debe retornarse en formato JSON con estructura jerárquica. Para eventos complejos, agrupar actividades cronológicamente.
* **RF4:** El endpoint de métricas debe retornar estadísticas calculadas mediante una única consulta JPQL optimizada con `JOIN`, evitando N+1 queries.
* **RN1 (Privacidad de Métricas):** Los datos estadísticos (asistencia y encuestas) son de acceso restringido. Cualquier intento de acceso a los endpoints de informes por parte de un usuario sin permisos debe ser denegado.
* **RN2 (Estructura de Eventos):** Un evento simple no requiere carga de actividades secundarias, mientras que un evento complejo requiere al menos una (1) actividad programada en su agenda.

---

## 4. Restricciones Técnicas Específicas

* **Arquitectura y Stack:** Desarrollo bajo patrón MVC/Capas con Spring Boot 3.x (Java 21). El frontend en Next.js consumirá la API para renderizar las vistas y los gráficos de métricas.
* **DTOs y Proyección:** Uso estricto de DTOs (`AgendaPublicDTO`, `ReportMetricsDTO`). Para consultas complejas de métricas, se deben utilizar proyecciones JPA o consultas JPQL/Native optimizadas, evitando cargar entidades completas en memoria.
* **Manejo de Errores:** Las violaciones de acceso (ej. un participante queriendo ver las métricas) deben lanzar una `UnauthorizedReportAccessException`, la cual será interceptada por el `GlobalExceptionHandler` devolviendo un HTTP 403.
* **Validación de Entradas:** Los parámetros de búsqueda (como el ID del evento) en los Controladores deben estar validados con Bean Validation (`@NotNull`, `@Positive`).
* **Librerías de Exportación:** Se deberá utilizar **iText 7** (biblioteca `com.itextpdf:itext7-core`) para generación de PDFs con soporte para tablas, estilos y múltiples páginas. Configurar en `pom.xml` con versión ≥ 7.2.0.
* **Seguridad en Descargas:** Los archivos PDF deben ser generados bajo demanda (streaming) sin almacenarlos en disco. Header: `Content-Disposition: attachment; filename=\"agenda_evento_{eventId}_{timestamp}.pdf\"`.
* **Documentación:** Swagger/OpenAPI debe documentar la estructura de los DTOs de respuesta y los tipos de contenido (`application/pdf`, `application/json`).

---

## 5. Modelo de Datos

Las tablas seguirán la convención **plural** y **snake_case** en PostgreSQL. Para soportar la agenda y los eventos complejos, se requiere la siguiente estructura complementaria a la tabla `events`:

* `events` (Se asume existente, se destacan campos relevantes)
  * `id` (PK)
  * `is_complex` (BOOLEAN)
  * `start_date` (TIMESTAMP)
  * `end_date` (TIMESTAMP)

* `activities` (Para el desglose de agenda en eventos complejos)
  * `id` (PK, UUID)
  * `event_id` (FK)
  * `title` (VARCHAR(255), no nulo)
  * `description` (TEXT, nullable)
  * `start_time` (TIMESTAMP, no nulo)
  * `end_time` (TIMESTAMP, no nulo)
  * `speaker_id` (FK, apunta a `users.id`, nullable para actividades sin disertante)
  * `sequence_order` (INT, para ordenamiento)
  * `created_at` (TIMESTAMP)
  * Índice compuesto: (`event_id`, `start_time`)
  * Índice simple: `speaker_id` para JOIN eficiente

*(Nota: Las métricas de inscriptos, acreditados y encuestas se calculan a partir de uniones SQL `JOIN` y funciones de agregación `COUNT`, `AVG` sobre las tablas transaccionales definidas en specs anteriores, sin necesidad de persistirlas en tablas nuevas).*

---

## 5.1 Endpoints REST y Estructuras de Respuesta

### Agenda Pública
* **GET** `/api/v1/events/{eventId}/agenda?format=json|pdf`
  * Público, sin autenticación requerida
  * Parámetro `format`: `json` (default) o `pdf`
  * Respuesta JSON (200 OK) para format=json:
    ```json
    {
      "eventId": "uuid",
      "eventName": "string",
      "eventType": "SIMPLE|COMPLEJO",
      "startDate": "2026-05-15T09:00:00Z",
      "endDate": "2026-05-15T17:00:00Z",
      "totalDurationMinutes": 480,
      "mainSpeakers": ["nombre1", "nombre2"],
      "activities": [
        {
          "id": "uuid",
          "title": "Apertura",
          "description": "Bienvenida",
          "startTime": "2026-05-15T09:00:00Z",
          "endTime": "2026-05-15T09:30:00Z",
          "durationMinutes": 30,
          "speaker": {"id": "uuid", "name": "string", "role": "string"},
          "sequenceOrder": 1
        }
      ]
    }
    ```
  * Respuesta PDF (200 OK, format=pdf): `Content-Type: application/pdf` con agenda maquetada

### Métricas del Evento (Solo Organizadores)
* **GET** `/api/v1/events/{eventId}/report`
  * Requiere: Token JWT, rol ORGANIZADOR vinculado al evento
  * Respuesta (200 OK):
    ```json
    {
      "eventId": "uuid",
      "eventName": "string",
      "eventDate": "2026-05-15",
      "attendanceMetrics": {
        "totalRegistered": 150,
        "totalAccredited": 120,
        "attendanceRate": 80.0,
        "noShowRate": 20.0
      },
      "surveyMetrics": {
        "responseCount": 85,
        "responseRate": 56.67,
        "overallSatisfaction": 4.2,
        "questionBreakdown": [
          {
            "questionId": "uuid",
            "questionText": "¿Qué tan satisfecho estás?",
            "type": "ESCALA_NUMERICA",
            "average": 4.2,
            "distribution": {"1": 2, "2": 5, "3": 20, "4": 40, "5": 18}
          }
        ]
      },
      "generatedAt": "2026-05-16T14:30:00Z"
    }
    ```
  * Excepciones: `403 Forbidden` (sin rol), `404 Not Found` (evento inexistente)

* **GET** `/api/v1/events/{eventId}/report/pdf`
  * Requiere: Token JWT, rol ORGANIZADOR
  * Respuesta: PDF descargable con informe completo (200 OK)
  * Header: `Content-Disposition: attachment; filename="informe_evento_{eventId}_{fecha}.pdf"`

---

## 6. Plan de Tareas Detallado

1. **Entidades JPA:**
   * Crear `Activity` con anotaciones Lombok (`@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`)
   * Actualizar `Event` con relación `@OneToMany(mappedBy="event", cascade=CascadeType.ALL)` a `Activity`
   * Mapear columnas: `event_id`, `title`, `description`, `start_time`, `end_time`, `speaker_id`, `sequence_order`

2. **Consultas Optimizadas (Repository) - JPQL con Proyecciones:**
   * `EventRepository` método personalizado: Consulta JPQL para obtener métricas agregadas en una única ida a BD
     ```sql
     SELECT NEW map(e.id as eventId, e.name as eventName, COUNT(DISTINCT i.id) as totalInscritos,
            COUNT(DISTINCT CASE WHEN i.asistio = true THEN i.id END) as totalAcreditados)
     FROM Event e
     LEFT JOIN e.inscriptions i
     WHERE e.id = :eventId
     GROUP BY e.id, e.name
     ```
   * `ActivityRepository`: `findByEventIdOrderBySequenceOrder(UUID eventId)`
   * `SurveyRepository`: Consulta para calcular promedio por pregunta con `COUNT` y `AVG`
   * Usar `@Query` con proyecciones de interfaz para evitar cargar entidades completas

3. **DTOs Completos con JSR 303:**
   * `AgendaPublicDTO`: `eventId, eventName, eventType, startDate, endDate, totalDurationMinutes, mainSpeakers, activities`
   * `ActivityDTO`: `id, title, description, startTime, endTime, durationMinutes, speaker, sequenceOrder`
   * `SpeakerDTO`: `id, name, role`
   * `AttendanceMetricsDTO`: `totalRegistered, totalAccredited, attendanceRate, noShowRate`
   * `SurveyMetricsDTO`: `responseCount, responseRate, overallSatisfaction, questionBreakdown`
   * `QuestionBreakdownDTO`: `questionId, questionText, type, average, distribution (Map<String, Long>)`
   * `ReportMetricsDTO`: `eventId, eventName, eventDate, attendanceMetrics, surveyMetrics, generatedAt`
   * Todas las clases con anotaciones Lombok y JSR 303 (`@NotNull`, `@Positive`, `@DecimalMin`, `@DecimalMax`)

4. **Servicios de Negocio:**
   * Crear `AgendaService`:
     * Método `getPublicAgenda(UUID eventId)` → retorna `AgendaPublicDTO`
     * Lógica: Diferencia entre eventos simples (solo evento) y complejos (incluir activities ordenadas)
     * Consultar `events` y `activities` con JOINs optimizados
   * Crear `ReportService`:
     * Método `getEventMetrics(UUID eventId, UUID organizerId)` → Valida permisos y retorna `ReportMetricsDTO`
     * Verifica que organizador esté vinculado al evento
     * Ejecuta consultas JPQL para calcular: inscritos, acreditados, tasas, promedio encuestas
     * Calcula distribución de respuestas por pregunta (histograma)
     * Calcula `responseRate = (respuestasRecibidas / inscritos) * 100`
   * Crear `PdfGeneratorService`:
     * Método `generateAgendaPDF(AgendaPublicDTO)` → retorna `byte[]`
     * Método `generateReportPDF(ReportMetricsDTO)` → retorna `byte[]`
     * Utiliza iText 7 para:
       * Agenda: Tabla con actividades, estilos corporativos, header/footer
       * Informe: Portada, tabla de métricas, tablas de distribución de encuestas

5. **Controladores con OpenAPI:**
   * `AgendaController`:
     * `GET /api/v1/events/{eventId}/agenda` (público)
     * Parámetro query `format` con valores `json` (default), `pdf`
     * Responde según formato: `application/json` o `application/pdf`
   * `ReportController`:
     * `GET /api/v1/events/{eventId}/report` (privado)
     * `GET /api/v1/events/{eventId}/report/pdf` (privado)
     * Inyecta `@RequestAttribute("userId")` y `@RequestAttribute("roles")` del contexto de seguridad
     * Documentar con `@Operation`, `@ApiResponses`, `@Parameter`
     * Validar con `@PathVariable @Positive UUID eventId`

6. **Generación de PDFs con iText 7:**
   * Dependencia: `com.itextpdf:itext7-core:7.2.0+`
   * Clase `PdfGenerator`:
     * `generateAgendaPDF(AgendaPublicDTO)`:
       * Header: Nombre evento, fecha
       * Body: Tabla de actividades (title, horario, disertante, duración)
       * Footer: Página N de M, fecha de generación
       * Márgenes: 1cm, fuente Arial 11pt, tablas con bordes
     * `generateReportPDF(ReportMetricsDTO)`:
       * Página 1: Portada con nombre evento, fecha, organizadores
       * Página 2: Tabla de asistencia (inscritos, acreditados, tasas)
       * Página 3+: Tabla de encuestas con distribución por pregunta, promedios
       * Footer: Documento confidencial, fecha generación

7. **Manejo de Excepciones:**
   * Crear excepciones personalizadas en `com.sgea.exception`:
     * `UnauthorizedReportAccessException` (rol no autorizado)
     * `EventNotFoundException` (evento no existe)
     * `NoActivitiesDefinedException` (evento complejo sin actividades)
   * Actualizar `GlobalExceptionHandler` para mapear a RFC 7807 (ProblemDetail)

8. **Frontend Next.js:**
   * Página `/events/[eventId]/agenda` (pública):
     * Componente `AgendaDisplay`: Renderizar tabla de actividades con horarios, disertantes
     * Botón "Descargar Agenda (PDF)" con ícono de descarga
     * Mostrar disertantes con avatar, nombre y rol
     * Responsive design para móvil y escritorio
   * Página `/dashboard/events/[eventId]/report` (protegida):
     * Middleware: Verificar rol ORGANIZADOR antes de renderizar
     * Componentes:
       * `AttendanceCard`: Tarjeta con números (inscritos vs acreditados), tasas en porcentaje, colores distintivos
       * `SurveyMetricsCard`: Promedio de satisfacción con icono de estrella (4.2/5.0)
       * `SatisfactionChart`: Gráfico de barras/pastel para distribución de respuestas (usar `recharts` o `chart.js`)
       * `ResponseRateMetric`: Porcentaje de respuestas recibidas
       * `QuestionBreakdownTable`: Tabla con desglose de respuestas por pregunta
       * `DownloadReportButton`: Botón para descargar PDF del informe
     * Estados React: `loading`, `error`, `success` con notificaciones toast
     * Polling con `useEffect` para actualizar cada 30 segundos (opcional)

---

## 7. Estrategia de Verificación Detallada

### Pruebas Unitarias (JUnit 5 + Mockito)
* **Test 1 - Cálculo de Tasa de Asistencia:**
  * Setup: Mock repositorio con 50 inscriptos, 25 acreditados
  * Ejecución: `reportService.calculateAttendanceRate(50, 25)`
  * Assert: Retorna `50.0` exactamente, `noShowRate == 50.0`
* **Test 2 - Promedio de Encuestas:**
  * Setup: Respuestas de encuesta con escala 1-5: {2, 3, 4, 4, 5}
  * Ejecución: `reportService.calculateAverageSatisfaction(respuestas)`
  * Assert: Retorna `3.6` con dos decimales
* **Test 3 - Diferenciación Evento Simple vs Complejo:**
  * Setup: Event(isComplex=false) y Event(isComplex=true) con 3 actividades
  * Ejecución: `agendaService.getPublicAgenda(eventSimple)` vs `getPublicAgenda(eventComplejo)`
  * Assert: Simple retorna `activities.isEmpty()`, Complejo retorna `size() == 3`
* **Test 4 - Ordenamiento de Actividades:**
  * Setup: 3 actividades con `sequence_order: 3, 1, 2`
  * Assert: Lista retornada está ordenada `[1, 2, 3]`

### Pruebas de Seguridad (Spring Security Test + @WithMockUser)
* **Test 1 - RBAC: Acceso No Autorizado:**
  * Setup: Usuario con rol PARTICIPANTE
  * Request: GET `/api/v1/events/{eventId}/report`
  * Assert: Status `403 Forbidden`, body contiene `"type": "about:blank/http-exception"`
* **Test 2 - RBAC: Acceso Autorizado:**
  * Setup: Usuario ORGANIZADOR del evento
  * Request: GET `/api/v1/events/{eventId}/report`
  * Assert: Status `200 OK`, JSON retorna `ReportMetricsDTO` con todos los campos
* **Test 3 - Público Sin Autenticación:**
  * Request: GET `/api/v1/events/{eventId}/agenda` sin token
  * Assert: Status `200 OK`, retorna `AgendaPublicDTO` (sin necesidad de login)
* **Test 4 - PDF Restringuido a Organizador:**
  * Setup: Usuario PARTICIPANTE
  * Request: GET `/api/v1/events/{eventId}/report/pdf`
  * Assert: Status `403 Forbidden`

### Pruebas de Integración (Spring Boot Test + TestContainers PostgreSQL)
* **Test 1 - Endpoint Agenda JSON:**
  * Setup: BD real con evento + 2 actividades via Flyway/fixture
  * Request: GET `/api/v1/events/{eventId}/agenda?format=json`
  * Assert: Status `200 OK`, JSON parseable, `activities.length == 2`, ordenadas por `sequence_order`, `totalDurationMinutes` correcto
* **Test 2 - Endpoint Agenda PDF:**
  * Request: GET `/api/v1/events/{eventId}/agenda?format=pdf`
  * Assert: Status `200 OK`, `Content-Type: application/pdf`, header HTTP contiene `%PDF-1.4` (primeros bytes)
* **Test 3 - Endpoint Métricas con Datos Reales:**
  * Setup: 100 inscripciones, 70 marcadas como acreditadas, 50 respuestas de encuesta
  * Request: GET `/api/v1/events/{eventId}/report`
  * Assert: `attendanceMetrics.totalRegistered == 100`, `totalAccredited == 70`, `attendanceRate == 70.0`, `noShowRate == 30.0`
* **Test 4 - Generación de PDF de Informe:**
  * Request: GET `/api/v1/events/{eventId}/report/pdf`
  * Assert: Status `200 OK`, `Content-Type: application/pdf`, verificar que archivo contiene tabla de métricas (parsear con iText API)
* **Test 5 - Validación de Parámetros:**
  * Request: GET `/api/v1/events/invalid-uuid/agenda`
  * Assert: Status `400 Bad Request` (UUID inválido)

### Pruebas de Frontend (Playwright)
* **Test 1 - Página Pública de Agenda:**
  * Navegar a `/events/{eventId}/agenda`
  * Assert: Título del evento visible, tabla con actividades (horarios, disertantes), botón "Descargar PDF" presente
  * Acción: Click en botón descarga → archivo PDF se descarga sin errores
* **Test 2 - Panel de Métricas (Solo Organizador):**
  * Login como Organizador, navegar a `/dashboard/events/{eventId}/report`
  * Assert: Tarjeta de asistencia muestra: "150 inscritos", "120 acreditados", "80%"
  * Assert: Gráfico de encuestas renderiza (canvas visible), promedio de satisfacción: "4.2 / 5.0"
* **Test 3 - Acceso Denegado (Participante):**
  * Login como Participante, intentar navegar a `/dashboard/events/{eventId}/report`
  * Assert: Redirige a `/403` o página de acceso denegado con mensaje
* **Test 4 - Descarga de PDF Funcional:**
  * Click en "Descargar Informe (PDF)"
  * Assert: Archivo descargado con nombre `informe_evento_{eventId}_{timestamp}.pdf`, contiene tablas de métricas
* **Test 5 - Responsividad:**
  * Viewport mobile (375px), verificar que tabla de actividades es scrollable horizontalmente
  * Viewport desktop (1920px), verificar que tabla se renderiza completa