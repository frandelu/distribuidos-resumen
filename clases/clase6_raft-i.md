# Clase 6 — Raft I

## [Apuntes de Emma](https://drive.google.com/file/d/1OVpJj8eza6oALuboZzP67MKPHMByy1Nu/view)

## Objetivo de Raft

Raft es un algoritmo de consenso pensado para resolver el problema de la **replicación de máquinas de estado** (*state machine replication*).

La idea central es que varias máquinas reciban las mismas operaciones, en el mismo orden, para que todas terminen en el mismo estado. Si la máquina de estado es determinística, entonces replicar el orden de operaciones alcanza para replicar el estado.

Raft se usa como una capa inferior sobre la cual puede construirse un sistema de storage, una base de datos, un key-value store u otro servicio distribuido. La aplicación no tiene que resolver por sí misma el consenso: le entrega comandos a Raft, y Raft se encarga de replicarlos de manera segura.

---

## Problemas que motivan Raft

### Escrituras no atómicas en GFS

En Google File System podían quedar réplicas inconsistentes ante fallas durante una escritura.

Ejemplo:

1. Un cliente intenta escribir un registro.
2. El primary replica el dato hacia las secondary replicas.
3. Una réplica recibe el dato, pero otra no.
4. La operación falla desde el punto de vista del cliente.
5. El cliente reintenta.
6. El dato puede terminar duplicado en algunas réplicas y ausente en otras.

Eso genera dos problemas:

- **Huecos o regiones inválidas** en alguna réplica.
- **Duplicados** en otras réplicas.

GFS no resolvía esto internamente. Le trasladaba la responsabilidad al cliente:

- Para duplicados: usar identificadores únicos.
- Para datos corruptos o incompletos: usar checksums o validaciones de formato.

Esto no es consistencia eventual. Es una inconsistencia que puede quedar permanentemente si nadie la corrige.

Raft apunta a evitar este tipo de problema con **atomic write** a nivel distribuido: una operación queda aceptada de forma segura o no queda aceptada.

---

## Atomic write distribuido

Una escritura atómica distribuida significa que una operación ocurre de forma segura en el sistema o no ocurre.

No alcanza con que un nodo la escriba localmente. Para poder decirle `OK` al cliente, la operación tiene que quedar replicada de forma tal que el sistema pueda recuperarla aunque haya fallas toleradas.

En Raft, una operación individual se considera segura cuando queda persistida en una **mayoría** de nodos. Recién ahí se puede aplicar a la máquina de estado y responder `OK` al cliente.

Esto da una garantía fuerte:

> Si el cliente recibió `OK`, la operación no se pierde mientras no fallen más nodos que los que el sistema fue diseñado para tolerar.

---

## Consistencia fuerte y linealizabilidad

Raft busca proveer **consistencia fuerte**, formalmente cercana a la idea de **linealizabilidad**.

La propiedad esperada es:

1. Un cliente escribe un valor.
2. El sistema responde `OK`.
3. A partir de ese momento, las lecturas posteriores deben observar ese valor o uno más nuevo.

Esto contrasta con consistencia eventual, donde una escritura puede responder `OK` y luego una lectura posterior todavía puede ver un valor viejo durante un tiempo.

Raft, en su versión base, prioriza consistencia sobre disponibilidad. Ante una partición de red, la parte que no puede garantizar seguridad debe dejar de aceptar operaciones, incluso si eso implica responder error.

---

## Single Point of Failure y split brain

Sistemas como GFS y MapReduce tenían un nodo central:

- En GFS: el **master**.
- En MapReduce: el **coordinador**.

Ese nodo central simplifica el diseño, pero introduce un **single point of failure**. Si ese nodo cae, todo el sistema queda afectado.

La solución natural es replicar ese nodo central. Pero replicarlo sin cuidado puede producir **split brain**.

### Split brain

El split brain ocurre cuando una red particionada hace que dos grupos de nodos crean estar actuando como sistema válido e independiente.

Ejemplo:

- En un lado de la partición, un cliente escribe `x = 15`.
- En el otro lado de la partición, otro cliente escribe `x = 20`.
- Ambos lados aceptan escrituras.
- El sistema termina con dos verdades incompatibles.

