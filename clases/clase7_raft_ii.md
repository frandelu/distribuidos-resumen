# Clase 7 — Raft II

## [Apuntes de Emma](https://drive.google.com/file/d/1FcaEAqVGvVee-Z4JJ6XRvsPyHnz8Rsvt/view)

## Objetivo

Raft resuelve replicación de máquina de estados manteniendo un log consistente entre varios nodos. La primera parte cubre elección de líder, términos, mayorías y `AppendEntries`. Esta segunda parte completa las piezas que hacen que el algoritmo siga siendo correcto cuando aparecen fallas, particiones, reinicios, logs divergentes y nodos atrasados.

Los temas centrales son:

- restricción de elección de líder;
- sincronización y rollback de logs;
- propiedad de coincidencia del log;
- persistencia mínima necesaria;
- costo real de persistir;
- compactación de logs mediante snapshots;
- instalación de snapshots en followers muy atrasados;
- consecuencias para clientes cuando no reciben respuesta.

---

## Restricción de elección

Un follower no vota automáticamente a cualquier candidato. Para responder afirmativamente a un `RequestVote`, deben cumplirse dos condiciones importantes.

Primero, un nodo puede votar una sola vez por término. Si ya votó por un candidato en el término actual, no puede votar a otro candidato en ese mismo término. Esto evita que dos candidatos distintos consigan mayorías incompatibles.

Segundo, el candidato debe tener un log suficientemente actualizado. No se debe votar a un candidato que tenga un log más viejo que el del votante.

La regla práctica es:

```text
Votar "sí" si:
1. no voté por otro candidato en este término.
2. el log del candidato está al menos tan actualizado como el mío.
```

El log del candidato se considera actualizado usando dos datos:

```text
lastLogTerm
lastLogIndex
```

La comparación se hace así:

```text
El candidato está más actualizado si:

1. candidate.lastLogTerm > my.lastLogTerm

o, si los términos son iguales:

2. candidate.lastLogTerm == my.lastLogTerm
   y candidate.lastLogIndex >= my.lastLogIndex
```

El término tiene prioridad sobre el largo del log. Un log más largo no necesariamente es más seguro si su último término es más viejo. Esto es clave para no elegir un líder que pueda borrar entradas ya comiteadas.

---

## Por qué no alcanza con votar por cualquier candidato

Supongamos un clúster de cinco nodos. Un líder recibe una operación, la guarda en su log y la replica a dos nodos más. Cuando esos dos followers responden `OK`, la operación queda en una mayoría. Por lo tanto, está comiteada.

Luego ocurre una partición de red. El líder anterior queda del lado minoritario con un follower. Del otro lado quedan tres nodos, que forman mayoría.

El líder viejo puede seguir creyendo que es líder. Puede recibir pedidos de clientes y agregar entradas nuevas a su log local, incluso replicarlas al único follower que todavía ve. Pero no puede comitearlas, porque no logra mayoría. El cliente no debe recibir `OK` por esas nuevas operaciones.

Del lado mayoritario, los tres nodos dejan de recibir heartbeats y empiezan una nueva elección. Para que el sistema sea seguro, el nuevo líder debe contener la información ya comiteada antes de la partición. Si se eligiera un nodo desactualizado, ese nodo podría sobrescribir o borrar entradas que ya habían sido confirmadas a un cliente. Eso rompería la garantía de Raft.

La restricción de elección evita ese caso: un nodo desactualizado no consigue los votos necesarios porque al menos un nodo de la mayoría tiene la entrada comiteada y rechaza votar por un candidato más viejo.

---

## La partición minoritaria no avanza

Una partición minoritaria puede seguir “viva” desde el punto de vista local, pero no puede avanzar el estado comiteado.

Puede ocurrir que:

1. el líder viejo quede en la partición minoritaria;
2. siga aceptando requests localmente;
3. agregue entradas a su log;
4. intente replicarlas;
5. no consiga mayoría;
6. no pueda responder `OK`.

Esto puede generar logs divergentes, pero no genera divergencia de estado comiteado. Las entradas que se agregaron sin mayoría podrán ser descartadas más tarde.

---

## Entradas no comiteadas: pueden sobrevivir o desaparecer

Hay un caso menos intuitivo: una operación puede haber sido escrita en un solo follower antes de que el líder muera.

Ejemplo:

1. El líder recibe una operación.
2. La guarda en su log.
3. La replica solamente a un follower.
4. Muere antes de obtener mayoría.
5. El cliente nunca recibe `OK`.

A partir de ahí pueden pasar dos cosas.

Si el follower que tiene la entrada más nueva se convierte en líder, esa entrada puede propagarse al resto y terminar comiteada.

