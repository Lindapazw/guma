# RF02 - Inicio de sesión (Backend)

Para el backend del requerimiento **RF02 – Inicio de sesión**, voy a crear un **endpoint POST** llamado  
`/api/auth/login`.  
Este endpoint será el encargado de manejar el proceso de autenticación de los usuarios registrados en el sistema.  

El flujo comienza en el **controlador** (`AuthController`), donde defino el método anotado con `@PostMapping("/login")`. Este método recibe un objeto `LoginRequest`, que contiene los dos datos necesarios para iniciar sesión:  

```java
private String email;
private String contrasena;
```  

Estos campos se relacionan directamente con las columnas de la tabla **USUARIOS**:  
- `email` → columna `email` (`VARCHAR(255) NOT NULL UNIQUE`)  
- `contrasena` → columna `contrasena` (`VARCHAR(255) NOT NULL`)  

Cuando el usuario envía sus datos, el controlador envía la información al **servicio de autenticación** (`AuthService`), donde se implementa toda la lógica del inicio de sesión.  

Dentro del servicio, lo primero que realizo son las **validaciones**:  
- Que el correo electrónico tenga un formato válido.  
- Que la contraseña no esté vacía.  

Si alguna validación falla, el servicio lanza una excepción personalizada, y el controlador responde con un código **HTTP 400 (Bad Request)** junto con un mensaje descriptivo del error.  

Si los datos son válidos, el siguiente paso es **verificar las credenciales**. Para esto, busco el usuario en la tabla `USUARIOS` usando el campo `email`.  
Si el correo no está registrado, o la contraseña no coincide con la almacenada en la columna `contrasena`, devuelvo un código **HTTP 401 (Unauthorized)** con el mensaje `"Credenciales inválidas"`.  

En caso de que la contraseña sea correcta, recupero la información del usuario, incluyendo los campos `id_rol` (que hace referencia al rol del usuario en la tabla `ROLES`) y `verified`, que indica si la cuenta fue verificada.  
Si el campo `verified` es igual a 0, puedo decidir si el sistema permite o no el acceso. Si no lo permite, devuelvo un **HTTP 403 (Forbidden)** indicando que la cuenta aún no ha sido verificada.  

Cuando el inicio de sesión es exitoso, el backend actualiza el campo `ultima_conexion` con la fecha y hora actual. Este valor pertenece a la tabla `USUARIOS` y se usa para registrar cuándo fue la última vez que el usuario accedió al sistema.  

Por último, genero un **token JWT** que contiene los datos básicos del usuario: su `id_usuario`, `email`, `id_rol` y el valor de `verified`. Este token se envía al cliente como parte de la respuesta para que pueda acceder a las funcionalidades del sistema.  

Si todo sale correctamente, el servidor responde con un **HTTP 200 (OK)** y un cuerpo como el siguiente:  
```json
{
  "mensaje": "Inicio de sesión exitoso",
  "usuario": {
    "email": "ejemplo@mail.com",
    "verified": true,
    "idRol": 1
  },
  "token": "jwt-generado"
}
```  

Si las credenciales son incorrectas o el usuario no existe, el backend responde con:  
```json
{
  "mensaje": "Credenciales inválidas"
}
```  

Además, agrego un **manejador global de excepciones** (`@ControllerAdvice`) para interceptar cualquier error inesperado y mantener respuestas consistentes.  

En resumen, este requerimiento implementa toda la **lógica de autenticación** del sistema, utilizando los datos de la tabla **USUARIOS** (campos `email`, `contrasena`, `id_rol`, `verified`, `ultima_conexion`), validando las credenciales ingresadas y gestionando las respuestas según cada caso. También deja registrada la última conexión del usuario y prepara el entorno para la gestión de sesiones mediante tokens, manteniendo la integridad del modelo definido en la base de datos **guma**.