# RF03 - Crear/Actualizar perfil de usuario (Backend)

Para el backend del requerimiento **RF03 – Crear/Actualizar perfil de usuario**, voy a implementar un **endpoint POST o PUT** (dependiendo de si se crea o se actualiza) llamado  
`/api/perfil` o `/api/perfil/{id}`.  
Este endpoint se encargará de crear o modificar la información del perfil personal del usuario autenticado, asociándolo al registro correspondiente en la tabla `USUARIOS`.  

El flujo comienza en el **controlador** (`PerfilUsuarioController`), donde defino los métodos `@PostMapping` y `@PutMapping`.  
Ambos reciben un objeto `PerfilUsuarioRequest`, que contendrá los campos que el usuario puede registrar o actualizar desde el módulo “Mi Perfil”:  

```java
private String nombre;
private String apellido;
private Integer dni;
private String email;
private String telefono;
private Integer idSexo;
private Integer idDireccion;
private Integer idRedSocial;
private Integer fotoPerfil;
private Date fechaNacimiento;
```

Estos campos corresponden directamente con las columnas definidas en la tabla **PERFIL_USUARIOS**:  
- `nombre` → `nombre VARCHAR(100) NOT NULL`  
- `apellido` → `apellido VARCHAR(100) NOT NULL`  
- `dni` → `dni INT NOT NULL UNIQUE`  
- `email` → `email VARCHAR(255) NOT NULL UNIQUE`  
- `telefono` → `telefono VARCHAR(20)`  
- `id_sexo` → FK a `SEXOS(id_sexo)`  
- `id_direccion` → FK a `DIRECCIONES(id_direccion)`  
- `id_red_social` → FK a `REDES_SOCIALES(id_red_social)`  
- `foto_perfil` → FK a `IMAGES(id_images)`  
- `fecha_nacimiento` → `DATE NOT NULL`  
- `verificado` → `TINYINT(1) DEFAULT 0`  
- `id_usuario` → FK a `USUARIOS(id_usuario)`  

El **servicio** (`PerfilUsuarioService`) se encarga de la lógica principal. Primero verifica que el usuario esté autenticado (utilizando su `id_usuario` obtenido del token). Luego, realiza las validaciones necesarias:  
- Que el campo `nombre` no esté vacío.  
- Que el `email` tenga formato válido.  
- Que el `dni` sea un número válido y no esté repetido.  
- Que los IDs referenciados (`id_sexo`, `id_direccion`, `id_red_social`, `foto_perfil`) existan en sus tablas correspondientes.  

Si el usuario aún no tiene un perfil, el backend ejecuta un **INSERT** en la tabla `PERFIL_USUARIOS`, asociando el nuevo registro al `id_usuario` autenticado.  
Si ya existe un perfil, el backend ejecuta un **UPDATE** sobre ese mismo registro, modificando solo los campos que fueron enviados en la solicitud.  

Por ejemplo, para crear un nuevo perfil, el backend prepararía algo así:  
```java
PerfilUsuario perfil = new PerfilUsuario();
perfil.setIdUsuario(usuarioAutenticado.getId());
perfil.setNombre(request.getNombre());
perfil.setApellido(request.getApellido());
perfil.setDni(request.getDni());
perfil.setEmail(request.getEmail());
perfil.setTelefono(request.getTelefono());
perfil.setIdSexo(request.getIdSexo());
perfil.setFechaNacimiento(request.getFechaNacimiento());
perfil.setVerificado(false);
```

Una vez que los datos son validados y persistidos, el sistema devuelve una respuesta:  
- Si se creó un nuevo perfil, responde con **HTTP 201 (Created)** y un mensaje `"Perfil creado correctamente"`.  
- Si se actualizó un perfil existente, responde con **HTTP 200 (OK)** y un mensaje `"Perfil actualizado correctamente"`.  

Ejemplo de respuesta exitosa:  
```json
{
  "mensaje": "Perfil actualizado correctamente",
  "perfil": {
    "nombre": "Marina",
    "apellido": "Gómez",
    "dni": 44123890,
    "email": "marina.gomez@mail.com",
    "telefono": "3815012244"
  }
}
```

Además, agrego un **manejador global de excepciones** para interceptar posibles violaciones de restricciones (como `UNIQUE` en `dni` o `email`), devolviendo un mensaje legible al usuario.  

En resumen, el backend del **RF03** gestiona la creación y actualización de los datos del perfil en la tabla **PERFIL_USUARIOS**, manteniendo la relación 1:1 con la tabla **USUARIOS** a través de `id_usuario`.  
Esto garantiza la sincronización entre la información del usuario y su perfil, permitiendo que las futuras operaciones (como registrar o editar mascotas) se asocien correctamente con el dueño dentro del sistema **GUMA**, asegurando trazabilidad y consistencia de la información.