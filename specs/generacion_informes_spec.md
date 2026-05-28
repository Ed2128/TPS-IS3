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

### **HU2: Panel de Métricas del Evento (Enriquecida - OWASP A01:2021)**
Como Organizador, quiero acceder a un *dashboard* con el informe del evento, para analizar su rendimiento general y la recepción del público de forma segura.
* **Criterios de Aceptación:**
  * El sistema debe verificar el rol de "Organizador" del usuario solicitante mediante la validación del Token JWT.
  * **Control de Acceso Basado en Datos (Anti-IDOR):** El sistema debe validar que el `userId` del Organizador extraído del token esté asociado explícitamente en la base de datos como administrador del `eventId` solicitado. Si no hay relación de pertenencia, se debe denegar el acceso.
  * El dashboard debe mostrar el nombre del evento, fecha, duración y una comparativa cuantitativa entre la cantidad de inscriptos y los efectivamente acreditados.
  * Debe incluir un resumen estadístico de las encuestas de satisfacción (ej. promedio de puntuaciones).

### **HU3: Exportación del Informe de Rendimiento (Enriquecida - OWASP A05:2021)**
Como Organizador, quiero descargar el informe consolidado en formato PDF, para tener un respaldo documental de las métricas del evento sin comprometer la disponibilidad del servidor.
* **Criterios de Aceptación:**
  * El PDF generado debe contener toda la información de la HU2, maquetada de forma institucional.
  * **Protección de Recursos (Rate Limiting):** El endpoint de descarga del PDF de rendimiento debe contar con una restricción de tasa de peticiones para mitigar sobrecargas por ráfagas maliciosas.

---

## 3. Requisitos Funcionales y Reglas de Negocio

* **RF1:** El sistema debe calcular dinámicamente la tasa de asistencia real aplicando la fórmula: $Tasa = \frac{Acreditados}{Inscriptos} \times 100$.
* **RF2:** La generación del PDF de la agenda o del informe debe realizarse en el servidor mediante iText 7 y devolverse con `Content-Type: application/pdf`.
* **RF3:** La agenda pública (simple o compleja) debe retornarse en formato JSON con estructura jerárquica. Para eventos complejos, agrupar actividades cronológicamente.
* **RF4:** El endpoint de métricas debe retornar estadísticas calculadas mediante una única consulta JPQL optimizada con `JOIN`, evitando N+1 queries.
* **RF5 (Seguridad - OWASP A01: Broken Access Control):** Validación de Relación de Dominio. El backend interceptará las peticiones dirigidas a los endpoints de reportes y comprobará que el identificador del usuario autenticado posea permisos directos de edición sobre la entidad del evento en cuestión antes de procesar los datos de negocio.
* **RF6 (Seguridad - OWASP A05: Security Misconfiguration / DoS):** Control de Inundación de Recursos. Se implementará una política de Rate Limiting (ej. mediante la librería Bucket4j) en los endpoints de exportación de documentos PDF corporativos, restringiendo a un máximo de 5 solicitudes de descarga por minuto por usuario autenticado.
* **RN1 (Privacidad de Métricas):** Los datos estadísticos (asistencia y encuestas) son de acceso restringido. Cualquier intento de acceso a los endpoints de informes por parte de un usuario sin permisos debe ser denegado.
* **RN2 (Estructura de Eventos):** Un evento simple no requiere carga de actividades secundarias, mientras que un evento complejo requiere al menos una (1) actividad programada en su agenda.

---

## 4. Restricciones Técnicas Específicas

* **Arquitectura y Stack:** Desarrollo bajo patrón MVC/Capas con Spring Boot 3.x (Java 21). El frontend en Next.js consumirá la API para renderizar las vistas y los gráficos de métricas.
* **DTOs y Proyección:** Uso estricto de DTOs (`AgendaPublicDTO`, `ReportMetricsDTO`). Para consultas complejas de métricas, se deben utilizar proyecciones JPA o consultas JPQL/Native optimizadas, evitando cargar entidades completas en memoria.
* **Manejo de Errores:** Las violaciones de acceso (ej. un participante queriendo ver las métricas o un organizador consultando un evento ajeno) deben lanzar una `UnauthorizedReportAccessException`, la cual será interceptada por el `GlobalExceptionHandler` devolviendo un HTTP 403 Forbidden bajo el estándar RFC 7807 (ProblemDetail).
* **Validación de Entradas:** Los parámetros de búsqueda (como el ID del evento) en los Controladores deben estar validados con Bean Validation (`@NotNull`, `@Positive`, `@UUID`).
* **Librerías de Exportación:** Se deberá utilizar **iText 7** (biblioteca `com.itextpdf:itext7-core`) para generación de PDFs con soporte para tablas, estilos y múltiples páginas. Configurar en `pom.xml` con versión ≥ 7.2.0.
* **Seguridad en Descargas y Sanitización de Salidas (OWASP A03:2021-Injection):** Los archivos PDF deben ser generados bajo demanda (streaming) sin almacenarlos en disco. Header: `Content-Disposition: attachment; filename=\"agenda_evento_{eventId}_{timestamp}.pdf\"`. Todos los campos de texto dinámicos procedentes de la base de datos (títulos, descripciones o respuestas textuales de encuestas) deben ser sanitizados previamente para evitar ataques de inyección de contenido en el motor de renderizado del documento.
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

