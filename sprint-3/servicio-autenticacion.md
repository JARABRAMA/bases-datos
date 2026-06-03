# Servicio de Autenticación

## Modelo relacional refinado del servicio 

![Modelo relacional de autenticación refinado](resources/relational-model-authentication.png)

## Script de creación de la base de datos

```sql
create table auth.tokens (
  expirated_at timestamp(6), 
  expiration_date timestamp(6) not null, 
  token_id uuid not null, 
  user_id uuid unique, 
  token_hash varchar(255) not null unique, 
  token_type varchar(255) not null check (
    (
      token_type in ('ACCESS', 'REFRESH')
    )
  ), 
  primary key (token_id)
);
create table auth.users (
  is_active boolean not null, 
  created_at timestamp(6) not null, 
  user_id uuid not null, 
  lastname varchar(50) not null, 
  name varchar(50) not null, 
  email varchar(255) not null unique, 
  password_hash varchar(255) not null, 
  primary key (user_id)
);
create table auth.activation_tokens (
  attempts INT DEFAULT 0 not null, 
  invalidated BOOLEAN DEFAULT FALSE not null, 
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP not null, 
  expires_at timestamp(6) not null, 
  id uuid not null, 
  user_id uuid not null, 
  code_hash varchar(255) not null, 
  primary key (id)
);
create table auth.tokens (
  expirated_at timestamp(6), 
  expiration_date timestamp(6) not null, 
  token_id uuid not null, 
  user_id uuid unique, 
  token_hash varchar(255) not null unique, 
  token_type varchar(255) not null check (
    (
      token_type in ('ACCESS', 'REFRESH')
    )
  ), 
  primary key (token_id)
);
create table auth.two_factor_auth_tokens (
  attempts INT DEFAULT 0 not null, 
  invalidated BOOLEAN DEFAULT FALSE not null, 
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP not null, 
  expires_at timestamp(6) not null, 
  id uuid not null, 
  user_id uuid not null, 
  code_hash varchar(255) not null, 
  primary key (id)
);
create table auth.users (
  is_active BOOLEAN DEFAULT FALSE not null, 
  created_at timestamp(6) not null, 
  user_id uuid not null, 
  lastname varchar(50) not null, 
  name varchar(50) not null, 
  email varchar(255) not null unique, 
  password_hash varchar(255) not null, 
  primary key (user_id)
);
alter table 
  if exists auth.activation_tokens 
add 
  constraint FKgtry5rof27b5hdur1a11y0gsn foreign key (user_id) references auth.users;
alter table 
  if exists auth.tokens 
add 
  constraint FK2dylsfo39lgjyqml2tbe0b0ss foreign key (user_id) references auth.users;
alter table 
  if exists auth.two_factor_auth_tokens 
add 
  constraint FKe7aqfopyjm7wfc0ohceqbsm2y foreign key (user_id) references auth.users;

```

## Volumen de datos aproximado por tablas

### Tabla `auth.users`

| Campo | Tamaño aprox |
|---|---|
| user_id | 16 B |
| name | 30 B |
| lastname | 30 B |
| password_hash | 60 B |
| created_at | 8 B |
| is_active | 1 B |
| email | 50 B |

Ningún campo admite nulos, por lo que no hay mapa de bits. Los campos de longitud variable son: `name`, `lastname`, `password_hash`, `email`.

$$
L = 4 \times \text{N campos de longitud variable} + \sum \text{size(campos de longitud fija)} + \sum \text{tamaño estimado de cada campo de longitud variable}
$$

$$
L = 4 \times 4 + (16 + 8 + 1) + (30 + 30 + 60 + 50)
$$

$$
L = 211\ \text{bytes}
$$

Calculando el número de registros por página $F_r$, asumiendo $P = 4096\ \text{bytes}$:

$$
4096 = 1 + 1 + 4F_r + 211F_r \implies F_r = \frac{4094}{215} \approx 19\ \text{registros/página}
$$

Estimando $T_r = 1\,000\,000$ usuarios en un año:

$$
B_r = \frac{1\,000\,000}{19} \approx 52\,632\ \text{páginas}
$$

$$
\text{size(Relación)} = 52\,632 \times 4\ \text{KB} \approx 2.10\ \text{GB}
$$

