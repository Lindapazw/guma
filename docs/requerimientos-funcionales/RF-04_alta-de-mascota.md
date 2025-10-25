# RF04 - Registro de mascota (Backend)

Para el backend del requerimiento **RF04 – Registro de mascota**, voy a crear un **endpoint POST** llamado  
`/api/mascotas`.  
Este endpoint registra una nueva mascota y la asocia al **perfil del usuario autenticado**. La operación es **transaccional**: si falla algún paso, no queda nada a medias.

En el **controlador** (`MascotasController`) defino `@PostMapping("/mascotas")`, que recibe un `MascotaRequest` con los campos que el usuario puede cargar:

```java
private String nombre;                 // REQUERIDO -> MASCOTAS.nombre
private Integer idRaza;                // REQUERIDO -> MASCOTAS.id_raza (FK RAZAS.id_raza)
private Integer idSexo;                // REQUERIDO -> MASCOTAS.id_sexo (FK SEXOS.id_sexo)
private Integer idEstadoVital;         // REQUERIDO -> MASCOTAS.id_estado_vital (FK ESTADO_VITAL.id_estado_vital)

private LocalDate fechaNacimiento;     // opcional -> MASCOTAS.fecha_nacimiento
private Integer edadAproximada;        // opcional -> MASCOTAS.edad_aproximada
private BigDecimal peso;               // opcional -> MASCOTAS.peso
private Integer idEstadoReproductivo;  // opcional -> MASCOTAS.id_estado_reproductivo (FK)
private Integer idNivelActividad;      // opcional -> MASCOTAS.id_nivel_actividad (FK)
private Integer idTipoAlimentacion;    // opcional -> MASCOTAS.id_tipo_alimentacion (FK)
private String  alimentoDescripcion;   // opcional -> MASCOTAS.alimento_descripcion
private Integer idTemperamento;        // opcional -> MASCOTAS.id_temperamento (FK)
private LocalDate ultimaFechaCelo;     // opcional -> MASCOTAS.ultima_fecha_celo
private Integer numeroCrias;           // opcional -> MASCOTAS.numero_crias
private String  descripcion;           // opcional -> MASCOTAS.descripcion
private Integer imageId;               // opcional -> MASCOTAS.image_id (FK IMAGES.id_images)
private String  qrId;                  // opcional -> MASCOTAS.qr_id (CHAR(36))
private Integer idColor;               // opcional -> MASCOTAS.id_color (FK)
private Integer idTamano;              // opcional -> MASCOTAS.id_tamano (FK)
private Integer idRangoEdad;           // opcional -> MASCOTAS.id_rango_edad (FK)
```

Estos campos mapean 1:1 con la tabla **MASCOTAS**. La **vinculación con el dueño** se resuelve con una segunda inserción en **MASCOTA_DUENO** usando el `id_perfil_usuario` del usuario autenticado.

En el **servicio** (`MascotasService`) hago el flujo completo:

1) **Autenticación y dueño**  
Obtengo el `id_usuario` del token y busco su perfil en `PERFIL_USUARIOS` (relación 1:1).  
• Si el usuario **no tiene perfil**, devuelvo **403 (Forbidden)** con “Debes completar tu perfil antes de registrar una mascota”.

2) **Validaciones de negocio**  
• `nombre`, `idRaza`, `idSexo`, `idEstadoVital` **obligatorios** (en tu BDD son NOT NULL o FK requeridas).  
• Formatos y rangos (peso ≥ 0, fechas válidas, `qrId` con longitud 36 si viene).  
• Consistencia: o bien `fechaNacimiento` o `edadAproximada`, pero no ambas si tu política lo requiere (regla opcional).  

3) **Validaciones de integridad referencial** (previas para evitar romper la FK)  
• Existe `RAZAS.id_raza = idRaza` y su especie asociada (FK interna de RAZAS).  
• Existe `SEXOS.id_sexo = idSexo`.  
• Existe `ESTADO_VITAL.id_estado_vital = idEstadoVital`.  
• Para los opcionales con FK (actividad, alimentación, temperamento, color, tamaño, rango_edad, estado_reproductivo, image_id) verifico existencia **solo si vienen**.  
Si alguna FK no existe, devuelvo **404 (Not Found)** indicando el catálogo faltante (p. ej., “idRaza inexistente”).

4) **Transacción (ACID)**  
4.1. Inserto en **MASCOTAS** con los campos recibidos.  
• `activo` queda por defecto en **1** (tu schema lo define DEFAULT 1).  
• Guardo el `id_mascota` generado.  
4.2. Inserto en **MASCOTA_DUENO**:  
• `id_mascota` = el generado.  
• `id_perfil_usuario` = perfil del usuario autenticado.  
• `fecha_alta` = hoy.  
• `activo` = 1.  
4.3. `COMMIT`. Si algo falla (FK, NOT NULL, etc.), hago `ROLLBACK` y devuelvo el error mapeado.

5) **Respuestas del endpoint**  
• **201 (Created)** con:  
```json
{
  "mensaje": "Mascota registrada",
  "idMascota": 123,
  "dueno": {
    "idPerfilUsuario": 456
  }
}
```  
• **400 (Bad Request)** por validaciones de formato/reglas (ej.: nombre vacío, peso negativo).  
• **403 (Forbidden)** si el usuario no tiene perfil.  
• **404 (Not Found)** si alguna FK referida no existe (raza, sexo, etc.).  

6) **Seguridad y hardening**  
• El dueño **no se toma del body**: siempre del token → evita registrar a nombre de terceros.  
• Rate limiting para mitigar abuso.  
• Logs con correlation-id.  
• Sanitización básica de `descripcion`/`alimentoDescripcion`.

7) **Resumen de mapeo exacto a la BDD**  
**Inserción en MASCOTAS** (obligatorios y opcionales):  
- Obligatorios: `nombre`, `id_raza` (FK RAZAS), `id_sexo` (FK SEXOS), `id_estado_vital` (FK ESTADO_VITAL).  
- Opcionales: `fecha_nacimiento`, `edad_aproximada`, `peso`, `id_estado_reproductivo`, `id_nivel_actividad`, `id_tipo_alimentacion`, `alimento_descripcion`, `id_temperamento`, `ultima_fecha_celo`, `numero_crias`, `descripcion`, `image_id`, `qr_id`, `id_color`, `id_tamano`, `id_rango_edad`, `activo` (DEFAULT 1).  

**Inserción en MASCOTA_DUENO**:  
- `id_mascota` (FK a MASCOTAS, con `ON DELETE CASCADE`).  
- `id_perfil_usuario` (FK a PERFIL_USUARIOS, con `ON DELETE CASCADE`).  
- `fecha_alta` (DATE, requerido por negocio).  
- `activo` (DEFAULT 1).

Con esto cierro el RF04: el backend registra la mascota en **MASCOTAS** y crea la relación de tenencia en **MASCOTA_DUENO**, manteniendo la integridad con todos los catálogos (`RAZAS`, `SEXOS`, `ESTADO_VITAL`, etc.) definidos en tu esquema **GUMA**.