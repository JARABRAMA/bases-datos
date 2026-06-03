# Servicio de tareas


## Modelo relacional 

![Modelo relacional del servicio de tareas](./resources/rational-model-tasks.png)


## Script de creación de la base de datos

```sql
create table task_management.guest_tasks (
  guest_id uuid not null, 
  task_id uuid not null, 
  primary key (guest_id, task_id)
);
create table task_management.guests (
  is_active boolean not null, 
  created_at timestamp(6), 
  guest_id uuid not null, 
  email varchar(255), 
  last_name varchar(255), 
  name varchar(255), 
  password_hash varchar(255), 
  primary key (guest_id)
);
create table task_management.priorities (
  priority_id uuid not null, 
  name varchar(255) not null, 
  primary key (priority_id)
);
create table task_management.status (
  status_id uuid not null, 
  name varchar(50) not null, 
  primary key (status_id)
);
create table task_management.tasks (
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP, 
  deadline timestamp(6) not null, 
  guest_id uuid, 
  home_id uuid not null, 
  priority_id uuid not null, 
  status_id uuid not null, 
  task_id uuid not null, 
  name varchar(100) not null, 
  description varchar(255) not null, 
  primary key (task_id)
);
alter table 
  if exists task_management.tasks 
add 
  constraint FKnq0d4mra8tpuwwak86ctvhfsb foreign key (priority_id) references task_management.priorities;
alter table 
  if exists task_management.tasks 
add 
  constraint FKfmlm4rxoa19247blv9g96eacd foreign key (status_id) references task_management.status;

```



## Volumen de Datos Estimado

### Tamaño por registro

#### `guests`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `guest_id` | Fijo (UUID) | 16 bytes |
| `active` | Fijo (BOOLEAN) | 1 byte |
| `created_at` | Fijo (TIMESTAMP) | 8 bytes |
| `email` | Variable (VARCHAR) | 25 bytes |
| `last_name` | Variable (VARCHAR) | 20 bytes |
| `name` | Variable (VARCHAR) | 10 bytes |
| `password_hash` | Variable (VARCHAR) | 80 bytes |
| Mapa de bits | 7 columnas | 1 byte |

**`L = (4 × 4) + (25 + 20 + 10 + 80) + (16 + 1 + 8) + 1 = 177 bytes`**

Ahora se calculará el número de registros por página, asumiendo que el tamaño de la página es de $4 KB$ es decir 4096 bytes, y utilizando la siguiente fórmula

$$
P = 1 + 1 + 4F_r + F_r L
$$
$$
4096 = 1 + 1 + 4F_r + 177 F_r
$$
$$
4094 = 181F_r
$$
$$
F_r = \frac{4094}{181} = 22.6 \approx 22\ \text{registros por paginas}
$$

Ahora calcularemos el número de páginas que ocupa la relación en el disco $B_r$, usando una estimación de cuántas tuplas tendrá la tabla en un año $T_r$. Usando la siguiente relación:

$$
B_r = \frac{T_r}{F_r} 
$$

Estimando que en un año habrá 1 millón de usuarios registrados en un año. 

$$
B_r = \frac{1 000 000}{22} = 45454.545 \approx 45455 \text{ paginas}
$$

Finalmente, para saber cuánto espacio ocupa la relación dentro del disco.

$$
size(Relacion) = B_r \times P = 45455 \times 4KB = 181820KB
$$

Así que la relación estará ocupando aproximadamente $0.18182GB$ dentro del disco

---

#### `tasks`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `task_id` | Fijo (UUID) | 16 bytes |
| `priority_id` | Fijo (UUID) | 16 bytes |
| `id_home` | Fijo (UUID) | 16 bytes |
| `status_id` | Fijo (UUID) | 16 bytes |
| `guest_id` | Fijo (UUID) | 16 bytes |
| `deadline` | Fijo (TIMESTAMP) | 8 bytes |
| `created_at` | Fijo (TIMESTAMP) | 8 bytes |
| `description` | Variable (TEXT) | 100 bytes |
| `name` | Variable (VARCHAR) | 15 bytes |
| Mapa de bits | 9 columnas | 2 bytes |

**`L = (4 × 2) + (100 + 15) + (16×5 + 8×2) + 2 = 221 bytes`**

Ahora se calculará el número de registros por página, para la tabla tareas de la misma manera que se realizó anteriormente

$$
P = 1 + 1 + 4F_r + F_r L
$$
$$
4096 = 1 + 1 + 4F_r + 221 F_r
$$
$$
4094 = 225F_r
$$
$$
F_r = \frac{4094}{225} = 18.19 \approx 18\ \text{registros por páginas}
$$

Ahora calcularemos el número de páginas que ocupa la relación en el disco $B_r$, usando una estimación de cuántas tuplas tendrá la tabla en un año $T_r$. Usando la siguiente relación:

$$
B_r = \frac{T_r}{F_r} 
$$

Como se estimó un millón de usuarios en un año, aproximadamente cada usuario tendrá una tarea diaria, por lo cual en un mes cada usuario tendrá 30 tareas. Por lo tanto en un mes tendremos aproximadamente 30 millones de tareas

$$
B_r = \frac{30000000}{18} = 1666666.66 \approx 1666667 \text{ paginas}
$$

Finalmente para saber cuánto espacio ocupa la relación dentro del disco.

$$
size(Relacion) = B_r \times P = 1666667 \times 4KB = 6666668KB
$$


Así que la relación estará ocupando aproximadamente $6.666668GB$ dentro del disco cada mes

En un año tendríamos $6.666668GB \times 12 = 80.000016GB$

---

#### `status` y `priority`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `id` | Fijo (UUID) | 16 bytes |
| `name` | Variable (VARCHAR) | 8 bytes |
| Mapa de bits | 2 columnas | 1 byte |

**`L = (4 × 1) + 8 + 16 + 1 = 29 bytes`**

Debido al valor tan pequeño que ocuparán ambas tablas no se realizará la estimación en disco ya que es insignificante

---

### Proyección total anual (1.000.000 usuarios)

| Tabla | Registros | Tamaño por registro | **Total** |
|---|---:|:---:|---:|
| `guests` | 1.000.000 | 177 bytes | **~0.18 GB** |
| `tasks` | 360.000.000 | 221 bytes | **~80.000016 GB** |
| `status` | 10 | 29 bytes | ~290 bytes |
| `priority` | 10 | 29 bytes | ~290 bytes |