---

### Tabla `auth.tokens`

| Campo | Tamaño |
|---|---|
| token_id | 16 B |
| token_hash | 60 B |
| expiration_date | 8 B |
| expirated_at | 8 B |
| user_id | 16 B |
| token_type | 20 B |

Solo `expirated_at` admite nulos → mapa de bits de 1 byte. Campos de longitud variable: `token_hash`, `token_type`.

$$
L = 4 \times 2 + (16 + 8 + 8 + 16) + (60 + 20) + 1 = 137\ \text{bytes}
$$

$$
4096 = 1 + 1 + 4F_r + 137F_r \implies F_r = \frac{4094}{141} \approx 29\ \text{registros/página}
$$

Estimando $T_r = 10\,000\,000$ tokens en un año:

$$
B_r = \frac{10\,000\,000}{29} \approx 344\,828\ \text{páginas}
$$

$$
\text{size(Relación)} = 344\,828 \times 4\ \text{KB} \approx 1.315\ \text{GB}
$$

---

### Tabla `auth.activation_tokens`

| Campo | Tamaño |
|---|---|
| id | 16 B |
| user_id | 16 B |
| code_hash | 60 B |
| created_at | 8 B |
| expires_at | 8 B |
| attempts | 4 B |
| invalidated | 1 B |

Ningún campo admite nulos. Campos de longitud variable: `code_hash`.

$$
L = 4 \times 1 + (16 + 16 + 8 + 8 + 4 + 1) + 60
$$

$$
L = 4 + 53 + 60 = 117\ \text{bytes}
$$

$$
4096 = 1 + 1 + 4F_r + 117F_r \implies F_r = \frac{4094}{121} \approx 33\ \text{registros/página}
$$

Los tokens de activación se generan una vez por registro de usuario y se invalidan rápidamente, por lo que se estima un volumen equivalente al de usuarios activos más un margen por reintentos: $T_r = 1\,500\,000$ tokens en un año.

$$
B_r = \frac{1\,500\,000}{33} \approx 45\,455\ \text{páginas}
$$

$$
\text{size(Relación)} = 45\,455 \times 4\ \text{KB} \approx 177.6\ \text{MB}
$$

---

### Tabla `auth.two_factor_auth_tokens`

| Campo | Tamaño |
|---|---|
| id | 16 B |
| user_id | 16 B |
| code_hash | 60 B |
| created_at | 8 B |
| expires_at | 8 B |
| attempts | 4 B |
| invalidated | 1 B |

La estructura es idéntica a `activation_tokens`, por lo que el tamaño de registro es el mismo:

$$
L = 4 \times 1 + (16 + 16 + 8 + 8 + 4 + 1) + 60 = 117\ \text{bytes}
$$

$$
F_r = 33\ \text{registros/página}
$$

A diferencia de los tokens de activación, los tokens 2FA se generan en cada inicio de sesión de usuarios que tengan habilitada esa funcionalidad. Estimando que el 30% de los usuarios usan 2FA y realizan en promedio 5 inicios de sesión por mes, se obtiene $T_r \approx 1\,800\,000$ tokens en un año.

$$
B_r = \frac{1\,800\,000}{33} \approx 54\,546\ \text{páginas}
$$

$$
\text{size(Relación)} = 54\,546 \times 4\ \text{KB} \approx 213.1\ \text{MB}
$$

---

### Resumen de volumen estimado

| Tabla | $F_r$ (reg/pág) | $T_r$ estimado | $B_r$ (páginas) | Tamaño aprox |
|---|---|---|---|---|
| `users` | 19 | 1,000,000 | 52,632 | 2.10 GB |
| `tokens` | 29 | 10,000,000 | 344,828 | 1.315 GB |
| `activation_tokens` | 33 | 1,500,000 | 45,455 | 177.6 MB |
| `two_factor_auth_tokens` | 33 | 1,800,000 | 54,546 | 213.1 MB |
| **Total** | — | — | — | **≈ 3.8 GB** |

> Las estimaciones de $T_r$ para `activation_tokens` y `two_factor_auth_tokens` asumen que se aplica una política de limpieza periódica (purga de tokens expirados), ya que de lo contrario el volumen acumulado crecería indefinidamente junto con el de `tokens`.