Una vez que eso pasa, no hay una reconciliación automática obvia. En sistemas críticos, como financieros, esto es especialmente grave.

El objetivo de Raft es evitar a toda costa el split brain. Para eso usa la noción de **mayoría**.

---

## Modelo de fallas

Raft tolera fallas de tipo **fail-stop**:

- Un nodo deja de responder.
- Una red se particiona.
- Un mensaje no llega.
- Una máquina cae y luego vuelve.

Desde el punto de vista del resto del sistema, no siempre es posible distinguir si murió el nodo o si se cortó la red. En ambos casos, simplemente dejó de responder.

Raft **no** tolera fallas bizantinas. Eso significa que no está diseñado para nodos maliciosos, comprometidos o incorrectamente implementados que responden cualquier cosa.

Todos los nodos deben seguir correctamente el protocolo. Si un nodo vota dos veces cuando no debe, acepta entradas inválidas o viola las reglas del algoritmo, las garantías se rompen.

---

## Mayorías

Una **mayoría** es la mitad + 1 de los nodos.

Para `N` nodos:

```text
M = floor(N / 2) + 1
```

Ejemplos:

| N | Mayoría M |
|---|-----------|
| 3 | 2 |
| 5 | 3 |
| 7 | 4 |

En sistemas distribuidos se prefieren números impares de nodos porque permiten que, ante una partición, exista una partición con mayoría y otra sin mayoría.

Ejemplo con `N = 5`:

- Si la red se divide en `3 + 2`, el grupo de 3 puede seguir.
- Si se divide en `4 + 1`, el grupo de 4 puede seguir.
- El grupo minoritario no debe aceptar escrituras ni elegir líder.

El valor de `N` es parte de la configuración del sistema. No cambia automáticamente cuando un nodo cae. Si un sistema fue configurado con `N = 5`, sigue siendo `N = 5` aunque momentáneamente haya dos nodos caídos.

---

## Tolerancia a fallas

Con `N = 2F + 1`, el sistema puede tolerar `F` fallas.

| N | Mayoría | Fallas toleradas |
|---|---------|------------------|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

Cuanto más fallas se quieren tolerar, más nodos hacen falta y más comunicación se requiere.

En la práctica, configuraciones comunes son:

- `N = 3`: tolera 1 falla.
- `N = 5`: tolera 2 fallas.

---

## Quórums

La idea de mayoría es un caso particular de la idea de **quórum**.

La propiedad fundamental de los quórums es:

> Dos quórums cualquiera deben tener al menos un nodo en común.

Esto es importante porque permite garantizar que una lectura suficientemente grande se cruza con una escritura previamente aceptada.

Frase de Emma: **Toda mayoria tiene un servidor actualizado si la escritura fue en mayoria.**

### Quórum de lectura y escritura

Se puede definir:

- `W`: quórum de escritura.
- `R`: quórum de lectura.
- `N`: cantidad total de nodos.

Para que toda lectura intersecte con toda escritura:

```text
W + R > N
```

Ejemplo con `N = 5`:

- `W = 3`
- `R = 3`
- `W + R = 6 > 5`

Si una escritura fue aceptada por 3 nodos y luego una lectura consulta 3 nodos, necesariamente al menos uno de los nodos leídos vio esa escritura.

Raft usa el caso particular más simple:

```text
W = R = mayoría
```

Esto combina dos propiedades:

1. Ante una partición, siempre queda a lo sumo una partición con mayoría.
2. Toda mayoría intersecta con cualquier mayoría anterior.

De esa combinación surge una garantía clave:

> Si una operación fue escrita en mayoría, cualquier mayoría futura contiene al menos un nodo que conoce esa operación.

---

## Historia breve de los algoritmos de consenso

El problema de consenso tolerante a fallas y particiones no se resolvió al comienzo de la computación distribuida. Algunos hitos:

- **Viewstamped Replication** — 1988, Liskov y Oki.
- **Paxos** — 1989, Lamport.
- **Raft** — 2014, Ongaro y Ousterhout.

Paxos es famoso por ser difícil de entender e implementar. Raft surge con la intención explícita de ser más comprensible.

