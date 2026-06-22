# Clase 9 - Consistencia de caches

## [Apuntes de Emma](https://drive.google.com/file/d/1TofDiVxODoqtW5dvQ01H_LoE2hddhfj3/view)

## Objetivo

**Los caches se usan para aumentar la capacidad del sistema y proteger la base de datos**, no solamente para que una consulta individual "se sienta" mas rapida.

En una aplicacion web clasica, primero puede existir una unica maquina con un servidor HTTP, por ejemplo Apache o Nginx, conectada a una base de datos relacional como MySQL. El primer paso para escalar suele ser separar la capa web y poner varias instancias stateless detras de un load balancer. Esa capa escala facil porque no guarda estado: si se cae una instancia, se reemplaza por otra.

La base de datos relacional no escala de la misma manera. Tiene estado persistente, discos, indices, locks y restricciones de consistencia. Por eso, cuando la capa web empieza a escalar, el cuello de botella suele pasar a ser la base de datos. Una forma comun de aliviarla es agregar caches.

## Modelo basico con teoria de colas

Un servidor puede pensarse como un sistema al que entran requests y del que salen responses.

Se definen tres magnitudes principales:

- `lambda`: tasa de arribos, es decir, cuantos requests entran por unidad de tiempo.
- `delta`: tasa de salidas, es decir, cuantos responses salen por unidad de tiempo.
- `N(t)`: cantidad de clientes o requests dentro del sistema en un instante.

Si entran requests mas rapido de lo que salen, el sistema empieza a acumular trabajo pendiente. La cola crece y el tiempo de espera aumenta.

```text
dN(t) / dt = lambda(t) - delta(t)
```

Cuando el sistema esta estable, la tasa de entrada y la tasa de salida se equilibran:

```text
lambda = delta
dN(t) / dt = 0
N(t) = constante
```

La estabilidad no significa que no haya requests, sino que el sistema no acumula deuda indefinidamente.

## Ley de Little

La Ley de Little relaciona la cantidad promedio de clientes en el sistema, la tasa promedio de arribos y el tiempo promedio dentro del sistema:

```text
L = lambda * W
```

Donde:

- `L`: cantidad promedio de clientes dentro del sistema.
- `lambda`: tasa promedio de arribos.
- `W`: tiempo promedio dentro del sistema.

Tambien puede despejarse:

```text
W = L / lambda
lambda = L / W
```

En sistemas web, esta misma idea puede interpretarse asi:

```text
throughput = paralelismo / tiempo promedio por request
```

Entonces, para aumentar throughput hay dos perillas principales:

1. Aumentar el paralelismo.
2. Reducir el tiempo promedio de cada request.

Agregar mas servidores aumenta el paralelismo. Agregar cache reduce el tiempo promedio de muchos requests.

## Ejemplo: cafe

Si entran 45 personas por hora a un cafe y cada persona permanece 25 minutos, la cantidad promedio de personas dentro del local es:

```text
L = lambda * W
L = 45 personas/hora * (25/60) horas
L = 18,75 personas
```

El local deberia dimensionarse para alrededor de 19 personas promedio, o mas si se quiere margen.

## Ejemplo: consultorio medico

Si hay 6 pacientes dentro del sistema y llegan 10 pacientes por hora:

```text
W = L / lambda
W = 6 / 10 horas
W = 0,6 horas
W = 36 minutos
```

El tiempo promedio dentro del sistema es de 36 minutos.

El sistema puede tener distintas estructuras internas. Por ejemplo, una unica cola y un medico, o varios medicos atendiendo en paralelo. La Ley de Little permite razonar el sistema desde afuera sin importar demasiado su estructura interna, siempre que se consideren las mismas fronteras del sistema.

## Aplicacion a una base de datos

Una base de datos puede pensarse como un sistema con un maximo de conexiones o threads procesando requests en paralelo.

```text
throughput (lambda) = paralelismo (L) / tiempo_promedio_por_query (W)
```

El paralelismo puede venir de parametros como `max_connections`, cantidad de workers, cantidad de threads o cantidad de instancias. El tiempo promedio por query depende de la complejidad de las consultas, los indices, el estado de la base, la red y otros factores.

Para aumentar el throughput se puede:

- aumentar el paralelismo;
- reducir el tiempo promedio de las queries;
- hacer ambas cosas.

Aumentar el paralelismo no siempre alcanza. En bases de datos relacionales, demasiadas conexiones pueden empeorar el rendimiento por locks, contencion interna, uso de memoria y overhead de scheduling.

Reducir el tiempo promedio por request suele ser una estrategia mas efectiva cuando hay muchas lecturas repetidas. Ahi aparecen los caches.

## Cache para reducir el tiempo promedio

