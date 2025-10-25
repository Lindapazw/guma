# RF10 - Baja de mascotas (Backend)

Para el backend del requerimiento **RF10 – Baja de mascotas**, voy a exponer un **endpoint DELETE** que realiza una **baja lógica del vínculo** entre el usuario autenticado y una mascota específica. **No elimina** la mascota ni los vínculos de otros co‑dueños.

`/api/mascotas/{idMascota}/mi-vinculo`

La operación marca `activo = 0` en la tabla `MASCOTA_DUENO` para el par (`id_mascota`, `id_perfil_usuario` del usuario autenticado).

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api")
public class MascotaVinculoController {

    private final MascotaVinculoService service;

    public MascotaVinculoController(MascotaVinculoService service) {
        this.service = service;
    }

    @DeleteMapping("/mascotas/{idMascota}/mi-vinculo")
    public ResponseEntity<BajaVinculoResponse> darDeBaja(
            @PathVariable Integer idMascota,
            Authentication auth
    ) {
        int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
        BajaVinculoResponse resp = service.bajaLogicaVinculo(idUsuario, idMascota);
        return ResponseEntity.ok(resp);
    }
}
```

---

## 2) DTOs

```java
public record BajaVinculoResponse(
        String mensaje,
        Integer idMascota,
        Integer idPerfilUsuario,
        boolean sigueVisibleParaOtros
) {}
```

---

## 3) Servicio (Java)

```java
@Service
public class MascotaVinculoService {

    private final PerfilUsuarioRepository perfilRepo;
    private final MascotaDuenoRepository mdRepo;

    public MascotaVinculoService(PerfilUsuarioRepository perfilRepo, MascotaDuenoRepository mdRepo) {
        this.perfilRepo = perfilRepo;
        this.mdRepo = mdRepo;
    }

    @Transactional
    public BajaVinculoResponse bajaLogicaVinculo(int idUsuario, int idMascota) {
        Integer idPerfil = perfilRepo.findIdPerfilByIdUsuario(idUsuario)
                .orElseThrow(() -> new ForbiddenException("Debes completar tu perfil para gestionar vínculos"));

        // 1) Validar que exista vínculo activo
        MascotaDueno vinculo = mdRepo.findActivoByIdMascotaAndIdPerfilUsuario(idMascota, idPerfil)
                .orElseThrow(() -> new NotFoundException("No tienes un vínculo activo con esta mascota"));

        // 2) Baja lógica del vínculo
        vinculo.setActivo(false);
        mdRepo.save(vinculo);

        // 3) Chequear si la mascota queda visible por otros (existen otros vínculos activos)
        boolean hayOtrosActivos = mdRepo.existsOtroVinculoActivo(idMascota, idPerfil);

        return new BajaVinculoResponse(
                "Baja realizada: ya no verás esta mascota en tu listado",
                idMascota,
                idPerfil,
                hayOtrosActivos
        );
    }
}
```

---

## 4) Repositorios (interfaces sugeridas)

```java
public interface PerfilUsuarioRepository {
    Optional<Integer> findIdPerfilByIdUsuario(Integer idUsuario);
}

public interface MascotaDuenoRepository {
    Optional<MascotaDueno> findActivoByIdMascotaAndIdPerfilUsuario(Integer idMascota, Integer idPerfilUsuario);
    boolean existsOtroVinculoActivo(Integer idMascota, Integer idPerfilExcluido);
    MascotaDueno save(MascotaDueno entity);
}
```

Implementación SQL típica:
```sql
-- Encontrar vínculo activo del usuario autenticado
SELECT *
FROM MASCOTA_DUENO
WHERE id_mascota = :idMascota
  AND id_perfil_usuario = :idPerfil
  AND activo = 1;

-- Baja lógica del vínculo
UPDATE MASCOTA_DUENO
SET activo = 0
WHERE id_mascota = :idMascota
  AND id_perfil_usuario = :idPerfil;

-- ¿Sigue visible para otros usuarios?
SELECT EXISTS (
  SELECT 1 FROM MASCOTA_DUENO
  WHERE id_mascota = :idMascota
    AND id_perfil_usuario <> :idPerfil
    AND activo = 1
) AS hayOtrosActivos;
```

---

## 5) Reglas y validaciones

- Se afecta **solo** el vínculo del usuario autenticado; no se permite pasar otro `id_perfil_usuario` por request.
- La mascota **no se elimina** de `MASCOTAS`; queda intacta.
- No es necesario impedir "último vínculo"; si el usuario era el único dueño, la mascota queda sin vínculos activos pero permanece en la base (histórico preservado). Si tu política exige impedirlo, se puede agregar un check y devolver **409 (Conflict)**.

---

## 6) Respuestas HTTP

- **200 OK** → baja lógica realizada.
- **403 Forbidden** → el usuario no tiene perfil.
- **404 Not Found** → el usuario no posee un vínculo activo con esa mascota.

Ejemplo de éxito:
```json
{
  "mensaje": "Baja realizada: ya no verás esta mascota en tu listado",
  "idMascota": 23,
  "idPerfilUsuario": 67,
  "sigueVisibleParaOtros": true
}
```

---

## 7) Mapeo a la BDD `guma`

**Tabla:** `MASCOTA_DUENO`  
- Se actualiza `activo` de **1 → 0** para la fila del usuario autenticado.  
- No se tocan `MASCOTAS` ni los vínculos de otros perfiles.  
- Se preserva la trazabilidad histórica (no hay `DELETE`).

---

## 8) Seguridad y trazabilidad

- Id de dueño **derivado del token**; evita manipulación del vínculo.  
- Transacción `@Transactional` para atomicidad.  
- Log de auditoría (correlation‑id) con `idMascota`, `idPerfil` y timestamp.  

Con esto, **RF10** queda implementado: el usuario puede eliminar **su** vínculo activo con una mascota mediante **baja lógica** en `MASCOTA_DUENO`, manteniendo íntegros los demás vínculos y el registro de la mascota en la BDD **GUMA**.
