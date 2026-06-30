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

## 31. Sistemas distribuidos

**Pregunta:**  
Un sistema distribuido puede pensarse igual que una computadora multicore, porque en ambos casos hay varios procesadores ejecutando tareas en paralelo sobre una memoria compartida.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

En una computadora multicore, los procesadores suelen compartir memoria. En un sistema distribuido, cada nodo tiene su propia memoria local y se comunica con los demás mediante mensajes.

Esta diferencia cambia completamente el modelo de programación. No hay memoria global inmediata, no hay locks locales compartidos entre nodos y no existe una visión instantánea única del estado del sistema.

</details>

---

## 32. Sistemas distribuidos

**Pregunta:**  
Una falla parcial ocurre cuando una parte del sistema falla, pero otras partes continúan funcionando.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Las fallas parciales son una característica central de los sistemas distribuidos.

Un nodo puede estar caído, otro puede seguir respondiendo, la red entre dos máquinas puede estar rota y un tercer nodo puede tener una visión diferente del estado. Esto hace que detectar y manejar fallas sea más difícil que en un sistema de una sola máquina.

</details>

---

## 33. RPC

**Pregunta:**  
Si un cliente hace una llamada RPC y no recibe respuesta, siempre puede concluir que el servidor nunca ejecutó la operación.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

La falta de respuesta no permite distinguir entre varios casos:

- el request nunca llegó;
- el servidor recibió el request pero falló antes de ejecutarlo;
- el servidor ejecutó la operación pero la respuesta se perdió;
- la red está lenta;
- el servidor está vivo pero saturado.

Por eso, los sistemas distribuidos deben diseñar cuidadosamente la semántica de reintentos e idempotencia.

</details>

---

## 34. MapReduce

**Pregunta:**  
En MapReduce, si una tarea `map` falla, el framework puede reejecutarla porque su salida depende únicamente del input y de la función `map`, siempre que esta sea determinística.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

La tolerancia a fallas de MapReduce se apoya fuertemente en la reejecución de tareas.

Si una tarea `map` falla, el master puede reasignarla a otro worker. Como la tarea procesa el mismo split de entrada y la función `map` debería ser determinística, se espera que produzca la misma salida intermedia.

Esto sería mucho más difícil si las tareas tuvieran efectos laterales externos o dependieran de estado mutable no controlado.

</details>

---

## 35. MapReduce

**Pregunta:**  
El uso de combiners en MapReduce siempre preserva la corrección del resultado, sin importar qué operación esté implementando el reducer.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Un combiner solo es seguro cuando la operación permite combinar resultados parciales sin cambiar la semántica final.

Por ejemplo, sumar apariciones de palabras permite usar un combiner, porque sumar parcialmente y luego volver a sumar da el mismo resultado.

Pero no todas las operaciones cumplen esta propiedad. Si el reducer depende del orden, de todos los valores completos o de una operación no asociativa, usar un combiner podría cambiar el resultado.

</details>

---

## 36. MapReduce

**Pregunta:**  
El master de MapReduce puede convertirse en un punto crítico del sistema porque mantiene el estado de las tareas y coordina la asignación de trabajo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

El master mantiene metadata sobre tareas `idle`, `in-progress` y `completed`, asigna trabajo a workers, detecta fallas y registra la ubicación de archivos intermedios.

Esto simplifica el diseño, pero lo convierte en un componente central. Si el master falla, el job puede verse afectado, aunque los datos estén distribuidos entre muchos workers.

</details>

---

## 37. Sharding

**Pregunta:**  
El sharding facilita las consultas que involucran joins o transacciones entre muchos shards, porque cada máquina puede resolver su parte de manera independiente sin coordinación.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El sharding ayuda cuando las operaciones pueden resolverse dentro de un único shard o cuando las claves están bien distribuidas.

Pero si una consulta necesita datos de varios shards, aparecen problemas adicionales: joins distribuidos, agregaciones globales, transacciones distribuidas, coordinación entre nodos y posibles cuellos de botella.

Por eso, el diseño de la clave de particionado es una decisión muy importante.

</details>

---

## 38. Replicación

**Pregunta:**  
En una máquina de estados replicada, si todas las réplicas parten del mismo estado inicial y aplican las mismas operaciones en el mismo orden, entonces terminan en el mismo estado final, suponiendo operaciones determinísticas.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Esta es la idea central de la replicación por máquina de estados.

El problema difícil no es aplicar operaciones localmente, sino lograr que todas las réplicas acuerden el mismo conjunto de operaciones y el mismo orden. De ahí surge la necesidad de logs replicados, consenso, líderes, quórums o protocolos similares.

</details>

---

## 39. Logs replicados

**Pregunta:**  
El log en un sistema replicado sirve solamente como mecanismo de auditoría; no participa en la definición del orden de las operaciones.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El log es una abstracción central porque define el orden total de las operaciones.

