# RF09 - Editar mascotas (Backend)

Para el backend del requerimiento **RF09 – Editar mascotas**, voy a exponer un **endpoint PUT**:

`/api/mascotas/{idMascota}`

Actualiza datos de una mascota **activa** y solo si el usuario autenticado está **vinculado** (dueño o co-dueño) vía `MASCOTA_DUENO.activo = 1`. La operación respeta las FKs de catálogos y mantiene la integridad referencial.

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api")
public class MascotasController {

  private final MascotasService service;

  public MascotasController(MascotasService service) { this.service = service; }

  @PutMapping("/mascotas/{idMascota}")
  public ResponseEntity<MascotaDetalleDto> editar(
      @PathVariable Integer idMascota,
      @RequestBody @Valid EditarMascotaRequest body,
      Authentication auth
  ) {
    int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
    MascotaDetalleDto dto = service.editarMascota(idUsuario, idMascota, body);
    return ResponseEntity.ok(dto);
  }
}
```

---

## 2) Request DTO

```java
public class EditarMascotaRequest {
  @NotBlank(message="El nombre no puede quedar vacío")
  private String nombre;                   // MASCOTAS.nombre

  private Integer idRaza;                  // FK RAZAS.id_raza
  private Integer idSexo;                  // FK SEXOS.id_sexo
  private LocalDate fechaNacimiento;       // MASCOTAS.fecha_nacimiento
  private Integer edadAproximada;          // MASCOTAS.edad_aproximada
  private BigDecimal peso;                 // MASCOTAS.peso
  private Integer idEstadoReproductivo;    // FK ESTADO_REPRODUCTIVOS.id_estado_reproductivo
  private Integer idNivelActividad;        // FK NIVEL_ACTIVIDAD.id_nivel_actividad
  private Integer idTipoAlimentacion;      // FK TIPO_ALIMENTACION.id_tipo_alimentacion
  private String  alimentoDescripcion;     // MASCOTAS.alimento_descripcion
  private Integer idTemperamento;          // FK TEMPERAMENTOS.id_temperamento
  private LocalDate ultimaFechaCelo;       // MASCOTAS.ultima_fecha_celo
  private Integer numeroCrias;             // MASCOTAS.numero_crias
  private String  descripcion;             // MASCOTAS.descripcion
  private Integer imageId;                 // FK IMAGES.id_images
  private Integer idColor;                 // FK COLOR.id_color
  private Integer idTamano;                // FK TAMANO.id_tamano
  private Integer idRangoEdad;             // FK RANGO_EDAD.id_rango_edad
}
```

> Todos los campos son opcionales salvo `nombre`. El endpoint permite actualizaciones parciales (solo lo que se envíe en el body).

---

## 3) Servicio (Java)

```java
@Service
public class MascotasService {

  private final MascotasRepository repo;
  private final PerfilUsuarioRepository perfilRepo;
  private final CatalogosRepository catRepo;

  public MascotasService(MascotasRepository repo, PerfilUsuarioRepository perfilRepo, CatalogosRepository catRepo) {
    this.repo = repo; this.perfilRepo = perfilRepo; this.catRepo = catRepo;
  }

  @Transactional
  public MascotaDetalleDto editarMascota(int idUsuario, int idMascota, EditarMascotaRequest r) {
    Integer idPerfil = perfilRepo.findIdPerfilByIdUsuario(idUsuario)
        .orElseThrow(() -> new ForbiddenException("Debes completar tu perfil para editar mascotas"));

    // 1) Verificar vínculo activo y que la mascota esté activa
    if (!repo.existsVinculoActivo(idMascota, idPerfil))
      throw new NotFoundException("Mascota no encontrada para este usuario");
    if (!repo.isActiva(idMascota))
      throw new ConflictException("Solo se pueden editar mascotas activas");

    // 2) Validaciones de negocio
    if (r.getPeso() != null && r.getPeso().signum() < 0)
      throw new BadRequestException("El peso no puede ser negativo");
    if (r.getFechaNacimiento() != null && r.getEdadAproximada() != null)
      throw new BadRequestException("Usar fecha de nacimiento o edad aproximada, no ambas");

    // 3) Validar FKs opcionales solo si vienen
    catRepo.validarFkSiViene("RAZAS", "id_raza", r.getIdRaza());
    catRepo.validarFkSiViene("SEXOS", "id_sexo", r.getIdSexo());
    catRepo.validarFkSiViene("ESTADO_REPRODUCTIVOS", "id_estado_reproductivo", r.getIdEstadoReproductivo());
    catRepo.validarFkSiViene("NIVEL_ACTIVIDAD", "id_nivel_actividad", r.getIdNivelActividad());
    catRepo.validarFkSiViene("TIPO_ALIMENTACION", "id_tipo_alimentacion", r.getIdTipoAlimentacion());
    catRepo.validarFkSiViene("TEMPERAMENTOS", "id_temperamento", r.getIdTemperamento());
    catRepo.validarFkSiViene("IMAGES", "id_images", r.getImageId());
    catRepo.validarFkSiViene("COLOR", "id_color", r.getIdColor());
    catRepo.validarFkSiViene("TAMANO", "id_tamano", r.getIdTamano());
    catRepo.validarFkSiViene("RANGO_EDAD", "id_rango_edad", r.getIdRangoEdad());

    // 4) Update parcial sobre MASCOTAS
    repo.updateMascota(idMascota, r);

    // 5) Devolver detalle actualizado (reusa fetch del RF08)
    return repo.fetchDetalle(idMascota)
        .orElseThrow(() -> new IllegalStateException("La mascota debería existir tras el update"));
  }
}
```

---

## 4) Repositorios

```java
public interface PerfilUsuarioRepository {
  Optional<Integer> findIdPerfilByIdUsuario(Integer idUsuario);
}

