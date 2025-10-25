# RF13 - Gestionar foto de perfil de usuario (Backend)

Para el backend del requerimiento **RF13 – Gestionar foto de perfil de usuario**, se exponen dos endpoints dentro del módulo **Mi Perfil**:

- **Subir/Reemplazar foto de perfil**: `POST /api/usuarios/perfil/foto`  
  Recibe un archivo (multipart/form-data).  
- **Quitar foto de perfil**: `DELETE /api/usuarios/perfil/foto`  
  Quita la referencia actual (`foto_perfil = NULL`), sin eliminar el archivo físico inmediatamente.

La imagen se guarda como **recurso en la tabla IMAGES** (`IMAGES.link`), sin almacenar binarios, y se referencia desde `PERFIL_USUARIOS.foto_perfil` (FK a `IMAGES.id_images`).

---

## 1) Controlador (Java)

```java
@RestController
@RequestMapping("/api/usuarios/perfil")
public class PerfilFotoController {

  private final PerfilFotoService service;

  public PerfilFotoController(PerfilFotoService service) {
    this.service = service;
  }

  // Subir o reemplazar foto de perfil
  @PostMapping(value = "/foto", consumes = MediaType.MULTIPART_FORM_DATA_VALUE)
  public ResponseEntity<PerfilFotoResponse> subirFoto(
      @RequestPart("file") MultipartFile file,
      Authentication auth
  ) {
    int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
    PerfilFotoResponse resp = service.subirOReemplazarFoto(idUsuario, file);
    return ResponseEntity.status(HttpStatus.CREATED).body(resp);
  }

  // Quitar foto de perfil
  @DeleteMapping("/foto")
  public ResponseEntity<PerfilFotoResponse> quitarFoto(Authentication auth) {
    int idUsuario = ((JwtUser) auth.getPrincipal()).getIdUsuario();
    PerfilFotoResponse resp = service.quitarFoto(idUsuario);
    return ResponseEntity.ok(resp);
  }
}
```

---

## 2) DTOs

```java
public record PerfilFotoResponse(
    Integer idPerfilUsuario,
    Integer imageId, // puede ser null si se quitó
    String  link,    // puede ser null si se quitó
    String  mensaje
) {}
```

---

## 3) Servicio (Java)

```java
@Service
public class PerfilFotoService {

  private static final Set<String> FORMATOS_PERMITIDOS = Set.of("image/jpeg", "image/png");
  private static final long MAX_BYTES = 5L * 1024 * 1024; // 5 MB

  private final PerfilUsuarioRepository perfilRepo;
  private final ImagesRepository imagesRepo;
  private final StorageGateway storage;

  public PerfilFotoService(PerfilUsuarioRepository perfilRepo, ImagesRepository imagesRepo, StorageGateway storage) {
    this.perfilRepo = perfilRepo;
    this.imagesRepo = imagesRepo;
    this.storage = storage;
  }

  @Transactional
  public PerfilFotoResponse subirOReemplazarFoto(int idUsuario, MultipartFile file) {
    PerfilUsuario perfil = perfilRepo.findByUsuario(idUsuario)
        .orElseThrow(() -> new ForbiddenException("Perfil no encontrado para el usuario"));

    if (file == null || file.isEmpty()) throw new BadRequestException("Archivo requerido");
    if (!FORMATOS_PERMITIDOS.contains(file.getContentType()))
      throw new BadRequestException("Formato inválido. Solo JPG/PNG");
    if (file.getSize() > MAX_BYTES)
      throw new BadRequestException("Tamaño máximo excedido");

    // Subir archivo a almacenamiento controlado
    String objectKey = "perfiles/" + perfil.getIdPerfilUsuario() + "/avatar_" + System.currentTimeMillis();
    String link = storage.upload(objectKey, file);

    // Registrar imagen
    Integer imageId = imagesRepo.insert(link);

    // Actualizar perfil con nuevo image_id
    Integer oldImageId = perfil.getFotoPerfil();
    perfilRepo.setFotoPerfil(perfil.getIdPerfilUsuario(), imageId);

    if (oldImageId != null) imagesRepo.marcarObsoleta(oldImageId);

    return new PerfilFotoResponse(perfil.getIdPerfilUsuario(), imageId, link, "Foto de perfil actualizada correctamente");
  }

  @Transactional
  public PerfilFotoResponse quitarFoto(int idUsuario) {
    PerfilUsuario perfil = perfilRepo.findByUsuario(idUsuario)
        .orElseThrow(() -> new ForbiddenException("Perfil no encontrado para el usuario"));

    Integer imageId = perfil.getFotoPerfil();
    if (imageId == null) throw new NotFoundException("El usuario no tiene foto de perfil");

    perfilRepo.clearFotoPerfil(perfil.getIdPerfilUsuario());
    imagesRepo.marcarObsoleta(imageId);

    return new PerfilFotoResponse(perfil.getIdPerfilUsuario(), null, null, "Foto de perfil eliminada");
  }
}
```

---

## 4) Repositorios

```java
public interface PerfilUsuarioRepository {
  Optional<PerfilUsuario> findByUsuario(Integer idUsuario);
  void setFotoPerfil(Integer idPerfilUsuario, Integer imageId); // UPDATE PERFIL_USUARIOS SET foto_perfil = ?
  void clearFotoPerfil(Integer idPerfilUsuario);                // UPDATE PERFIL_USUARIOS SET foto_perfil = NULL
}

public interface ImagesRepository {
  Integer insert(String link);           // INSERT INTO IMAGES(link) VALUES (?)
  void marcarObsoleta(Integer id);       // marca para limpieza futura
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
UPDATE PERFIL_USUARIOS SET foto_perfil = :imageId WHERE id_perfil_usuario = :idPerfil;
UPDATE PERFIL_USUARIOS SET foto_perfil = NULL WHERE id_perfil_usuario = :idPerfil;
```

---

## 6) Mapeo exacto a la BDD `guma`

- **IMAGES**: `id_images` (PK), `link` (TEXT NOT NULL)
- **PERFIL_USUARIOS**: `foto_perfil` (FK → `IMAGES.id_images`, nullable)

Reemplazar foto → crear nuevo registro en `IMAGES` y apuntar `PERFIL_USUARIOS.foto_perfil` al nuevo `id_images`  
Quitar foto → `PERFIL_USUARIOS.foto_perfil = NULL`

---

## 7) Validaciones y reglas

- Formatos permitidos: **JPG/PNG**
- Tamaño máximo: configurable (ej. 2–5 MB)
- Una sola foto activa por usuario
- Solo el **propio usuario autenticado** puede modificar su foto
- La eliminación solo afecta la referencia, no borra el archivo físico ni la fila de `IMAGES`

---

## 8) Respuestas HTTP

- **201 Created** → subida/reemplazo exitoso
- **200 OK** → foto quitada
- **400 Bad Request** → archivo faltante, formato o tamaño inválido
- **403 Forbidden** → sin permisos
- **404 Not Found** → no tiene foto de perfil

---

## 9) Ejemplo de respuesta JSON

```json
{
  "idPerfilUsuario": 12,
  "imageId": 88,
  "link": "https://cdn.guma.com/perfiles/12/avatar_1730000000000.png",
  "mensaje": "Foto de perfil actualizada correctamente"
}
```

---

Con esto, **RF13** queda implementado: permite al usuario autenticado **gestionar su foto de perfil** mediante relación entre `PERFIL_USUARIOS.foto_perfil` e `IMAGES.link`, preservando trazabilidad, evitando almacenamiento binario en BD y habilitando reemplazo o eliminación controlada.