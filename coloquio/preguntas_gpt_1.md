# Preguntas nuevas de examen - Sistemas Distribuidos I

Set de preguntas nuevas en formato **afirmación + respuesta justificada**, basadas en el modelo de examen y los resúmenes de clases.

---

## 1. Introducción / RPC

**Pregunta:**  
La principal ventaja de RPC es que permite tratar una llamada remota exactamente igual que una llamada local, por lo que el programador no necesita preocuparse por latencia, timeouts, fallas parciales o pérdida de mensajes.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

RPC intenta dar una abstracción cómoda para invocar operaciones remotas, pero una llamada remota nunca es idéntica a una llamada local.

Una llamada local ocurre dentro del mismo proceso o máquina, con memoria compartida y fallas relativamente simples. En cambio, una llamada remota atraviesa la red, puede demorarse, perderse, ejecutarse en el servidor sin que la respuesta vuelva, duplicarse por reintentos o fallar por una partición.

Por eso, aunque RPC simplifica la interfaz, no conviene ocultar completamente que la operación es distribuida.

</details>

---

## 2. MapReduce

**Pregunta:**  
En MapReduce, la cantidad de tareas `map` y `reduce` debe coincidir necesariamente con la cantidad de máquinas físicas disponibles en el clúster.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

`M` y `R` son cantidades lógicas de tareas, no cantidades de máquinas físicas.

Puede haber muchos más splits de entrada que workers disponibles. El master asigna tareas a medida que los workers quedan libres. Esto permite balancear mejor la carga, porque si una tarea termina rápido, ese worker puede recibir otra.

Lo mismo ocurre con las tareas reduce: los workers no son “máquinas map” o “máquinas reduce” de forma fija, sino nodos genéricos que ejecutan tareas asignadas por el coordinador.

</details>

---

## 3. MapReduce

**Pregunta:**  
El `shuffle` en MapReduce existe para garantizar que todos los valores asociados a una misma clave intermedia lleguen al mismo reducer.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

La función `map` emite pares intermedios `(k2, v2)`. Luego, el sistema debe agrupar todos los valores que tengan la misma clave `k2`.

Para eso se usa una función de particionado, típicamente:

```text
hash(key) mod R
```

De esta manera, todas las apariciones de una misma clave van al mismo reducer. Si la clave `"hola"` aparece en muchos mappers distintos, todos esos pares deben llegar al mismo reducer para poder calcular correctamente el resultado.

</details>

---

## 4. Storage distribuido

**Pregunta:**  
El sharding por sí solo mejora la tolerancia a fallas, porque al repartir los datos entre varias máquinas cada dato queda automáticamente protegido ante la caída de un nodo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El sharding reparte los datos entre varias máquinas, pero no necesariamente los replica.

Si un dato está solamente en un shard y la máquina que contiene ese shard falla, ese dato puede quedar inaccesible o perderse. El sharding ayuda a escalar capacidad, throughput y paralelismo, pero la durabilidad y la alta disponibilidad requieren replicación.

En la práctica, muchos sistemas combinan ambas técnicas: particionan los datos en shards y luego replican cada shard en varias máquinas.

</details>

---

## 5. Replicación primary-backup

**Pregunta:**  
En un esquema primary-backup, si el primary responde `OK` al cliente antes de que todas las réplicas estén actualizadas, una lectura posterior desde una réplica puede devolver un valor viejo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Si el primary confirma la escritura antes de que todos los backups hayan aplicado el cambio, algunas réplicas pueden quedar atrasadas temporalmente.

Esto mejora la latencia de escritura y la disponibilidad, pero permite lecturas stale. Si un cliente escribe `x = 1` y luego lee desde una réplica que todavía tiene `x = 0`, el sistema no se comporta como una única copia linealizable.

Para obtener lecturas más fuertes, una opción simple es leer desde el primary o usar algún mecanismo de coordinación adicional.

</details>

---

## 6. GFS

**Pregunta:**  
En GFS, el master participa en el camino de datos de todas las lecturas y escrituras para poder garantizar consistencia fuerte entre chunkservers.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El master de GFS administra metadata: namespace, ubicación de chunks, réplicas, leases, versiones y coordinación general.