Raft separa el problema en dos grandes partes:

1. **Elección de líder**.
2. **Replicación del log**.

Esa separación hace que el algoritmo sea más fácil de estudiar, implementar y razonar.

---

## Estructura básica de Raft

Raft se ubica debajo de la aplicación.

```text
Cliente
   |
Aplicación / Máquina de estado
   |
Raft
   |
Red / otros nodos
```

La aplicación puede ser, por ejemplo, un key-value store:

```text
put(k, v)
get(k)
delete(k)
```

Pero Raft no necesita entender qué significa cada comando. Para Raft, el comando es opaco: puede tratarse simplemente como un arreglo de bytes.

Lo importante es que todos los nodos reciban esos comandos en el mismo orden.

---

## El log

La estructura central de Raft es el **log**.

Cada nodo mantiene su propio log. El objetivo de Raft es lograr que los logs de los nodos sean consistentes.

La aplicación no aplica directamente una operación cuando llega del cliente. Primero la operación se agrega al log de Raft. Luego Raft replica esa entrada en otros nodos. Recién cuando la entrada está segura, se aplica a la máquina de estado.

Esto evita que un nodo aplique operaciones que luego podrían perderse.

---

## Caso feliz de escritura en Raft

Supongamos un sistema de 3 nodos:

```text
S1   S2   S3
```

`S2` es el líder.

Flujo:

1. El cliente envía una operación al líder, por ejemplo:

   ```text
   put(k, v)
   ```

2. La aplicación del líder le pasa el comando a Raft.
3. Raft agrega el comando al log local del líder.
4. El líder envía `AppendEntries` a los followers.
5. Los followers guardan la entrada en sus logs y responden `ACK`.
6. Cuando el líder recibe respuestas de una mayoría, la entrada queda **committed**.
7. El líder aplica la operación a su máquina de estado.
8. El líder responde `OK` al cliente.
9. El líder informa a los followers que esa entrada está committed.
10. Los followers aplican la entrada a sus propias máquinas de estado.

En `N = 3`, la mayoría es 2. Entonces alcanza con que el líder y un follower tengan la entrada persistida para que la operación pueda considerarse segura.

---

## Qué significa `committed`

Una entrada del log está **committed** cuando está almacenada en una mayoría de nodos.

No es exactamente el mismo concepto de `COMMIT` de una base de datos relacional. En Raft, significa que esa entrada ya no puede perderse mientras se mantengan las hipótesis de fallas toleradas.

Una entrada committed:

- Puede aplicarse a la máquina de estado.
- No debe ser borrada.
- Debe aparecer en los líderes futuros.

Una entrada no committed:

- Puede estar en algunos logs.
- Puede no haber sido confirmada al cliente.
- Puede ser sobrescrita por un líder futuro.

---

## Garantías del `OK` al cliente

Cuando el cliente recibe `OK`, tiene dos garantías:

1. La operación está persistida en una mayoría.
2. La operación fue aplicada por el líder a la máquina de estado.

Eso significa que la operación no depende de una sola máquina. Si el líder cae, otro nodo que participó de esa mayoría puede ayudar a recuperar la operación.

---

## RPCs principales de Raft

Raft usa dos RPCs principales:

### `AppendEntries`

Sirve para varias cosas:

- Replicar entradas del log.
- Enviar heartbeats.
- Informar el `commitIndex`.
- Mantener a los followers sincronizados con el líder.

Un heartbeat es, en la práctica, un `AppendEntries` vacío. No contiene nuevas entradas, pero le indica a los followers que el líder sigue vivo.

El `commitIndex` suele viajar dentro de `AppendEntries`: el líder informa hasta qué posición del log considera segura para aplicar.

### `RequestVote`

Sirve para pedir votos durante una elección de líder.

Un candidato envía `RequestVote` a los demás nodos. Si obtiene votos de una mayoría, se convierte en líder.

---

## Estados de un nodo

Cada nodo de Raft está en uno de estos estados:

- **Follower**
- **Candidate**
- **Leader**

Todos los nodos arrancan como followers.

Un follower acepta heartbeats del líder. Si pasa demasiado tiempo sin recibir heartbeats, asume que no hay líder disponible e inicia una elección.

