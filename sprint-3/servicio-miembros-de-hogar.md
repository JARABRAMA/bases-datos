# Servicio de Miembros de Hogar 

## Modelo relacional refinado

![Modelo relacional de miembros de hogar refinado](resources/relational-model-membership.png)


## Script de creación de la base de datos
```sql
CREATE SCHEMA IF NOT EXISTS users;
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE IF NOT EXISTS users.person (
    person_id   UUID            PRIMARY KEY,
    name        VARCHAR(50)     NOT NULL,
    last_name   VARCHAR(50)     NOT NULL,
    email       VARCHAR(255)    NOT NULL UNIQUE,
    password    VARCHAR(255)    NOT NULL,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW(),
    active      BOOLEAN         NOT NULL DEFAULT TRUE
);

CREATE TABLE IF NOT EXISTS users.home (
    home_id     UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(100)    NOT NULL UNIQUE,
    created_at  TIMESTAMP       NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS users.roles (
    role_id     UUID            PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(50)     NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS users.member_home (
    home_id     UUID NOT NULL,
    person_id   UUID NOT NULL,
    role_id     UUID NOT NULL,
    PRIMARY KEY (home_id, person_id),
    FOREIGN KEY (home_id)   REFERENCES users.home(home_id)     ON UPDATE CASCADE ON DELETE CASCADE,
    FOREIGN KEY (person_id) REFERENCES users.person(person_id) ON UPDATE CASCADE ON DELETE CASCADE,
    FOREIGN KEY (role_id)   REFERENCES users.roles(role_id)    ON UPDATE CASCADE ON DELETE RESTRICT
);
```



## Analisis de volumen de datos

### Tamaño por registro

#### `person`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `person_id`  | Fijo (UUID)        | 16 bytes |
| `active`     | Fijo (BOOLEAN)     | 1 byte   |
| `created_at` | Fijo (TIMESTAMP)   | 8 bytes  |
| `name`       | Variable (VARCHAR) | 10 bytes |
| `last_name`  | Variable (VARCHAR) | 20 bytes |
| `email`      | Variable (VARCHAR) | 25 bytes |
| `password`   | Variable (VARCHAR) | 80 bytes |
| Mapa de bits | 7 columnas         | 1 byte   |

Campos de longitud variable: `name`, `last_name`, `email`, `password` (4 campos).

$$
L = 4 \times 4 + (10 + 20 + 25 + 80) + (16 + 1 + 8) + 1 = 177\ \text{bytes}
$$

Ahora se calculará el número de registros por página, asumiendo que el tamaño de la página es de $4 KB$ es decir $4096$ bytes, usando la siguiente fórmula:

$$
P = 1 + 1 + 4F_r + F_r L
$$
$$
4096 = 1 + 1 + 4F_r + 177 F_r
$$
$$
4094 = 181 F_r
$$
$$
F_r = \frac{4094}{181} = 22.6 \approx 22\ \text{registros por página}
$$

Estimando que en un año habrá 1 millón de personas registradas:

$$
B_r = \frac{T_r}{F_r} = \frac{1\,000\,000}{22} = 45\,454.5 \approx 45\,455\ \text{páginas}
$$

$$
size(\text{Relación}) = B_r \times P = 45\,455 \times 4KB = 181\,820 KB
$$

Es decir, aproximadamente $0.18 GB$ en disco.

---

#### `home`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `home_id`    | Fijo (UUID)        | 16 bytes |
| `created_at` | Fijo (TIMESTAMP)   | 8 bytes  |
| `name`       | Variable (VARCHAR) | 20 bytes |
| Mapa de bits | 3 columnas         | 1 byte   |

Campos de longitud variable: `name` (1 campo).

$$
L = 4 \times 1 + 20 + (16 + 8) + 1 = 49\ \text{bytes}
$$

$$
P = 1 + 1 + 4F_r + 49 F_r
$$
$$
4094 = 53 F_r
$$
$$
F_r = \frac{4094}{53} = 77.24 \approx 77\ \text{registros por página}
$$

Asumiendo un promedio de 4 personas por hogar, con 1 millón de personas tendríamos aproximadamente $250\,000$ hogares:

$$
B_r = \frac{250\,000}{77} = 3\,246.7 \approx 3\,247\ \text{páginas}
$$

$$
size(\text{Relación}) = 3\,247 \times 4KB = 12\,988 KB \approx 0.013 GB
$$

---

#### `member_home`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `home_id`   | Fijo (UUID) | 16 bytes |
| `person_id` | Fijo (UUID) | 16 bytes |
| `role_id`   | Fijo (UUID) | 16 bytes |
| Mapa de bits | 3 columnas | 1 byte   |

Sin campos de longitud variable, por lo cual no aplica el factor $4 \times N_{var}$:

$$
L = 0 + (16 + 16 + 16) + 1 = 49\ \text{bytes}
$$

$$
P = 1 + 1 + 4F_r + 49 F_r
$$
$$
4094 = 53 F_r
$$
$$
F_r = \frac{4094}{53} = 77.24 \approx 77\ \text{registros por página}
$$

Estimando que en promedio cada persona pertenece a 1 hogar, tendríamos $1\,000\,000$ membresías:

$$
B_r = \frac{1\,000\,000}{77} = 12\,987\ \text{páginas}
$$

$$
size(\text{Relación}) = 12\,987 \times 4KB = 51\,948 KB \approx 0.052 GB
$$

---

#### `roles`

| Campo | Tipo | Tamaño estimado |
|---|---|---|
| `role_id` | Fijo (UUID)        | 16 bytes |
| `name`    | Variable (VARCHAR) | 8 bytes  |
| Mapa de bits | 2 columnas      | 1 byte   |

$$
L = 4 \times 1 + 8 + 16 + 1 = 29\ \text{bytes}
$$

Debido al valor tan pequeño y a que el catálogo de roles tendrá apenas un puñado de filas (`ADMIN`, `MEMBER`, etc.), no se realiza la estimación en disco — el tamaño es despreciable.

---

### Proyección total anual (1.000.000 de usuarios)

| Tabla | Registros | Tamaño por registro | **Total** |
|---|---:|:---:|---:|
| `person`      | 1.000.000 | 177 bytes | **~0.18 GB**  |
| `home`        |   250.000 |  49 bytes | **~0.013 GB** |
| `member_home` | 1.000.000 |  49 bytes | **~0.052 GB** |
| `roles`       |        ~5 |  29 bytes | ~145 bytes    |

**Total aproximado del microservicio: ~0.245 GB / año** (≈ 245 MB).

A diferencia del microservicio de tareas, el crecimiento aquí es **lineal con el número de usuarios** (no exponencial con la actividad), por lo que el storage se mantiene controlado.
