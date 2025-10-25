# RF12 - Gestionar avatar de mascota (Backend)

Para el backend del requerimiento **RF12 – Gestionar avatar de mascota**, se exponen dos endpoints:

- **Subir/Reemplazar avatar**: `POST /api/mascotas/{idMascota}/avatar`  
  Recibe un archivo de imagen (multipart/form-data).
- **Quitar avatar**: `DELETE /api/mascotas/{idMascota}/avatar`  
  Quita la referencia actual, realizando baja lógica sin borrar archivos.

La imagen **no se guarda como binario en la BD**, sino que se almacena en un sistema de archivos o CDN, registrando su **link** en la tabla `IMAGES` (`IMAGES.link`). La tabla `MASCOTAS` referencia el avatar con `image_id` (FK a `IMAGES.id_images`).

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api")
public class MascotaAvatarController {

  private final MascotaAvatarService service;

  public MascotaAvatarController(MascotaAvatarService service) {
    this.service = service;
  }

  @PostMapping(value = "/mascotas/{idMascota}/avatar", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
  public ResponseEntity<MascotaAvatarResponse> subirAvatar(
      @PathVariable Integer idMascota,
      @RequestPart("file") MultipartFile file,
      Authentication auth
  ) {
    int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
    MascotaAvatarResponse resp = service.subirOReemplazarAvatar(idUsuario, idMascota, file);
    return ResponseEntity.status(HttpStatus.CREATED).body(resp);
  }

  @DeleteMapping("/mascotas/{idMascota}/avatar")
  public ResponseEntity<MascotaAvatarResponse> quitarAvatar(
      @PathVariable Integer idMascota,
      Authentication auth
  ) {
    int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
    MascotaAvatarResponse resp = service.quitarAvatar(idUsuario, idMascota);
    return ResponseEntity.ok(resp);
  }
}
```

---

## 2) DTOs

```java
public record MascotaAvatarResponse(
    Integer idMascota,
    Integer imageId,    // puede ser null si se quitó
    String  link,       // puede ser null si se quitó
    String  mensaje
) {}
```

---

## 3) Servicio (Java)

```java
@Service
public class MascotaAvatarService {

  private static final Set<String> FORMATOS_PERMITIDOS = Set.of("image/jpeg", "image/png");
  private static final long MAX_BYTES = 5L * 1024 * 1024; // 5 MB

  private final PerfilUsuarioRepository perfilRepo;
  private final MascotasRepository mascotasRepo;
  private final ImagesRepository imagesRepo;
  private final StorageGateway storage;

  public MascotaAvatarService(PerfilUsuarioRepository perfilRepo,
                              MascotasRepository mascotasRepo,
                              ImagesRepository imagesRepo,
                              StorageGateway storage) {
    this.perfilRepo = perfilRepo;
    this.mascotasRepo = mascotasRepo;
    this.imagesRepo = imagesRepo;
    this.storage = storage;
  }

  @Transactional
  public MascotaAvatarResponse subirOReemplazarAvatar(int idUsuario, int idMascota, MultipartFile file) {
    Integer idPerfil = perfilRepo.findIdPerfilByIdUsuario(idUsuario)
        .orElseThrow(() -> new ForbiddenException("Debes completar tu perfil"));

    if (!mascotasRepo.existsVinculoActivo(idMascota, idPerfil))
      throw new NotFoundException("Mascota no encontrada para este usuario");
    if (!mascotasRepo.isActiva(idMascota))
      throw new ConflictException("Solo se puede gestionar el avatar de mascotas activas");

    if (file == null || file.isEmpty()) throw new BadRequestException("Archivo requerido");
    if (!FORMATOS_PERMITIDOS.contains(file.getContentType()))
      throw new BadRequestException("Formato inválido. Solo JPG/PNG");
    if (file.getSize() > MAX_BYTES)
      throw new BadRequestException("Tamaño máximo excedido");

    // Subir archivo al storage
    String objectKey = "mascotas/" + idMascota + "/avatar_" + System.currentTimeMillis();
    String link = storage.upload(objectKey, file);

    // Registrar imagen
    Integer imageId = imagesRepo.insert(link);

    // Actualizar mascota
    Integer oldImageId = mascotasRepo.getImageId(idMascota).orElse(null);
    mascotasRepo.setImageId(idMascota, imageId);

    if (oldImageId != null) imagesRepo.marcarObsoleta(oldImageId);

    return new MascotaAvatarResponse(idMascota, imageId, link, "Avatar actualizado correctamente");
  }

