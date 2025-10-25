# RF01 - Registro de usuario (Backend)

Para el backend del requerimiento **RF01 – Registro de usuario**, voy a implementar un **endpoint POST** llamado  
`/api/auth/register`.  
Este endpoint se encargará de recibir desde el cliente tres datos principales: **nombre**, **correo electrónico** y **contraseña**.  

En Java, el flujo comienza en el **controlador** (`UsuarioController`), donde defino un método público anotado con `@PostMapping("/register")`. Ese método recibirá un objeto `UsuarioRequest`, que representa los datos que el usuario ingresa en el formulario. Este objeto tendrá los siguientes campos:  
```java
private String nombre;
private String email;
private String contrasena;
```

Estos datos se corresponden con las columnas definidas en la tabla **USUARIOS** de la base de datos:  
- `email` → columna `email` (`VARCHAR(255) NOT NULL UNIQUE`)  
- `contrasena` → columna `contrasena` (`VARCHAR(255) NOT NULL`)  
- `nombre` → no se guarda directamente en `USUARIOS`, pero se utilizará más adelante al crear el registro en la tabla `PERFIL_USUARIOS`, que tiene la columna `nombre VARCHAR(100) NOT NULL`.  

Una vez recibido el objeto, el controlador lo envía al **servicio** (`UsuarioService`), donde implemento toda la lógica del registro.  

Dentro del servicio, primero valido los datos:  
- Que `nombre` no esté vacío.  
- Que `email` tenga un formato válido (usando una expresión regular).  
- Que `contrasena` tenga al menos 8 caracteres.  

Si alguna validación falla, el servicio lanza una excepción personalizada, que el controlador captura para devolver un mensaje de error con código **HTTP 400 (Bad Request)** y el detalle del problema.  

Si los datos son válidos, el siguiente paso es **simular el registro** creando un objeto `Usuario` con los mismos nombres de atributos que las columnas de la tabla `USUARIOS`. Por ejemplo:  
```java
Usuario nuevoUsuario = new Usuario();
nuevoUsuario.setEmail(request.getEmail());
nuevoUsuario.setContrasena(request.getContrasena());
nuevoUsuario.setIdRol(1); // valor por defecto: usuario estándar
nuevoUsuario.setVerified(false);
nuevoUsuario.setUltimaConexion(null);
```

Por ahora no voy a guardar nada en la base de datos. En cambio, devuelvo una respuesta simulando el resultado que obtendría si el insert se realizara correctamente.  

Si todo salió bien, el backend devuelve un **HTTP 201 (Created)** con un mensaje como:  
```json
{
  "mensaje": "Cuenta creada correctamente",
  "usuario": {
    "email": "ejemplo@mail.com",
    "verified": false
  }
}
```

En caso de error, devuelve algo como:  
```json
{
  "mensaje": "El correo electrónico no tiene un formato válido"
}
```

Finalmente, agrego un **manejador global de excepciones** (`@ControllerAdvice`) para capturar cualquier error inesperado y devolver siempre una respuesta clara y controlada al usuario.  

En resumen, esta primera parte del backend se centra en la **lógica de validación y simulación del registro**, preparando la estructura para que más adelante se conecte con la base de datos **guma** y ejecute el `INSERT` en la tabla **USUARIOS** (campos `email`, `contrasena`, `id_rol`, `verified`, `ultima_conexion`), junto con la creación automática del perfil en **PERFIL_USUARIOS**.