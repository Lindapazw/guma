# RF11 - Baja global de mascota (Backend)

Para el backend del requerimiento **RF11 – Baja global de mascota**, voy a exponer un **endpoint PATCH** de uso **exclusivo para administradores**. La operación aplica **baja lógica** sobre la mascota (marca `MASCOTAS.activo = 0`) y desactiva **todos** sus vínculos en `MASCOTA_DUENO` (`activo = 0`). No elimina filas.

`/api/admin/mascotas/{idMascota}/baja-global`

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api/admin")
public class AdminMascotasController {

    private final AdminMascotasService service;

    public AdminMascotasController(AdminMascotasService service) {
        this.service = service;
    }

    @PatchMapping("/mascotas/{idMascota}/baja-global")
    public ResponseEntity<BajaGlobalResponse> bajaGlobal(
            @PathVariable Integer idMascota,
            Authentication auth
    ) {
        int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
        BajaGlobalResponse resp = service.bajaGlobalMascota(idUsuario, idMascota);
        return ResponseEntity.ok(resp);
    }
}
```

---

## 2) DTOs

```java
public record BajaGlobalResponse(
        Integer idMascota,
        boolean yaEstabaInactiva,
        int vinculosDesactivados,
        String mensaje
) {}
```

---

## 3) Servicio (Java)

```java
@Service
public class AdminMascotasService {

    private final UsuariosRepository usuariosRepo;
    private final MascotasRepository mascotasRepo;
    private final MascotaDuenoRepository mdRepo;

    public AdminMascotasService(UsuariosRepository u, MascotasRepository m, MascotaDuenoRepository md) {
        this.usuariosRepo = u; this.mascotasRepo = m; this.mdRepo = md;
    }

    @Transactional
    public BajaGlobalResponse bajaGlobalMascota(int idUsuario, int idMascota) {
        // 1) Verificar rol administrador
        if (!usuariosRepo.tieneRolAdmin(idUsuario)) {
            throw new ForbiddenException("Solo el administrador puede ejecutar esta acción");
        }

        // 2) Verificar existencia de mascota
        Mascota m = mascotasRepo.findById(idMascota)
                .orElseThrow(() -> new NotFoundException("Mascota no encontrada"));

        boolean yaInactiva = (m.getActivo() != null && m.getActivo() == 0);

        // 3) Marcar mascota como inactiva
        if (!yaInactiva) {
            mascotasRepo.setActivo(idMascota, false);
        }

        // 4) Desactivar TODOS los vínculos activos en MASCOTA_DUENO
        int desactivados = mdRepo.desactivarVinculosDeMascota(idMascota); // devuelve cantidad de filas afectadas

        return new BajaGlobalResponse(
                idMascota,
                yaInactiva,
                desactivados,
                yaInactiva ? "La mascota ya estaba inactiva; se desactivaron vínculos igualmente." :
                             "Baja global aplicada con éxito"
        );
    }
}
```

---

## 4) Repositorios (interfaces sugeridas)

```java
public interface UsuariosRepository {
    boolean tieneRolAdmin(Integer idUsuario); // valida contra USUARIOS.id_rol / ROLES
}

public interface MascotasRepository {
    Optional<Mascota> findById(Integer idMascota);
    void setActivo(Integer idMascota, boolean activo);
}

public interface MascotaDuenoRepository {
    int desactivarVinculosDeMascota(Integer idMascota); // UPDATE ... SET activo = 0 WHERE id_mascota = ? AND activo = 1
}
```

---

## 5) SQL conceptual

```sql
-- (A) Verificar rol admin (ejemplo)
SELECT 1
FROM USUARIOS u
JOIN ROLES r ON r.id_rol = u.id_rol
WHERE u.id_usuario = :idUsuario AND r.rol = 'ADMIN';

-- (B) Marcar mascota inactiva
UPDATE MASCOTAS
SET activo = 0
WHERE id_mascota = :idMascota;

-- (C) Desactivar TODOS los vínculos activos de esa mascota
UPDATE MASCOTA_DUENO
SET activo = 0
WHERE id_mascota = :idMascota
  AND activo = 1;
```

---

## 6) Reglas y validaciones

- **Solo administrador** puede ejecutar la baja global (según `USUARIOS.id_rol` ↔ `ROLES`).  
- La operación es **idempotente**: si la mascota ya está inactiva, se informa en la respuesta y se desactivan vínculos de todas formas.  
- No elimina filas: se conserva la trazabilidad histórica y los datos para reportes.  
- Transacción atómica: si falla alguna parte, se deshace todo.  
- No hay endpoint para revertir (según restricción); cualquier reactivación sería un proceso separado.

---

## 7) Respuestas HTTP

- **200 OK** → baja global aplicada (o ya existente).  
- **403 Forbidden** → el usuario autenticado no es administrador.  
- **404 Not Found** → la mascota no existe.  
- **500 Internal Server Error** → error no controlado (se loguea con correlation‑id).

Ejemplo de éxito:
```json
{
  "idMascota": 321,
  "yaEstabaInactiva": false,
  "vinculosDesactivados": 3,
  "mensaje": "Baja global aplicada con éxito"
}
```

---

## 8) Mapeo a la BDD `guma`

- **MASCOTAS**: se actualiza columna `activo` a **0** para `id_mascota` objetivo.  
- **MASCOTA_DUENO**: se actualiza columna `activo` a **0** para **todas** las filas con ese `id_mascota`.  
- **USUARIOS/ROLES**: se valida el permiso de administrador mediante `USUARIOS.id_rol` y `ROLES.rol`.

---

## 9) Seguridad y auditoría

- Autenticación obligatoria + verificación estricta de rol administrador.  
- Se registran logs de auditoría con `idUsuarioAdmin`, `idMascota`, cantidad de vínculos afectados y timestamp.  
- El endpoint vive bajo `/api/admin` para reforzar la política de acceso.

Con esto, **RF11** queda implementado: la baja global **despublica** la mascota del entorno activo, desactiva todos los vínculos en `MASCOTA_DUENO` y preserva la trazabilidad completa en la BDD **GUMA** sin eliminar información.
