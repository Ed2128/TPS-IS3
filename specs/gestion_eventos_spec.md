# Spec: Gestión de Eventos

## 1. Objetivo y Contexto
Permitir a los administradores u organizadores crear y configurar nuevos eventos académicos en el sistema. Esto define detalles cruciales como fechas de realización y cupos límite para que el evento sea público y los participantes puedan inscribirse.

## 2. Historias de Usuario y Criterios de Aceptación

**HU-02: Creación de Evento**
Como **Organizador**, quiero **crear un nuevo evento estableciendo sus fechas y cupos**, para **que los participantes puedan inscribirse posteriormente**.

**Criterios de Aceptación:**
* El sistema debe requerir obligatoriamente el título, tipo de evento, fecha de inicio, fecha de fin y cupo máximo.
* Tras la creación exitosa, el evento debe listarse en la vista pública.
* El sistema debe permitir filtrar el evento recién creado como un evento a futuro.

## 3. Requisitos Funcionales y Reglas de Negocio

* **RF-02:** El sistema debe registrar los datos del evento en la base de datos y generar un identificador único.
* **RN-21:** La fecha de inicio ingresada debe ser estrictamente mayor a la fecha y hora actual del sistema.
* **RN-22:** El sistema debe validar que las inscripciones no superen el cupo máximo definido en la creación.

## 4. Restricciones técnicas específicas de este módulo

* **Framework:** Spring Boot 3.x con Java 21.
* **Validación:** Implementar Bean Validation (JSR 303) usando anotaciones como `@FutureOrPresent` para las fechas y `@Min` / `@Max` para los cupos.
* **Manejo de Errores:** Las excepciones de validación deben ser interceptadas por el `GlobalExceptionHandler`, retornando una respuesta estándar con código HTTP 400.
* **Inmutabilidad:** La recepción de datos desde el frontend (Next.js) debe realizarse exclusivamente mediante un DTO (`EventoCreateDTO`).
* **Base de Datos:** Los datos se persistirán en la tabla `eventos`, respetando la nomenclatura en plural y snake_case en PostgreSQL.

## 5. Modelo de datos de este módulo

```java
@Entity
@Table(name = "eventos")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Evento {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "El título es obligatorio")
    @Column(name = "titulo")
    private String titulo;

    @NotNull(message = "La fecha de inicio es obligatoria")
    @FutureOrPresent(message = "La fecha debe ser actual o futura")
    @Column(name = "fecha_inicio")
    private LocalDateTime fechaInicio;

    @Column(name = "cupo_maximo")
    private Integer cupoMaximo;
}
## 6. Plan de Tareas
Base de Datos: Generar script de migración SQL para crear la tabla eventos.

Backend (Domain): Crear la entidad Evento y el DTO EventoCreateDTO.

Backend (Repository): Implementar EventoRepository extendiendo de JpaRepository.

Backend (Service): Desarrollar EventoService.crearEvento() aplicando las reglas de negocio de fechas.

Backend (Controller): Desarrollar EventoController exponiendo el endpoint POST /api/eventos.

Documentación: Registrar el endpoint y sus modelos en Swagger/OpenAPI.

## 7. Estrategia de Verificación
Test Unitario (Service): Verificar que el método de creación lance una excepción si se envía una fecha de inicio en el pasado.

Test de Integración (Controller): Simular una petición HTTP POST con un cuerpo JSON inválido (sin título) y comprobar que el sistema responda con un HTTP 400 Bad Request.