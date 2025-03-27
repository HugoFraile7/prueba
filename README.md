# Memoria Práctica Cassandra
## 1. Diseño de las Tablas
-Diseña una base de datos Cassandra para dar servicio a las lecturas y escrituras anteriores. Argumenta tus decisiones de diseño.

- En este apartado se va a justificar el diseño de una base de datos optimizada para un sistema de rankings de mazmorras, justificando las decisiones tomadas en la modelización de las tablas. Como es sabido tenemos que diseñar nuestras tablas para que estas puedan responder correctamente a todas las consultas de lectura y escritura. La clave de particion se establecera para que se puedan realizar las lecturas correctas y se buscará que los datos esten repartidos, la clave de clústering por su parte buscará proporcionarnos una manera de ordenar los datos según las consultas de lectura deseadas.
- 
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
- **Clave primaria:** `(country, idD)` agrupa los datos por país y mazmorra, permitiendo particionar la tabla de forma eficiente, ya que en el enunciado se específica que se desea que los jugadores puedan consultar el top 5 por pais para cada mazmorra.
- **Clustering Order:** `time_minutes ASC, date ASC, email ASC` permite ordenar los tiempos de menor a mayor, facilitando la consulta de los mejores registros, como sabemos en el enunciado especifica que se quiere consultar el top 5 de jugadores más rapidos para una mazmorra determinada, se incluye date con el objetivo de dar prioridad de orden a un jugador que hizo ese tiempo antes. Se incluye y email por que es preciso garantizar que la primary key sea única.
## LECTURA:

-Query genérica: SELECT idD, dungeon_name, email, userName, time_minutes, date  FROM hall_of_fame WHERE country = <country_name> AND idD=<dungeon_id> LIMIT 5;
-Se debe iterar entre las distintas dungeons (idD) en un lenguaje externo como Python.Esto es necesario porque Cassandra no permite bucles dentro de sus consultas. Se puede utilizar un bucle for en Python para iterar sobre un SET de python de todos los dungeon_id guardados en el csv. Cada iteración ejecuta una consulta específica para cada dungeon en Cassandra y se produce cada vez que un usuario completa una mazmorra determinada.

-Query de ejemplo:
SELECT idD, dungeon_name, email, userName, time_minutes, date  FROM hall_of_fame WHERE country = 'ko_KR' AND idd=3 LIMIT 5;

## ESCRITURA:
-Query genérica: INSERT INTO hall_of_fame (country, idD, dungeon_name, email, time_minutes, date)
VALUES (<country_name>,<idD>,<dungeon_name>,<email>,<UserName>,<time_minutes>,toTimestamp(now()));

En esta query se incluye el campo 'country' como parte de la clave. Es necesario para permitir la ejecución eficiente de la consulta de lectura, en el enunciado se especifica que las consultas son específicas por países. Esto optimiza el rendimiento y cumple con los requisitos del problema.

-Query ejemplo: INSERT INTO hall_of_fame (country, idD, dungeon_name, email, userName, time_minutes, date)
VALUES ('ja_JP', 0, 'Burghap, Prison of the Jealous Hippies', 'aabe@example.net', 'minorusuzuki', 20, toTimestamp(now());

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
- **Clave primaria:** `(email, idD)` particiona los datos por usuario y mazmorra, permitiendo particionar la tabla de manera eficiente ya que se desea, que un jugador especifico pueda consultar los tiempos que ha tardado en completar una mazmorra determinada. 
- **Clustering Order:** `time_minutes ASC, date ASC` permite consultar los tiempos ordenados de menor a mayor, ya que se desea, que un jugador pueda consultar los tiempos que ha tardado en consultar una mazmorra determinada en orden descendente, volvemos a incluir la fecha para que se pueda establecer un orden de prerioridad por aquellos registros que se han conseguido antes.


## Lectura
-Query genérica: SELECT time_minutes, date FROM user_statistics WHERE email = <email> AND idD = <dungeon_id>;
-Query ejemplo: SELECT time_minutes, date FROM user_statistics WHERE email = 'aabe@example.net' AND idD = 0;

## Escritura:
Cada vez que un usuario ejecute una mazmorra se realizará el siguiente INSERT:

-Query genérica:INSERT INTO user_statistics (idD, email, time_minutes, date) VALUES (<dungeon_id>, <email>, <tiempo en minutos>, toTimestamp(now()));

-Query ejemplo:INSERT INTO user_statistics (idD, email, time_minutes, date) VALUES (0, 'aabe@example.net', 20, toTimestamp(now()));

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
- **Clave primaria:** `(country, idE)` permite agrupar los datos por país y evento, ya que como se especifica en el enunciado se quiere saber para una horda determinada (idE) que jugador ha matado mas monstros.
- **Uso de `counter`:** Se utiliza para actualizar la cantidad de monstruos eliminados de manera eficiente, esta variable counter es necesaria ya que necesitamos contar los monstruos que ha matado cada jugador además de ser necesaria para las consultas de lectura y escritura. Hay que resaltar que al ser una variable tipo counter no podemos usarla como clave de clustering.
- **Ordenación:** `email, user_name` garantiza que los datos se puedan recorrer sin colisiones.









