
# Arquitectura del Proyecto GUMA (Java + Swing + JDBC)

## Visión general

La aplicación GUMA se basa en una arquitectura modular de **5 capas**.  
Cada capa cumple una función específica, y la comunicación entre ellas se realiza de forma **unidireccional**.

```mermaid
flowchart LR
  UI[frontend (Swing)] --> APP[application (facade + dto)]
  APP --> BE[backend (services + ports)]
  BE --> DM[domain (entities + rules)]
  BE -->|Ports| DT[data (jdbc impl)]
  DT --> DM
```

Dependencias:
- `frontend` → `application` → `backend` → `domain`
- `data` implementa los puertos declarados por `backend`
- Ninguna capa accede directamente a otra que no sea su inmediata dependiente

---

## 1. Capa DOMAIN

**Propósito:** Representa el núcleo del sistema: entidades, reglas del negocio y tipos fundamentales.

**Contenido:**
- `entities/` – Clases que reflejan las tablas de la base de datos (Usuario, Mascota, PerfilUsuario, etc.)
- `enums/` – Enumeraciones de valores fijos (Sexo, EstadoVital, etc.)
- `valueobjects/` – Objetos de valor (Email, Dni, etc.)
- `services/` – Lógica de dominio pura sin dependencias externas

**Reglas:**
- No depende de ninguna otra capa
- No accede a base de datos
- No importa librerías de UI ni JDBC

---

## 2. Capa BACKEND

**Propósito:** Contiene la lógica de negocio y las reglas de aplicación. Define los *puertos* de acceso a datos.

**Contenido:**
- `services/` – Casos de uso y reglas de negocio (UsuarioService, MascotaService...)
- `ports/`
  - `persistence/` – Interfaces de repositorios (UsuarioRepository, MascotaRepository...)
  - `storage/` – Interfaces para almacenamiento de imágenes u otros medios
- `exceptions/` – Excepciones de negocio (`BusinessException`, etc.)

**Reglas:**
- Depende solo de `domain`
- No accede directamente a la base de datos
- Define *qué necesita*, no *cómo* se obtiene

---

## 3. Capa DATA

**Propósito:** Implementa la lógica de acceso a datos. Conecta con MySQL mediante JDBC.

**Contenido:**
- `jdbc/` – Implementaciones de los repositorios definidos en `backend.ports`
- `mapper/` – Mapeos entre ResultSet y entidades del dominio
- `config/` – Inicialización de la conexión (`DataSource`)

**Reglas:**
- Implementa interfaces de `backend.ports`
- Depende de `backend` (interfaces) y `domain` (entidades)
- No incluye lógica de negocio
- Usa `try-with-resources` y `PreparedStatement` parametrizado

---

## 4. Capa APPLICATION

**Propósito:** Orquesta los casos de uso. Traduce entre DTOs y objetos de dominio.

**Contenido:**
- `dto/` – Objetos de transferencia para entrada/salida (UsuarioDTO, MascotaDTO...)
- `mapper/` – Conversores DTO ↔ entidad
- `facade/` – Fachadas o servicios de aplicación (AuthFacade, MascotaFacade, etc.)
- `commands_queries/` – Comandos y consultas específicos

**Reglas:**
- Depende de `backend` y `domain`
- No ejecuta SQL ni accede a repositorios
- Coordina transacciones y maneja errores de aplicación

---

## 5. Capa FRONTEND

**Propósito:** Interfaz gráfica del usuario en **Swing**. Representa la capa de presentación.

**Contenido:**
- `ui/` – Ventanas, paneles y diálogos (LoginDialog, MainFrame, MascotaFormDialog...)
- `viewmodel/` – Modelos de datos para la UI
- `actions/` – Clases que ejecutan acciones y comunican con `application`

**Reglas:**
- Depende solo de `application`
- Nunca accede a base de datos
- Toda operación pesada o I/O ocurre fuera del hilo de interfaz (EDT)

---

## 6. Capa SHARED (opcional)

**Propósito:** Centraliza clases comunes y utilidades transversales.

**Contenido:**
- `util/` – Helpers genéricos (Validadores, FechaUtils...)
- `validation/` – Reglas de validación
- `result/` – Tipos genéricos (`Result<T>`, `Either`, etc.)

---

## 7. Estructura de carpetas

```
guma/
  src/
    com/guma/domain/
    com/guma/backend/
    com/guma/data/
    com/guma/application/
    com/guma/frontend/
    com/guma/shared/
  resources/
    application.properties
  lib/
    mysql-connector-j-8.x.x.jar
    jbcrypt-0.4.jar
  out/
  build.sh / build.bat
  run.sh   / run.bat
```

---

## 8. Flujo de un caso de uso

Ejemplo: **Registrar Mascota**
1. `frontend` → Usuario completa formulario → `MascotaFormDialog`
2. `application` → `MascotaFacade.createMascota(dto)` valida datos y llama a servicios
3. `backend` → `MascotaService` crea la entidad y llama a `MascotaRepository.save`
4. `data` → `MascotaRepositoryJdbc` ejecuta el `INSERT`
5. `domain` → Entidad `Mascota` valida coherencia interna
6. `application` → Devuelve `MascotaDTO` a la UI
7. `frontend` → Actualiza la interfaz

---

## 9. Sin Maven ni Gradle

El proyecto se compila y ejecuta con scripts simples:

### build.sh
```bash
#!/usr/bin/env bash
set -e
mkdir -p out
CP="lib/*:resources"
javac -encoding UTF-8 -source 17 -target 17 -cp "$CP" -d out $(find src -name "*.java")
```

### run.sh
```bash
#!/usr/bin/env bash
CP="out:lib/*:resources"
java -cp "$CP" com.guma.frontend.Main
```

---

## 10. Principios rectores

- **Separación estricta de capas**
- **Reglas de negocio centralizadas en backend**
- **Dominio inmutable y sin dependencias externas**
- **UI desacoplada de persistencia**
- **Uso responsable de JDBC con PreparedStatement**
- **Todo acceso a base fuera del EDT (Event Dispatch Thread)**

---

Esta arquitectura permite un desarrollo ordenado, mantenible y sin frameworks externos, cumpliendo con los requerimientos académicos del proyecto.
