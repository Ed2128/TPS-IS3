# Spec: Gestión de Roles de Usuario

## 1. Objetivo y Contexto
Administrar los niveles de acceso dentro de la plataforma mediante la asignación de roles específicos (organizador, participante, disertante). Esto permite restringir o habilitar funcionalidades críticas según el perfil del usuario autenticado.

## 2. Historias de Usuario y Criterios de Aceptación

**HU-03: Asignación de Rol**
Como **Administrador**, quiero **asignar el rol de Disertante u Organizador a un usuario registrado**, para **otorgarle los permisos necesarios sobre los eventos**.

**Criterios de Aceptación:**
* El endpoint de modificación de roles debe ser accesible únicamente para usuarios con perfil administrador.
* El sistema debe actualizar el rol del usuario de forma persistente.
* El cambio debe verse reflejado en las autorizaciones del usuario en su próxima interacción con el sistema.

## 3. Requisitos Funcionales y Reglas de Negocio

* **RF-03:** El sistema debe proveer un mecanismo para actualizar el campo de rol de un usuario existente.
* **RN-31:** Todo usuario recién registrado en el sistema obtiene el rol de "Participante" por defecto.

## 4. Restricciones técnicas específicas de este módulo

* **Seguridad:** Implementar Spring Security para interceptar la ruta de actualización y requerir explícitamente el rol `ADMIN`.
* **Arquitectura:** Respetar la separación por capas mediante `UsuarioController`, `UsuarioService` y `UsuarioRepository`.
* **Manejo de Errores:** En caso de que un usuario no autorizado intente acceder, el `GlobalExceptionHandler` debe gestionar la respuesta y retornar un HTTP 403 Forbidden.
* **Base de Datos:** El rol se almacenará en la columna `rol` de la tabla `usuarios`, mapeado preferentemente como un Enum de Java.

## 5. Modelo de datos de este módulo

```java
@Entity
@Table(name = "usuarios")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Usuario {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank
    @Column(name = "email")
    private String email;

    @Enumerated(EnumType.STRING)
    @Column(name = "rol")
    private RolUsuario rol; 
}

// Enum para los roles
public enum RolUsuario {
    PARTICIPANTE,
    ORGANIZADOR,
    DISERTANTE,
    ADMIN
}

## 6. Plan de Tareas
Backend (Domain): Crear el enumerador RolUsuario y actualizar la entidad Usuario incorporando este campo.

Backend (Repository): Asegurar que UsuarioRepository permita la actualización de la entidad.

Backend (Service): Implementar el método UsuarioService.actualizarRol(Long usuarioId, RolUsuario nuevoRol).

Seguridad: Modificar la configuración de Spring Security (SecurityConfig) para restringir el nuevo endpoint.

Backend (Controller): Exponer el endpoint PATCH /api/usuarios/{id}/rol.

7. Estrategia de Verificación
Test de Seguridad: Ejecutar una petición HTTP PATCH al endpoint utilizando el token de un usuario con rol PARTICIPANTE y validar que el servidor retorne HTTP 403.

Test de Integración: Ejecutar una petición HTTP PATCH válida con un token de administrador y verificar en la base de datos de prueba que el campo rol del usuario objetivo haya sido actualizado correctamente.