Pero los datos no pasan por el master. En una lectura, el cliente consulta al master para saber qué chunkservers tienen el chunk, y luego lee directamente desde un chunkserver. En una escritura, el cliente consulta quién es la primary replica y cuáles son las secondaries, pero el envío de datos ocurre directamente entre cliente y réplicas.

Esto evita que el master se convierta en un cuello de botella para tráfico pesado de datos.

</details>

---

## 7. GFS

**Pregunta:**  
El `record append` de GFS garantiza semántica exactly-once: si un cliente reintenta una operación, el registro aparecerá una sola vez en el archivo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

`record append` garantiza que el sistema elige el offset donde agregar el registro y permite que múltiples clientes agreguen datos concurrentemente.

Sin embargo, ante fallas y reintentos, un registro puede aparecer duplicado. También pueden aparecer regiones de padding o réplicas con contenido parcialmente distinto.

Por eso, GFS traslada parte de la responsabilidad a la aplicación: por ejemplo, usar identificadores únicos para detectar duplicados o checksums para detectar registros inválidos.

</details>

---

## 8. Raft

**Pregunta:**  
En Raft, una entrada se considera `committed` cuando fue escrita en el log del líder, incluso si todavía no fue replicada a otros nodos.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Una entrada no queda committed por estar solamente en el líder.

En Raft, una entrada queda committed cuando está almacenada en una mayoría de nodos. Recién ahí el líder puede aplicarla a la máquina de estados y responder `OK` al cliente.

Esto asegura que, si el líder cae, la operación no se pierda siempre que no fallen más nodos que los tolerados por la configuración.

</details>

---

## 9. Raft

**Pregunta:**  
En una elección de líder de Raft, un candidato con un log más largo siempre se considera más actualizado que un candidato con un log más corto.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Raft compara primero el `lastLogTerm` y recién después el `lastLogIndex`.

Un candidato está más actualizado si su último término es mayor. Solo si los términos son iguales se usa el largo del log como desempate.

Esto evita elegir como líder a un nodo que tiene muchas entradas, pero de términos viejos, y que podría borrar entradas más nuevas o incluso ya comprometidas.

</details>

---

## 10. Raft

**Pregunta:**  
Si un cliente envía una operación a un líder Raft y no recibe respuesta, entonces puede asumir con seguridad que la operación no fue ejecutada.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Si el cliente no recibe respuesta, no puede saber si la operación fue comiteada o no.

Puede haber pasado que el líder haya replicado la entrada en mayoría y haya caído antes de responder. En ese caso, la operación podría sobrevivir y aplicarse más adelante.

También puede haber pasado que la operación haya quedado solamente en una minoría y luego sea descartada por un nuevo líder.

Por eso, las aplicaciones suelen usar identificadores únicos de operación, deduplicación o mecanismos de idempotencia para manejar reintentos.

</details>

---

## 11. Raft

**Pregunta:**  
En Raft, los valores `currentTerm`, `votedFor` y el `log[]` deben persistirse en disco para mantener la seguridad del algoritmo ante reinicios.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

`currentTerm` debe persistirse para que un nodo no olvide términos más nuevos después de reiniciar.

`votedFor` debe persistirse porque un nodo no puede votar dos veces en el mismo término. Si vota, se cae y reinicia, debe recordar ese voto.

El `log[]` debe persistirse porque un `OK` a `AppendEntries` significa que esa entrada fue almacenada de forma durable. Si un follower responde `OK` y luego pierde la entrada, puede romper la mayoría usada por el líder.

</details>

---

## 12. Linealizabilidad

**Pregunta:**  
Leer siempre desde el nodo que cree ser líder alcanza para garantizar lecturas linealizables en Raft.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Puede ocurrir una partición de red donde el líder viejo queda aislado en una minoría, mientras la mayoría elige un nuevo líder.

El líder viejo puede seguir creyendo que es líder y responder lecturas con información vieja. Si mientras tanto el nuevo líder acepta escrituras, una lectura respondida por el líder viejo puede violar linealizabilidad.