Si todas las réplicas aplican el mismo log en el mismo orden, llegan al mismo estado. Por eso, en sistemas como Raft, el objetivo del protocolo es mantener un log replicado consistente entre nodos.

</details>

---

## 40. GFS

**Pregunta:**  
En GFS, el tamaño grande de los chunks reduce la cantidad de metadata que debe manejar el master.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

GFS usa chunks grandes, típicamente de 64 MB.

Esto reduce la cantidad total de chunks para archivos muy grandes. Al haber menos chunks, el master debe mantener menos metadata: menos mappings de archivo a chunk y menos ubicaciones de réplicas.

La contracara es que GFS queda más orientado a archivos grandes y cargas batch que a muchos archivos pequeños.

</details>

---

## 41. GFS

**Pregunta:**  
El lease de la primary replica en GFS ayuda a evitar que dos réplicas actúen simultáneamente como primary del mismo chunk.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

El master otorga a una réplica un permiso temporal para actuar como primary de un chunk.

Mientras el lease está vigente, esa réplica define el orden de las mutaciones. Si el master pierde contacto con ella, debe tener cuidado antes de elegir una nueva primary, porque la primary vieja podría seguir viva pero aislada.

El lease limita temporalmente la autoridad de la primary y ayuda a evitar split brain.

</details>

---

## 42. Raft

**Pregunta:**  
En Raft, los heartbeats son mensajes especiales totalmente distintos de `AppendEntries`.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

En Raft, un heartbeat puede verse como un `AppendEntries` vacío.

No necesariamente lleva nuevas entradas de log, pero sirve para indicar que el líder sigue vivo, evitar que los followers inicien elecciones y comunicar información como el `commitIndex`.

</details>

---

## 43. Raft

**Pregunta:**  
En un clúster Raft de 5 nodos, una partición de 2 nodos puede seguir aceptando escrituras si uno de esos nodos era el líder antes de la partición.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Con 5 nodos, la mayoría es 3.

Aunque el líder viejo quede en la partición de 2 nodos, no puede comitear nuevas entradas porque no consigue mayoría. Puede intentar agregar entradas localmente o replicarlas al follower que todavía ve, pero no debe responder `OK` al cliente.

La partición con mayoría es la única que puede elegir líder y avanzar de forma segura.

</details>

---

## 44. Raft

**Pregunta:**  
Raft garantiza que toda entrada escrita alguna vez en algún log sobrevivirá a futuras elecciones de líder.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Raft no garantiza que sobrevivan todas las entradas escritas en algún nodo.

Una entrada no comiteada puede sobrevivir si el nodo que la tiene se convierte en líder y logra replicarla. Pero también puede ser descartada si se elige como líder un nodo que no la tenía.

Lo que Raft garantiza es que no se pierden entradas ya comiteadas.

</details>

---

## 45. Raft

**Pregunta:**  
Si dos logs tienen una entrada con el mismo índice y el mismo término, Raft asume que esa entrada contiene el mismo comando.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Esta es la Log Matching Property.

La combinación `(index, term)` permite detectar puntos de coincidencia entre logs. Si un follower tiene una entrada con el mismo índice y término que el líder, entonces se considera que los logs coinciden hasta ese punto.

Esto permite que el líder borre sufijos conflictivos y sincronice followers atrasados o divergentes.

</details>

---

## 46. ZooKeeper

**Pregunta:**  
ZooKeeper está pensado principalmente para almacenar grandes archivos de datos, de manera similar a GFS.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

ZooKeeper no está diseñado como almacenamiento masivo de archivos.

Su modelo de datos parece un filesystem jerárquico, con paths y znodes, pero está pensado para coordinación: configuración, locks, elección de líder, watches y metadata pequeña.

Para archivos grandes o procesamiento batch, sistemas como GFS encajan mucho mejor.

</details>

---

## 47. ZooKeeper

**Pregunta:**  
La operación `sync` en ZooKeeper puede usarse para acercar una lectura a una semántica más fuerte, haciendo que el servidor se ponga al día antes de responder.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

ZooKeeper permite lecturas desde followers para mejorar throughput, pero esas lecturas pueden estar atrasadas.

La operación `sync` sirve para que el cliente fuerce una sincronización con el líder antes de leer. Esto permite obtener una lectura más actualizada que una lectura local común.

No significa que todas las lecturas de ZooKeeper sean linealizables por defecto, sino que existe un mecanismo para pedir mayor frescura cuando hace falta.

</details>

---

## 48. Caches

**Pregunta:**  
Un cache con hit ratio alto reduce la carga sobre la base de datos porque muchas lecturas se resuelven sin consultar el storage persistente.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

El cache no solo mejora la latencia individual de ciertas consultas, sino que también protege a la base de datos.

