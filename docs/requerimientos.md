# Requerimientos Funcionales - Sistema GUMA (Gestión Unificada de Mascotas)

| Código | Caso de Uso | Actor | Prioridad | Comentario |
|---------|--------------|--------|------------|-------------|
| **RF01 – Registro de usuario** | UC01 – Registrarse | Usuario | Obligatorio | Alta con email/clave y verificación. |
| **RF02 – Inicio de sesión** | UC02 – Iniciar sesión | Usuario | Obligatorio | Autenticación y sesión activa. |
| **RF03 – Crear/Actualizar perfil** | UC03 – Crear/Actualizar Perfil de Usuario | Usuario | Obligatorio | Datos personales, dirección y foto opcional. |
| **RF04 – Alta de mascota** | UC04 – Crear Mascota | Usuario | Obligatorio | Especie, raza, sexo, color, tamaño, estado; vinculada al usuario. |
| **RF05 – Agregar co-dueño** | UC05 – Agregar Co-dueño | Usuario | Opcional | Comparte tenencia con otro usuario verificado. |
| **RF06 – Remover co-dueño** | UC06 – Remover Co-dueño | Usuario | Opcional | Baja lógica del vínculo compartido. |
| **RF07 – Búsqueda de mascotas** | UC07 – Consultar/Buscar Mascotas | Usuario | Obligatorio | Filtros por especie, raza, edad, tamaño, estado. |
| **RF08 – Ver detalle de mascota** | UC08 – Ver Detalle de Mascota | Usuario | Obligatorio | Ficha completa con relaciones y multimedia. |
| **RF09 – Editar mascota** | UC09 – Editar Mascota | Usuario | Obligatorio | Actualiza atributos y relaciones. |
| **RF10 – Baja para mí** | UC10 – Baja de Mascota (para mí) | Usuario | Obligatorio | Desvincula al usuario manteniendo la mascota activa. |
| **RF11 – Baja global** | UC11 – Baja Global de Mascota (archivar) | Admin | Opcional | Archiva mascota y oculta de búsquedas. |
| **RF12 – Avatar de mascota** | UC12 – Gestionar Avatar de Mascota | Usuario | Opcional | Alta/actualización de imagen principal. |
| **RF13 – Foto de perfil** | UC13 – Gestionar Foto de Perfil de Usuario | Usuario | Opcional | Alta/actualización de foto de perfil. |