---

## Election timeout

Cada follower tiene un **election timeout**.

Si el follower no recibe heartbeats antes de que expire ese timeout:

1. Pasa a estado candidate.
2. Incrementa el término actual.
3. Se vota a sí mismo.
4. Envía `RequestVote` a los demás nodos.
5. Espera votos.

Si obtiene mayoría, se convierte en líder.

El election timeout debe ser mayor que el intervalo de heartbeats. Si fuera demasiado corto, los followers iniciarían elecciones innecesarias todo el tiempo.

---

## Términos

Raft organiza el tiempo lógico en **terms**.

Cada elección pertenece a un término. Cada término puede tener:

- Un líder.
- Ningún líder, si la elección queda dividida.
- Nunca más de un líder válido para ese término.

Los términos aumentan cuando se inicia una nueva elección.

Si un nodo recibe un mensaje con un término mayor que el propio, actualiza su término y reconoce que está atrasado. Si era líder o candidato, vuelve a follower.

---

## Elección de líder

Ejemplo con `N = 5`:

```text
S1 S2 S3 S4 S5
```

Mayoría:

```text
M = 3
```

Flujo:

1. No hay líder.
2. Un nodo expira su election timeout.
3. Ese nodo se convierte en candidate.
4. Incrementa el term.
5. Se vota a sí mismo.
6. Envía `RequestVote` a los demás.
7. Si recibe dos votos externos, suma tres votos contando el propio.
8. Al llegar a mayoría, se convierte en leader.

El líder comienza a enviar heartbeats periódicos para evitar nuevas elecciones.

---

## Elección durante una partición

Con `N = 5`, si la red se divide en `3 + 2`:

- El lado con 3 nodos puede elegir líder.
- El lado con 2 nodos no puede elegir líder.

El lado minoritario puede tener candidatos, pero nunca llega a mayoría.

Esto evita el split brain: no pueden existir dos líderes válidos al mismo tiempo en particiones distintas, porque dos mayorías siempre se intersectan.

---

## Voto dividido

Puede ocurrir un **voto dividido** si dos candidatos aparecen al mismo tiempo.

Ejemplo:

- Un líder cae.
- Dos followers expiran su timeout casi simultáneamente.
- Ambos se convierten en candidates.
- Algunos nodos votan a uno y otros votan al otro.
- Nadie alcanza mayoría.

En ese caso, el término puede quedar sin líder. Luego expira otro timeout y se inicia una nueva elección en un término posterior.

Para reducir la probabilidad de voto dividido, los election timeouts se randomizan. Cada nodo usa un timeout ligeramente distinto.

---

## Randomización del election timeout

Si todos los nodos tuvieran exactamente el mismo timeout, sería común que muchos se vuelvan candidates al mismo tiempo.

Para evitarlo, se usa un rango:

```text
electionTimeout = minTimeout + random(jitter)
```

Esto hace que, estadísticamente, un nodo expire antes que los demás y pueda iniciar la elección con ventaja.

Los heartbeats deben llegar bastante antes de que venza el timeout más corto. Si no, el sistema puede pasar demasiado tiempo eligiendo líderes y no avanzar.

---

## Reglas para otorgar votos

Un follower no vota a cualquier candidato. Hay dos reglas centrales.

### 1. Un voto por término

Cada nodo puede votar a lo sumo una vez por term.

Esto debe persistirse en disco. Si un nodo vota, se cae, revive y olvida por quién votó, podría votar dos veces en el mismo término y romper las garantías del algoritmo.

Por eso `currentTerm` y `votedFor` son parte del estado persistente de Raft.

### 2. Solo votar candidatos actualizados

Un follower solo debe votar a un candidato cuyo log esté al menos tan actualizado como el propio.

Esto evita elegir como líder a un nodo que podría borrar entradas que ya fueron committed.

La comparación de “log actualizado” se basa en:

- El término de la última entrada del log.
- El índice de la última entrada del log.

El detalle completo se estudia junto con la replicación y reparación de logs, pero la intuición es:

> No se debe votar a un candidato que no conoce información que el votante sí conoce y que podría ser necesaria para preservar entradas committed.