Un cache como Memcached o Redis puede responder un `GET(k)` en alrededor de 1 ms, mientras que una base de datos relacional puede tardar bastante mas, por ejemplo 10 ms o mas segun la query.

Si el cache tiene un hit ratio del 98%, el tiempo promedio queda:

```text
W = 0,98 * 1 ms + 0,02 * 10 ms
W = 1,18 ms
```

Sin cache, si todas las consultas fueran a la base y tardaran 10 ms, el tiempo promedio seria 10 ms.

Con cache, el tiempo promedio baja a 1,18 ms. Como el throughput es aproximadamente:

```text
throughput = paralelismo / W
```

bajar `W` aumenta la capacidad maxima del sistema. En este ejemplo, el sistema puede soportar alrededor de 8 veces mas throughput para ese tipo de requests.

El cache tambien protege la base de datos: si el 98% de las lecturas se resuelven en cache, solo el 2% llega a la base.

## Tipos de cache

### Look-aside cache

En un cache look-aside, la aplicacion conoce tanto el cache como la base de datos.

El flujo de lectura es:

```text
read(k):
    v = cache.get(k)

    if v is null:
        v = db.fetch(k)
        cache.set(k, v)

    return v
```

La aplicacion primero mira el cache. Si hay hit, responde desde el cache. Si hay miss, consulta la base de datos, guarda el resultado en cache y responde.

Este es el modelo usado en muchos sistemas reales porque es simple de implementar con herramientas como Memcached o Redis.

### Look-through cache

En un cache look-through, la aplicacion solo habla con el cache. El cache sabe como ir a buscar el dato a la base de datos si no lo tiene.

```text
app -> cache -> db
```

El cache abstrae el acceso a la base. Esto puede esconder parte de la complejidad de consistencia, pero requiere que el cache tenga logica de negocio o sepa como traducir claves a queries. Por eso es menos comun en sistemas generales.

## Arquitectura de Facebook con Memcache

Facebook escala una aplicacion web con varias regiones, frontends stateless, clusters de Memcache y bases de datos MySQL shardeadas y replicadas.

La estructura general es:

```text
usuarios
   |
load balancer / routing
   |
frontends stateless
   |
clusters de Memcache
   |
MySQL shardeado y replicado
```

Los frontends son stateless. Los caches estan distribuidos en muchos nodos. La base de datos esta particionada y replicada entre regiones.

En una region, los nodos de Memcache no son necesariamente replicas exactas entre si. Pueden ser shards: distintas claves se asignan a distintos nodos.

Una forma simple de asignar claves seria:

```text
shard = hash(k) % cantidad_de_nodos
```

En sistemas reales se usa consistent hashing para reducir el movimiento de claves cuando se agregan o quitan nodos.

## Lecturas con look-aside cache

El algoritmo basico de lectura es:

```text
read(k):
    v = memcache.get(k)

    if v is null:
        v = db.fetch(k)
        memcache.set(k, v)

    return v
```

Si hay hit, la base de datos no participa.

Si hay miss, se consulta la base y luego se puebla el cache.

## Escrituras e invalidacion

En una escritura, no conviene actualizar directamente el valor en el cache. La estrategia usada es invalidar el cache:

```text
write(k, v):
    db.write(k, v)
    memcache.delete(k)
```

La invalidacion hace que la proxima lectura no encuentre el valor viejo en cache y tenga que ir a la base de datos a buscar el valor actualizado.

Esto introduce consistencia eventual: puede haber una ventana entre el `write` en la base y el `delete` en el cache donde alguien lea el valor viejo. Pero esa ventana deberia ser corta.

El TTL puede servir como red de seguridad, pero no deberia ser el mecanismo principal de consistencia. Si se espera media hora a que expire una clave, el sistema puede devolver datos viejos durante media hora. Si se pone un TTL demasiado corto, el cache pierde efectividad porque baja el hit ratio.

## Por que invalidar y no hacer set

Una escritura podria parecer mas directa asi:

```text
write(k, v):
    db.write(k, v)
    memcache.set(k, v)
```

Pero eso puede generar race conditions.

### Race condition 1: escrituras concurrentes y cache pisado

Supongamos dos clientes:

```text
C1:
    db.write(x = 1)
    memcache.set(x = 1)

C2:
    db.write(x = 2)
    memcache.set(x = 2)
```

Si las operaciones se intercalan asi:

```text
C1: db.write(x = 1)
C2: db.write(x = 2)
C2: memcache.set(x = 2)
C1: memcache.set(x = 1)
```

El resultado final queda:

```text
DB:       x = 2
Memcache: x = 1
```

El cache queda con un valor viejo.

Con invalidacion, en cambio:

```text
C1: db.write(x = 1)
C2: db.write(x = 2)
C2: memcache.delete(x)
C1: memcache.delete(x)
```