*(Nota: Las métricas de inscriptos, acreditados y encuestas se calculan a partir de uniones SQL `JOIN` y funciones de aggregación `COUNT`, `AVG` sobre las tablas transaccionales definidas en specs anteriores, sin necesidad de persistirlas en tablas nuevas).*

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

### Métricas del Evento (Solo Organizadores Autorizados - OWASP A01)
* **GET** `/api/v1/events/{eventId}/report`
  * Requiere: Token JWT, rol ORGANIZADOR asociado explícitamente como dueño/gestor de dicho evento en BD.
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
  * Excepciones: `403 Forbidden` (sin rol u organizador no vinculado al recurso), `404 Not Found` (evento inexistente)

### Exportación de Informes Protegida (Solo Organizadores Autorizados - OWASP A05)
* **GET** `/api/v1/events/{eventId}/report/pdf`
  * Requiere: Token JWT, rol ORGANIZADOR vinculado al evento. Aplica control de cuotas Bucket4j.
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
     * **Validación Cruzada Anti-IDOR:** Comprueba en base de datos la relación explícita entre el `organizerId` solicitante y el `eventId`.
     * Ejecuta consultas JPQL para calcular: inscritos, acreditados, tasas, promedio encuestas
     * Calcula distribución de respuestas por pregunta (histograma)
     * Calcula `responseRate = (respuestasRecibidas / inscritos) * 100`
   * Crear `PdfGeneratorService`:
     * Método `generateAgendaPDF(AgendaPublicDTO)` → retorna `byte[]`
     * Método `generateReportPDF(ReportMetricsDTO)` → retorna `byte[]`
     * Utiliza iText 7 aplicando **sanitización estricta de cadenas de caracteres** contra ataques de inyección.

5. **Controladores con OpenAPI e Interceptores de Seguridad:**
   * `AgendaController`: Endpoint público para formatos JSON/PDF.
   * `ReportController`: Manejo de endpoints privados inyectando `@RequestAttribute("userId")`. Integra el filtro/interceptor de control de tasa (Rate Limiting) de Spring Boot para bloquear ráfagas abusivas sobre solicitudes binarias (.pdf).

---

## 7. Estrategia de Verificación Detallada

### Pruebas Unitarias (JUnit 5 + Mockito)
* **Test 1 - Cálculo de Tasa de Asistencia:** Mock con 50 inscriptos, 25 acreditados. Retorna `50.0` exactamente, `noShowRate == 50.0`.
* **Test 2 - Promedio de Encuestas:** Respuestas de encuesta con escala 1-5: {2, 3, 4, 4, 5}. Retorna `3.6` con dos decimales.
* **Test 3 - Diferenciación Evento Simple vs Complejo:** Comprueba bifurcación de lógica según bandera `isComplex`.
* **Test 4 - Ordenamiento de Actividades:** 3 actividades con `sequence_order: 3, 1, 2` son devueltas en orden `[1, 2, 3]`.

### Pruebas de Seguridad (Spring Security Test + OWASP Verification)
* **Test 1 - RBAC: Acceso No Autorizado:** Usuario con rol PARTICIPANTE. Request: GET `/api/v1/events/{eventId}/report`. Assert: Status `403 Forbidden`.
* **Test 2 - RBAC: Acceso Autorizado:** Usuario ORGANIZADOR del evento correspondiente. Request: GET `/api/v1/events/{eventId}/report`. Assert: Status `200 OK`.
* **Test 3 - Público Sin Autenticación:** Request a endpoint de agenda sin cabecera Authorization. Assert: Status `200 OK`.
* **Test 4 - PDF Restringido a Organizador:** Usuario PARTICIPANTE solicita PDF de reportes. Assert: Status `403 Forbidden`.
* **Test 5 - Prevención de IDOR (OWASP A01):**
  * Setup: Usuario autenticado con rol `ORGANIZADOR` asociado a un "Evento A".
  * Request: GET `/api/v1/events/{evento_B}/report` (intento de acceso de lectura a datos de un evento ajeno).
  * Assert: Status `403 Forbidden` (La validación de dominio del backend detecta la incompatibilidad e interrumpe el flujo).
* **Test 6 - Mitigación DoS por Inundación (OWASP A05):**
  * Setup: Realizar 6 peticiones consecutivas al endpoint `/api/v1/events/{eventId}/report/pdf` en menos de un minuto usando las mismas credenciales.
  * Assert: Las primeras 5 peticiones devuelven `200 OK`, la sexta petición retorna de inmediato un Status `429 Too Many Requests`.

### Pruebas de Integración (Spring Boot Test + TestContainers PostgreSQL)
* Cobertura completa sobre parseo JSON/PDF y mapeo de excepciones personalizadas bajo estándar RFC 7807.

### Pruebas de Frontend (Playwright)
* Verificación de responsividad móvil, estados React ante errores 403/429 y renderizado de gráficos estadísticos.
