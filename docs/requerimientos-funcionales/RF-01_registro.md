# RF01 – Registro de usuario (Paso a paso para Java puro + JDBC)  

> **Contexto**: Implementar **RF01 – Registro de usuario** contra tu **BDD MySQL existente** (script provisto por vos). **Sin Maven/Gradle**, solo JDK y JDBC. No se crean ni modifican tablas; se inserta exclusivamente en `USUARIOS` y se deja `verified = 0`. El perfil (`PERFIL_USUARIOS`) se atiende en **RF03**.

---

## 1) Supuestos exactos del esquema involucrado
**Tabla `USUARIOS`** (según tu BDD):
- `id_usuario INT AUTO_INCREMENT PK`
- `email VARCHAR(255) NOT NULL UNIQUE`
- `contrasena VARCHAR(255) NOT NULL`
- `id_rol INT NOT NULL` → FK a `ROLES(id_rol)`
- `verified TINYINT(1) NOT NULL DEFAULT 0`
- `ultima_conexion DATETIME NULL`

**Requisito de negocio RF01**
- Alta con **email/clave** y **verificación** (representada por `verified = 0` tras el alta).

**Restricciones relevantes**
- Email **único** (constraint UNIQUE).
- Debe existir un **id_rol** válido previamente cargado en `ROLES`.

---

## 2) Preparación del entorno (sin Maven)
1. **JDK 17+** instalado (`java -version` / `javac -version`).
2. **MySQL 8** corriendo con la base `guma` creada con tu script.
3. **Conector JDBC de MySQL** (archivo `mysql-connector-j-<vers>.jar`):
   - Guardalo en una carpeta, por ejemplo `lib/` del proyecto.
   - Al **compilar** y **ejecutar**, incluilo en el **classpath** manualmente (ver ejemplo más abajo).
4. Asegurate de tener al menos **un rol** cargado en `ROLES` (por ejemplo `id_rol = 1`).

---

## 3) Configuración de conexión JDBC (Java → MySQL)
- **Driver**: `com.mysql.cj.jdbc.Driver`
- **URL** típica:  
  `jdbc:mysql://localhost:3306/guma?useSSL=false&serverTimezone=UTC&characterEncoding=utf8`
- **Credenciales**: usuario y password reales de tu MySQL.

**Qué debe hacer Copilot** (tareas concretas):
- Crear una clase `DB` con un método estático `Connection get()` que:
  - Cargue el driver (`Class.forName("com.mysql.cj.jdbc.Driver")`).
  - Devuelva `DriverManager.getConnection(url, user, pass)`.
- Manejar `SQLException` y fallar rápido con mensaje claro si no conecta.

---

## 4) Validaciones mínimas (antes de tocar la base)
- **Email** con una **regex** simple (case-insensitive).
- **Contraseña** con política mínima: **≥ 8** caracteres, debe incluir **letras y números**.
- **id_rol** numérico **existente** en `ROLES` (opcional validar con `SELECT 1 FROM ROLES WHERE id_rol=?`).

**Qué debe hacer Copilot**:
- Crear clase utilitaria `Validators` con:
  - `boolean isValidEmail(String email)`.
  - `boolean isValidPassword(String pwd)` (revisa longitud, al menos una letra y un dígito).

---

## 5) Hash de contraseña (sin librerías externas)
- Usar PBKDF2 nativo del JDK: **`PBKDF2WithHmacSHA256`**.
- Parámetros recomendados: **salt 16 bytes**, **iterations 120_000**, **keyLength 256 bits**.
- **Formato de almacenamiento** en `USUARIOS.contrasena` (string plano):  
  `pbkdf2$<iterations>$<saltBase64>$<hashBase64>`

**Qué debe hacer Copilot**:
- Crear clase `PBKDF2` con:
  - `String hash(char[] password)` → genera `salt`, deriva clave y compone el string anterior.
  - `boolean verify(char[] password, String stored)` → útil para RF02 (login).

---

## 6) Acceso a datos (DAO) y chequeos previos
**Consultas necesarias**:
1. **Unicidad de email** (antes de insertar):  
   ```sql
   SELECT 1 FROM USUARIOS WHERE email = ?
   ```
2. **Inserción del usuario** (si no existe el email):  
   ```sql
   INSERT INTO USUARIOS (email, contrasena, id_rol, verified, ultima_conexion)
   VALUES (?,?,?,?,NULL)
   ```

**Qué debe hacer Copilot**:
- Crear `UsuarioDAO` con métodos:
  - `boolean existsByEmail(String email)` usando `PreparedStatement` y `ResultSet.next()`.
  - `int insert(Usuario u)` que haga el `INSERT` y devuelva el `id_usuario` generado (usar `RETURN_GENERATED_KEYS`).

---

