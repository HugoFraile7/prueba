# Memoria Práctica Cassandra
## 1. Diseño de las Tablas
-Diseña una base de datos Cassandra para dar servicio a las lecturas y escrituras anteriores. Argumenta tus decisiones de diseño.

- En este apartado se va a justificar el diseño de una base de datos optimizada para un sistema de rankings de mazmorras, justificando las decisiones tomadas en la modelización de las tablas. Como es sabido tenemos que diseñar nuestras tablas para que estas puedan responder correctamente a todas las consultas de lectura y escritura. La clave de particion se establecera para que se puedan realizar las lecturas correctas y se buscará que los datos esten repartidos, la clave de clústering por su parte buscará proporcionarnos una manera de ordenar los datos según las consultas de lectura deseadas.
### 2.1. Tabla `hall_of_fame`
```sql
CREATE TABLE hall_of_fame (
    country text,
    idD int,
    time_minutes int,
    date timestamp,
    email text,
    dungeon_name text,
    userName text,
    PRIMARY KEY ((country, idD), time_minutes, date, email)
) WITH CLUSTERING ORDER BY (time_minutes ASC, date ASC, email ASC);
```
#### Justificación:
- **Clave primaria:** `(country, idD)` agrupa los datos por país y mazmorra, permitiendo particionar la tabla de forma eficiente, ya que en el enunciado se específica que se desea que los jugadores puedan consultar el top 5 por pais para cada mazmorra
- **Clustering Order:** `time_minutes ASC, date ASC, email ASC` permite ordenar los tiempos de menor a mayor, facilitando la consulta de los mejores registros, como sabemos en el enunciado especifica que se quiere consultar el top 5 de jugadores más rapidos para una mazmorra determinada, se incluye la fecha con el objetivo de dar prioridad de orden a un jugador que hizo ese tiempo antes.
- **Uso principal:** Obtener los mejores tiempos de cada país y mazmorra, optimizando consultas como `SELECT * FROM hall_of_fame WHERE country = 'ES' AND idD = 1 LIMIT 5;`.

### 2.2. Tabla `user_statistics`
```sql
CREATE TABLE user_statistics (
    email text,
    idD int,
    time_minutes int,
    date timestamp,
    PRIMARY KEY ((email, idD), time_minutes, date)
) WITH CLUSTERING ORDER BY (time_minutes ASC, date ASC);
```
#### Justificación:
- **Clave primaria:** `(email, idD)` particiona los datos por usuario y mazmorra. 
- **Clustering Order:** `time_minutes ASC, date ASC` permite consultar los tiempos ordenados de menor a mayor.
- **Uso principal:** Obtener el historial de tiempos de un usuario en cada mazmorra, optimizando consultas como `SELECT * FROM user_statistics WHERE email = 'usuario@example.com' AND idD = 1;`.

### 2.3. Tabla `top_horde`
```sql
CREATE TABLE top_horde (
    country text,
    idE int,
    user_name text,
    email text,
    n_killed counter,
    PRIMARY KEY ((country, idE), email, user_name)
);
```
#### Justificación:
- **Clave primaria:** `(country, idE)` permite agrupar los datos por país y evento.
- **Uso de `counter`:** Se utiliza para actualizar la cantidad de monstruos eliminados de manera eficiente.
- **Ordenación:** `email, user_name` garantiza que los datos se puedan recorrer sin colisiones.
- **Posible mejora:** Agregar `n_killed DESC` en el `CLUSTERING ORDER` para facilitar la consulta de los jugadores con más eliminaciones.
- **Uso principal:** Consultar el ranking de jugadores que han matado más monstruos en una horda.