Para hacer una lectura estrictamente linealizable, el líder debe asegurarse de que todavía tiene autoridad, por ejemplo consultando una mayoría o usando un mecanismo seguro de leases.

</details>

---

## 13. ZooKeeper

**Pregunta:**  
ZooKeeper sacrifica linealizabilidad completa en las lecturas para mejorar throughput, pero mantiene garantías útiles por cliente como `read your writes` y lecturas monotónicas.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

ZooKeeper está optimizado para cargas con muchas lecturas y pocas escrituras.

Las escrituras pasan por el líder y se ordenan mediante Zab, por lo que son linealizables. Las lecturas, en cambio, pueden ser respondidas localmente por followers para escalar mejor.

Eso significa que una lectura puede no ver la última escritura global. Pero ZooKeeper mantiene orden FIFO por cliente: un cliente no debería dejar de ver sus propias escrituras ni retroceder a una versión anterior a otra que ya observó.

</details>

---

## 14. Memcache

**Pregunta:**  
En un cache look-aside, ante una escritura conviene hacer `db.write(k, v)` y luego `memcache.set(k, v)`, porque así el cache queda actualizado inmediatamente y se evitan condiciones de carrera.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Actualizar el cache con `set` después de escribir en la base puede generar carreras.

Ejemplo:

```text
C1: db.write(x = 1)
C2: db.write(x = 2)
C2: memcache.set(x = 2)
C1: memcache.set(x = 1)
```

El resultado final sería:

```text
DB:       x = 2
Memcache: x = 1
```

El cache queda con un valor viejo. Por eso suele preferirse invalidar:

```text
db.write(k, v)
memcache.delete(k)
```

El `delete` es idempotente y la próxima lectura repuebla el cache desde la base.

</details>

---

## 15. Memcache

**Pregunta:**  
El TTL debería ser el mecanismo principal de consistencia de un cache, porque garantiza que tarde o temprano los valores viejos desaparezcan.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El TTL sirve como red de seguridad, pero no debería ser el mecanismo principal de consistencia.

Si el TTL es largo, el sistema puede devolver datos viejos durante mucho tiempo. Si el TTL es muy corto, baja el hit ratio y el cache pierde utilidad, porque muchas lecturas vuelven a pegarle a la base.

La estrategia principal suele ser invalidar activamente el cache cuando cambia el dato en la base.

</details>

---

## 16. Dynamo

**Pregunta:**  
Dynamo prioriza disponibilidad sobre consistencia fuerte, por lo que puede aceptar escrituras concurrentes sobre una misma clave y reconciliarlas posteriormente.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Dynamo fue diseñado para estar siempre disponible para escrituras, incluso ante fallas o particiones.

A diferencia de Raft, no usa un líder global ni un log global. Dos partes del sistema pueden aceptar escrituras concurrentes sobre la misma clave. Luego, Dynamo usa relojes vectoriales para detectar si las versiones son causales o concurrentes.

Si las versiones son concurrentes y no se pueden fusionar automáticamente, el cliente o la aplicación deben participar en la reconciliación.

</details>

---

## 17. Dynamo

**Pregunta:**  
Los relojes de Lamport permiten distinguir perfectamente cuándo dos eventos son concurrentes.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Los relojes de Lamport garantizan que:

```text
si a → b, entonces C(a) < C(b)
```

Pero no vale la inversa. Que `C(a) < C(b)` no implica necesariamente que `a` haya causado a `b`.

Dos eventos concurrentes pueden terminar con timestamps distintos. Por eso los relojes de Lamport sirven para construir un orden compatible con la causalidad, pero no detectan perfectamente concurrencia.

Para detectar concurrencia de forma más precisa se usan relojes vectoriales.

</details>

---

## 18. Dynamo

**Pregunta:**  
El `hinted handoff` implica que, si una réplica original está caída, otra réplica pasa a ser dueña permanente de esos datos.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

En `hinted handoff`, un nodo alternativo guarda temporalmente una escritura que estaba destinada a otro nodo.

El hint indica algo como:

```text
esta escritura era para S3;
guardala mientras S3 está caído;
cuando S3 vuelva, entregásela.
```

