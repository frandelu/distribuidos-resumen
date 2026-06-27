# Clase 4 — Sistemas de storage: replicación y sharding

## [Apuntes de Emma](https://drive.google.com/file/d/1FcaEAqVGvVee-Z4JJ6XRvsPyHnz8Rsvt/view)

## 1. Sistemas de storage

Los sistemas de *storage* son sistemas cuyo objetivo principal es almacenar y recuperar datos. En esta categoría entran, entre otros, los **file systems** y las **bases de datos**.

A diferencia de los sistemas orientados a cómputo, como MapReduce, los sistemas de storage son más difíciles de distribuir porque su esencia es mantener **estado**. Un sistema de cómputo puede muchas veces dividir el trabajo en partes independientes y reintentar una parte si falla. En cambio, un sistema de almacenamiento no puede simplemente “reintentar” sin pensar qué pasó con los datos ya escritos, qué réplicas los recibieron, en qué orden se aplicaron las operaciones o si una lectura puede ver una versión vieja.

Los objetivos principales de un sistema de storage distribuido son:

- **Escalabilidad horizontal**: poder agregar más máquinas para aumentar capacidad de almacenamiento, throughput o capacidad de procesamiento.
- **Tolerancia a fallas / alta disponibilidad**: que el sistema siga funcionando aunque fallen nodos individuales, discos o enlaces de red.
- **Durabilidad de datos**: que los datos no se pierdan aunque una máquina deje de funcionar.

En una arquitectura típica, por ejemplo una aplicación web con varios servidores y una base de datos, escalar los servidores web suele ser relativamente simple: se agregan más instancias detrás de un load balancer. La parte difícil suele ser la base de datos, porque no alcanza con duplicarla sin definir cómo se reparten los datos, cómo se sincronizan las copias y qué garantías de consistencia se ofrecen.

## 2. Diferencia con MapReduce

MapReduce servía para resolver problemas de cómputo masivo. Su estrategia principal era:

1. Particionar el problema en muchos fragmentos.
2. Ejecutar esos fragmentos en paralelo.
3. Reintentar las tareas que fallaban.

Eso funcionaba bien porque los `map` y `reduce` debían ser **deterministas** y, en general, **stateless**. Si una tarea fallaba, se podía volver a ejecutar con el mismo input y producir el mismo output.

En storage no ocurre lo mismo. Un sistema de storage existe justamente porque tiene estado. Una base de datos, un sistema de archivos o un log persistente guardan información que cambia con el tiempo. Por eso, la tolerancia a fallas exige técnicas más cuidadosas: no basta con reejecutar, sino que hay que saber qué operaciones fueron aplicadas, en qué orden y en qué réplicas.

## 3. Técnicas principales

Para construir sistemas de storage distribuidos aparecen dos técnicas fundamentales:

1. **Particionado o sharding**.
2. **Replicación**.

En la práctica suelen combinarse. Un sistema puede dividir los datos en shards y, a la vez, replicar cada shard en varias máquinas.

## 4. Sharding o particionado

El sharding consiste en dividir un dataset grande en varias partes y ubicar cada parte en una máquina o grupo de máquinas diferente.

Por ejemplo, una tabla lógica con millones de registros puede dividirse en varios fragmentos. Cada fragmento se guarda en un nodo distinto. De esa forma, el sistema completo puede almacenar más datos que una sola máquina y puede responder más consultas en paralelo.

Para que esto funcione se necesita un **mecanismo de nombres** o de ubicación. Es decir, alguna forma de mapear cada dato o registro a su ubicación física.

Ese mapeo puede implementarse de distintas maneras:

- Una tabla de metadatos que indique dónde está cada fragmento.
- Una función de hash sobre una clave.
- Consistent hashing.
- Un master o coordinador que conozca la ubicación de cada bloque o shard.

La idea abstracta es siempre la misma:

```text
Dato / registro / clave  --->  ubicación física
```

### Beneficios del sharding

El sharding aporta varias ventajas.

Primero, permite **escalabilidad horizontal**. Si una máquina ya no alcanza, se pueden agregar más máquinas y repartir los datos.

Segundo, permite **fallas parciales**. Si falla un shard, no necesariamente cae todo el sistema. Puede degradarse una parte del servicio, pero el resto puede seguir funcionando.