El resultado es que la clave se borra del cache. La proxima lectura va a la base y trae `x = 2`. El `delete` es idempotente: borrar dos veces no rompe nada.

## Sharding de Memcache

Memcache guarda pares clave-valor en memoria. Como una sola maquina tiene memoria limitada, se distribuyen claves entre varias maquinas.

```text
frontend
   |
   +--> memcache shard 0
   +--> memcache shard 1
   +--> memcache shard 2
```

La clave se mapea de forma deterministica a un shard. En una version simple:

```text
shard = md5(k) % N
```

En sistemas grandes se prefiere consistent hashing para que agregar o quitar nodos no obligue a remapear casi todas las claves.

En Facebook, cada request de frontend puede necesitar muchas claves: usuario, timeline, posts, fotos, contadores, relaciones, etc. Por eso un mismo frontend puede terminar pegandole a muchos shards de Memcache por cada request.

## Replicacion entre regiones

Facebook usa varias regiones. Las escrituras van a la region primaria de la base de datos. Otras regiones pueden tener replicas secundarias para lecturas.

```text
Region primaria:
    frontend -> memcache -> MySQL primary

Region secundaria:
    frontend -> memcache -> MySQL secondary
```

Si un frontend de una region secundaria recibe una escritura, no puede escribir en su replica local si esa replica es read-only. Debe mandar la escritura al primary.

Luego la base primaria replica el cambio hacia las replicas secundarias.

El problema es que los caches de otras regiones pueden quedar con valores viejos. Invalidar solo el cache local no alcanza.

## Invalidacion cross-region

Cuando se escribe en la region primaria:

```text
1. Se escribe en MySQL primary.
2. Se invalida el Memcache local.
3. El cambio se replica hacia otras regiones.
4. Los caches remotos tambien deben invalidarse.
```

No conviene que cada frontend conozca todos los caches de todas las regiones. Eso acoplaria demasiado las regiones.

La solucion de Facebook usa un sistema llamado `mcsqueal`/`mcsqueal` en el paper y una idea parecida a Change Data Capture: observar el log de la base de datos y producir invalidaciones.

La base de datos tiene un write-ahead log. Un proceso observa ese log, detecta updates y genera deletes en los caches correspondientes.

La idea general es:

```text
MySQL WAL / binlog
      |
proceso de CDC
      |
delete(k) en caches afectados
```

Herramientas modernas como Debezium implementan una idea similar: leer cambios desde el log de la base y publicarlos, por ejemplo, en Kafka.

## Embedding de claves de cache en queries

Una forma de ayudar al sistema de invalidacion es incluir en la query informacion sobre que claves de cache deben invalidarse.

Ejemplo conceptual:

```sql
UPDATE users
SET name = 'foo'
WHERE id = 123
-- keys: users:123
```

El proceso que lee el log puede extraer la clave `users:123` y saber que cache invalidar sin tener que entender profundamente la semantica de la query.

## Invalidacion local y por log

Puede haber dos caminos de invalidacion:

1. El frontend invalida el cache local inmediatamente despues de escribir.
2. El sistema que lee el log de la base invalida caches al observar el cambio.

El primer camino reduce la ventana de inconsistencia local. El segundo sirve como respaldo y permite invalidar caches remotos.

Si el frontend logra escribir en la base pero falla al invalidar el cache local, el lector del log eventualmente vera el cambio y enviara la invalidacion.

## Race condition 2: region secundaria lee antes de replicar

Supongamos dos regiones:

```text
Region primaria:
    DB primary

Region secundaria:
    DB secondary
    Memcache local
```

Un frontend de la region secundaria recibe una escritura. Como la escritura debe ir al primary:

```text
1. El frontend manda write(k, v2) al primary.
2. Invalida su Memcache local.
3. La replicacion hacia la DB secundaria tarda un poco.
```

Ahora otro frontend de la region secundaria quiere leer `k`:

```text
1. Busca k en Memcache local.
2. Hay miss, porque se invalido.
3. Va a la DB secundaria.
4. La DB secundaria todavia no recibio v2.
5. Lee v1.
6. Guarda v1 en Memcache.
```

Resultado:

```text
DB primary:    k = v2
DB secondary:  eventualmente k = v2
Memcache sec:  k = v1
```

El cache de la region secundaria quedo repoblado con un valor viejo.

### Solucion: remote marker o RK

Antes de mandar la escritura al primary, el frontend de la region secundaria marca en su Memcache local que esa clave esta siendo modificada remotamente.

```text
set(RK)
write primary
delete(k)
```

Luego, en la lectura:

```text
read(k):
    v = memcache.get(k)

    if v is miss:
        marker = memcache.get(RK)

        if marker is hit:
            leer desde primary
        else:
            leer desde replica local
```