Si el 98% de las lecturas se responden desde cache, solo el 2% llega a la base. Esto reduce la presión sobre conexiones, CPU, locks, índices y disco.

Por eso los caches se usan como mecanismo de escalabilidad, no solamente como optimización cosmética.

</details>

---

## 49. Caches

**Pregunta:**  
El problema de thundering herd ocurre cuando muchos clientes intentan rellenar simultáneamente la misma clave de cache luego de un miss.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Si una clave muy popular desaparece del cache, muchos clientes pueden detectar miss al mismo tiempo y consultar la base para reconstruir el mismo valor.

Esto puede generar una avalancha de consultas repetidas a la base de datos. Facebook usa leases en Memcache para limitar cuántos clientes tienen permiso para rellenar una clave en un intervalo determinado.

</details>

---

## 50. Dynamo

**Pregunta:**  
En Dynamo, el uso de consistent hashing busca evitar que agregar o quitar un nodo obligue a redistribuir casi todas las claves del sistema.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Con hashing simple del tipo:

```text
server = hash(key) mod N
```

si cambia `N`, muchísimas claves cambian de servidor.

Consistent hashing organiza servidores y claves en un anillo. Al agregar un nodo, solo se mueve una porción del rango de claves, no todo el dataset. Esto permite escalar de forma incremental.

</details>

---

## 51. Dynamo

**Pregunta:**  
En Dynamo, los nodos virtuales sirven para mejorar el balance de carga y facilitar la redistribución de datos cuando cambia la membresía del clúster.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Cada máquina física puede aparecer muchas veces en el anillo como varios nodos virtuales.

Esto reduce la varianza en la cantidad de claves asignadas a cada máquina. También permite que, al agregar un nodo nuevo, reciba pequeñas porciones de muchos nodos existentes en lugar de una única porción grande.

</details>

---

## 52. Dynamo

**Pregunta:**  
Los Merkle trees permiten comparar réplicas de manera eficiente porque, si los hashes de un subárbol coinciden, no hace falta comparar todas las claves de ese rango.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Un Merkle tree resume rangos de datos mediante hashes.

Si dos réplicas tienen la misma raíz para un rango, se asume que ese rango coincide. Si difieren, se baja en el árbol hasta encontrar qué subrangos o claves son distintas.

Esto permite reparar divergencias sin transferir toda la tabla.

</details>

---

## 53. Gossip

**Pregunta:**  
Un protocolo de gossip requiere que exista un coordinador central que le envíe periódicamente a todos los nodos la visión correcta del clúster.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Gossip es una estrategia descentralizada.

Cada nodo mantiene una visión local del estado del clúster y periódicamente intercambia información con otros nodos. La información se propaga como un rumor.

No todos los nodos conocen inmediatamente la misma información, pero el sistema converge eventualmente.

</details>

---

## 54. Transacciones distribuidas

**Pregunta:**  
La serializabilidad exige que una ejecución concurrente de transacciones sea equivalente a alguna ejecución serial de esas mismas transacciones.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Una ejecución serializable puede tener operaciones intercaladas internamente, pero el resultado debe ser equivalente a ejecutar las transacciones una por una en algún orden.

Por ejemplo, si una transacción transfiere saldo entre dos cuentas y otra lee ambas cuentas, la lectura no debería observar un estado intermedio imposible bajo cualquier orden serial.

</details>

---

## 55. Two-Phase Locking

**Pregunta:**  
En Two-Phase Locking, una transacción puede liberar un lock y luego pedir otro lock nuevo, siempre que todavía no haya hecho commit.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

La regla central de 2PL es que hay dos fases:

1. fase expansiva: se adquieren locks;
2. fase contractiva: se liberan locks.

Una vez que la transacción libera un lock, ya no puede pedir locks nuevos. Esta restricción es la que permite garantizar serializabilidad.

En strict 2PL, los locks se mantienen hasta commit o abort.

</details>

---

## 56. Two-Phase Commit

**Pregunta:**  
Los mensajes `commit` y `abort` en 2PC conviene que sean idempotentes, porque pueden reenviarse durante recuperación o ante pérdida de respuestas.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

En sistemas distribuidos, los mensajes pueden perderse o duplicarse.

Si el coordinador no recibe respuesta a un `commit`, puede reenviarlo. El participante debe poder recibir el mismo `commit(tx_id)` más de una vez sin aplicar efectos duplicados ni romper su estado.

Lo mismo vale para `abort`.

</details>

---

## 57. Bitcoin

**Pregunta:**  
Bitcoin resuelve el problema de double spend imponiendo un líder fijo que ordena todas las transacciones, de manera similar a Raft.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Bitcoin no tiene un líder fijo.