No cambia de forma permanente la preference list ni la propiedad lógica del dato. Es un mecanismo para mantener disponibilidad y durabilidad durante fallas temporales.

</details>

---

## 19. Transacciones distribuidas

**Pregunta:**  
En Two-Phase Commit, alcanza con que una mayoría de participantes responda `OK` al `prepare` para que el coordinador pueda decidir `commit`.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Two-Phase Commit no funciona por mayoría. Requiere unanimidad.

Si todos los participantes responden `OK` al `prepare`, el coordinador puede decidir `commit`. Si alguno responde `NO`, o no se puede preparar correctamente, debe abortar.

Esto es distinto de Raft, donde una operación puede confirmarse con mayoría. En 2PC, como se busca atomicidad entre todos los participantes, todos deben estar preparados para confirmar.

</details>

---

## 20. Transacciones distribuidas

**Pregunta:**  
Después de responder `OK` al `prepare`, un participante puede abortar localmente si pasa demasiado tiempo sin recibir la decisión final del coordinador.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Después de responder `OK` al `prepare`, el participante queda en un punto de no retorno.

Ya prometió que podrá confirmar si el coordinador decide `commit`. Si abortara unilateralmente por timeout, podría pasar que otros participantes sí hayan recibido `commit`, rompiendo la atomicidad.

Por eso 2PC puede bloquear: un participante preparado debe esperar la decisión final del coordinador, salvo que exista algún mecanismo adicional de recuperación o replicación del coordinador.

</details>

---

## 21. DynamoDB

**Pregunta:**  
En DynamoDB, las operaciones no transaccionales simples y las operaciones transaccionales siguen exactamente el mismo camino interno, pasando siempre por el Transaction Coordinator.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Las operaciones comunes, como un `GetItem` o un `PutItem` no transaccional, pueden ir directamente al storage node correspondiente a la partición.

Las operaciones transaccionales, en cambio, pasan por un Transaction Coordinator. Ese coordinador se encarga de coordinar el protocolo necesario para que varias operaciones sobre distintas particiones se apliquen todas juntas o ninguna.

La distinción es importante porque evita pagar el costo de coordinación transaccional en operaciones simples.

</details>

---

## 22. Bitcoin

**Pregunta:**  
Bitcoin puede usar una votación del tipo “un nodo, un voto” porque cada nodo de la red representa una identidad real difícil de falsificar.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Bitcoin es una red permissionless: cualquiera puede crear nodos o identidades.

Si se usara “un nodo, un voto”, un atacante podría crear miles de identidades falsas y controlar la votación. Esto se conoce como ataque Sybil.

Por eso Bitcoin reemplaza la votación por Proof of Work: la probabilidad de proponer el próximo bloque depende del poder de cómputo, no de la cantidad de identidades declaradas.

</details>

---

## 23. Bitcoin

**Pregunta:**  
La finalidad en Bitcoin es probabilística: una transacción incluida en un bloque puede volverse más difícil de revertir a medida que se agregan bloques encima, pero no tiene la misma finalidad inmediata que una entrada committed en Raft.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

En Raft, una entrada committed queda preservada por las reglas de mayoría y elección de líder.

En Bitcoin, pueden aparecer forks temporales: dos mineros pueden encontrar bloques válidos casi al mismo tiempo. La red termina siguiendo la cadena con más trabajo acumulado, y la otra rama puede quedar abandonada.

Cuantos más bloques se agregan encima de una transacción, más costoso se vuelve revertirla. Pero la garantía es probabilística, no una confirmación inmediata equivalente a Raft.

</details>

---

## 24. Spanner

**Pregunta:**  
Las transacciones read-only de Spanner usan Two-Phase Commit y locks distribuidos, igual que las transacciones read-write, para garantizar consistencia externa.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Spanner optimiza especialmente las transacciones read-only.

Las transacciones read-write usan locks, Two-Phase Locking, Two-Phase Commit y replicación mediante Paxos/Multi-Paxos.