Mientras exista `RK`, las lecturas no deben repoblar el cache desde la replica local porque esa replica puede estar atrasada. Deben ir al primary o a una fuente suficientemente actualizada.

Cuando la replicacion llega a la region secundaria, el proceso que observa el log borra el marker `RK`.

## Thundering herd

El thundering herd aparece cuando muchos clientes intentan resolver el mismo miss de cache al mismo tiempo.

Ejemplo: una hot key, como un post muy popular o un tweet viral.

Mientras la clave esta en cache, todo funciona bien:

```text
miles de requests -> Memcache hit
```

Pero si la clave se invalida:

```text
miles de requests -> Memcache miss
miles de requests -> DB
```

Todos los frontends ven el miss al mismo tiempo y todos van a la base de datos. La base puede saturarse o caerse.

El cache estaba protegiendo la base; al invalidar una hot key, esa proteccion desaparece de golpe.

## Solucion con leases

Facebook modifica Memcache para que, ante un miss, solo un cliente tenga permiso para ir a la base de datos y repoblar el cache.

El flujo es:

```text
C1: get(k) -> miss + lease
C2: get(k) -> miss + retry
C3: get(k) -> miss + retry
```

El primer cliente recibe un lease. Ese lease es un numero aleatorio, por ejemplo de 64 bits, que Memcache asocia temporalmente a la clave.

```text
tabla interna:
k | value | lease
```

El cliente con lease va a la base de datos:

```text
v = db.read(k)
memcache.set(k, v, lease)
```

Memcache acepta el `set` solo si el lease coincide.

Los demas clientes reciben `retry`, esperan unos milisegundos y vuelven a intentar. Con suerte, para ese momento el primer cliente ya repoblo el cache.

Esto evita que miles de frontends le peguen simultaneamente a la base de datos.

## Race condition 3: lectura vieja repuebla cache despues de una escritura

Otra race condition ocurre entre una lectura con miss y una escritura.

```text
C1:
    get(k) -> miss
    read_db(k) -> v1
    set(k, v1)

C2:
    write_db(k, v2)
    delete(k)
```

Si se intercalan asi:

```text
C1: get(k) -> miss
C1: read_db(k) -> v1
C2: write_db(k, v2)
C2: delete(k)
C1: set(k, v1)
```

El resultado queda:

```text
DB:       k = v2
Memcache: k = v1
```

La lectura vieja repuebla el cache despues de que la escritura ya invalido la clave.

### Leases como optimistic locking

El lease tambien soluciona este caso.

Cuando C1 hace `get(k)` y obtiene `miss + lease`, Memcache registra el lease.

Si C2 hace `delete(k)`, Memcache borra la clave y tambien invalida el lease asociado.

Luego, cuando C1 intenta:

```text
set(k, v1, lease)
```

Memcache verifica el lease. Como el lease ya no coincide o ya no existe, rechaza el `set`.

Eso funciona como un `check-and-set` u optimistic locking:

```text
set(k, v, lease):
    if current_lease == lease:
        guardar valor
    else:
        rechazar
```

El valor viejo no vuelve a entrar al cache.

## TTL y leases

El lease debe tener un TTL corto. Si el cliente que recibio el lease muere antes de repoblar el cache, no puede quedar bloqueando la clave para siempre.

Despues de unos milisegundos o segundos, el lease expira y otro cliente puede intentar repoblar.

## Check-and-set

Operaciones como `check-and-set` permiten actualizar un valor solo si cierta condicion sigue siendo valida.

En este caso, la condicion es que el lease coincida. En otros sistemas puede ser una version, un timestamp, un token o un ETag.

La idea es la misma que optimistic locking:

```text
lei version V
quiero escribir si la version actual sigue siendo V
si cambio, fallo y reintento
```

## Lecciones practicas

Invalidar caches es parte central del diseño. No alcanza con poner TTLs largos y esperar que el sistema se corrija solo.

Usar `delete` en vez de `set` evita race conditions porque el delete es idempotente.

Los caches look-aside son simples, pero trasladan la responsabilidad de consistencia a la aplicacion.

Los sistemas multi-region necesitan invalidacion remota. Una tecnica practica es observar el log de la base de datos y emitir invalidaciones.

Las hot keys pueden causar thundering herd cuando se invalidan. El problema no es solo que haya miss, sino que muchos clientes hagan miss al mismo tiempo.

Los leases permiten que un solo cliente repueble el cache y que los demas esperen.

Los leases tambien evitan que una lectura vieja repueble el cache despues de una escritura concurrente.

El objetivo principal de un cache en sistemas grandes es aumentar throughput y proteger la base de datos. La mejora de latencia es un efecto secundario util, pero no es la unica razon para usarlo.