public interface MascotasRepository {
  boolean existsVinculoActivo(Integer idMascota, Integer idPerfilUsuario);
  boolean isActiva(Integer idMascota);
  void updateMascota(Integer idMascota, EditarMascotaRequest r);   // construye UPDATE dinámico
  Optional<MascotaDetalleDto> fetchDetalle(Integer idMascota);
}

public interface CatalogosRepository {
  void validarFkSiViene(String tabla, String columnaId, Integer valor); // no hace nada si valor==null
}
```

---

## 5) SQL conceptual (UPDATE parcial)

```sql
UPDATE MASCOTAS
SET
  nombre = :nombre,
  id_raza = COALESCE(:idRaza, id_raza),
  id_sexo = COALESCE(:idSexo, id_sexo),
  fecha_nacimiento = :fechaNacimiento,
  edad_aproximada = :edadAproximada,
  peso = :peso,
  id_estado_reproductivo = COALESCE(:idEstadoReproductivo, id_estado_reproductivo),
  id_nivel_actividad = COALESCE(:idNivelActividad, id_nivel_actividad),
  id_tipo_alimentacion = COALESCE(:idTipoAlimentacion, id_tipo_alimentacion),
  alimento_descripcion = :alimentoDescripcion,
  id_temperamento = COALESCE(:idTemperamento, id_temperamento),
  ultima_fecha_celo = :ultimaFechaCelo,
  numero_crias = :numeroCrias,
  descripcion = :descripcion,
  image_id = COALESCE(:imageId, image_id),
  id_color = COALESCE(:idColor, id_color),
  id_tamano = COALESCE(:idTamano, id_tamano),
  id_rango_edad = COALESCE(:idRangoEdad, id_rango_edad)
WHERE id_mascota = :idMascota;
```

---

## 6) Respuestas HTTP

- **200 OK** → devuelve el detalle actualizado.
- **400 Bad Request** → validaciones de negocio (peso negativo, doble fecha/edad, nombre vacío).
- **403 Forbidden** → el usuario no tiene perfil.
- **404 Not Found** → no hay vínculo activo con esa mascota.
- **409 Conflict** → mascota inactiva (no editable).

---

## 7) Mapeo a la BDD `guma`

Edita columnas de **MASCOTAS**:  
`nombre`, `id_raza`, `id_sexo`, `fecha_nacimiento`, `edad_aproximada`, `peso`, `id_estado_reproductivo`, `id_nivel_actividad`, `id_tipo_alimentacion`, `alimento_descripcion`, `id_temperamento`, `ultima_fecha_celo`, `numero_crias`, `descripcion`, `image_id`, `id_color`, `id_tamano`, `id_rango_edad`.  
No modifica `activo` ni relaciones en **MASCOTA_DUENO** (se respetan vínculos activos).

---

## 8) Seguridad y consistencia

- El **dueño/co-dueño** se verifica por `MASCOTA_DUENO` (no se acepta id de dueño por body).  
- Transacción `@Transactional` para atomicidad.  
- Sanitización de campos tipo texto (`descripcion`, `alimentoDescripcion`).  
- Logs con correlation-id para trazabilidad.  

---

Con esto, **RF09** queda implementado: el backend actualiza los datos de una mascota dentro de los límites definidos por las reglas del sistema, garantizando integridad referencial con los catálogos y consistencia con los vínculos activos de co‑dueños en **MASCOTA_DUENO**.