Tercero, mejora el **paralelismo de acceso**. Si los datos están distribuidos, distintos clientes o procesos pueden leer distintos shards al mismo tiempo. Esto aumenta el throughput del sistema.

Sin embargo, el sharding no resuelve todos los problemas. Si una consulta necesita datos de varios shards, aparecen dificultades adicionales. También se complica hacer joins, transacciones distribuidas o mantener invariantes globales.

## 5. Replicación

La replicación consiste en guardar copias del mismo dato en varias máquinas.

Mientras que el sharding divide el dataset, la replicación duplica información. Por ejemplo, el dato `x = 1` puede existir en tres nodos distintos. Si un nodo falla, todavía hay otras copias disponibles.

La replicación sirve principalmente para:

- Mejorar disponibilidad.
- Evitar pérdida de datos.
- Permitir lecturas desde varias réplicas.
- Tolerar fallas de nodos o discos.
- Ubicar copias en distintos racks, zonas o datacenters.

El problema central de la replicación no es copiar el dato una vez, sino mantener las réplicas **sincronizadas** a medida que llegan escrituras.

## 6. Tipos de fallas

### Fallas fail-stop

Una falla *fail-stop* es aquella en la que un nodo deja de responder. Desde afuera no siempre se sabe si:

- El nodo murió.
- La red se cortó.
- El request nunca llegó.
- El request llegó, se ejecutó, pero la respuesta se perdió.

Para el cliente o para otro nodo, todos esos casos se manifiestan parecido: no hay respuesta.

Este es el tipo de falla principal que suelen asumir muchos algoritmos de replicación, como Raft. El algoritmo supone que los nodos que siguen vivos ejecutan correctamente el protocolo.

### Fallas bizantinas

Una falla bizantina ocurre cuando un nodo se comporta de forma incorrecta, arbitraria o maliciosa. Por ejemplo, un nodo podría decir que guardó `x = 1`, pero en realidad guardar `x = 2`, o enviar respuestas contradictorias.

Los algoritmos vistos en esta parte no resuelven fallas bizantinas. Para eso se necesitan protocolos más complejos. Raft, por ejemplo, no tolera que un nodo implemente mal el algoritmo o mienta: asume que los nodos correctos siguen el protocolo.

## 7. Mecanismos de replicación

Hay dos grandes enfoques para replicar estado:

1. **State transfer / snapshot**.
2. **Replicated State Machine**.

## 8. State transfer o snapshot

El enfoque de *state transfer* consiste en copiar el estado completo de una máquina a otra.

Por ejemplo, si un nodo tiene una base de datos, se toma una copia del contenido y se transfiere a otro nodo. Esto es conceptualmente simple, pero tiene problemas importantes:

- Puede ser lento si el estado ocupa muchos GB o TB.
- Mientras se copia, el sistema puede seguir recibiendo escrituras.
- Si no se coordina bien, el snapshot puede quedar inconsistente.

Una forma simple de evitar inconsistencias sería “parar el mundo”: dejar de aceptar escrituras mientras se copia el estado. Pero eso afecta disponibilidad y no escala bien.

Por eso, los snapshots suelen usarse combinados con otros mecanismos. Por ejemplo, se puede cargar un snapshot reciente en una réplica nueva y luego aplicarle las operaciones posteriores desde un log hasta ponerla al día.

## 9. Replicated State Machine

El enfoque de **máquina de estados replicada** parte de una idea simple:

Si dos réplicas parten del mismo estado inicial y reciben las mismas operaciones en el mismo orden, entonces terminan en el mismo estado final, siempre que las operaciones sean deterministas.

```text
Estado inicial S0
+ operación 1
+ operación 2
+ operación 3
= mismo estado final en todas las réplicas
```

Los requisitos son:

- Todas las réplicas parten del mismo estado inicial.
- Todas reciben las mismas operaciones.
- Todas las aplican en el mismo orden.
- Las operaciones son deterministas.

El problema difícil es garantizar el **mismo orden** en todas las réplicas.

Si hay un único cliente, el orden puede parecer fácil. Pero en un sistema real hay múltiples clientes enviando operaciones concurrentemente. Un nodo puede recibir primero `O1` y luego `O2`, mientras que otro puede recibir primero `O2` y luego `O1`. Si esas operaciones no conmutan, las réplicas pueden terminar en estados distintos.

