# Servicio de atenticacion

Este es el entregable de bases de datos para el microservicio de autenticacion.

## Modelo entidad relacion (nivel logico)

![diagrama modelo relacional](./relational-logico.jpg)

## Modelo de datos relacional (nivel fisico)

![diagrama modelo relacional](./relational.jpg)

Para este microseravicio. Sera solamente necesaria dos tablas la de usuarios y la de tokens. La tabla de usarios se sincronizara con cada uno de los microservicios que tambien tengan la entidad usuarios en su dominio mediante una cola de mensajeria en azure.

## Scritp SQL que inicializa la base de datos
Como esta base la base de datos que sera usada para el programa sera la misma para todos los microservicios, se separan solo por `schemas`

```{sql}
DROP TABLE IF EXISTS auth.tokens;

DROP TABLE IF EXISTS auth.users;

DROP TYPE IF EXISTS auth.token_type;

CREATE SCHEMA IF NOT EXISTS auth;

CREATE TABLE IF NOT EXISTS auth.users (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid (),
    name VARCHAR(50) NOT NULL,
    lastname VARCHAR(50) NOT NULL,
    password_hash TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    is_active BOOLEAN NOT NULL DEFAULT FALSE,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS auth.tokens (
    token_id UUID PRIMARY KEY DEFAULT gen_random_uuid (),
    token_hash TEXT NOT NULL,
    expiration_date TIMESTAMP NOT NULL,
    expired_at TIMESTAMP DEFAULT NULL,
    user_id UUID NOT NULL REFERENCES auth.users (user_id),
    token_type VARCHAR(50) NOT NULL
)
```

## Consultas indentificadas segun el Sprint 1 y 2

1. Consultar un token por su hash (verificar autenticacion)

`SELECT * FROM auth.tokens WHERE token_hash = :hash`

2. Revocar todos los tokens de un usuario (logout)

`UPDATE auth.tokens SET expirated_at = NOW() WHERE user_id = :userId`


## Volumen de datos aproximado por tablas 

### Tabla de usuarios 
En este caso ninguno de los datos puede ser nullos asi que no tenemos mapa de bits. En la tabla definimos aproximadamente el tamaño de cada uno de los tipos de datos
Campos de longitud variable: name, lastname, password_hash, email


$$
L = 4 \times \text{\# campos de longitud variable}
+ \sum \text{size(campos de longitud fija)}
+ \text{size(mapa de bits)}
+ \sum \text{tamaño estimado de cada campo de longitud variable}
\tag{1}
$$

$$
L=4 \times 4 +(16+8+1)+ (30+30+60+50) 
$$

$$
L = 211 B
$$


| Campo             | Tamaño aprox |
| ----------------- | ------------ |
| user_id           | 16 B         |
| name              | 30 B         |
| lastname          | 30 B         |
| password_hash     | 60 B         |
| created_at        | 8 B          |
| is_active         | 1 B          |
| email             | 50 B         |


tamaño de tabla por tupla = `211 Bytes`

En caso de que nuestro programa llege a tener 1 millon de usuarios el tamaño de la base de la tabla seria de `211.000.000 Bytes` que serian `211 MB`

### Tabla de tokens
Esta tabla tampoco adimitira solo admitira `expirated_at` como nulo. 
campos de longitud variable: token_hash, token_type

$$
L=4 \times 2 +(16+8+8+16)+(60+20)+1
$$

$$
L = 137B
$$

| Campo           | Tamaño |
| --------------- | ------ |
| token_id        | 16 B   |
| token_hash      | 60 B   |
| expiration_date | 8 B    |
| expired_at      | 8 B    |
| user_id         | 16 B   |
| token_type      | 20 B   |

tamaño de cada tupla `137 Bytes`

en el caso de que hayan unos 10 millones de registros con tokens (se experan que haya hasta 10 veces mas tokens que usaurios) el tamño de la tabla sera `1370MB`
