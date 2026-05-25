# Spec: Inscripción de Participantes (Híbrida)

## 1. Objetivo y Contexto
Gestionar el registro de asistentes al evento, permitiendo tanto la auto-inscripción de usuarios registrados en la plataforma como la carga manual por parte de administradores para participantes externos. El sistema debe garantizar la integridad de los datos y el control de cupos en tiempo real.

## 2. Historias de Usuario y Criterios de Aceptación
**HU-03: Inscripción Autónoma (Virtual)**
* **Como** Usuario Registrado, **quiero** inscribirme a un evento desde la web, **para** asegurar mi lugar de forma inmediata.

**HU-04: Inscripción Manual (Ventanilla)**
* **Como** Administrador, **quiero** cargar los datos de una persona que no tiene usuario, **para** que figure en la lista de asistencia y reciba su certificado luego.

**Criterios de Aceptación:**
* El sistema debe permitir la inscripción manual solicitando únicamente DNI, Nombre y Email.
* Al iniciar el formulario de inscripción, el sistema debe realizar una "reserva" temporal del cupo para evitar sobreventas.
* El sistema debe validar que el DNI ingresado no esté ya inscripto en el mismo evento.
- **Control de Seguridad (OWASP - Broken Access Control & Mass Assignment):** El sistema debe garantizar que el endpoint de inscripción manual (`/api/inscripciones/manual`) rechace cualquier solicitud que no provenga de una sesión activa con rol `ADMIN`. Asimismo, se debe validar estrictamente que los DTOs de entrada sean inmutables y no permitan la inyección de parámetros heredados (Mass Assignment) para alterar campos críticos como el `id_usuario` en registros manuales.

## 3. Requisitos Funcionales y Reglas de Negocio
* **RN-13 (Restricción de Acceso):** Para la inscripción virtual, es obligatorio haber iniciado sesión en la plataforma.
* **RN-14 (Inscripción Manual):** El Administrador puede inscribir personas sin cuenta; estos datos se consideran "efímeros" para fines del evento.
* **RN-15 (Regla del "Empecé antes"):** Si un usuario inicia el trámite antes de la hora de cierre, el sistema debe permitirle finalizarlo aunque el horario límite expire durante el proceso.
* **RN-16 (Validación de Identidad):** No se permiten registros duplicados con el mismo número de DNI para un mismo evento.

## 4. Restricciones técnicas específicas
* **Framework:** Spring Boot 3.x con Java 21.
* **Gestión de Concurrencia:** Usar @Transactional y bloqueo optimista (version) en JPA para reservas temporales; implementar expiración con ScheduledExecutorService o Quartz para liberar cupos expirados.
* **Seguridad:** Inscripción virtual requiere rol `PARTICIPANTE` (autenticado). Inscripción manual requiere rol `ADMIN`. Usar Spring Security.
* **Validación:** Aplicar Bean Validation (JSR 303) con anotaciones como `@NotNull`, `@Email`, `@Pattern` en DTOs y entidades.
* **Manejo de Errores:** Implementar GlobalExceptionHandler para respuestas consistentes (ej. 409 para cupo lleno, 400 para validaciones).
* **Inmutabilidad:** Usar DTOs para requests/responses (ej. InscripcionRequestDTO inmutable).
* **Base de Datos:** Tablas en plural y snake_case (ej. `inscripciones`). Usar JPA/Hibernate con PostgreSQL; índices en `dni_participante` y `id_evento` para unicidad.
* **Documentación API:** Incluir endpoints en Swagger/OpenAPI.
* **Frontend:** Next.js para formularios; consumir APIs REST con manejo de concurrencia (polling para reservas).
- **Seguridad (Mitigación R3):** Interceptar los endpoints de inscripción mediante Spring Security. [cite_start]El endpoint `/api/inscripciones/virtual` se blindará con `@PreAuthorize("hasRole('PARTICIPANTE')")` y el endpoint `/api/inscripciones/manual` con `@PreAuthorize("hasRole('ADMIN')")`[cite: 4]. [cite_start]Cualquier petición que no adjunte un token JWT válido con las declaraciones de rol (*claims*) correspondientes será rechazada en la capa de filtrado de seguridad antes de procesar la lógica de negocio[cite: 4].
- **Manejo Seguro de Errores de Autorización:** Cuando Spring Security lance una excepción de tipo `AccessDeniedException`, el `GlobalExceptionHandler` debe interceptarla y transformarla en una respuesta estructurada con código de estado `HTTP 403 Forbidden`, omitiendo cualquier detalle técnico de las trazas del sistema (*Stack Traces*) para evitar el reconocimiento de la arquitectura por parte de atacantes externos.

## 5. Modelo de datos de este módulo
Entidad **Inscripcion** completa con JPA:

```java
@Entity
@Table(name = "inscripciones", uniqueConstraints = @UniqueConstraint(columnNames = {"dni_participante", "id_evento"}))
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Inscripcion {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull
    @Column(name = "id_evento")
    private Long idEvento;

    // Para virtual
    @Column(name = "id_usuario")
    private Long idUsuario;

    // Para manual
    @Column(name = "dni_participante")
    private Long dniParticipante;

    @Email
    @Column(name = "email_contacto")
    private String emailContacto;

    @Enumerated(EnumType.STRING)
    @Column(name = "tipo_registro")
    private TipoRegistro tipoRegistro; // VIRTUAL, MANUAL_ADMIN

    @Column(name = "fecha_inicio_tramite")
    private LocalDateTime fechaInicioTramite;

    @Column(name = "estado_cupo")
    private String estadoCupo; // RESERVADO, CONFIRMADO, EXPIRADO

    @Version
    private Long version; // Para bloqueo optimista

    // Relaciones opcionales
    @ManyToOne
    @JoinColumn(name = "id_evento", insertable = false, updatable = false)
    private Evento evento;
}

public enum TipoRegistro {
    VIRTUAL, MANUAL_ADMIN
}
```

DTOs:
```java
@Data
@Builder
public class InscripcionRequestDTO {
    private final Long idEvento;
    private final Long dniParticipante; // Para manual
    private final String emailContacto;
    private final TipoRegistro tipoRegistro;
}

@Data
@Builder
public class InscripcionResponseDTO {
    private final Long id;
    private final String estadoCupo;
    private final LocalDateTime fechaInicioTramite;
}
```

## 6. Plan de Tareas
1. **Base de Datos:** Ejecutar migración para crear tabla `inscripciones` con constraints únicos en `dni_participante` + `id_evento`. Agregar índices y campos para concurrencia.
2. **Backend - Entidad y Repository:** Crear `Inscripcion` entity con validaciones. Implementar `InscripcionRepository` extendiendo `JpaRepository<Inscripcion, Long>` con métodos como `findByDniParticipanteAndIdEvento(Long dni, Long eventoId)`, `findReservasExpiradas(LocalDateTime now)`.
3. **Backend - Service:** Implementar `InscripcionService` con métodos:
   - `InscripcionResponseDTO iniciarInscripcion(InscripcionRequestDTO request)`: Reservar cupo temporalmente, validar unicidad DNI, setear `fechaInicioTramite` y `estadoCupo = RESERVADO`.
   - `void confirmarInscripcion(Long inscripcionId)`: Cambiar a `CONFIRMADO` si no expirado.
   - `void liberarReservasExpiradas()`: Método programado para expirar reservas >10 min.
   - `boolean validarHorarioLimite(Long inscripcionId)`: Comparar `fechaInicioTramite` con `evento.fechaLimite`.
4. **Backend - Controller:** Crear `InscripcionController` con endpoints:
   - `POST /api/inscripciones/virtual`: Para inscripción virtual (rol PARTICIPANTE), validar sesión.
   - `POST /api/inscripciones/manual`: Para inscripción manual (rol ADMIN), sin requerir usuario.
   - `PUT /api/inscripciones/{id}/confirmar`: Confirmar reserva.
5. **Backend - Excepciones:** Crear `InscripcionException` (ej. CupoLlenoException, DniDuplicadoException), manejadas por `GlobalExceptionHandler`.
6. **Backend - Scheduler:** Usar `@Scheduled` para liberar reservas expiradas periódicamente.
7. **Frontend (Virtual):** En Next.js, formulario de inscripción con validación en tiempo real; llamar a POST virtual y manejar reserva con polling.
8. **Frontend (Admin):** Vista para carga manual con campos DNI/Email; llamar a POST manual.
9. **Documentación:** Agregar endpoints a Swagger con ejemplos de request/response.

## 7. Estrategia de Verificación
* **Test de Horario Límite:** Iniciar inscripción a las 22:59 (cierre 23:00) y terminar a las 23:02; el sistema debe procesarla exitosamente validando RN-15.
* **Test de DNI Duplicado:** Intentar inscribir manualmente un DNI ya registrado virtualmente y verificar `DniDuplicadoException` con mensaje de error.
* **Test Unitario - Service:** Verificar `iniciarInscripcion()` reserve cupo y lance excepción si DNI duplicado. Probar `liberarReservasExpiradas()` expire reservas correctamente.
* **Test de Integración - Controller:** Simular POST a `/api/inscripciones/virtual` con usuario autenticado y verificar reserva en BD. Simular POST manual sin auth y verificar creación.
* **Test de Concurrencia:** Simular múltiples requests simultáneos para el último cupo y verificar que solo uno confirme.
* **Test de Seguridad:** Verificar que endpoint virtual rechace usuarios sin rol PARTICIPANTE (HTTP 403); endpoint manual requiera ADMIN.
* **Test de Validación:** Enviar request con email inválido y verificar Bean Validation errores (ej. 400 con detalles).
* **Test E2E:** En frontend, iniciar inscripción virtual, confirmar, y verificar en admin la lista; probar carga manual y descarga de certificado posterior.
.
## Desarrollo finalizado y verificado