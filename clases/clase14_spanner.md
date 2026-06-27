# Clase 14 — Spanner

## [Apuntes de Emma](https://drive.google.com/file/d/1laiLq7uyB_KTtml3HvSKUyiC-xVVtntd/view)

## Idea general

Spanner es una base de datos distribuida de Google que busca ofrecer transacciones ACID en un sistema distribuido globalmente. El objetivo es poder usar una abstracción parecida a una base SQL tradicional, con operaciones del estilo:

```sql
BEGIN;
READ x;
READ y;
x = x + y;
COMMIT;
```

La diferencia es que internamente los datos no están en una única máquina, sino particionados, replicados y distribuidos entre distintos data centers.

Spanner combina varios mecanismos ya vistos en sistemas distribuidos:

- particionado de tablas;
- replicación por grupos de consenso;
- transacciones distribuidas;
- Two-Phase Commit;
- Two-Phase Locking;
- MVCC;
- relojes físicos con incertidumbre acotada mediante TrueTime.

El paper es complejo porque no introduce una sola técnica aislada, sino que combina muchas técnicas para lograr una base de datos distribuida, global y con consistencia fuerte.

## Modelo de datos y particionado

Spanner trabaja con tablas similares a las de SQL. Las tablas tienen columnas y una clave primaria obligatoria.

La clave primaria es importante porque Spanner particiona la tabla en rangos de clave primaria. Cada rango forma un shard. A diferencia de DynamoDB, que usa particionado por hash, Spanner usa rangos ordenados de primary key.

Conceptualmente:

```text
Tabla
+-----+-----+-----+
| PK  | ... | ... |
+-----+-----+-----+
|  1  |     |     |
|  2  |     |     |
| ... |     |     |
+-----+-----+-----+

Rangos de PK:
[0, 1000)
[1000, 5000)
[5000, ...)
```

Cada shard está replicado en un grupo de replicación. Esos grupos pueden estar distribuidos en distintos data centers. En el paper se usa Paxos, más precisamente una variante tipo Multi-Paxos, para mantener sincronizadas las réplicas. Para entender la clase alcanza pensarlo como algo similar a Raft: hay un grupo de réplicas que acuerda un log y mantiene consistente el estado de ese shard.

La arquitectura se parece en parte a DynamoDB: muchos shards, cada uno replicado. La diferencia fuerte aparece en cómo Spanner implementa transacciones más generales.

## Tipos de transacciones

Spanner distingue dos tipos principales de transacciones:

1. **Read-write transactions**: leen y escriben datos.
2. **Read-only transactions**: solo leen datos.

Esta distinción es fundamental porque se implementan de formas muy distintas.

Las transacciones read-write usan mecanismos clásicos de bases de datos distribuidas:

- Two-Phase Commit;
- Two-Phase Locking;
- grupos Paxos para persistir el estado de los participantes.

Las transacciones read-only están mucho más optimizadas:

- no usan locks;
- no usan Two-Phase Commit;
- leen desde réplicas locales;
- usan MVCC;
- usan TrueTime para elegir timestamps consistentes.

La razón de esta optimización es que la enorme mayoría de las transacciones son de solo lectura. Si las lecturas se pueden resolver rápido, sin coordinación global y sin locks, el sistema gana muchísima performance.

## Read-write transactions

Una transacción read-write puede tocar datos ubicados en distintos shards. Cada shard involucrado funciona como un participante de la transacción. Como cada shard está replicado mediante Paxos, cada participante en realidad representa a un grupo de réplicas.

La secuencia general es:

1. El cliente inicia la transacción.
2. Hace lecturas sobre distintos participantes.
3. Cada lectura obtiene un read lock.
4. El cliente calcula localmente los valores intermedios.
5. Al momento de escribir, manda los writes a los participantes correspondientes.
6. Cada write requiere un write lock.
7. Se elige uno de los participantes como transaction coordinator.
8. El coordinador ejecuta Two-Phase Commit.
9. Una vez hecho el commit, se liberan los locks.

Un esquema simplificado:

```text
Cliente              P1                         P2
   |                 |                          |
   |---- read ------>|                          |   adquiere read lock
   |<--- value ------|                          |
   |---------------- read -------------------->|   adquiere read lock
   |<--------------- value --------------------|
   |                 |                          |
   |---- elige P1 como coordinator ------------|
   |                 |                          |
   |---- write ----->|                          |   adquiere write lock
   |---------------- write ------------------->|   adquiere write lock
   |                 |                          |
   |---- commit ---->|                          |
   |                 |---- prepare ----------->|
   |                 |<--- ok -----------------|
   |                 |---- commit ------------>|
   |                 |<--- ok -----------------|
   |                 |                          |
   |                 | libera locks            | libera locks
```