Si se convierte en líder otro nodo que no tenía esa entrada, esa entrada puede borrarse del follower que sí la tenía.

Ambas posibilidades son correctas desde el punto de vista de Raft, porque el cliente nunca recibió confirmación. Cuando un cliente no recibe respuesta, no sabe si la operación terminó comiteada o no. El sistema puede terminar en cualquiera de los dos estados sin violar una promesa hecha al cliente.

Esto implica que las aplicaciones construidas sobre Raft deben pensar la semántica de reintentos. Si una operación no es idempotente, reenviarla puede duplicar efectos. Para evitarlo, suelen usarse identificadores únicos de operación, deduplicación o semánticas de tipo “exactly once” implementadas por la aplicación.

---

## Log rollback / Sincronización de logs

Cuando se elige un nuevo líder, su log pasa a ser la fuente de verdad. Los followers deben sincronizarse con él.

Raft usa `AppendEntries` tanto para replicar entradas nuevas como para corregir logs divergentes. Cada `AppendEntries` incluye, entre otros campos:

```text
term
leaderId
prevLogIndex
prevLogTerm
entries[]
leaderCommit
```

Los campos más importantes para sincronizar son:

```text
prevLogIndex
prevLogTerm
entries[]
```

La idea es que el follower solo acepta nuevas entradas si puede verificar que su log coincide con el del líder justo antes de la entrada nueva.

---

## Log Matching Property

La propiedad de coincidencia del log dice:

```text
Si dos entradas en dos logs distintos tienen el mismo índice y el mismo término,
entonces contienen el mismo comando.
```

Esta propiedad permite usar `(index, term)` como punto de sincronización.

Ejemplo:

```text
Índice: 10  11  12  13

S1:      3
S2:      3   3   4
S3:      3   3   5   6
```

Si `S3` es líder y quiere replicar la entrada del índice 13, envía:

```text
prevLogIndex = 12
prevLogTerm  = 5
entries      = [6]
```

Si `S2` tiene en el índice 12 el término 4, rechaza el `AppendEntries`, porque no coincide el `prevLogTerm`.

Entonces el líder retrocede y prueba con un prefijo anterior:

```text
prevLogIndex = 11
prevLogTerm  = 3
entries      = [5, 6]
```

Ahora `S2` sí tiene en el índice 11 una entrada del término 3. Ese punto coincide. Entonces puede aceptar las nuevas entradas, truncar lo que tenía desde el punto divergente y reemplazarlo por lo enviado por el líder.

El resultado queda:

```text
S2: 3  3  5  6
S3: 3  3  5  6
```

Las entradas no comiteadas que estaban en conflicto se descartan. Las entradas comiteadas no se pierden porque la restricción de elección impide elegir como líder a un nodo que no contenga el prefijo comiteado necesario.

---

## Truncado del log

Cuando un follower acepta un `AppendEntries`, pueden pasar dos situaciones.

Si el follower no tiene el `prevLogIndex` o lo tiene con otro término, responde rechazo. El líder debe retroceder y probar con un punto anterior.

Si el follower sí tiene el `prevLogIndex` con el `prevLogTerm` correcto, entonces hay coincidencia de prefijo. Desde ahí, el follower puede borrar cualquier sufijo conflictivo y agregar las entradas que envía el líder.

En forma simplificada:

```text
on AppendEntries(prevLogIndex, prevLogTerm, entries):
    if log[prevLogIndex].term != prevLogTerm:
        reject

    delete conflicting entries after prevLogIndex
    append new entries
    accept
```

El algoritmo básico puede retroceder de a una posición por vez. Eso funciona, pero puede ser lento si un follower quedó muy atrasado o divergió mucho. Por eso existen optimizaciones donde el follower devuelve más información sobre el conflicto, permitiendo que el líder salte más rápido hacia atrás.

---

## `leaderCommit`

Además de replicar entradas, el líder informa a los followers hasta dónde está comiteado el log. Eso viaja en `AppendEntries` mediante `leaderCommit`.

Un follower puede tener entradas en el log, pero no aplicarlas todavía a la máquina de estados hasta saber que están comiteadas.

El flujo es:

1. el líder replica entradas;
2. consigue mayoría;
3. actualiza su `commitIndex`;
4. aplica las entradas comiteadas a su aplicación;
5. informa `leaderCommit` a los followers en `AppendEntries`;
6. los followers avanzan su `commitIndex`;
7. aplican las entradas a la aplicación en orden.

Esto mantiene la regla fundamental: la aplicación solo ve entradas comiteadas y en el mismo orden en todos los nodos.

---

## Election restriction en ejemplos

Si hay tres servidores con estos últimos términos:

```text
S1: 5  6  7
S2: 5  8
S3: 5  8
```

`S1` tiene el log más largo, pero su último término es 7. `S2` y `S3` tienen último término 8. Por lo tanto, `S1` no está más actualizado que ellos. No debe ser elegido líder.

Si se eligiera a `S1`, podría borrar entradas del término 8 que quizá ya estaban comiteadas. Por eso el término pesa más que el largo.

En cambio, `S2` o `S3` sí pueden ser elegidos, porque tienen el término más nuevo.

---

## Ejemplo grande: no hay un único líder posible

En escenarios con muchos nodos y logs muy divergentes, puede haber varios candidatos válidos. Raft no necesita que exista un único candidato correcto. Necesita que cualquier candidato que pueda obtener mayoría no borre entradas ya comiteadas.

La elección puede depender del orden de timeouts y mensajes. Distintos líderes posibles pueden llevar a distintos sufijos finales del log, pero todos preservan el prefijo comiteado.

Esto es central:

```text
Raft no garantiza que sobreviva toda entrada escrita en algún nodo.
Raft garantiza que no se pierde una entrada comiteada.
```

Una entrada escrita en minoría puede sobrevivir o desaparecer. Una entrada escrita en mayoría debe sobrevivir.

---

## Persistencia

Hay tres piezas de estado que deben persistirse en disco:

```text
currentTerm
votedFor
log[]
```

Todo lo demás puede reconstruirse o recalcularse.

`currentTerm` debe persistirse porque un nodo puede enterarse de términos más nuevos aunque no reciba entradas nuevas. Si reinicia y olvida el término actual, podría volver a actuar con información vieja.

`votedFor` debe persistirse porque un nodo no puede votar dos veces en el mismo término. Si vota, se cae y revive, debe recordar por quién votó.

`log[]` debe persistirse porque un `OK` a `AppendEntries` significa que la entrada está segura en ese nodo. Si el nodo responde `OK` y luego pierde la entrada al reiniciar, rompe la garantía que el líder usó para formar mayoría.

---

## Orden correcto: primero disco, después respuesta

La regla práctica es:

```text
primero persistir;
después responder OK.
```

Para `AppendEntries`:

```text
AppendEntries -> OK
significa:
    la entrada está en el log persistente
    el término relevante está persistido
```

Para `RequestVote`:

```text
RequestVote -> OK
significa:
    el voto está persistido
    el término relevante está persistido
```

Responder antes de persistir es incorrecto. Puede ocurrir que un nodo responda `OK`, se caiga, reinicie y haya perdido la información que prometió tener.

---

## `write` no alcanza: hace falta `fsync`

En sistemas operativos, una llamada a `write()` no necesariamente escribe en disco físico. Puede dejar los datos en buffers del sistema operativo.

Para garantizar persistencia real se necesita algo como:

```text
write()
fsync()
```

Recién después de `fsync()` se puede responder `OK` con la garantía de que el dato sobrevivirá a un reinicio.

Esto tiene costo de performance. Hacer `fsync()` por cada entrada puede ser muy lento. Una optimización común es acumular varias entradas y sincronizarlas juntas en lote. El orden se mantiene, pero las respuestas se retienen hasta que el lote queda persistido.

---

## Log compaction
=90
El log de Raft crece indefinidamente si nunca se borra. Eso trae dos problemas:

1. ocupa cada vez más disco;
2. reiniciar un nodo sería cada vez más lento, porque habría que reconstruir la aplicación aplicando el log desde el principio de los tiempos.

La solución es compactar el log usando snapshots.

La aplicación mantiene el estado derivado del log: por ejemplo, una base key-value, una tabla, un árbol de archivos o cualquier otra máquina de estados determinista.

En cierto momento, Raft puede pedirle a la aplicación una foto consistente de su estado. Ese snapshot representa el resultado de aplicar el log hasta cierto índice.

Ejemplo:

```text
log aplicado hasta índice 12
snapshot = estado de la aplicación después de aplicar hasta 12
```

Raft guarda el snapshot junto con metadata:

```text
lastIncludedIndex = 12
lastIncludedTerm  = término de la entrada 12
```

Después puede borrar del log las entradas anteriores o iguales a ese índice, porque ya están representadas dentro del snapshot.

Conceptualmente:

```text
antes:
[1][2][3][4][5][6][7][8][9][10][11][12][13][14]

snapshot hasta 12

después:
snapshot(12) + [13][14]
```

Los índices no se reinician. La entrada 13 sigue siendo la entrada 13 aunque el prefijo haya sido eliminado físicamente.

---

## Restore

Cuando un nodo reinicia, no necesita aplicar todo el log desde cero. Puede:

