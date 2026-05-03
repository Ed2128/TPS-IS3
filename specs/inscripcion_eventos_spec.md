# Spec: Gestión de Inscripciones y Control de Cupos

## 1. Objetivo y Contexto
Este módulo regula el proceso por el cual un usuario se registra en un evento. Es crítico porque debe garantizar que no se sobrepase el cupo máximo y que las inscripciones se realicen dentro del periodo permitido.

## 2. Historias de Usuario y Criterios de Aceptación
### HU-01: Inscripción Autónoma
**Como** participante registrado  
**Quiero** inscribirme a un evento académico  
**Para** asegurar mi lugar y recibir información posterior.

**Criterios de Aceptación:**
- **Caso Exitoso:** Si hay cupo y la fecha actual es anterior a la fecha límite, registrar la inscripción con estado "CONFIRMADA".
- **Caso Cupo Lleno:** Si el número de inscritos es igual al `cupo_maximo`, el sistema debe rechazar la solicitud con un mensaje de error[cite: 2].
- **Caso Fecha Excedida:** Si la fecha actual es posterior a `fecha_limite_inscripcion`, la operación se rechaza[cite: 2].

## 3. Requisitos Funcionales y Reglas de Negocio
- **RF-01:** Validar existencia del usuario y del evento antes de procesar.
- **RN-01:** Un usuario no puede inscribirse dos veces al mismo evento.
- **RN-02:** Si el evento tiene `cupo_minimo` y no se alcanza, el sistema debe permitir inscripciones igualmente, pero marcar el evento como "Pendiente de Confirmación".

## 4. Restricciones Técnicas
- **Framework:** Spring Boot 3.x.
- **Seguridad:** Solo usuarios con rol `PARTICIPANTE` pueden acceder al endpoint de inscripción.
- **Validación:** Usar anotaciones `@Min` y `@NotNull` en los modelos

## 5. Modelo de Datos
Entidad `Inscripcion`:
- `id`: Long (PK)
- `id_usuario`: Long (FK)
- `id_evento`: Long (FK)
- `fecha_inscripcion`: LocalDateTime
- `estado`: String (CONFIRMADA, CANCELADA)

## 6. Plan de Tareas
1. Crear la entidad `Inscripcion` con JPA.
2. Crear `InscripcionRepository` extendiendo de `JpaRepository`.
3. Implementar `InscripcionService` con la lógica de validación de cupos y fechas.
4. Crear `InscripcionController` con un método POST `/api/inscripciones`.

## 7. Estrategia de Verificación
- **Test Unitario:** Verificar que el método de servicio lance una excepción personalizada (`CupoExcedidoException`) cuando el cupo esté lleno.
- **Test de Integración:** Simular una petición POST y verificar que se cree el registro en la base de datos H2/PostgreSQL.