  @Transactional
  public MascotaAvatarResponse quitarAvatar(int idUsuario, int idMascota) {
    Integer idPerfil = perfilRepo.findIdPerfilByIdUsuario(idUsuario)
        .orElseThrow(() -> new ForbiddenException("Debes completar tu perfil"));

    if (!mascotasRepo.existsVinculoActivo(idMascota, idPerfil))
      throw new NotFoundException("Mascota no encontrada para este usuario");
    if (!mascotasRepo.isActiva(idMascota))
      throw new ConflictException("Solo se puede gestionar el avatar de mascotas activas");

    Integer imageId = mascotasRepo.getImageId(idMascota)
        .orElseThrow(() -> new NotFoundException("La mascota no tiene avatar asignado"));

    mascotasRepo.clearImageId(idMascota);
    imagesRepo.marcarObsoleta(imageId);

    return new MascotaAvatarResponse(idMascota, null, null, "Avatar eliminado");
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
  Optional<Integer> getImageId(Integer idMascota);
  void setImageId(Integer idMascota, Integer imageId);
  void clearImageId(Integer idMascota);
}

public interface ImagesRepository {
  Integer insert(String link);
  void marcarObsoleta(Integer id);
}

public interface StorageGateway {
  String upload(String objectKey, MultipartFile file);
  void delete(String objectKey);
}
```

---

## 5) SQL conceptual

```sql
INSERT INTO IMAGES (link) VALUES (:link);
UPDATE MASCOTAS SET image_id = :newImageId WHERE id_mascota = :idMascota;
UPDATE MASCOTAS SET image_id = NULL WHERE id_mascota = :idMascota;
```

---

## 6) Mapeo a la BDD `guma`

- **IMAGES**: `id_images` (PK), `link` (TEXT NOT NULL).
- **MASCOTAS**: `image_id` (FK → `IMAGES.id_images`, nullable).
- Reemplazar avatar = crear nuevo registro en `IMAGES` y apuntar `MASCOTAS.image_id` al nuevo ID.
- Quitar avatar = `MASCOTAS.image_id = NULL`.

---

## 7) Validaciones y reglas

- Formatos permitidos: **JPG/PNG**.
- Tamaño máximo configurable (2–5 MB).
- Una sola imagen activa por mascota.
- Solo usuarios vinculados (dueño o co-dueño) y mascotas activas pueden editar su avatar.
- Eliminación solo de referencia; no se borran registros ni archivos físicos inmediatamente.

---

## 8) Respuestas HTTP

- **201 Created** → subida/reemplazo exitoso.
- **200 OK** → avatar eliminado.
- **400 Bad Request** → formato/tamaño inválido.
- **403 Forbidden** → usuario sin perfil.
- **404 Not Found** → mascota no encontrada o sin avatar.
- **409 Conflict** → mascota inactiva.

---

## 9) Ejemplo de respuesta JSON

```json
{
  "idMascota": 42,
  "imageId": 101,
  "link": "https://cdn.guma.com/mascotas/42/avatar_1730000000000.png",
  "mensaje": "Avatar actualizado correctamente"
}
```

---

Con esto, **RF12** queda implementado: la gestión del avatar se realiza sin guardar binarios en la BD, manteniendo una relación directa entre `MASCOTAS.image_id` y `IMAGES.link`, garantizando trazabilidad, seguridad y limpieza futura del almacenamiento.