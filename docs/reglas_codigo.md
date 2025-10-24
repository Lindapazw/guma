# CODE_RULES.md — Reglas de código obligatorias (Java 17 + Swing + JDBC)

> Este documento es un contrato técnico. Toda contribución (humana o asistida por IA) **DEBE** cumplir estas reglas sin excepción.

## 0) Alcance y supuestos

- Arquitectura en 5 capas: **domain** → **backend** → **data** → **application** → **frontend (Swing)**.
- Sin Maven/Gradle. Dependencias en `lib/`. Compilación/ejecución vía `build.sh/.bat` y `run.sh/.bat`.
- Persistencia con **JDBC** (MySQL Connector/J). Sin ORM.

---

## 1) Principios generales

1. **Respetar estrictamente el DER**: no crear tablas/campos/relaciones que no existan; no persistir datos derivados. Mantener 1:1 entidades↔tablas.
2. **Separación de capas**:
   - `frontend` (Swing) **no** accede a repos ni SQL; solo llama a `application`.
   - `application` orquesta casos de uso (DTOs, mappers, transacciones) y usa **servicios** de `backend`.
   - `backend` concentra las **reglas de negocio** y declara **ports** (interfaces) de persistencia y servicios externos.
   - `data` implementa los **ports** con JDBC. Sin lógica de negocio.
   - `domain` no depende de nada externo.
3. **DTOs en fronteras**: `application` expone/consume DTOs. **Nunca** filtrar entidades del dominio a la UI.
4. **Transacciones**: definir límites claros en `application` o en la capa que gestione el `DataSource`. Escrituras atómicas.
5. **Validación en capas**:
   - **Entrada** (DTOs) en `application`.
   - **Reglas** de negocio en `backend` y/o `domain`.
6. **Errores útiles**: usar excepciones con códigos/mensajes accionables. No silenciar errores.

---

## 2) Estilo y nomenclatura

- **Idioma**: Todo el código (clases, métodos, variables, comentarios, mensajes) **DEBE** escribirse en **español**. Prohibido usar inglés excepto para palabras técnicas sin traducción razonable (e.g., `DTO`, `Repository`, `Exception`).
- **Paquetes**: `com.guma.<capa>.<subpaquete>` (todo en minúsculas).
- **Clases/Interfaces**: `PascalCase` (e.g., `MascotaService`, `UsuarioRepository`).
- **Métodos/variables**: `camelCase`.
- **Constantes**: `UPPER_SNAKE_CASE`.
- **Nombres semánticos**: evitar abreviaturas crípticas.
- **Formateo**: 2–4 espacios, sin tabs. Archivos UTF-8.

---

## 3) Buenas prácticas POO

- **SOLID**: clases pequeñas y enfocadas; dependencias por interfaces (ports).
- **Tell, don’t ask**: servicios expongan operaciones de alto nivel.
- **Composición sobre herencia**.
- **Inmutabilidad** donde aplique (`final`).
- **No duplicar lógica** (DRY): extraer a `service` o `util`.

---

## 4) Acceso a datos (JDBC)

1. **Conexión**: `DriverManager.getConnection(...)` desde un `DataSource` propio; propiedades en `resources/application.properties`.
2. **SQL parametrizado**: **siempre** `PreparedStatement` con `?`; prohibido concatenar strings con datos de usuario.
3. **Ciclo JDBC** (patrón obligatorio):
   ```java
   try (Connection con = dataSource.get();
        PreparedStatement ps = con.prepareStatement(SQL)) {
       // setX(...)
       try (ResultSet rs = ps.executeQuery()) {
           // mapear
       }
   }
   ```
4. **Mapeo**: `data.mapper.*` transforma `ResultSet` → entidades de `domain`. Sin lógica de negocio.
5. **Transacciones**: `con.setAutoCommit(false)` y `commit()/rollback()` donde corresponda; ámbito mínimo.
6. **Índices/consultas**: diseñar SELECTs por caso de uso; evitar `SELECT *` en productivo (permitido en prototipos controlados).
7. **Errores**: capturar `SQLException` y envolver en excepción de infraestructura con contexto (sin exponer credenciales ni SQL completo).

---

## 5) Validación y excepciones

- **Precondiciones**: validar DTOs en `application` (no nulos, formatos, rangos, coherencia inter-campo).
- **Reglas de negocio**: lanzar `BusinessException` (runtime) con código/causa.
- **Checked vs runtime**: checked para condiciones recuperables externas (I/O), runtime para violaciones de dominio.
- **Mensajes**: claros, sin datos sensibles.