1. cargar el snapshot;
2. restaurar la aplicación al estado del snapshot;
3. aplicar las entradas posteriores al snapshot.

Ejemplo:

```text
snapshot hasta 12
log restante: 13, 14, 15

restore:
    cargar snapshot
    aplicar 13
    aplicar 14
    aplicar 15
```

Esto acelera la recuperación y evita guardar un log infinito.

---

## InstallSnapshot

La compactación introduce un problema: puede haber followers tan atrasados que necesiten entradas que el líder ya compactó y borró.

Ejemplo:

```text
Follower:
    solo tiene entradas viejas

Líder:
    ya compactó el log hasta 12
    solo conserva desde 13 en adelante
```

Si el líder intenta sincronizar retrocediendo con `AppendEntries`, llega un punto donde ya no tiene las entradas antiguas necesarias para encontrar un prefijo común.

En ese caso usa otra RPC:

```text
InstallSnapshot
```

El líder envía el snapshot al follower. El follower:

1. recibe el snapshot;
2. descarta el estado/log que ya no sirve;
3. instala el snapshot en la aplicación;
4. queda ubicado en `lastIncludedIndex`;
5. recibe luego las entradas posteriores mediante `AppendEntries`.

Esto permite recuperar followers muy atrasados aunque el líder ya haya compactado el prefijo del log.

---

## Cuándo se puede borrar log

Un nodo solo puede compactar entradas que ya están:

```text
comiteadas
aplicadas a la máquina de estados
incluidas en un snapshot consistente
```

No puede borrar entradas no comiteadas, porque todavía podrían ser necesarias para decidir el estado futuro del sistema.

Cada servidor puede decidir localmente cuándo compactar. No hace falta que todos tomen snapshots al mismo tiempo. Lo importante es que cada snapshot represente un estado consistente de la aplicación y que incluya la metadata necesaria para seguir comparando logs.

---

## Lecturas y consistencia

Toda la discusión se enfocó en escrituras. Las lecturas agregan otro problema.

Si hay una partición, la mitad minoritaria puede tener datos viejos. Si se permite leer de cualquier follower, una lectura puede devolver información desactualizada.

Para consistencia fuerte o linealizabilidad, no alcanza con leer localmente de cualquier nodo. Hay que asegurarse de que la lectura observe un estado comiteado y vigente respecto del líder actual. Este punto se conecta con la discusión posterior sobre linealizabilidad y sistemas como ZooKeeper.

---

## Resumen operativo de Raft II

La seguridad de Raft depende de varias piezas que encajan:

1. Un nodo vota una sola vez por término.
2. Un nodo solo vota candidatos con logs actualizados.
3. El líder elegido puede imponer su log sobre los followers.
4. `AppendEntries` usa `prevLogIndex` y `prevLogTerm` para detectar divergencias.
5. Los followers rechazan entradas si no coincide el prefijo.
6. El líder retrocede hasta encontrar un punto común.
7. Las entradas conflictivas no comiteadas se truncan.
8. Las entradas comiteadas no se pierden.
9. `currentTerm`, `votedFor` y `log` deben persistirse.
10. Se debe persistir antes de responder `OK`.
11. Los snapshots evitan logs infinitos.
12. `InstallSnapshot` permite recuperar followers demasiado atrasados.

La garantía importante no es que todo lo que alguna vez apareció en un log sobreviva. La garantía importante es:

```text
Toda entrada comiteada sobrevive y se aplica en el mismo orden en todos los nodos.
```

---

## Puntos importantes para implementar

Para implementar Raft correctamente, conviene prestar especial atención a:

- `RequestVote`:
  - un voto por término;
  - comparación correcta de `lastLogTerm` y `lastLogIndex`;
  - persistir `currentTerm` y `votedFor` antes de responder.

- `AppendEntries`:
  - validar `prevLogIndex` y `prevLogTerm`;
  - rechazar si no coincide el prefijo;
  - truncar sufijos conflictivos;
  - agregar entradas nuevas;
  - persistir antes de responder `OK`;
  - actualizar `commitIndex` con `leaderCommit`.

- Sincronización:
  - manejar followers atrasados;
  - retroceder `nextIndex`;
  - optimizar rechazos si hace falta.

- Snapshots:
  - generar snapshots solo con entradas aplicadas;
  - guardar `lastIncludedIndex` y `lastIncludedTerm`;
  - compactar el log sin perder la numeración lógica;
  - implementar `InstallSnapshot` para followers atrasados.

- Clientes:
  - un timeout no implica que la operación no haya ocurrido;
  - los reintentos deben ser seguros;
  - la aplicación debe manejar idempotencia o deduplicación cuando sea necesario.