Garantizar ese orden común lleva al problema de **consenso**.

## 10. El log como abstracción central

Para resolver la replicación aparece una abstracción fundamental: el **log**.

Un log es una secuencia append-only de operaciones. Cada operación se agrega al final y ocupa una posición. Esa posición define el orden total de las operaciones.

```text
[ O1 ][ O2 ][ O3 ][ O4 ][ O5 ]
```

Si todas las réplicas aplican el mismo log en el mismo orden, se mantienen sincronizadas.

El log combina dos cosas:

- Las operaciones que modifican el estado.
- El orden en que esas operaciones deben aplicarse.

Por eso, resolver la replicación muchas veces equivale a resolver cómo construir y mantener un log replicado.

## 11. El log en bases de datos

Las bases de datos relacionales ya usan logs internamente. Un ejemplo típico es el **Write-Ahead Log** o **WAL**.

El WAL registra operaciones antes de que los cambios queden definitivamente reflejados en las páginas de datos. Esto permite recuperar la base ante fallas y sostener propiedades transaccionales.

Si una transacción queda incompleta, el log permite revertirla. Si una transacción fue confirmada, el log permite rehacer sus cambios si todavía no estaban completamente persistidos.

En sistemas replicados, ese mismo log puede emitirse a réplicas. Las réplicas consumen las operaciones en el mismo orden y actualizan su estado.

## 12. El log en Raft

Raft puede entenderse como un algoritmo para mantener un log replicado.

Una aplicación que usa Raft suele tener dos capas:

```text
Aplicación / storage
--------------------
Raft / log replicado
```

Cuando un cliente quiere escribir, la operación se agrega al log de Raft. Raft replica esa entrada en otros nodos. Cuando la entrada está suficientemente replicada, se considera comprometida y se aplica a la máquina de estados de la aplicación.

El log también sirve como forma de versionado. Si una réplica aplicó hasta la entrada 10 y otra hasta la entrada 11, se sabe que la primera está atrasada respecto de la segunda.

## 13. Active-active con log externo

Una forma de resolver la replicación es poner el log por fuera de las réplicas.

Por ejemplo, un sistema como Kafka puede actuar como log ordenado. Los clientes escriben operaciones en Kafka y varias aplicaciones consumen ese log en el mismo orden.

```text
Cliente -> Log externo -> Réplica A
                     -> Réplica B
                     -> Réplica C
```

En este esquema, las réplicas son activas y consumen el mismo flujo de operaciones. El problema del orden se delega al sistema de log.

Esta estrategia se parece a un esquema *active-active*: varias instancias procesan el mismo log y construyen vistas o estados derivados.

El punto importante es que el problema no desaparece: queda encapsulado en el sistema que implementa el log. Kafka, por ejemplo, también necesita mecanismos propios para mantener orden, particiones, durabilidad y replicación.

## 14. Primary-backup

El esquema **primary-backup** es una forma muy común de replicación.

Hay un nodo especial llamado **primary**. Las escrituras van al primary. El primary aplica la operación localmente y la replica hacia los backups.

```text
Cliente --write--> Primary --> Backup 1
                         --> Backup 2
                         --> Backup 3
```

Las lecturas pueden hacerse desde el primary o desde las réplicas, dependiendo de las garantías que se quieran ofrecer.

Este esquema también aparece con otros nombres:

- Master / slaves.
- Writer / read replicas.
- Leader / followers.
- Primary / backups.

La ventaja es que el primary decide unilateralmente el orden de las operaciones. Como una sola máquina ordena las escrituras, se evita el problema de consenso para cada operación en la versión más simple del sistema.

### Consistencia eventual

Si el primary responde al cliente antes de que todas las réplicas estén actualizadas, las réplicas pueden quedar atrasadas.

Esto permite más performance y disponibilidad, pero introduce **consistencia eventual**. Si un cliente escribe y luego lee desde una réplica atrasada, puede ver una versión vieja.

En términos más formales, ese sistema no es necesariamente **linealizable**.

Para obtener lecturas fuertemente consistentes, una estrategia simple es leer siempre desde el primary. Otra posibilidad es coordinar más estrictamente con las réplicas, pero eso aumenta la latencia y reduce disponibilidad.