Two-Phase Locking garantiza serializabilidad: aunque internamente las operaciones de distintas transacciones se intercalen, el resultado es equivalente a ejecutar las transacciones una por una en algún orden serial.

Two-Phase Commit garantiza atomicidad entre participantes: todos comitean o todos abortan.

La diferencia con un 2PC más simple es que el estado del coordinador y de los participantes está replicado mediante Paxos. Eso evita que el algoritmo quede bloqueado indefinidamente si se cae una máquina. Si el líder de un grupo falla, otro puede tomar su lugar y continuar desde el estado persistido.

El costo es la latencia. Las read-write transactions son más lentas porque requieren locks, coordinación, consenso dentro de cada grupo de réplicas y Two-Phase Commit entre participantes.

## Read-only transactions

Las transacciones read-only son el caso más importante para optimizar. La idea es que una lectura pueda resolverse:

- desde una réplica local;
- sin locks;
- sin coordinar con todos los participantes;
- sin bloquear escrituras;
- leyendo un snapshot consistente.

Esto requiere resolver un problema: si una transacción lee `x` e `y`, ambos valores deben pertenecer al mismo snapshot lógico de la base de datos.

No sirve leer `x` de un estado viejo e `y` de un estado nuevo si ese estado combinado nunca existió.

Ejemplo:

```text
T1: x = 1, y = 2
T2: x = 3, y = 4
```

Una lectura que devuelve:

```text
x = 1
y = 4
```

puede ser serializable según cómo se la ordene, pero no representa necesariamente el estado actual esperado si la lectura ocurrió después de `T2`. Para Spanner no alcanza con leer un snapshot cualquiera: además debe respetar el orden del tiempo físico.

## External consistency

Spanner busca una propiedad llamada **external consistency**.

La idea es:

```text
Si T1 hace commit antes de que T2 empiece,
entonces T2 debe ver los efectos de T1.
```

Ese “antes” se refiere al tiempo físico real, no solo a causalidad lógica.

Esto combina dos propiedades:

- **serializabilidad**: las transacciones se comportan como si se ejecutaran una por una;
- **linealizabilidad**: si una operación termina antes de que otra empiece, la segunda debe observar los efectos de la primera.

Una lectura puede ser serializable y aun así no ser linealizable. Por ejemplo, si una escritura `x = 3, y = 4` ya terminó y luego se hace una lectura, esa lectura no debería devolver el estado anterior `x = 1, y = 2`.

Spanner quiere consistencia fuerte: después de escribir y recibir `OK`, una lectura posterior debe poder ver lo escrito.

## Timestamp ordering: repaso

Una forma de implementar lecturas consistentes es asignar timestamps a las versiones de los datos.

Ejemplo:

```text
K   V    TS
x   v1   10
y   v2   12
```

Una transacción read-only con timestamp `13` puede leer ambos valores:

```text
read-only @13
read x -> v1, porque 10 < 13
read y -> v2, porque 12 < 13
```

Pero una transacción read-only con timestamp `11` tendría problemas:

```text
read-only @11
read x -> v1, porque 10 < 11
read y -> falla, porque 12 > 11
```

En un sistema simple, la transacción debería abortar y reintentarse con un timestamp posterior.

## MVCC: Multi-Version Concurrency Control

MVCC evita abortar lecturas cuando hay versiones anteriores disponibles.

En lugar de guardar solo el último valor de cada clave, se guardan varias versiones, cada una asociada a un timestamp.

```text
(K, TS)    V
(x, 10)    v1
(y, 9)     v2
(y, 12)    v3
```

Si una transacción lee en timestamp `11`:

```text
read-only @11
read x -> v1  (versión más reciente de x con TS <= 11)
read y -> v2  (versión más reciente de y con TS <= 11)
```

Aunque exista `y` en timestamp `12`, esa versión pertenece al futuro respecto de la lectura. Por eso se lee la versión anterior.

La regla general es:

```text
Para leer una clave K en timestamp T,
se devuelve la versión de K con mayor timestamp <= T.
```

Esto permite leer snapshots consistentes sin locks. También permite hacer lecturas históricas: si se elige un timestamp del pasado, se puede reconstruir el estado de la base en ese momento.

PostgreSQL y MySQL también usan ideas de MVCC, aunque normalmente no guardan todas las versiones históricas para siempre. Spanner explota esta técnica en un sistema distribuido global.

## Problema: réplicas desactualizadas

MVCC resuelve el problema de elegir una versión correcta dentro de una réplica. Pero todavía queda otro problema: la réplica local puede estar desactualizada.

Ejemplo:

