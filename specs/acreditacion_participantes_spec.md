# Spec: Sistema de Acreditación de Participantes

## 1. Objetivo y Contexto
Este módulo gestiona el registro de asistencia en tiempo real durante el evento académico. Permite al personal del evento marcar a los participantes como "Acreditados", lo cual es una condición obligatoria para la posterior generación de certificados de asistencia.

## 2. Historias de Usuario y Criterios de Aceptación
### HU-02: Acreditación por Personal del Evento
**Como** personal del evento (Rol: Organizador/Staff)  
**Quiero** registrar la entrada de un participante  
**Para** validar su asistencia y permitir su ingreso al recinto.

**Criterios de Aceptación:**
- **Validación de Inscripción:** El sistema solo debe permitir acreditar a usuarios que tengan una inscripción previa con estado "CONFIRMADA".
- **Estado Duplicado:** Si un usuario ya ha sido acreditado, el sistema debe mostrar un aviso indicando la hora de la primera acreditación y no duplicar el registro.
- **Feedback Visual:** Al realizar la acreditación exitosa, debe mostrar los datos principales del participante (Nombre, DNI/ID y Tipo de Participante).

## 3. Requisitos Funcionales y Reglas de Negocio
- **RF-02:** El sistema debe permitir la búsqueda de inscritos por ID de usuario o nombre.
- **RN-03:** Solo se puede realizar la acreditación en la fecha de realización del evento establecida en la gestión del mismo.
- **RN-04:** Una vez acreditado, el campo `asistencia` en el registro de inscripción debe cambiar a `TRUE`.

## 4. Restricciones Técnicas
- **Endpoints:** Implementar `/api/acreditacion/{eventoId}/{usuarioId}` usando el verbo PATCH o PUT para actualizar el estado de asistencia.
- **Seguridad:** Acceso restringido mediante Spring Security. Solo usuarios con roles `ORGANIZADOR` o `STAFF` pueden ejecutar esta acción.
- **Rendimiento:** La consulta de búsqueda de participantes debe estar indexada por DNI/ID para garantizar respuestas en milisegundos.

## 5. Modelo de Datos
Este módulo interactúa principalmente con la entidad `Inscripcion` definida en la spec anterior, añadiendo o actualizando:
- `asistio`: Boolean (Default: false)
- `fecha_acreditacion`: LocalDateTime (Auditoría de cuándo ingresó)

## 6. Plan de Tareas
1. Modificar la entidad `Inscripcion` para incluir los campos de asistencia.
2. Crear un método en `InscripcionRepository` para buscar por `eventoId` y `usuarioId`.
3. Desarrollar la lógica en `AcreditacionService` que valide la fecha del evento antes de marcar la asistencia.
4. Crear el controlador para la interfaz de acreditación.

## 7. Estrategia de Verificación
- **Prueba de Regla de Negocio:** Intentar acreditar a un usuario en un evento que no es el día de la fecha actual y verificar que el sistema devuelva un error de "Evento no iniciado".
- **Prueba de Integración:** Verificar que el cambio en el estado `asistio` se persista correctamente en la base de datos PostgreSQL