---

## Logs divergentes

Los logs no siempre avanzan prolijamente iguales. Ante fallas, pueden diverger.

Ejemplo simple:

```text
S1: [3]
S2: [3, 3]
S3: [3, 3]
```

Esto puede pasar si `S2` era líder:

1. Replica la primera entrada a todos.
2. Replica una segunda entrada solo a `S3`.
3. Muere antes de replicarla a `S1`.

La segunda entrada está en mayoría (`S2` y `S3`), por lo tanto puede estar committed.

Si luego `S1` fuera elegido líder, podría intentar borrar la segunda entrada de `S2` y `S3`, lo cual sería incorrecto. Por eso Raft no debe permitir que `S1` sea elegido si su log está menos actualizado.

---

## Entradas no committed

Otro caso posible:

```text
S1: [3, 3, 5]
S2: [3, 3, 4]
S3: [3, 3]
```

Las entradas `4` y `5` pueden haber sido escritas por líderes distintos que murieron antes de alcanzar mayoría.

En ese caso:

- El cliente que envió la operación `4` puede no haber recibido respuesta.
- El cliente que envió la operación `5` también puede no haber recibido respuesta.
- Raft puede terminar conservando una de esas entradas y descartando la otra, dependiendo de qué líder sea elegido después.

Esto no viola la semántica, porque a falta de `OK`, el cliente no sabe si la operación fue aceptada o no. El cliente puede hacer retry o consultar el estado.

La regla importante es:

> Las entradas committed no se borran. Las entradas no committed pueden ser sobrescritas.

---

## Relación entre mayoría, elección y log

La mayoría aparece en dos lugares:

1. Para elegir líderes.
2. Para confirmar entradas del log.

Esto es lo que hace que las piezas encajen.

Si una entrada fue committed, entonces está en una mayoría. Cualquier líder futuro tuvo que ser elegido por una mayoría. Como dos mayorías se intersectan, algún nodo que votó al nuevo líder conoce la entrada committed o impide votar a un candidato que no esté suficientemente actualizado.

Esa combinación evita que un líder futuro borre información ya confirmada.

---

## CAP y disponibilidad

Ante una partición de red, no se puede mantener todo al mismo tiempo:

- Consistencia fuerte.
- Disponibilidad total.
- Tolerancia a particiones.

Raft elige consistencia. La partición minoritaria no puede aceptar escrituras. Dependiendo de la implementación, incluso puede rechazar lecturas para no devolver datos viejos.

En la versión base del paper, Raft apunta a que las lecturas también respeten la consistencia fuerte, por lo que un nodo en una minoría no debería responder como si pudiera garantizar actualidad.

---

## Para implementar Raft

La implementación suele dividirse en partes:

1. Elección de líder.
2. Replicación básica del log.
3. Persistencia.
4. Snapshots.

La primera parte se concentra en:

- Estados `Follower`, `Candidate`, `Leader`.
- `RequestVote`.
- `AppendEntries` vacío como heartbeat.
- Election timeout.
- Terms.
- Voto único por término.
- Conversión correcta entre estados.
- Randomización de timeouts.

Los tests suelen ser exigentes con los tiempos. No alcanza con que el algoritmo funcione “eventualmente”: debe restaurarse en un plazo razonable y evitar elecciones innecesarias.

---

## Resumen final

Raft resuelve consenso para replicar una máquina de estados. Su objetivo es que varias máquinas apliquen las mismas operaciones en el mismo orden, incluso ante fallas fail-stop y particiones de red.

Las ideas centrales son:

- Usar un líder para ordenar operaciones.
- Replicar cada operación en un log.
- Confirmar entradas solo cuando están en mayoría.
- Elegir líderes solo con mayoría.
- Usar términos para ordenar épocas de liderazgo.
- Evitar split brain impidiendo que una minoría avance.
- No borrar entradas committed.
- Permitir sobrescribir entradas no committed.
- Persistir información crítica como el log, el término actual y el voto emitido.

Raft combina mayorías, quórums, logs y elección de líder para garantizar que, si un cliente recibe `OK`, la operación queda protegida por el sistema y podrá recuperarse aunque falle una parte de los nodos.