```text
Réplica líder:
(x, 10)
(x, 11)

Réplica local:
(x, 10)
```

Si se hace una lectura `read-only @12` contra la réplica local, esa réplica podría devolver `(x, 10)` aunque ya exista `(x, 11)`. Eso rompe la consistencia fuerte.

## Safe time

Para evitar leer desde una réplica que todavía no recibió todas las escrituras relevantes, Spanner usa la idea de **safe time**.

Cada réplica mantiene información sobre hasta qué timestamp está segura de estar actualizada.

La lectura en timestamp `T` solo puede resolverse cuando la réplica sabe que ya recibió todo lo anterior a `T`.

Una forma intuitiva de verlo:

```text
Quiero leer @12.
La réplica solo sabe que está actualizada hasta @10.
No puede responder todavía.

Luego recibe alguna entrada con timestamp @13.
Como el log avanza en orden, eso implica que ya recibió todo hasta @13.
Ahora puede responder la lectura @12.
```

Safe time introduce una espera, pero evita devolver valores viejos desde una réplica local.

## Problema: relojes físicos no sincronizados

Hasta ahora se usaron timestamps como si todas las máquinas tuvieran relojes perfectamente sincronizados. Eso no es posible en un sistema distribuido real.

Los relojes físicos tienen drift: se atrasan o adelantan. Además, pedir la hora a otro servidor tiene latencia. Por eso no se puede asumir que dos máquinas conocen exactamente el mismo tiempo físico.

En read-write transactions el problema no aparece de la misma forma porque el locking pesimista y Two-Phase Locking ordenan las operaciones. Pero en read-only transactions, que dependen de timestamps físicos, los relojes desincronizados pueden romper la consistencia.

Hay dos casos:

### Lector adelantado

Si una lectura usa un timestamp demasiado adelantado, la lectura sigue siendo correcta, pero puede ser lenta.

Ejemplo: si el sistema está cerca de `12`, pero una lectura llega marcada como `400`, deberá esperar safe time hasta que la réplica esté segura de haber recibido todo hasta `400`. La lectura no es incorrecta, pero puede quedar esperando demasiado.

### Lector atrasado

Si una lectura usa un timestamp atrasado, puede leer datos del pasado.

Ejemplo:

```text
T1 @0:  x = 1
T2 @10: x = 2

Después de T2, se ejecuta una lectura real.
Pero el reloj del lector dice @5.
```

La lectura `read x @5` devolvería `x = 1`, aunque físicamente ocurrió después de que `x = 2` ya había sido escrito. Eso viola linealizabilidad.

En DynamoDB una lectura de un valor viejo puede ser aceptable por diseño. En Spanner no: Spanner busca consistencia fuerte.

## TrueTime

La solución de Google es **TrueTime**.

TrueTime no devuelve un instante exacto. Devuelve un intervalo:

```text
TT.now() = [earliest, latest]
```

La garantía es:

```text
el tiempo real actual está dentro de ese intervalo
```

Es decir:

```text
earliest <= tiempo real <= latest
```

TrueTime también ofrece primitivas como:

```text
TT.after(t)
TT.before(t)
```

`TT.after(t)` permite saber si el tiempo real ya pasó definitivamente un timestamp `t`.

La clave no es conocer el tiempo exacto, sino conocer una cota segura de incertidumbre.

## Cómo se implementa TrueTime

La arquitectura usa servidores de tiempo llamados **time masters**, conectados a fuentes de tiempo precisas como GPS o relojes atómicos.

Los servidores comunes consultan periódicamente a los time masters y reciben un intervalo de incertidumbre. Localmente hacen avanzar ese intervalo usando su reloj propio, pero como los relojes locales tienen drift, la incertidumbre crece con el tiempo.

Por eso se vuelve a consultar a los time masters cada cierto intervalo. La idea es mantener pequeño el intervalo `[earliest, latest]`.

Los relojes atómicos y GPS no eliminan la incertidumbre, pero la reducen. Menor incertidumbre implica menores esperas en los algoritmos de lectura y commit.

## Elegir timestamps con TrueTime

Aunque TrueTime devuelve intervalos, MVCC necesita timestamps puntuales. Por eso Spanner debe elegir un timestamp concreto para cada transacción.

### Timestamp de una read-only transaction

Para una lectura se usa:

```text
TS_read = TT.now().latest
```

Se elige `latest` porque es un timestamp garantizado en el futuro respecto del tiempo real actual. Eso evita que una lectura quede marcada con un tiempo demasiado viejo y lea datos del pasado.

Si se eligiera `earliest`, podría ocultar escrituras ya realizadas.

### Timestamp de una escritura

Para una transacción read-write, la elección es más delicada.

