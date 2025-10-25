# RF07 - Consultar/Buscar mascotas (Backend)

Para el backend del requerimiento **RF07 – Consultar/Buscar mascotas**, expongo un **endpoint GET**:

`/api/mascotas?nombre=&idEspecie=&idRaza=&idEstadoVital=&idRangoEdad=&page=0&size=20`

Lista **solo** las mascotas **activas** (`MASCOTAS.activo = 1`) asociadas al usuario autenticado (dueño o co‑dueño) vía `MASCOTA_DUENO` (`activo = 1`). Incluye filtros opcionales y paginación.

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api")
public class MascotasController {

    private final MascotasService mascotasService;

    public MascotasController(MascotasService mascotasService) {
        this.mascotasService = mascotasService;
    }

    @GetMapping("/mascotas")
    public ResponseEntity<PageResponse<MascotaResumenDto>> listar(
            @RequestParam(required = false) String nombre,
            @RequestParam(required = false) Integer idEspecie,
            @RequestParam(required = false) Integer idRaza,
            @RequestParam(required = false) Integer idEstadoVital,
            @RequestParam(required = false) Integer idRangoEdad,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            Authentication auth
    ) {
        int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
        FiltrosMascota filtros = new FiltrosMascota(nombre, idEspecie, idRaza, idEstadoVital, idRangoEdad);
        PageResponse<MascotaResumenDto> resp = mascotasService.buscarMascotas(idUsuario, filtros, page, size);
        return ResponseEntity.ok(resp);
    }
}
```

---

## 2) DTOs / Records

```java
public record FiltrosMascota(
        String nombre,
        Integer idEspecie,
        Integer idRaza,
        Integer idEstadoVital,
        Integer idRangoEdad
) {}

public record MascotaResumenDto(
        Integer idMascota,
        String nombre,
        String especie,
        String raza,
        Integer edadAproximada,
        LocalDate fechaNacimiento,
        String estadoVital,
        String foto
) {}

public record PageResponse<T>(
        int page,
        int size,
        long total,
        List<T> items
) {}
```

---

## 3) Servicio (Java)

```java
@Service
public class MascotasService {

    private final MascotasRepository repo;
    private final PerfilUsuarioRepository perfilRepo;

    public MascotasService(MascotasRepository repo, PerfilUsuarioRepository perfilRepo) {
        this.repo = repo;
        this.perfilRepo = perfilRepo;
    }

    @Transactional(readOnly = true)
    public PageResponse<MascotaResumenDto> buscarMascotas(int idUsuario, FiltrosMascota f, int page, int size) {
        if (page < 0 || size < 1 || size > 100) throw new BadRequestException("Paginación inválida");

        Integer idPerfil = perfilRepo.findIdPerfilByIdUsuario(idUsuario)
                .orElseThrow(() -> new ForbiddenException("Debes completar tu perfil para ver tus mascotas"));

        long total = repo.countByPerfilAndFiltros(idPerfil, f);
        List<MascotaResumenDto> items = repo.findByPerfilAndFiltros(idPerfil, f, page, size);
        return new PageResponse<>(page, size, total, items);
    }
}
```

---

## 4) Repositorios (interfaces sugeridas)

```java
public interface PerfilUsuarioRepository {
    Optional<Integer> findIdPerfilByIdUsuario(Integer idUsuario);
}

public interface MascotasRepository {
    long countByPerfilAndFiltros(Integer idPerfilUsuario, FiltrosMascota f);
    List<MascotaResumenDto> findByPerfilAndFiltros(Integer idPerfilUsuario, FiltrosMascota f, int page, int size);
}
```

---

## 5) SQL/JPQL base (conceptual)

```sql
SELECT m.id_mascota, m.nombre, r.name AS raza, e.especie AS especie,
       m.edad_aproximada, m.fecha_nacimiento,
       ev.estado AS estado_vital,
       img.link AS foto
FROM MASCOTAS m
JOIN MASCOTA_DUENO md   ON md.id_mascota = m.id_mascota AND md.activo = 1
JOIN PERFIL_USUARIOS pu ON pu.id_perfil_usuario = md.id_perfil_usuario
JOIN RAZAS r            ON r.id_raza = m.id_raza
JOIN ESPECIES e         ON e.id_especie = r.id_especie
JOIN ESTADO_VITAL ev    ON ev.id_estado_vital = m.id_estado_vital
LEFT JOIN IMAGES img    ON img.id_images = m.image_id
WHERE pu.id_perfil_usuario = :idPerfil
  AND m.activo = 1
  AND (:nombre        IS NULL OR m.nombre LIKE CONCAT('%', :nombre, '%'))
  AND (:idEspecie     IS NULL OR e.id_especie = :idEspecie)
  AND (:idRaza        IS NULL OR m.id_raza = :idRaza)
  AND (:idEstadoVital IS NULL OR m.id_estado_vital = :idEstadoVital)
  AND (:idRangoEdad   IS NULL OR m.id_rango_edad = :idRangoEdad)
ORDER BY m.id_mascota DESC
LIMIT :size OFFSET :offset;
```

> Nota: `:offset = page * size`.

---

## 6) Validaciones y respuestas HTTP

- **200 OK** → listado paginado.
- **400 Bad Request** → paginación inválida o tipos de filtros incorrectos.
- **403 Forbidden** → el usuario no posee perfil asociado.

---

## 7) Mapeo a la BDD `guma`

- **MASCOTAS**: `id_mascota`, `nombre`, `id_raza`, `id_estado_vital`, `edad_aproximada`, `fecha_nacimiento`, `image_id`, `id_rango_edad`, `activo`.
- **MASCOTA_DUENO**: vínculo con `id_perfil_usuario` (solo `activo = 1`).
- **RAZAS** ↔ **ESPECIES**: para filtrar `idEspecie` y resolver nombres.
- **ESTADO_VITAL**: para el estado.
- **IMAGES**: `link` para la foto (LEFT JOIN porque puede ser null).

---

## 8) Seguridad y performance

- El universo de búsqueda se limita por el **perfil del usuario autenticado** (no se acepta id de dueño por request).
- Índices recomendados (sin alterar esquema): considerar índices en `MASCOTA_DUENO(id_perfil_usuario, activo)`, `MASCOTAS(activo, id_raza, id_estado_vital, id_rango_edad)` para acelerar filtros.
- Rate limiting suave y orden estable por `m.id_mascota DESC`.

---

## 9) Ejemplo de respuesta JSON

```json
{
  "page": 0,
  "size": 20,
  "total": 3,
  "items": [
    {
      "idMascota": 123,
      "nombre": "Luna",
      "especie": "Canina",
      "raza": "Labrador",
      "edadAproximada": 4,
      "fechaNacimiento": null,
      "estadoVital": "Viva",
      "foto": "https://cdn.../luna.jpg"
    }
  ]
}
```

Con esto, **RF07** queda listo: lista únicamente mascotas **activas** y **vinculadas** al usuario mediante `MASCOTA_DUENO`, con filtros por `nombre`, `idEspecie`, `idRaza`, `idEstadoVital`, `idRangoEdad`, y respuesta paginada con datos claves de la mascota.