## 7) Flujo principal del registro (consola)
**Entradas**: `email`, `password` (dos veces), `id_rol`.  
**Pasos**:
1. Pedir `email` y **validar** con `Validators.isValidEmail(...)`.
2. Pedir `password` y **repetición**; validar igualdad y política con `Validators.isValidPassword(...)`.
3. Pedir `id_rol` y convertir a `int`; opcionalmente validar que **existe** en `ROLES`.
4. **Comprobar unicidad** de `email` con `UsuarioDAO.existsByEmail(...)`.  
   - Si **existe**, abortar con mensaje: “Ya existe un usuario con ese email.”
5. **Hashear** la contraseña con `PBKDF2.hash(passwordChars)`.
6. Construir objeto `Usuario` con: `email`, `hash`, `id_rol`, `verified=false`.
7. Ejecutar `UsuarioDAO.insert(...)`.  
   - Éxito: mostrar `id_usuario` devuelto y mensaje de confirmación.
   - Error: capturar `SQLException`:
     - Si el `SQLState` es `23000` (violación de unicidad), informar claramente.
     - Para otros códigos, registrar el mensaje y finalizar.
8. **Postcondición**: existe nueva fila en `USUARIOS` con `verified=0` y `ultima_conexion=NULL`.

**Qué debe hacer Copilot**:
- Crear clase `RegistrarUsuario` (con `main`) que implemente exactamente los 8 pasos anteriores usando `Scanner`/`Console`.
- No crear transacción manual: es **un solo INSERT** (autocommit basta).

---

## 8) Mensajería y errores esperados
- **Email inválido** → “Email inválido.”
- **Contraseñas no coinciden** → “Las contraseñas no coinciden.”
- **Contraseña débil** → “La contraseña debe tener al menos 8 caracteres, con letras y números.”
- **id_rol inválido** → “id_rol inválido.”
- **Email duplicado** (por constraint UNIQUE o chequeo previo) → “Ya existe un usuario con ese email.”
- **Error SQL genérico** → Mostrar `e.getMessage()` y terminar.

---

## 9) Comandos sin Maven (ejemplo con `mysql-connector-j.jar` en `lib/`)
Dentro del directorio raíz del proyecto que tenga `src/`:

**Compilar**  
```bash
javac -cp "lib/mysql-connector-j.jar" -d out $(find src -name "*.java")
```

**Ejecutar**  
```bash
java -cp "out:lib/mysql-connector-j.jar" guma.app.RegistrarUsuario
```
> En Windows, reemplazá `:` por `;` en el classpath.

---

## 10) Pruebas manuales rápidas (SQL)
- Verificar inserción:  
  ```sql
  SELECT id_usuario, email, id_rol, verified, ultima_conexion
  FROM USUARIOS
  WHERE email = 'tu-correo@dominio.com';
  ```
- Probar caso duplicado intentando registrar **mismo email**.
- Probar con `id_rol` inexistente y observar el error FK si activaste la validación por consulta previa (recomendado).

---

## 11) Checklist de calidad (aplicable a RF01)
- [ ] No se crean ni alteran tablas. Solo `INSERT` en `USUARIOS`.
- [ ] Email validado y **único**.
- [ ] Contraseña **hash** PBKDF2 con **salt** e **iterations** altos (≥ 120k).
- [ ] Uso de `PreparedStatement` en todas las consultas.
- [ ] Manejo de `SQLException` con mensajes útiles.
- [ ] `verified` queda en **0** (pendiente verificación real).
- [ ] `ultima_conexion` permanece **NULL** (se setea en RF02).
- [ ] No se toca `PERFIL_USUARIOS` (queda para RF03).

---

## 12) Qué **no** debe hacer RF01 (por tu BDD y reglas)
- ❌ No crear filas en `PERFIL_USUARIOS` (tiene campos NOT NULL que corresponden a RF03).
- ❌ No forzar `verified=1` (no hay verificación por email implementada aquí).
- ❌ No insertar roles ni catálogos desde esta función.
- ❌ No almacenar contraseñas en texto plano.

---

## 13) Encargo explícito para Copilot (guía de tareas)
1. **Crear clases**: `DB`, `Validators`, `PBKDF2`, `Usuario`, `UsuarioDAO`, `RegistrarUsuario`.
2. **En `DB`**: método `Connection get()` con URL/USER/PASS parametrizables.
3. **En `Validators`**: métodos para email y contraseña.
4. **En `PBKDF2`**: `hash(...)` y `verify(...)` con `PBKDF2WithHmacSHA256`, 120k iteraciones, 16 bytes de salt, 256 bits.
5. **En `UsuarioDAO`**: `existsByEmail(email)` y `insert(Usuario)` con `PreparedStatement` y `RETURN_GENERATED_KEYS`.
6. **En `RegistrarUsuario`** (main): implementar el **flujo de 8 pasos** de la sección 7.
7. **Mensajes**: usar exactamente los textos enumerados en la sección 8.
8. **Classpath**: preparar instrucciones de compilación/ejecución sin Maven (sección 9).
9. **Respeto del esquema**: Nunca crear/alterar tablas; solo `INSERT` en `USUARIOS` con `verified=0`.

Con esto, RF01 queda implementado de forma consistente con tu BDD y listo para integrarse con RF02 (login) y RF03 (perfil).