Las read-only, en cambio, intentan evitar locks y coordinación global. Leen un snapshot consistente usando MVCC y timestamps, apoyándose en TrueTime y safe time para no leer desde réplicas atrasadas.

Esto permite resolver muchas lecturas desde réplicas locales con menor latencia.

</details>

---

## 25. Spanner

**Pregunta:**  
La consistencia externa de Spanner exige que, si una transacción `T1` hace commit antes de que `T2` empiece, entonces `T2` debe observar los efectos de `T1`.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

La consistencia externa combina serializabilidad con respeto del orden del tiempo real.

No alcanza con que exista algún orden serial equivalente. Además, ese orden debe respetar que si una transacción terminó antes de que otra empezara, la segunda no puede ignorar los efectos de la primera.

Spanner logra esto usando timestamps, MVCC, TrueTime y mecanismos como commit wait para manejar la incertidumbre de los relojes físicos.

</details>

---

## 26. Sistemas de mensajería

**Pregunta:**  
En SQS, cuando un consumidor recibe un mensaje mediante `ReceiveMessage`, el mensaje se elimina inmediatamente de la cola.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Cuando un consumidor recibe un mensaje, SQS no lo borra inmediatamente.

El mensaje queda invisible para otros consumidores durante el `visibility timeout`. Si el consumidor termina correctamente, debe llamar a `DeleteMessage`.

Si el consumidor falla antes de borrar el mensaje, el timeout expira y el mensaje vuelve a estar disponible. Por eso SQS ofrece semántica at-least-once y los consumidores deben ser idempotentes.

</details>

---

## 27. Sistemas de mensajería

**Pregunta:**  
En un modelo publisher-subscriber, todos los suscriptores interesados reciben una copia del mensaje; en un modelo producer-consumer, varios consumidores compiten por procesar mensajes de una misma cola.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

En producer-consumer, la cola funciona como un buffer de tareas. Si hay varios consumidores, cada mensaje debería ser procesado por uno de ellos.

En publisher-subscriber, el productor publica en un tópico y el sistema distribuye el mensaje a todos los suscriptores.

Por eso pub/sub sirve mejor para eventos de negocio que deben ser observados por varios servicios distintos, mientras que producer-consumer sirve mejor para distribuir trabajo entre workers equivalentes.

</details>

---

## 28. Stream processing

**Pregunta:**  
En procesamiento de streams, para agrupar eventos por ventanas temporales siempre conviene usar processing time, porque es el tiempo que conoce con certeza el sistema.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Para muchos problemas interesa el `event time`, es decir, el momento en que el evento ocurrió realmente.

El `processing time` depende de cuándo el sistema recibió o procesó el evento. Si hay retrasos, retries o eventos fuera de orden, agrupar por processing time puede producir resultados incorrectos desde el punto de vista del negocio.

Por ejemplo, una búsqueda hecha a las 8:01 pero procesada a las 8:10 debería contarse en la ventana de las 8:01 si se busca medir actividad real por minuto.

</details>

---

## 29. Stream processing

**Pregunta:**  
Un watermark permite decidir cuándo cerrar una ventana temporal, indicando hasta qué event time se considera que el stream ya está completo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Un watermark es una marca temporal que avanza de forma monótona e indica que ya no deberían llegar eventos con `event_time` menor a ese valor.

Si una ventana termina a las 8:05 y el watermark llega a 8:05, el sistema puede emitir o cerrar esa ventana.

En streams particionados, el watermark global suele estar limitado por la partición más atrasada, porque para cerrar una ventana se necesita estar seguro de que no faltan eventos relevantes de ninguna partición.

</details>

---

## 30. Stream processing

**Pregunta:**  
El replay de eventos en sistemas de streaming permite reconstruir estado, corregir bugs o recalcular métricas, siempre que la fuente de eventos conserve el histórico necesario.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Si la fuente de eventos es un log persistente, como Kafka, un consumidor puede volver a leer desde una posición anterior.

Esto permite reconstruir estado después de una falla, reprocesar eventos con una versión corregida del código o recalcular métricas históricas.

El replay es una diferencia importante frente a sistemas de mensajería donde los mensajes desaparecen una vez consumidos y confirmados.

</details>
