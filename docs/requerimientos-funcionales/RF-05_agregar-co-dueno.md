# RF05 - Agregar co-dueño (Backend)

Para el backend del requerimiento **RF05 – Agregar co-dueño**, voy a crear un **endpoint POST** llamado  
`/api/mascotas/{idMascota}/coduenos`.  
Este endpoint permitirá al **dueño principal** de una mascota asociar otros usuarios registrados como co-dueños, manteniendo la relación entre la mascota y varios perfiles dentro del sistema.

El flujo comienza en el **controlador** (`CoDuenoController`), donde defino el método `@PostMapping("/mascotas/{idMascota}/coduenos")`.  
El método recibe un objeto `CoDuenoRequest` con los datos necesarios para asociar el nuevo co-dueño:

```java
public class CoDuenoRequest {
    private String emailCoDueno; // correo del usuario ya registrado
}
```

Estos datos se corresponden con los campos que luego se insertan en la tabla **MASCOTA_DUENO**:  
- `id_mascota` → FK a `MASCOTAS.id_mascota`  
- `id_perfil_usuario` → FK a `PERFIL_USUARIOS.id_perfil_usuario` (del co-dueño)  
- `fecha_alta` → `DATE NOT NULL`  
- `activo` → `TINYINT(1) DEFAULT 1`  

El **servicio** (`CoDuenoService`) se encarga de implementar la lógica principal del proceso.  

Primero, el sistema identifica al usuario autenticado a través del token, que representa al **dueño principal**. Luego, valida que este usuario sea efectivamente dueño de la mascota seleccionada verificando su relación en la tabla `MASCOTA_DUENO`. Si no tiene ese vínculo, el sistema devuelve **403 (Forbidden)** indicando que solo el dueño principal puede agregar co-dueños.

Una vez validada la tenencia, el sistema busca al co-dueño que se desea asociar. Para ello consulta en las tablas:  
- `USUARIOS.email` para verificar que el usuario exista.  
- `PERFIL_USUARIOS` para obtener el `id_perfil_usuario` correspondiente a ese usuario.

Si el co-dueño no está registrado o no tiene perfil, se devuelve **404 (Not Found)** con el mensaje `"El usuario indicado no existe o no tiene perfil activo"`.

Antes de crear la nueva relación, el backend realiza una validación para asegurar que **no exista ya un vínculo activo** entre esa mascota y el mismo perfil de usuario.  
Esto se verifica con una consulta a `MASCOTA_DUENO`:
```sql
SELECT * FROM MASCOTA_DUENO 
WHERE id_mascota = ? AND id_perfil_usuario = ? AND activo = 1;
```
Si existe, el sistema devuelve **409 (Conflict)** con el mensaje `"El usuario ya está asociado a esta mascota"`.

Si las validaciones son correctas, se realiza un **INSERT** en la tabla `MASCOTA_DUENO` con los siguientes valores:  
- `id_mascota` = ID de la mascota seleccionada.  
- `id_perfil_usuario` = ID del co-dueño obtenido.  
- `fecha_alta` = fecha actual.  
- `activo` = 1.  

Ejemplo del flujo en código Java:
```java
MascotaDueno nuevoCoDueno = new MascotaDueno();
nuevoCoDueno.setIdMascota(idMascota);
nuevoCoDueno.setIdPerfilUsuario(perfilCoDueno.getId());
nuevoCoDueno.setFechaAlta(LocalDate.now());
nuevoCoDueno.setActivo(true);

mascotaDuenoRepository.save(nuevoCoDueno);
```

Si la inserción es exitosa, el backend responde con un código **HTTP 201 (Created)** y un cuerpo como este:
```json
{
  "mensaje": "Co-dueño agregado correctamente",
  "mascota": {
    "id": 23
  },
  "coDueno": {
    "email": "co.dueno@mail.com",
    "idPerfilUsuario": 67
  }
}
```

Si ocurre una violación de clave foránea (por ejemplo, el `id_mascota` no existe), se devuelve **400 (Bad Request)** indicando el error exacto.

Por último, el servicio actualiza la lista de co-dueños de la mascota en memoria o la devuelve en la respuesta del endpoint, mostrando los usuarios vinculados.

En resumen, el backend del **RF05** amplía la relación **N-N** entre las tablas **MASCOTAS** y **PERFIL_USUARIOS** mediante la tabla intermedia **MASCOTA_DUENO**, permitiendo asociar múltiples perfiles a una misma mascota.  
El flujo garantiza que:
- Solo el dueño principal puede agregar co-dueños.  
- No se creen duplicaciones activas (único vínculo activo por combinación mascota-usuario).  
- Cada co-dueño debe existir previamente en el sistema y tener un perfil válido.  

De esta forma, el sistema mantiene la trazabilidad completa de la tenencia compartida dentro de **GUMA**, asegurando integridad y consistencia de los datos.