El orden de las transacciones surge de la cadena de bloques aceptada por la red. Los mineros compiten mediante Proof of Work para proponer bloques. Cuando aparece un bloque válido, otros nodos lo verifican y construyen encima.

Esto reemplaza al líder estable de Raft por una elección probabilística basada en poder de cómputo.

</details>

---

## 58. Bitcoin

**Pregunta:**  
Modificar una transacción vieja en Bitcoin cambia el hash del bloque que la contiene y obliga a rehacer el trabajo de los bloques posteriores si se quisiera mantener una cadena válida.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Cada bloque contiene el hash del bloque anterior.

Si se modifica una transacción vieja, cambia el hash de su bloque. Como el bloque siguiente incluye ese hash, también queda invalidado, y así sucesivamente.

Para reescribir la historia, un atacante tendría que rehacer el Proof of Work de ese bloque y de todos los bloques posteriores, y además alcanzar o superar a la cadena honesta.

</details>

---

## 59. Spanner

**Pregunta:**  
MVCC permite que una lectura en Spanner obtenga una versión histórica consistente de una clave, en lugar de bloquearse necesariamente esperando la última escritura.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

MVCC guarda múltiples versiones de una clave, cada una asociada a un timestamp.

Si una transacción lee en timestamp `T`, obtiene la versión más reciente con timestamp menor o igual a `T`.

Esto permite lecturas de snapshot sin locks, siempre que la réplica tenga información suficiente para saber que está actualizada hasta ese timestamp.

</details>

---

## 60. Spanner

**Pregunta:**  
El problema de usar relojes físicos en sistemas distribuidos es que todos los relojes están perfectamente sincronizados, pero leerlos es demasiado lento.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El problema es justamente que los relojes físicos no están perfectamente sincronizados.

Cada máquina puede tener drift, latencia al consultar fuentes de tiempo y diferencias respecto de otras máquinas. Spanner usa TrueTime para trabajar con intervalos de incertidumbre, no con un instante exacto perfecto.

Esa incertidumbre debe ser considerada para garantizar consistencia externa.

</details>

---

## 61. Mensajería

**Pregunta:**  
Una dead-letter queue sirve para almacenar mensajes que fallaron repetidamente y no pudieron ser procesados correctamente por los consumidores.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

Un mensaje problemático, o poison pill, puede hacer fallar a todos los consumidores que intentan procesarlo.

Para evitar que vuelva infinitamente a la cola principal, se puede configurar una cantidad máxima de intentos. Al superar ese límite, el mensaje se mueve a una dead-letter queue.

Esto permite inspeccionarlo, corregir el bug o reprocesarlo manualmente.

</details>

---

## 62. Mensajería

**Pregunta:**  
El patrón SNS + SQS permite que cada consumidor tenga su propia cola y procese eventos a su propio ritmo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

SNS puede hacer fan-out del evento a varias colas SQS.

Cada consumidor lee desde su propia cola. Si un consumidor está caído, sus mensajes se acumulan sin afectar necesariamente a los demás. Esto desacopla la publicación del evento de la disponibilidad y velocidad de cada servicio consumidor.

</details>

---

## 63. Stream processing

**Pregunta:**  
Un stream es un dataset cerrado, por lo que el sistema puede esperar a tener todos los eventos antes de empezar a procesar.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

Un stream es potencialmente infinito o unbounded.

Los eventos siguen llegando continuamente. Por eso, los sistemas de stream processing procesan eventos a medida que aparecen y usan mecanismos como ventanas temporales, watermarks y estado incremental.

Esperar a “tener todo” convertiría el problema en batch, pero en streaming ese final no existe.

</details>

---

## 64. Stream processing

**Pregunta:**  
Un join entre dos streams suele requerir mantener estado temporal, porque puede llegar primero el evento de un stream y más tarde el evento relacionado del otro.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es verdadera.**

En un stream-stream join, los eventos relacionados pueden llegar desordenados o con distinta latencia.

Por ejemplo, para correlacionar una push notification con una apertura de app dentro de los 5 minutos siguientes, el sistema debe recordar temporalmente eventos de push y esperar posibles accesos relacionados.

El estado se puede descartar cuando el watermark o la ventana indique que ya no deberían llegar eventos relevantes.

</details>

---

## 65. Stream processing

**Pregunta:**  
El microbatching elimina por completo el problema de eventos tardíos, porque cada lote pequeño representa exactamente todos los eventos de ese intervalo.

<details>
<summary><strong>Ver respuesta</strong></summary>

**La afirmación es falsa.**

El microbatching reduce la latencia frente al batch tradicional, pero no elimina los eventos tardíos.

Un evento con `event_time` de las 8:00:05 puede llegar después de que el microbatch correspondiente ya fue procesado. El sistema todavía debe decidir si ignora el evento, emite una corrección, reabre la ventana o usa watermarks.

</details>