No alcanza con elegir arbitrariamente un valor del intervalo. Si se elige un timestamp demasiado en el futuro, una lectura posterior podría no ver la escritura. Si se elige un timestamp demasiado en el pasado, se puede reescribir historia y romper lecturas históricas.

Spanner usa dos reglas:

1. **Start rule**.
2. **Commit wait**.

## Start rule

Cuando comienza la fase de commit, se obtiene un intervalo de TrueTime:

```text
TT.now() = [earliest, latest]
```

El timestamp de la transacción se fija como:

```text
TS_commit = latest
```

Ese timestamp queda asociado a la escritura.

## Commit wait

Después de elegir `TS_commit`, la transacción no se hace visible inmediatamente. El sistema espera hasta estar seguro de que ese timestamp quedó en el pasado.

Es decir, espera hasta que:

```text
TT.after(TS_commit) == true
```

Recién cuando el timestamp elegido está definitivamente en el pasado:

- se hace visible el commit;
- se responde `OK` al cliente;
- las lecturas posteriores quedan obligadas a tener timestamps mayores.

Esto garantiza external consistency.

La secuencia conceptual es:

```text
prepare OK de todos
start rule: TS_commit = TT.now().latest
commit wait: esperar hasta que TT.after(TS_commit)
hacer visible el commit
responder OK al cliente
```

La espera de commit wait es el costo de usar relojes físicos con incertidumbre. TrueTime minimiza ese costo al mantener intervalos pequeños.

## Por qué commit wait garantiza consistencia fuerte

Supongamos que una escritura `T1` recibe timestamp `12`, pero el tiempo real todavía podría estar antes de `12`. Si Spanner respondiera inmediatamente al cliente, una lectura posterior podría recibir un timestamp menor o no ver la escritura.

Con commit wait, Spanner no responde hasta que `12` ya quedó definitivamente en el pasado. Entonces cualquier lectura que empiece después del `OK` elegirá `TT.now().latest`, que necesariamente será mayor que `12`. Por MVCC, esa lectura verá la versión escrita por `T1`.

La linealizabilidad se logra haciendo que el orden de timestamps respete el orden observable por los clientes.

## Costo de las esperas

Spanner introduce dos esperas posibles:

1. **Safe time** en lecturas: esperar a que la réplica local esté suficientemente actualizada para responder una lectura en cierto timestamp.
2. **Commit wait** en escrituras: esperar a que el timestamp elegido para el commit quede definitivamente en el pasado.

Estas esperas existen porque no se puede sincronizar perfectamente el tiempo físico. TrueTime no elimina el problema, pero acota la incertidumbre para que las esperas sean pequeñas.

## Resultado

Spanner logra:

- transacciones read-write ACID distribuidas;
- transacciones read-only muy rápidas;
- lecturas locales sin locks;
- snapshot isolation;
- external consistency;
- consistencia fuerte global;
- replicación entre data centers;
- uso de MVCC para leer snapshots;
- uso de TrueTime para ordenar transacciones con tiempo físico acotado.

El logro principal es permitir una base de datos globalmente distribuida con semántica similar a una base SQL tradicional. Las escrituras son más caras porque requieren coordinación distribuida, pero las lecturas, que son el caso dominante, se resuelven rápido leyendo versiones locales consistentes.

## Comparación rápida con DynamoDB

DynamoDB prioriza disponibilidad y baja latencia, aceptando consistencia eventual en varios casos. Spanner busca consistencia fuerte, aun pagando el costo de coordinación, MVCC y esperas asociadas a TrueTime.

```text
DynamoDB:
- particionado por hash;
- consistencia eventual en muchos casos;
- transacciones más limitadas;
- diseño orientado a disponibilidad y latencia.

Spanner:
- particionado por rangos de primary key;
- transacciones distribuidas más generales;
- consistencia fuerte;
- MVCC + TrueTime;
- lecturas locales consistentes;
- diseño global entre data centers.
```

## Resumen final

Spanner cierra el bloque de sistemas de storage mostrando una combinación de ideas distribuidas y de bases de datos clásicas.

Las read-write transactions usan técnicas conocidas: Two-Phase Commit, Two-Phase Locking y replicación por Paxos groups.

Las read-only transactions son la parte más interesante: usan MVCC para leer snapshots sin locks y TrueTime para que esos snapshots respeten el orden del tiempo real.

La idea clave de TrueTime es no fingir que se conoce el tiempo exacto. En cambio, se trabaja con intervalos de incertidumbre. Con esos intervalos, Spanner puede decidir cuándo un timestamp ya está definitivamente en el pasado y, por lo tanto, puede garantizar que las lecturas posteriores vean las escrituras anteriores.

