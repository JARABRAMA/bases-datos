#  Entregable Sprint 2

> Para este sprint encontrarás un archivo Markdown por cada microservicio en la carpeta `sprint-2`.

---

##  Contexto General — Esquema de Base de Datos

La aplicación usa una **base de datos relacional compartida**, separada por schemas:

| Schema | Microservicio | Información |
|---|---|---|
| `auth` | authentication | Usuarios, contraseñas cifradas, tokens |
| `users` | user-membership-service | Personas, hogares, roles de negocio, miembros del hogar |
| `task_management` | tasks | Tareas, estados, prioridades, invitados asignados |

---

##  Diagrama ER de Microservicios

Diagrama general que muestra las entidades de cada microservicio y cómo se conectan entre ellos:

- **Líneas continuas**: relaciones internas dentro de un mismo servicio.
- **Líneas punteadas**: comunicación entre microservicios.
  - `AUTH_User → MH_Person` y `AUTH_User → TASKS_User`: sincronización vía Azure Storage Queue.
  - `MH_Home → TASKS_Task`: referencia lógica por `home_id` (sin FK física, BDs separadas por schema).
- **Nodos en naranja** (`MH_Person`, `TASKS_User`): copias replicadas del `User` de Authentication.

![Diagrama ER de microservicios](./sprint-2/diagrama-er-microservicios.png)

---

##  Roles Técnicos de Base de Datos

> Los roles definidos no son roles funcionales de usuario como `ADMIN`, `MEMBER` o `GUEST`.  
> Son roles técnicos para controlar quién puede administrar, modificar, consultar o soportar la información.

---

###  `db_admin`

Rol administrador total de la base de datos.

- Puede crear, modificar y eliminar schemas, tablas, índices, constraints, funciones, usuarios y roles.
- Accede a toda la información de los schemas: `auth`, `users` y `task_management`.
- Es el único rol autorizado para ejecutar operaciones destructivas o estructurales críticas (`DROP`, `ALTER` profundo, eliminación de schemas, modificación de permisos globales).

>  Todas sus acciones deben ser auditadas, especialmente cambios sobre tablas, schemas, roles, permisos y eliminación de objetos.

---

###  `migration_role`

Rol usado exclusivamente para ejecutar migraciones controladas.

- Puede crear o modificar estructuras necesarias para la evolución del sistema: tablas, columnas, índices, relaciones, constraints y schemas.
- No debe usarse en tiempo de ejecución de las aplicaciones.
- No debe usarse por desarrolladores para consultas manuales.
- Su uso debe limitarse a pipelines de despliegue o procesos formales de migración.
- Trabaja sobre `auth`, `users` y `task_management`, pero únicamente con propósito estructural.

---

###  `developer_role`

Rol pensado para desarrollo y pruebas.

- Puede consultar, insertar y modificar datos de prueba en los schemas del sistema.
- Puede crear estructuras temporales o auxiliares para validar funcionalidades.

No debe acceder a información sensible:

| Schema | Datos restringidos |
|---|---|
| `auth` | `password_hash`, tokens |
| `users` | Contraseñas de personas |
| `task_management` | Hashes o datos sensibles de invitados |

>  En producción, este rol debe estar deshabilitado, bloqueado o no asignado a ningún usuario real. No puede eliminar tablas, schemas ni ejecutar cambios destructivos.

---

###  `readonly_role`

Rol de solo lectura global.

- Puede consultar información de `auth`, `users` y `task_management` para reportes, debugging o análisis.
- No puede insertar, actualizar, eliminar ni modificar estructura.
- Su objetivo es permitir observabilidad sin riesgo de alterar datos.

>  Si se requiere mayor seguridad, debería consultar vistas filtradas en lugar de tablas directas, para evitar exposición de datos sensibles.

---

###  `support_role`

Rol para soporte técnico operativo.

- Puede consultar información para diagnosticar problemas: hogares, miembros, tareas, estados, prioridades y asignaciones.
- Puede realizar actualizaciones puntuales y controladas, por ejemplo corregir el estado de una tarea en `task_management.tasks`.
- No puede eliminar registros, modificar estructura, cambiar roles, acceder a tokens, ver contraseñas ni modificar información crítica de autenticación.

> Su alcance debe limitarse a operaciones de soporte, no administración.

---

## Resumen de Control de Acceso

| Rol | Lectura | Escritura | Cambios estructurales | Datos sensibles | `DROP` |
|---|:---:|:---:|:---:|:---:|:---:|
| `db_admin` |  Total |  Total |  Sí |  Sí |  Sí |
| `migration_role` |  Limitada |  Limitada |  Sí |  Solo en migración |  No recomendado |
| `developer_role` |  Parcial |  Datos de prueba |  Temporal/pruebas |  No |  No |
| `readonly_role` | Sí |  No | No |  Depende de vistas |  No |
| `support_role` |  Operativa |  Muy limitada |  No |  No |  No |
