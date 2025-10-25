# RF06 - Remover co-dueño (Backend)

Para el backend del requerimiento **RF06 – Remover co-dueño**, voy a exponer un **endpoint DELETE**:

`/api/mascotas/{idMascota}/coduenos/{idPerfilUsuario}`

Este endpoint permite al **dueño principal** eliminar el vínculo activo entre una mascota y uno de sus co-dueños sin borrar registros físicos. Se realiza una **baja lógica** en la tabla `MASCOTA_DUENO` seteando `activo = 0`.

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api")
public class CoDuenoController {

    private final CoDuenoService coDuenoService;

    public CoDuenoController(CoDuenoService coDuenoService) {
        this.coDuenoService = coDuenoService;
    }

    @DeleteMapping("/mascotas/{idMascota}/coduenos/{idPerfilUsuario}")
    public ResponseEntity<RemoverCoDuenoResponse> removerCoDueno(
            @PathVariable Integer idMascota,
            @PathVariable Integer idPerfilUsuario,
            Authentication auth // de aquí saco el id_usuario dueño principal
    ) {
        RemoverCoDuenoResponse resp = coDuenoService.removerCoDueno(idMascota, idPerfilUsuario, auth);
        return ResponseEntity.ok(resp);
    }
}
```

**Respuesta estándar**:
```json
{
  "mensaje": "El co-dueño fue removido correctamente",
  "mascota": { "id": 23 },
  "coDuenoRemovido": { "idPerfilUsuario": 67 }
}
```

---

## 2) DTOs

```java
public record RemoverCoDuenoResponse(
        String mensaje,
        Integer idMascota,
        Integer idPerfilRemovido
) {}
```

---

## 3) Servicio (Java)

```java
@Service
public class CoDuenoService {

    private final MascotaDuenoRepository mascotaDuenoRepository;
    private final PerfilUsuarioRepository perfilUsuarioRepository;

    public CoDuenoService(MascotaDuenoRepository mdr, PerfilUsuarioRepository pur) {
        this.mascotaDuenoRepository = mdr;
        this.perfilUsuarioRepository = pur;
    }

    @Transactional
    public RemoverCoDuenoResponse removerCoDueno(Integer idMascota, Integer idPerfilCoDueno, Authentication auth) {
        Integer idUsuarioAuth = ((JwtUser) auth.getPrincipal()).getIdUsuario();

        // 1) Validar que el autenticado sea dueño principal (tenga vínculo activo)
        boolean esDuenoPrincipal = mascotaDuenoRepository.existsActivoByMascotaAndUsuario(idMascota, idUsuarioAuth);
        if (!esDuenoPrincipal) {
            throw new ForbiddenException("Solo el dueño principal puede remover co-dueños");
        }

        // 2) Recuperar relación activa co-dueño
        MascotaDueno relacion = mascotaDuenoRepository
                .findActivoByIdMascotaAndIdPerfilUsuario(idMascota, idPerfilCoDueno)
                .orElseThrow(() -> new NotFoundException("No se encontró un vínculo activo con ese usuario"));

        // 3) Regla: no permitir eliminar el último vínculo activo
        long countVinculosActivos = mascotaDuenoRepository.countActivosByIdMascota(idMascota);
        if (countVinculosActivos <= 1) {
            throw new ConflictException("No se puede eliminar el último vínculo activo de la mascota");
        }

        // 4) Baja lógica
        relacion.setActivo(false);
        mascotaDuenoRepository.save(relacion);

        return new RemoverCoDuenoResponse(
                "El co-dueño fue removido correctamente",
                idMascota,
                idPerfilCoDueno
        );
    }
}
```

---

## 4) Repositorios (interfaces sugeridas)

```java
public interface MascotaDuenoRepository {
    Optional<MascotaDueno> findActivoByIdMascotaAndIdPerfilUsuario(Integer idMascota, Integer idPerfilUsuario);
    boolean existsActivoByMascotaAndUsuario(Integer idMascota, Integer idUsuarioDuenoPrincipal);
    long countActivosByIdMascota(Integer idMascota);
    MascotaDueno save(MascotaDueno entity);
}

public interface PerfilUsuarioRepository {
    Optional<PerfilUsuario> findById(Integer idPerfilUsuario);
}
```

---

## 5) Mapeo exacto a la BDD `guma`

**Tabla involucrada:** `MASCOTA_DUENO`
- `id_mascota` (FK → `MASCOTAS.id_mascota`, `ON DELETE CASCADE`)
- `id_perfil_usuario` (FK → `PERFIL_USUARIOS.id_perfil_usuario`, `ON DELETE CASCADE`)
- `fecha_alta` (DATE NOT NULL)
- `activo` (TINYINT(1) NOT NULL DEFAULT 1) ← **Se setea en 0** para baja lógica

**UPDATE ejecutado** (conceptual):
```sql
UPDATE MASCOTA_DUENO
SET activo = 0
WHERE id_mascota = ? AND id_perfil_usuario = ?;
```

---

## 6) Reglas y respuestas HTTP

- **200 OK** → baja lógica realizada.
- **403 Forbidden** → usuario autenticado no es dueño principal.
- **404 Not Found** → no hay vínculo activo con ese `idPerfilUsuario`.
- **409 Conflict** → intento de eliminar el **último vínculo activo**.
- **500 Internal Server Error** → error no controlado (se loguea y se responde mensaje genérico).

---

## 7) Notas de seguridad

- El **dueño principal** se determina por el vínculo activo en `MASCOTA_DUENO` para el `id_usuario` del token.
- No se aceptar pasar el dueño por body: siempre del contexto de autenticación.
- Se audita en logs internos (no en DB) con correlation-id.

---

## 8) Ejemplo de respuesta de error

```json
{
  "mensaje": "Solo el dueño principal puede remover co-dueños",
  "codigo": 403
}
```

Con esto, RF06 queda implementado respetando tu modelo: **no se eliminan filas**, se preserva trazabilidad y se mantiene íntegra la relación **N–N** entre `MASCOTAS` y `PERFIL_USUARIOS` a través de `MASCOTA_DUENO`. 
