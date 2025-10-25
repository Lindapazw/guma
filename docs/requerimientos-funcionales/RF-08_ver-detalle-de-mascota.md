# RF08 - Ver detalle de mascota (Backend)

Para el backend del requerimiento **RF08 – Ver detalle de mascota**, voy a exponer un **endpoint GET**:

`/api/mascotas/{idMascota}`

Solo devuelve el detalle si la mascota está **activa** (`MASCOTAS.activo = 1`) y el usuario autenticado está **vinculado** (dueño o co-dueño) vía `MASCOTA_DUENO.activo = 1`.

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api")
public class MascotasController {

  private final MascotasService service;

  public MascotasController(MascotasService service) { this.service = service; }

  @GetMapping("/mascotas/{idMascota}")
  public ResponseEntity<MascotaDetalleDto> detalle(
      @PathVariable Integer idMascota,
      Authentication auth
  ) {
    int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
    MascotaDetalleDto dto = service.obtenerDetalle(idUsuario, idMascota);
    return ResponseEntity.ok(dto);
  }
}
```

---

## 2) DTOs

```java
public record CoDuenoDto(
  Integer idPerfilUsuario,
  String nombre,
  String apellido,
  String email
) {}

public record MascotaDetalleDto(
  Integer idMascota,
  String nombre,
  String especie,          // ESPECIES.especie
  String raza,             // RAZAS.name
  String sexo,             // SEXOS.sexo
  LocalDate fechaNacimiento,
  Integer edadAproximada,
  BigDecimal peso,
  String estadoReproductivo, // ESTADO_REPRODUCTIVOS.name
  String nivelActividad,     // NIVEL_ACTIVIDAD.descripcion
  String tipoAlimentacion,   // TIPO_ALIMENTACION.descripcion
  String alimentoDescripcion,
  String temperamento,       // TEMPERAMENTOS.descripcion
  LocalDate ultimaFechaCelo,
  Integer numeroCrias,
  String descripcion,
  String foto,               // IMAGES.link (via MASCOTAS.image_id)
  String color,              // COLOR.nombre
  String tamano,             // TAMANO.nombre
  String estadoVital,        // ESTADO_VITAL.estado
  String rangoEdad,          // RANGO_EDAD.nombre
  List<CoDuenoDto> coDuenos
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
    this.repo = repo; this.perfilRepo = perfilRepo;
  }

  @Transactional(readOnly = true)
  public MascotaDetalleDto obtenerDetalle(int idUsuario, int idMascota) {
    Integer idPerfil = perfilRepo.findIdPerfilByIdUsuario(idUsuario)
        .orElseThrow(() -> new ForbiddenException("Debes completar tu perfil para ver detalles"));

    // Verificar vínculo activo
    boolean vinculado = repo.existsVinculoActivo(idMascota, idPerfil);
    if (!vinculado) throw new NotFoundException("No se encontró la mascota para este usuario");

    // Traer detalle (solo si MASCOTAS.activo = 1)
    return repo.fetchDetalle(idMascota)
        .orElseThrow(() -> new NotFoundException("Mascota no encontrada o inactiva"));
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
  boolean existsVinculoActivo(Integer idMascota, Integer idPerfilUsuario);
  Optional<MascotaDetalleDto> fetchDetalle(Integer idMascota);
}
```

---

## 5) SQL/JPQL conceptual (detalle + co-dueños)

```sql
-- Detalle de la mascota (solo activa)
SELECT m.id_mascota, m.nombre,
       e.especie, r.name AS raza, s.sexo,
       m.fecha_nacimiento, m.edad_aproximada, m.peso,
       er.name AS estado_reproductivo,
       na.descripcion AS nivel_actividad,
       ta.descripcion AS tipo_alimentacion,
       m.alimento_descripcion,
       t.descripcion AS temperamento,
       m.ultima_fecha_celo, m.numero_crias, m.descripcion,
       img.link AS foto,
       c.nombre  AS color,
       tma.nombre AS tamano,
       ev.estado AS estado_vital,
       re.nombre AS rango_edad
FROM MASCOTAS m
JOIN RAZAS r              ON r.id_raza = m.id_raza
JOIN ESPECIES e           ON e.id_especie = r.id_especie
JOIN SEXOS s              ON s.id_sexo = m.id_sexo
JOIN ESTADO_VITAL ev      ON ev.id_estado_vital = m.id_estado_vital
LEFT JOIN ESTADO_REPRODUCTIVOS er ON er.id_estado_reproductivo = m.id_estado_reproductivo
LEFT JOIN NIVEL_ACTIVIDAD na      ON na.id_nivel_actividad = m.id_nivel_actividad
LEFT JOIN TIPO_ALIMENTACION ta    ON ta.id_tipo_alimentacion = m.id_tipo_alimentacion
LEFT JOIN TEMPERAMENTOS t         ON t.id_temperamento = m.id_temperamento
LEFT JOIN IMAGES img              ON img.id_images = m.image_id
LEFT JOIN COLOR c                 ON c.id_color = m.id_color
LEFT JOIN TAMANO tma              ON tma.id_tamano = m.id_tamano
LEFT JOIN RANGO_EDAD re           ON re.id_rango_edad = m.id_rango_edad
WHERE m.id_mascota = :idMascota
  AND m.activo = 1;

-- Co-dueños activos (incluye dueño principal)
SELECT pu.id_perfil_usuario, pu.nombre, pu.apellido, pu.email
FROM MASCOTA_DUENO md
JOIN PERFIL_USUARIOS pu ON pu.id_perfil_usuario = md.id_perfil_usuario
WHERE md.id_mascota = :idMascota
  AND md.activo = 1;
```

---

## 6) Respuestas HTTP

- **200 OK** → devuelve `MascotaDetalleDto`.
- **403 Forbidden** → el usuario no tiene perfil asociado.
- **404 Not Found** → no hay vínculo activo con esa mascota o la mascota está inactiva.

---

## 7) Ejemplo de respuesta JSON

```json
{
  "idMascota": 123,
  "nombre": "Luna",
  "especie": "Canina",
  "raza": "Labrador",
  "sexo": "Hembra",
  "fechaNacimiento": null,
  "edadAproximada": 4,
  "peso": 24.5,
  "estadoReproductivo": "Esterilizada",
  "nivelActividad": "Alta",
  "tipoAlimentacion": "Balanceado",
  "alimentoDescripcion": "Grain-free",
  "temperamento": "Sociable",
  "ultimaFechaCelo": null,
  "numeroCrias": 0,
  "descripcion": "Excelente con chicos",
  "foto": "https://cdn.../luna.jpg",
  "color": "Marrón",
  "tamano": "Grande",
  "estadoVital": "Viva",
  "rangoEdad": "Adulto",
  "coDuenos": [
    {"idPerfilUsuario": 456, "nombre": "Marina", "apellido": "Gómez", "email": "marina@mail.com"}
  ]
}
```

---

## 8) Notas finales

- Se consulta únicamente si el usuario está vinculado en `MASCOTA_DUENO`.
- La mascota debe tener `activo = 1`.
- Se incluyen datos enriquecidos de todos los catálogos (`RAZAS`, `ESPECIES`, `ESTADO_VITAL`, `COLOR`, `TAMANO`, etc.).
- Los co-dueños se obtienen del join con `PERFIL_USUARIOS`.
- No se modifican datos; el endpoint es de solo lectura.

Con esto, **RF08** queda implementado: entrega la ficha integral de la mascota con todos los datos definidos en la BDD **GUMA**, junto a la lista de co‑dueños activos, garantizando integridad y trazabilidad del registro.