### Falla del primary

Si falla un backup, el sistema puede seguir funcionando. Luego se puede crear una nueva réplica, cargarle un snapshot y ponerla al día aplicando el log restante.

Si falla el primary, la situación es más delicada. Puede haber escrituras aplicadas localmente en el primary que todavía no fueron replicadas. Según el protocolo, puede haber pérdida de datos o necesitarse una elección de nuevo líder.

Raft puede verse como una versión más sofisticada de primary-backup, donde el primary o líder no está fijo: se elige automáticamente y el algoritmo garantiza que no se pierdan entradas comprometidas.

## 15. Change Data Capture: Postgres + Debezium + Kafka

Una configuración práctica muy usada consiste en tomar el WAL de una base relacional, como Postgres, y publicarlo en un log externo como Kafka.

Un componente como Debezium puede conectarse a Postgres como si fuera una réplica, leer el WAL y publicar los cambios en Kafka.

Luego distintos consumidores pueden procesar esos cambios:

- Un data warehouse.
- Un motor de búsqueda de texto, como Elasticsearch u OpenSearch.
- Microservicios que necesitan enterarse de altas, bajas o modificaciones.
- Sistemas analíticos o dashboards.

```text
Postgres -> WAL -> Debezium -> Kafka -> Data warehouse
                                      -> Buscador de texto
                                      -> Microservicio
```

Esto permite construir vistas derivadas del mismo estado. Cada consumidor recibe las operaciones en orden y actualiza su propia representación.

La base principal define el orden; Kafka distribuye ese orden; los consumidores materializan distintas vistas.

## 16. Chain replication

**Chain replication** es otra forma de replicación. Los nodos se organizan como una cadena.

```text
Head -> Nodo intermedio -> Tail
```

Las escrituras entran por el **head**. Cada nodo aplica la operación y la reenvía al siguiente. El último nodo, el **tail**, confirma al cliente.

```text
Cliente --write--> Head -> Nodo intermedio -> Tail --ack--> Cliente
```

Las lecturas, en la versión básica, se hacen desde el tail.

Esto tiene una propiedad importante: si el tail responde, significa que la operación llegó hasta el final de la cadena. Por lo tanto, las lecturas desde el tail ven el estado comprometido.

### Ventajas

- El head no tiene que enviar cada operación a todas las réplicas, solo al siguiente nodo.
- Puede ofrecer consistencia fuerte si las lecturas se hacen desde el tail.
- La estructura es conceptualmente simple.

### Particularidad

No es un RPC tradicional, porque el request de escritura entra por un nodo y el ack puede volver desde otro. El cliente necesita conocer tanto el head como el tail o usar una librería que oculte esa complejidad.

## 17. Fallas en chain replication

### Falla del head

Si falla el head, el siguiente nodo puede ser promovido como nuevo head. Los clientes deben ser reconfigurados para escribir en ese nuevo nodo.

Si una escritura había llegado solo al head y este murió antes de reenviarla, el cliente no recibirá ack y podrá reintentar. Si la escritura ya había sido reenviada, continuará propagándose por la cadena.

### Falla del tail

Si falla el tail, el nodo anterior se convierte en el nuevo tail. Como ese nodo ya recibió las operaciones previas, puede responder lecturas y acks a partir de la nueva configuración.

### Falla de un nodo intermedio

Si falla un nodo del medio, se lo saltea y se reconecta la cadena. El head puede necesitar reenviar operaciones que no llegaron al tail. Para eso debe conservar cierta memoria de operaciones recientes.

### Agregar un nuevo nodo

Para agregar un nodo, una estrategia común es ubicarlo al final de la cadena. Primero se le envía un snapshot del estado y luego se le aplican las operaciones del log hasta que se ponga al día. Recién entonces puede pasar a formar parte activa de la cadena.

## 18. Split brain

Uno de los problemas más peligrosos en sistemas replicados es el **split brain**.

Ocurre cuando una partición de red divide al sistema en dos grupos y ambos creen que pueden seguir funcionando de forma independiente.

```text
DC1                 DC2
Nodos A, B   ||    Nodos C, D
Cliente 1          Cliente 2
```

Si cada lado se autoconfigura y acepta escrituras, el sistema deja de tener una única historia. Aparecen dos estados divergentes. Cuando la red vuelve, reconciliar esos estados puede ser muy difícil o incluso requerir intervención manual.

