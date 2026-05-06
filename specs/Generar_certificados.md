# Spec: Generación de Certificados PDF

## 1. Objetivo y Contexto
Automatizar la entrega de comprobantes de asistencia para que los alumnos puedan descargarlos de forma autónoma desde la plataforma. Este módulo depende directamente de que el evento haya finalizado y de que se haya registrado la asistencia del usuario.

## 2. Historias de Usuario y Criterios de Aceptación
**HU-01: Descarga de Comprobante**
* **Como** Participante del evento, **quiero** descargar mi certificado de asistencia en formato PDF, **para** poder acreditar el logro en mi currículum.

**Criterios de Aceptación:**
* El PDF debe incluir: Nombre del evento, nombre del alumno, DNI y fecha.
* Al hacer clic en "Descargar", el archivo debe abrirse en una pestaña nueva o descargarse automáticamente.

## 3. Requisitos Funcionales y Reglas de Negocio
* **RF-01:** El sistema debe generar el archivo PDF usando los datos de la base de datos.
* **RN-11:** Solo se habilita la descarga si el campo asistio es igual a true.
* **RN-12:** No se permiten descargas si el evento aún no ha finalizado.

## 4. Restricciones técnicas específicas
* **Framework:** Spring Boot 3.x con Java 21.
* **Librería PDF:** Usar iText (compatible con Java) para generar PDFs en el servidor.
* **Seguridad:** Solo usuarios con rol `PARTICIPANTE` pueden acceder al endpoint de descarga. Usar Spring Security.
* **Validación:** Aplicar Bean Validation (JSR 303) con anotaciones como `@NotNull`, `@AssertTrue` en DTOs y entidades.
* **Manejo de Errores:** Implementar GlobalExceptionHandler para respuestas consistentes (ej. 403 para acceso denegado, 400 para reglas de negocio).
* **Inmutabilidad:** Usar DTOs para comunicación externa (ej. CertificadoDTO con campos inmutables).
* **Base de Datos:** Tablas en plural y snake_case (ej. `inscripciones`). Usar JPA/Hibernate con PostgreSQL.
* **Documentación API:** Incluir endpoints en Swagger/OpenAPI.
* **Frontend:** Next.js para vistas; consumir APIs REST.

## 5. Modelo de datos de este módulo
Este módulo utiliza y añade campos a la entidad de Inscripción. Entidad completa con JPA:

```java
@Entity
@Table(name = "inscripciones")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Inscripcion {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Column(name = "id_usuario")
    private Long idUsuario;

    @NotNull
    @Column(name = "id_evento")
    private Long idEvento;

    @Column(name = "fecha_inscripcion")
    private LocalDateTime fechaInscripcion;

    @Column(name = "estado")
    private String estado;

    // Nuevos campos para certificados
    @Column(name = "asistio")
    @AssertTrue(message = "Debe marcar asistencia para generar certificado")
    private Boolean asistio = false;

    @Column(name = "fecha_emision")
    private LocalDateTime fechaEmision;

    @Column(name = "url_certificado")
    private String urlCertificado;

    // Relaciones (opcional, si se definen)
    @ManyToOne
    @JoinColumn(name = "id_usuario", insertable = false, updatable = false)
    private Usuario usuario;

    @ManyToOne
    @JoinColumn(name = "id_evento", insertable = false, updatable = false)
    private Evento evento;
}
```

DTO para respuestas:
```java
@Data
@Builder
public class CertificadoDTO {
    private final String nombreEvento;
    private final String nombreAlumno;
    private final String dni;
    private final LocalDate fecha;
}
```

## 6. Plan de Tareas
1. **Base de Datos:** Ejecutar migración SQL o usar Flyway para agregar columnas `asistio` (BOOLEAN DEFAULT FALSE), `fecha_emision` (TIMESTAMP), `url_certificado` (VARCHAR(255)) a la tabla `inscripciones`.
2. **Backend - Entidad y Repository:** Actualizar `Inscripcion` entity con nuevos campos y validaciones. Crear/actualizar `InscripcionRepository` extendiendo `JpaRepository<Inscripcion, Long>` con métodos como `findByIdUsuarioAndIdEventoAndAsistioTrue(Long idUsuario, Long idEvento)`.
3. **Backend - Service:** Implementar `CertificadoService` con métodos:
   - `boolean validarAsistencia(Long inscripcionId)`: Verificar `asistio == true` y evento finalizado.
   - `byte[] generarPDF(Long inscripcionId)`: Usar iText para crear PDF con datos de `CertificadoDTO`, lanzar `CertificadoException` si falla.
   - `void marcarAsistencia(Long inscripcionId)`: Actualizar `asistio` a true.
4. **Backend - Controller:** Crear `CertificadoController` con endpoints:
   - `POST /api/certificados/{inscripcionId}/marcar-asistencia`: Para admin marcar asistencia (rol ADMIN).
   - `GET /api/certificados/{inscripcionId}/descargar`: Generar y devolver PDF como stream (rol PARTICIPANTE), validar reglas.
5. **Backend - Excepciones:** Crear `CertificadoException` para errores específicos, manejada por `GlobalExceptionHandler`.
6. **Frontend (Admin):** En Next.js, crear vista `/admin/eventos/{id}/asistencias` con lista de inscritos y botón "Marcar Presente" que llame a POST endpoint.
7. **Frontend (Usuario):** En perfil de usuario, mostrar botón "Descargar Certificado" solo si `asistio == true` y evento finalizado; llamar a GET endpoint para descarga.
8. **Documentación:** Agregar endpoints a Swagger con ejemplos de request/response.

## 7. Estrategia de Verificación
* **Prueba de Inasistencia:** Intentar descargar el certificado con un usuario que tiene `asistio = false` y verificar que el sistema lance `CertificadoException` con mensaje "Asistencia no registrada" o no muestre el botón.
* **Test Unitario - Service:** Verificar `validarAsistencia()` lance excepción si `asistio = false` o evento no finalizado. Probar `generarPDF()` cree archivo válido con iText.
* **Test de Integración - Controller:** Simular POST a `/api/certificados/{id}/marcar-asistencia` y verificar actualización en BD. Simular GET a `/api/certificados/{id}/descargar` con usuario autorizado y verificar PDF descargado.
* **Test de Seguridad:** Verificar que endpoint de descarga rechace usuarios sin rol PARTICIPANTE (HTTP 403).
* **Test de Validación:** Enviar request inválido y verificar Bean Validation errores (ej. campos nulos).
* **Test E2E:** En frontend, marcar asistencia como admin, luego descargar como usuario y validar contenido PDF.