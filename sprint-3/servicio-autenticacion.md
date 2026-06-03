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

### Tabla de usuarios 

| Campo             | Tamaño aprox |
| ----------------- | ------------ |
| user_id           | 16 B         |
| name              | 30 B         |
| lastname          | 30 B         |
| password_hash     | 60 B         |
| created_at        | 8 B          |
| is_active         | 1 B          |
| email             | 50 B         |

En este caso ninguno de los datos puede ser nullos asi que no tenemos mapa de bits. En la tabla definimos aproximadamente el tamaño de cada uno de los tipos de datos
Campos de longitud variable: name, lastname, password_hash, email

$$
L = 4 \times \text{\ N campos de longitud variable} + \sum \text{size(campos de longitud fija)} + \text{size(mapa de bits)} + \sum \text{tamaño estimado de cada campo de longitud variable} 
$$

$$
L=4 \times 4 +(16+8+1)+ (30+30+60+50) 
$$

$$
L = 211\  \text{bytes}
$$

Ahora calcula re el numero de regsitros por pagina, asumiendo que el tamaño de la pagina es de $4 KB$ es decir $4096\ \text{bytes}$, y utilizando la siguente formula

$$
P = 1 + 1 + 4F_r + F_r L
$$
$$
4096 = 1 + 1 + 4F_r + 211 F_r
$$
$$
4094 = 215F_r
$$
$$
F_r = \frac{24094}{215} = 19.04 \approx 19\ \text{registros por paginas}
$$

Ahora calcularemos el numero de paginas que ocupa la relacion en el disco $B_r$, usando una estimacion de cuantas tuplas tendra la tabla en un año $T_r$. Usando la siguiente relacion:

$$
B_r = \frac{T_r}{F_r} 
$$

estimando que en un año habrán 1 millon de usuarios registrados en un año. 

$$
B_r = \frac{1 000 000}{19} = 52 631.578 \approx 52 632 \text{ paginas}
$$

Finalmente para saber cuanto espacio ocupa la relacion dentro del disco.

$$
size(Relacion) = B_r \times P = 52 632 \times 4KB = 2105360KB
$$

Asi que la relacion estara ocupando aproximadamente $2.10536GB$ dentro del disco

### Tabla de tokens

| Campo           | Tamaño |
| --------------- | ------ |
| token_id        | 16 B   |
| token_hash      | 60 B   |
| expiration_date | 8 B    |
| expired_at      | 8 B    |
| user_id         | 16 B   |
| token_type      | 20 B   |

Esta tabla tampoco adimitira solo admitira `expirated_at` como nulo. 
campos de longitud variable: token_hash, token_type

$$
L=4 \times 2 +(16+8+8+16)+(60+20)+1
$$

$$
L = 137B
$$

Calculare el $F_r$ usando la siguiente formula y asumiendo que $P=4096\ bytes$

$$
P = 1+1+4F_r +F_r L
$$

$$
4096 = 1+1+4F_r +137F_r
$$

$$
4094 = 141F_r
$$

$$
F_r = \frac{4094}{141} =29.03 \approx 29\ \text{registros por pagina}
$$

Ahora calculare el $B_r$  que es el numero de paginas que ocupa la relacion en disco, teniendo en cuenta que $T_r$ es el numero de tuplas que contiene la relacion. Estimando que dentro de un año se tendrán un 10 millones de tokens.

$$
B_r = \frac{T_r}{F_r} = \frac{10 000 000}{29} = 344 827.58 \approx 344 828\ \text{paginas}
$$

ahora para encontrar el tamaño total de la relacion en un año multiplico el peso de cada registro por el numero de registros por pagina y el numero total de paginas

$$
size(Relacion) = P  \times B_r
$$
p
teniendo en cuenta que $4096 = 4\times2^{10} = 4KB$

$$
size(Relacion) = 4KB \times 344 828 = 1379312KB \approx 1.315 GB
$$