Esto puede tener consecuencias graves, por ejemplo en sistemas financieros, pedidos de compra o inventarios.

Por eso, muchos sistemas prefieren sacrificar disponibilidad en una partición antes que permitir dos cerebros activos.

## 19. Servicio de configuración

Para evitar split brain, muchos sistemas usan un **servicio de configuración**.

Este servicio decide:

- Qué nodos forman parte del sistema.
- Quién es el head o tail en una cadena.
- Quién es el primary.
- Qué nodos están vivos.
- Cómo reconfigurar ante fallas.

El servicio de configuración pertenece al plano de control: no necesariamente pasan por él los datos de usuario, pero sí la metadata de cómo está organizado el sistema.

Una versión simple es tener una única máquina que decide la configuración. Esto evita split brain porque, ante una partición, el servicio queda de un lado o del otro. Solo el lado que puede hablar con el servicio sigue avanzando.

Una versión más robusta es implementar el servicio de configuración con un sistema tolerante a fallas, por ejemplo usando Raft. En ese caso, una mayoría de nodos decide cuál es la configuración válida.

Sistemas como ZooKeeper o etcd suelen usarse para este tipo de coordinación.

## 20. Relación con GFS y Raft

Google File System combina sharding y replicación. Los archivos se dividen en chunks y cada chunk se replica en varias máquinas. Además, usa un coordinador centralizado para mantener metadata y decidir ubicaciones.

Raft resuelve el problema de replicar un log de manera consistente y tolerante a fallas. En vez de depender de un primary fijo, el sistema elige un líder. El líder ordena las operaciones y las replica. Si falla, se elige otro líder sin perder las entradas comprometidas.

Estos temas aparecen como respuestas concretas a los problemas generales introducidos acá:

- Cómo dividir datos grandes.
- Cómo replicar datos.
- Cómo mantener el orden de escrituras.
- Cómo detectar fallas.
- Cómo evitar split brain.
- Cómo decidir qué nodo puede aceptar escrituras.

## 21. Comparación rápida

| Técnica | Objetivo principal | Idea central | Problema principal |
|---|---|---|---|
| Sharding | Escalar capacidad y throughput | Dividir el dataset en partes | Ubicar datos, joins, transacciones entre shards |
| Replicación | Tolerar fallas y aumentar disponibilidad | Copiar el mismo dato en varios nodos | Mantener copias sincronizadas |
| State transfer | Copiar estado completo | Snapshot de una réplica a otra | Lento, difícil con escrituras concurrentes |
| Replicated State Machine | Replicar operaciones | Mismo estado inicial + mismas operaciones + mismo orden | Conseguir orden total |
| Primary-backup | Ordenar escrituras fácilmente | Un primary decide el orden | Falla del primary, consistencia eventual |
| Active-active con log externo | Varias réplicas consumen un log | El log externo define el orden | El problema se delega al sistema de log |
| Chain replication | Replicar por cadena | Escrituras por head, lecturas/ack por tail | Reconfiguración y detección de fallas |
| Config service | Evitar split brain | Un servicio decide la configuración válida | El servicio también debe ser confiable |

## 22. Ideas clave

Un sistema de storage distribuido es difícil porque combina estado, concurrencia, fallas parciales y comunicación por red.

El sharding permite escalar dividiendo los datos, pero no garantiza por sí solo durabilidad ni consistencia.

La replicación mejora disponibilidad y durabilidad, pero exige resolver cómo mantener todas las copias sincronizadas.

La idea de máquina de estados replicada reduce el problema a aplicar las mismas operaciones en el mismo orden en todas las réplicas.

El log es la abstracción central para representar ese orden.

Primary-backup simplifica el orden usando un nodo especial, pero puede ofrecer solo consistencia eventual si las lecturas se hacen desde réplicas atrasadas.

Chain replication ofrece una forma elegante de obtener consistencia fuerte leyendo desde el tail, pero necesita un buen mecanismo de reconfiguración.

Split brain es uno de los problemas más peligrosos: el sistema puede dividirse en dos historias incompatibles.

Un servicio de configuración, muchas veces implementado con consenso, ayuda a decidir qué parte del sistema puede seguir operando ante fallas o particiones.
