# Memoria Práctica Cassandra
## 1. Diseño de las Tablas
-Diseña una base de datos Cassandra para dar servicio a las lecturas y escrituras anteriores. Argumenta tus decisiones de diseño.

- En este apartado se va a justificar el diseño de una base de datos optimizada para un sistema de rankings de mazmorras, justificando las decisiones tomadas en la modelización de las tablas. Como es sabido tenemos que diseñar nuestras tablas para que estas puedan responder correctamente a todas las consultas de lectura y escritura. La clave de particion se establecera para que se puedan realizar las lecturas correctas y se buscará que los datos esten repartidos, la clave de clústering por su parte buscará proporcionarnos una manera de ordenar los datos según las consultas de lectura deseadas.

### 1.1. Tabla `hall_of_fame`
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
## Lectura

- **Query genérica:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  SELECT idD, dungeon_name, email, userName, time_minutes, date  
  FROM hall_of_fame 
  WHERE country = <country_name> AND idD = <dungeon_id> 
  LIMIT 5;
  ```
- **Nota:**  
  Se debe iterar entre las distintas mazmorras (`idD`) en un lenguaje externo como Python. Esto es necesario porque Cassandra no permite bucles dentro de sus consultas. Se puede utilizar un bucle `for` en Python para iterar sobre un `SET` de Python que contenga todos los `dungeon_id` guardados en el CSV. Cada iteración ejecuta una consulta específica para cada mazmorra en Cassandra y se produce cada vez que un usuario completa una mazmorra determinada.

- **Query ejemplo:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  SELECT idD, dungeon_name, email, userName, time_minutes, date  
  FROM hall_of_fame 
  WHERE country = 'ko_KR' AND idD = 3 
  LIMIT 5;
  ```

## Escritura

- **Query genérica:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  INSERT INTO hall_of_fame (country, idD, dungeon_name, email, userName, time_minutes, date)
  VALUES (<country_name>, <idD>, <dungeon_name>, <email>, <userName>, <time_minutes>, toTimestamp(now()));
  ```
- **Nota:**  
  En esta query se incluye el campo `country` como parte de la clave. Es necesario para permitir la ejecución eficiente de la consulta de lectura, ya que en el enunciado se especifica que las consultas son específicas por países. Esto optimiza el rendimiento y cumple con los requisitos del problema.

- **Query ejemplo:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  INSERT INTO hall_of_fame (country, idD, dungeon_name, email, userName, time_minutes, date)
  VALUES ('ja_JP', 0, 'Burghap, Prison of the Jealous Hippies', 'aabe@example.net', 'minorusuzuki', 20, toTimestamp(now()));
  ```


### 1.2. Tabla `user_statistics`
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

- **Query genérica:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  SELECT time_minutes, date FROM user_statistics WHERE email = <email> AND idD = <dungeon_id>;
  ```
- **Query ejemplo:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  SELECT time_minutes, date FROM user_statistics WHERE email = 'aabe@example.net' AND idD = 0;
  ```

## Escritura

Cada vez que un usuario ejecute una mazmorra se realizará el siguiente `INSERT`:

- **Query genérica:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  INSERT INTO user_statistics (idD, email, time_minutes, date) 
  VALUES (<dungeon_id>, <email>, <tiempo en minutos>, toTimestamp(now()));
  ```
- **Query ejemplo:**  
  ```sql
  CONSISTENCY LOCAL_QUORUM;
  INSERT INTO user_statistics (idD, email, time_minutes, date) 
  VALUES (0, 'aabe@example.net', 20, toTimestamp(now()));
### 1.3. Tabla `top_horde`
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


## Lectura

- **Query genérica:**  
  ```sql
  CONSISTENCY ONE;
  SELECT userName, email, n_kills
  FROM top_horde
  WHERE country = <pais> AND event_id = <evento id>;
  ```
- **Query ejemplo:**  
  ```sql
  CONSISTENCY ONE;
  SELECT user_name, email, n_killed
  FROM top_horde
  WHERE country = 'de_DE' AND idE = 0;
  ```

En este caso, no se puede ordenar por `n_killed`, ya que al ser `counter`, no puede ser incluida en la clave de clustering. Esto provoca que el ordenamiento y el filtrado por el valor `k` introducido (número de personas a incluir en el top) debe hacerse con herramientas externas como Python. Por ejemplo, se podría usar un `DataFrame` de `pandas` para introducir el resultado y hacer las correspondientes agrupaciones, ordenamientos y filtrado por número de personas a incluir en el top.

## Escritura

- **Query genérica:**  
  ```sql
  CONSISTENCY ONE;
  UPDATE top_horde
  SET n_killed = n_killed + 1
  WHERE country = <country> AND eId = <evento id> AND email = <email> AND userName = <user name>;
  ```
- **Query ejemplo:**  
  ```sql
  CONSISTENCY ONE;
  UPDATE top_horde
  SET n_killed = n_killed + 1
  WHERE country = 'de_DE' AND idE = 0 AND email = 'qloeffler@example.org' AND user_name = 'abdul92';
  ```

Como `n_killed` es de tipo `counter`, se puede sumar uno cada vez que se produce la kill.
# 2.Exportar datos a csv.

-Crea las consultas .sql necesarias para exportar los datos de la base de datos relacional a ficheros .csv. Los ficheros deberán tener un formato acorde al diseño del punto 1.

# HALL OF FAME