---

## 6) Concurrencia y Swing (EDT)

- **Nunca** ejecutar I/O o acceso a DB en el **EDT** (hilo de interfaz).
- Operaciones de backend en hilos de trabajo; actualización de UI con `SwingUtilities.invokeLater(...)`.
- Cancelación/cierres limpios de recursos.

---

## 7) Logging y configuración

- **Sin** `System.out` en código final; usar `java.util.logging` (o un wrapper simple).
- Niveles: `INFO` para hitos, `FINE`/`FINER` para detalle, `SEVERE` en fallas.
- Archivos de propiedades en `resources/`. Nada de secretos en repositorio público.

---

## 8) Reglas “NUNCA” (prohibiciones absolutas)

- **Nunca** romper el DER ni su cardinalidad.
- **Nunca** acceder a DB desde `frontend`.
- **Nunca** exponer entidades de `domain` a la UI; usar DTOs.
- **Nunca** ocultar silenciosamente excepciones (`catch (e) {}` vacío).
- **Nunca** concatenar SQL con datos de usuario.
- **Nunca** hardcodear strings mágicos/ids (usar constantes o enums).
- **Nunca** duplicar lógica (refactorizar).
- **Nunca** bloquear el EDT con I/O.
- **Nunca** introducir dependencias externas sin documentarlas en `lib/` y `BUILD_RUN.md`.

---

## 9) Regla de _Revisión de funcionalidad existente_ (coherencia interna)

> **Antes de implementar cualquier nueva funcionalidad**, se debe revisar el repositorio para encontrar una funcionalidad similar y **copiar el patrón interno**, validando:

- Arquitectura y **capas** involucradas.
- **Archivos** y ubicaciones (paquetes) creados para casos análogos.
- **Convenciones de nombres** (clases, métodos, DTOs, repos).
- **Estilo de codificación** y estructura de paquetes.
- **Patrones** aplicados (puertos/adaptadores, mappers, facades).
- **Manejo de errores** y validaciones.
- **Dependencias** ya disponibles (evitar introducir nuevas si no es necesario).
- **Flujo de UI** (cómo la UI invoca `application` y cómo se actualiza).

Si no existe un caso similar, **documentar** el nuevo patrón propuesto y mantener consistencia con el resto del proyecto.

---

## 10) Regla de _TDD documental_ (tests primero)

> **Antes de implementar una nueva funcionalidad**, crear un archivo de especificación de pruebas en **`/docs/tests`** con el siguiente formato:

- Nombre: `YYYYMMDD_<feature>.md` (ej.: `20251024_crear_mascota.md`).
- Secciones mínimas:
  1. **Descripción** de la funcionalidad.
  2. **Precondiciones** (datos, estado del sistema).
  3. **Criterios de aceptación** (Given/When/Then o lista verificable).
  4. **Casos de prueba** (entradas, pasos, resultado esperado).
  5. **Restricciones** (performance, validaciones, seguridad).
  6. **Datos de prueba** (usuarios, especies/razas, etc.).
- La implementación **debe** cubrir todos los criterios definidos.
- Si durante el desarrollo cambia el alcance, **actualizar primero** este `.md` antes de modificar el código.

---

## 11) Contraseñas y seguridad

- Hash con **jBCrypt** (si se usa), nunca texto plano.
- No loguear tokens ni contraseñas.
- Sanitizar entradas que se muestren en UI (evitar inyección en componentes HTML-like o renderers).

---

## 12) Ejemplos de contratos por capa (mini-guía)

- **backend.ports.persistence.MascotaRepository**
  - `Mascota save(Mascota m)`
  - `Optional<Mascota> findById(int id)`
  - `List<Mascota> search(MascotaFilter f, int limit, int offset)`
- **application.facade.MascotaFacade**
  - `MascotaDTO createMascota(MascotaCreateDTO dto)`
  - `MascotaDTO getById(int id)`
  - `List<MascotaListItemDTO> search(MascotaSearchDTO dto)`

_(Firmas ilustrativas; la especificación exacta se define en `API_CONTRACT.md`.)_

---

## 13) Revisión y cumplimiento

- Todo PR o cambio debe enlazar al `.md` de pruebas en `/docs/tests`.
- Cambios que toquen múltiples capas requieren verificación de dependencias y actualización de docs afectadas (`FILES_TO_CREATE.md`, `API_CONTRACT.md` si aplica).
- Incumplimientos de este documento son motivo de rechazo automático del cambio.

---