Lo primero que tenemos que hacer es crear la tabla Hall of Fame, que usaremos para devolver los 5 mejores jugadores de cada mazmorra por pais.
```sql
SELECT WebUser.Country AS country, Dungeon.idD, CompletedDungeons.time AS time_minutes, CompletedDungeons.date, WebUser.email, Dungeon.name AS dungeon_name, WebUser.userName
FROM WebUser
INNER JOIN CompletedDungeons ON CompletedDungeons.email = WebUser.email
INNER JOIN Dungeon ON CompletedDungeons.idD = Dungeon.idD;
```

Seleccionamos el country, userName y email de la tabla WebUser; el idD de la tabla Dungeon; y el time y date de la tabla CompletedDungeons. Despues unimos WebUser con CompletedDungeons usando el campo email, y une CompletedDungeons con Dungeon usando el id de la mazmorra.


# ESTADISTICAS DE USUARIO

Despues crearemos la tabla que almacene las estadisticas de usuario para consultar los tiempos que han tardado en completar una mazmorra ordenados de menor a mayor.
```sql
SELECT WebUser.email, CompletedDungeons.idD, CompletedDungeons.time AS time_minutes, CompletedDungeons.date
FROM WebUser
INNER JOIN CompletedDungeons ON CompletedDungeons.email = WebUser.email;
```

Seleccionamos el email de la tabla WebUser, y el idD de la mazmorra, el tiempo en completarla, y la fecha cuando se completo. Despues unimos WebUser con CompletedDungeons usando el campo email.


 # HORDAS

Finalmente crearemos la tabla para el evento de Hordas, para poder consultar los usuarios que mas enemigos han matado en una dungeon en concreto.
```sql
SELECT WebUser.Country AS country, Kills.idE, WebUser.userName AS user_name, WebUser.email, COUNT(Kills.idM) AS n_killed
FROM WebUser
INNER JOIN Kills ON WebUser.email=Kills.email
GROUP BY WebUser.Country, Kills.idE, WebUser.userName, WebUser.email;
```


Seleccinamos el Country, el userName y el email de la tabla WebUser; el idE del evento y el idM de los monstruos de la tabal Kills. Este caso en particular, pues procesamos el numero de monstruos asesinados por el usuario y lo devolvemos como un int en la variable n_killed.

# 3. Preparación de cluster local.

-Prepara un cluster local de 3 nodos todos en el mismo rack y datacenter.

Creamos un archivo **compose.yml** que construye el cluster local de 3 nodos en el mismo rack y datacenter, para ello creamos primero una red personalizada tipo bridge llamada cassandra donde todos los nodos estaran conectados entre si y podran comunicarse.
Despues definimos los contenedores individuales, son tres y los llamaremos node1, node2, node3. Definimos la version de Cassandra que usan, en nuestro caso la 5 y añadimos una serie de configuraciones para verificar que funcione correctamente. Mapeamos los puertos de los nodos al mismo puerto del host, para conectarnos a todos los nodos desde el mismo puerto y le asignamos volumenes para que haya persistenca de datos. 
Finalmente añadimos la dependencia de nodos para controlar el inicio de los nodos y asegurarnos que los nodos funcionan correctamente antes de que se inicie el siguiente.

# 4. Crear diseño en cassandra y cargar los ficheros
- Haz un fichero .cql que creen tu diseño en Cassandra y cargue los ficheros .csv creados en el paso 2. Se debe utilizar un factor de replicación de 2 y tener en cuenta que se las pruebas se ejecutaran en un cluster local.


# 5. Si es necesario actualizar tablas.
- [OPCIONAL] Si el diseño lo necesita, actualiza la tabla de escrituras para incluir
cualquier modificación que sea necesaria en la información que se le debe
proporcionar al servidor.

-Debido a que el diseño de las tablas ha sido el correcto, no es preciso realizar ninguna modificacion en estas.

# 6. Consultas de lectura, escritura y nivel de consistencia .
-Haz un fichero .cql que realice las consultas de escritura y lectura necesarias.
Incluye el nivel de consistencia de cada consulta, teniendo en cuenta las
características de los diferentes rankings.

## 🔹 Consistencia en el Juego

El sistema usa diferentes niveles de consistencia en Cassandra según la criticidad de los datos y la tolerancia a la latencia.  

Las queries están en el archivo cql. A continuación se hablará de la consistencia de las queries expuestas en el apartado 1.

---

###  Hall of Fame  
**Consistencia: `LOCAL_QUORUM`**  
- Garantiza que la lectura refleje los datos más recientes confirmados.  
- Muestra el **TOP 5 de jugadores** por país y mazmorra.  
- La precisión es crucial, ya que los jugadores confían en estos rankings.  
- Sin embargo, en el **campamento** (menú, no gameplay), se tolera un ligero retraso.  
- Se usa la misma consistencia en la **escritura** para asegurar coherencia en la lectura.  

---

###  User Statistics  
**Consistencia: `LOCAL_QUORUM`**  
- Asegura que la lectura refleje los datos más recientes confirmados.  
- Similar a *Hall of Fame*, prioriza **consistencia sobre velocidad**.  
- Un retraso en la escritura no afecta al gameplay, según el enunciado.  

---

###  Top Horde  
**Consistencia: `ONE`**  
- Permite leer desde una **única réplica**, reduciendo latencia y mejorando fluidez.  
- Este leaderboard se consulta **durante el gameplay activo de las Hordas**, donde la latencia es crítica.  
- El enunciado permite cierta inconsistencia temporal ("bailar un poco"), pero evita divergencias extremas.  













