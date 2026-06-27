# Clase 12 — Transacciones distribuidas

## [Apuntes de Emma](https://drive.google.com/file/d/134xX2l3qWaulpErl4qCQRCyMJ8BI61DY/view)

## Problema general

Una **transacción distribuida** agrupa varias operaciones ejecutadas sobre distintos sistemas como si fueran una única operación lógica.

La idea básica es:

```text
varias operaciones distribuidas
           │
           ▼
una única unidad lógica
```

El objetivo es que el sistema no quede en un estado intermedio. O se ejecuta todo, o no se ejecuta nada.

## Ejemplo: reserva de vuelos

Un sistema de reserva de vuelos puede estar separado en varios subsistemas. Por ejemplo:

```text
Usuario
  │
  ▼
Frontend
  ├── reserve(2B) ──▶ Sistema de asientos
  └── emit(..., 2B) ──▶ Sistema de tickets
```

La operación completa debería reservar el asiento y emitir el ticket. Si solo ocurre una de las dos cosas, el sistema queda inconsistente.

Ejemplo de flujo feliz:

```text
Frontend ── reserve(2B) ──▶ Asientos
Frontend ◀──── OK ───────── Asientos
Frontend ── emit(..., 2B) ─▶ Tickets
Frontend ◀──── OK ───────── Tickets
```

El problema aparece cuando falla alguna parte del flujo.

Si falla antes de reservar el asiento, no pasó nada grave. Pero si el asiento ya fue reservado y el frontend muere antes de emitir el ticket, queda un asiento bloqueado sin ticket emitido.

```text
Frontend ── reserve(2B) ──▶ Asientos
Frontend ◀──── OK ───────── Asientos
Frontend muere antes de llamar a Tickets
```

Resultado:

```text
asiento reservado
ticket no emitido
```

También puede ocurrir lo inverso: el frontend llama a `emit`, el sistema de tickets emite el ticket, pero la respuesta se pierde. El frontend no sabe si el ticket fue emitido o no. Si intenta deshacer la reserva del asiento, podría quedar un ticket emitido sin asiento reservado.

Estos casos son típicos en sistemas distribuidos. No alcanza con pensar solamente en el camino feliz.

## Transacciones distribuidas implícitas

Muchas veces se hacen transacciones distribuidas sin nombrarlas como tales.

Ejemplo común:

```text
1. Escribir en una base de datos.
2. Mandar un mensaje a una cola.
```

Si se escribe en la base pero falla el envío del mensaje, otro sistema nunca se entera del cambio.

```text
DB write ── OK
Queue send ── falla
```

Si se manda primero el mensaje y luego falla la escritura en la base, otro sistema puede consumir un evento que todavía no tiene respaldo en la base de datos.

```text
Queue send ── OK
DB write ── falla
```

También puede pasar que el mensaje llegue demasiado rápido: otro sistema consume el evento, busca el dato en la base, no lo encuentra porque la escritura todavía no ocurrió, y aborta su procesamiento.

El problema no aparece solo en sistemas enormes como los de Amazon o Google. Aparece en cualquier arquitectura con múltiples servicios, bases de datos, colas o APIs externas.

## Propiedades ACID

Las transacciones suelen describirse con las propiedades **ACID**:

| Propiedad | Idea |
|---|---|
| Atomicidad | Todo o nada |
| Consistencia | Preservar invariantes internas de la base |
| Aislamiento | Evitar interferencias entre transacciones concurrentes |
| Durabilidad | Lo confirmado queda persistido |

En esta parte interesan especialmente dos propiedades:

```text
Atomicidad  ──▶ atomic write
Aislamiento ──▶ control de concurrencia
```

### Atomicidad

La atomicidad exige que una transacción se ejecute completa o no tenga efecto.

```text
reserve(2B) + emit(ticket)
```

Debe comportarse como una sola operación atómica. No debería quedar solamente una mitad aplicada.

### Consistencia

La consistencia de ACID no es la misma consistencia del teorema CAP ni la misma que linealizabilidad.

En ACID, consistencia significa preservar invariantes internas. Por ejemplo:

```text
una clave única no puede duplicarse
un saldo no puede violar una restricción
una foreign key debe seguir apuntando a algo válido
```

### Aislamiento

El aislamiento evita que dos transacciones concurrentes se mezclen de una forma que produzca resultados imposibles.

La propiedad fuerte asociada al aislamiento es **serializabilidad**.

### Durabilidad

La durabilidad exige que, una vez confirmada una transacción, sus efectos no se pierdan.

En una base de datos local suele lograrse escribiendo a disco. En un sistema distribuido suele lograrse replicando en varios discos o nodos.

## Serializabilidad

Una ejecución de transacciones es **serializable** si existe algún orden serial, una transacción por vez, que produce el mismo resultado.

```text
Ejecución concurrente
        │
        ▼
Debe equivaler a algún orden serial
```

Ejemplo inicial:

```text
x = 10
y = 10
```

Transacción 1:

```text
Tx1:
  add(x, 1)
  add(y, -1)
```

Transacción 2:

```text
Tx2:
  v1 = get(x)
  v2 = get(y)
  print(v1, v2)
```

Si `Tx1` ocurre antes que `Tx2`, el resultado visible para `Tx2` es:

```text
x = 11
y = 9
```

Si `Tx2` ocurre antes que `Tx1`, el resultado visible para `Tx2` es:

```text
x = 10
y = 10
```

Esos dos resultados son serializables porque corresponden a órdenes seriales válidos:

```text
Tx1 ; Tx2  ──▶  11, 9
Tx2 ; Tx1  ──▶  10, 10
```

Una intercalación problemática sería:

```text
add(x, 1)
v1 = get(x)
v2 = get(y)
add(y, -1)
```

En ese caso `Tx2` ve:

```text
x = 11
y = 10
```

Ese resultado no corresponde a ningún orden serial. No es legal bajo serializabilidad.

## Control de concurrencia

El **control de concurrencia** es el conjunto de mecanismos que evita ejecuciones no serializables.

Hay dos grandes enfoques:

```text
pesimista ──▶ usa locks
optimista ──▶ no usa locks tradicionales; valida antes de escribir
```

## Control pesimista

El control pesimista asume que puede haber conflictos. Por eso bloquea recursos antes de accederlos.

```text
antes de leer o escribir
        │
        ▼
obtener lock
```

Si el lock está libre, la transacción avanza. Si no está libre, la transacción espera.

La desventaja es que pueden aparecer bloqueos y deadlocks.

```text
Tx1 espera lock de B mientras tiene lock de A
Tx2 espera lock de A mientras tiene lock de B
```

## Two-phase locking

El algoritmo clásico para control pesimista es **two-phase locking** o **2PL**.

Tiene dos fases:

```text
1. Fase expansiva / growing
   Se obtienen locks.

2. Fase contractiva / shrinking
   Se liberan locks.
```

La regla central es:

```text
una vez que se libera un lock,
ya no se puede pedir ningún lock nuevo
```

Una variante fuerte es **strict 2PL** o **strong strict 2PL**:

```text
begin transaction
  obtener locks
  ejecutar operaciones
commit / abort
  liberar locks
```

Los locks se liberan después del `commit` o del `abort`. Esta forma garantiza serializabilidad fuerte.

## Locks en sistemas distribuidos

En un sistema distribuido, los locks suelen estar distribuidos junto con los datos.

Ejemplo:

```text
Participante 1: Sistema de asientos
  tabla asientos
  tabla/estructura de locks

Participante 2: Sistema de tickets
  tabla tickets
  tabla/estructura de locks
```

Cada participante guarda los locks de los recursos que administra.

```text
Asientos:
  asiento 2A ── lock: host1

Tickets:
  ticket 233 ── lock: host1
```

El algoritmo sigue siendo conceptualmente el mismo que en una base de datos local: primero se obtienen los locks necesarios, luego se confirma o aborta, y finalmente se liberan.

## Control optimista

El control optimista asume que la mayoría de las transacciones no van a entrar en conflicto.

No bloquea preventivamente. La transacción avanza y, antes de confirmar, se valida si todavía es segura.

```text
1. Ejecutar sin locks tradicionales.
2. Validar antes de escribir o confirmar.
3. Si no hay conflicto: commit.
4. Si hay conflicto: abort + retry.
```

Ventajas:

```text
no hay espera por locks
no hay deadlocks
```

Desventaja:

```text
el cliente debe tolerar aborts y reintentar
```

Este enfoque es útil cuando los conflictos son poco frecuentes.

## Atomic write distribuido

La atomicidad distribuida se suele resolver con **two-phase commit** o **2PC**.

El objetivo es que todos los participantes confirmen la transacción o todos la aborten.

```text
Participante 1
Participante 2
Participante 3
      ▲
      │
Transaction Coordinator
```

El sistema introduce un nuevo rol: el **transaction coordinator**. Ese coordinador orquesta la transacción.

Además, cada transacción tiene un identificador:

```text
transaction_id
```

Ese identificador viaja con las operaciones para que cada participante pueda asociarlas a la transacción correcta.

## Two-phase commit

El **two-phase commit** tiene dos fases principales:

```text
1. Prepare
2. Commit / Abort
```

### Fase 1: prepare

El coordinador pregunta a todos los participantes si están listos para confirmar.

```text
Coordinator ── prepare(tx_id) ──▶ P1
Coordinator ── prepare(tx_id) ──▶ P2
Coordinator ── prepare(tx_id) ──▶ P3
```

Cada participante responde:

```text
OK
```

o:

```text
NO
```

Para avanzar al commit, todos deben responder `OK`.

```text
todos OK ──▶ commit
alguno NO ──▶ abort
```

No alcanza con mayoría. No hay quórum. Tiene que ser unanimidad.

### Fase 2: commit o abort

Si todos respondieron `OK`, el coordinador manda `commit`.

```text
Coordinator ── commit(tx_id) ──▶ P1
Coordinator ── commit(tx_id) ──▶ P2
Coordinator ── commit(tx_id) ──▶ P3
```

Si algún participante respondió `NO`, o si no se puede completar la preparación, el coordinador manda `abort`.

```text
Coordinator ── abort(tx_id) ──▶ P1
Coordinator ── abort(tx_id) ──▶ P2
Coordinator ── abort(tx_id) ──▶ P3
```

## Punto de no retorno

El momento más importante del algoritmo es cuando un participante responde `OK` al `prepare`.

```text
prepare recibido
      │
      ▼
participante responde OK
      │
      ▼
punto de no retorno
```

Antes de responder `OK`, el participante puede abortar localmente sin problema. Todavía no prometió nada.

Después de responder `OK`, el participante queda comprometido: si luego recibe `commit`, debe poder confirmar.

Por eso, antes de responder `OK`, el participante debe persistir lo necesario.

```text
antes de responder OK:
  guardar estado en disco
  o replicar estado de forma durable
```

En un sistema distribuido, “guardar en disco” suele significar escribir en un mecanismo tolerante a fallas, no solo en memoria.

## Por qué un participante no puede timeoutar solo

Después de responder `OK`, un participante no puede decidir unilateralmente abortar por timeout.

Supongamos:

```text
P1 respondió OK
P2 respondió OK
Coordinator mandó commit a P1
Coordinator murió antes de mandar commit a P2
```

Si `P2` decide abortar solo por timeout, queda:

```text
P1 committed
P2 aborted
```

Eso rompe la atomicidad.

También podría pasar lo contrario: si un participante decide comitear solo, pero el coordinador había abortado con otros participantes, también se rompe el sistema.

Después del `prepare OK`, el participante debe esperar una decisión final del coordinador:

```text
commit
```

o:

```text
abort
```

Esta es una de las razones por las que 2PC puede bloquear: si el coordinador falla en el momento incorrecto, los participantes preparados quedan esperando.

## El coordinador debe tolerar fallas

Para que 2PC funcione, el coordinador debe ser tolerante a fallas o poder recuperarse.

El coordinador tiene que persistir su estado:

```text
prepare enviado a P1
P1 respondió OK
prepare enviado a P2
P2 respondió OK
decisión = commit
commit enviado a P1
...
```

Si el coordinador muere, otro proceso o el mismo coordinador al reiniciar debe poder reconstruir el estado y continuar.

Antes de mandar el primer `commit`, el coordinador debe haber persistido la decisión de commit. Si no lo hace, podría haber participantes que comitearon y luego no habría forma de saber que la decisión global era commit.

Los mensajes `commit` y `abort` suelen ser idempotentes. Si un participante recibe dos veces el mismo `commit`, no debería romperse.

```text
commit(tx_id)
commit(tx_id) repetido
        │
        ▼
mismo resultado
```

## DynamoDB y transacciones

DynamoDB originalmente ofrecía operaciones simples sobre items:

```text
put item
update item
delete item
get item
```

Una operación afectaba un item individual. Con el tiempo se agregaron operaciones transaccionales:

```text
TransactWriteItems
TransactGetItems
ConditionCheck / CheckItem
```

Estas transacciones son más limitadas que las de SQL. No permiten ejecutar lógica arbitraria dentro de una transacción larga. En cambio, se envía una bolsa de operaciones que deben aplicarse todas juntas o ninguna.

Ejemplo conceptual:

```text
TransactWriteItems:
  - update item A
  - put item B
  - condition check C
```

El sistema ejecuta todo el conjunto atómicamente o rechaza la transacción.

## Arquitectura base de DynamoDB

El flujo base de DynamoDB incluye:

```text
Cliente
  │
  ▼
Request Router
  │
  ├── consulta Partition Metadata System
  │
  ▼
Storage Nodes
```

El **request router** recibe el pedido. Para saber dónde está una clave, consulta el **partition metadata system**. Ese sistema indica en qué storage nodes vive la partición correspondiente.

Una transacción puede involucrar múltiples claves ubicadas en distintas particiones y, por lo tanto, en distintos grupos de storage nodes.

```text
Transacción:
  put(k1, v1) ──▶ partición A
  put(k2, v2) ──▶ partición B
  delete(k3) ──▶ partición C
```

El problema es lograr que todas esas operaciones queden confirmadas juntas.

## Transaction coordinator en DynamoDB

Para implementar transacciones, DynamoDB agrega un **transaction coordinator**.

```text
Cliente
  │
  ▼
Request Router
  │
  ├── operación común ──▶ Storage Nodes
  │
  └── operación transaccional ──▶ Transaction Coordinator
```

Si el request es un `put` común, puede ir directo al storage node correspondiente.

Si el request es transaccional, pasa por el transaction coordinator, que coordina el protocolo.

DynamoDB combina dos técnicas:

```text
Two-phase commit
Optimistic locking / optimistic concurrency control
```

## Prepare con operaciones incluidas

En 2PC clásico, el coordinador puede enviar operaciones antes y luego mandar `prepare`.

En DynamoDB, el `prepare` ya incluye las operaciones que se quieren ejecutar.

```text
Coordinator ── prepare(tx_id, operaciones) ──▶ Storage Node 1
Coordinator ── prepare(tx_id, operaciones) ──▶ Storage Node 2
```

Ejemplo:

```text
prepare(tx_id,
  put(k1, v1),
  update(k2, v2),
  condition_check(k3)
)
```

El storage node evalúa si puede aceptar esa parte de la transacción. Si puede, responde `OK`. Si no, responde `NO`.

Luego el coordinador decide:

```text
todos OK ──▶ commit
alguno NO ──▶ cancel / abort
```

Esta forma ayuda a mantener latencias más predecibles: la transacción llega como un paquete de operaciones acotado, no como una sesión arbitraria con lógica indefinida.

## Estados del transaction coordinator

El coordinador maneja una máquina de estados para cada transacción.

Estados típicos:

```text
preparing
committing
canceling
completed
```

Flujo simplificado:

```text
state = preparing
for storage_node in participantes:
    send_prepare(storage_node, tx)

if all_prepared_ok:
    state = committing
    for storage_node in participantes:
        send_commit(storage_node, tx)
    state = completed
else:
    state = canceling
    for storage_node in participantes:
        send_cancel(storage_node, tx)
    state = completed
```

El estado no puede vivir solamente en memoria. Si el coordinador muere, la transacción no puede quedar perdida.

## Ledger del coordinador

DynamoDB guarda el estado del coordinador en un **ledger** o log persistente.

Conceptualmente:

| tx_id | estado | updated_at |
|---|---|---|
| tx1 | preparing | t1 |
| tx2 | committing | t2 |
| tx3 | canceling | t3 |

Cada vez que la transacción avanza, el coordinador actualiza el ledger.

```text
preparing ──▶ committing ──▶ completed
```

El campo `updated_at` permite detectar transacciones trabadas.

## Recovery manager

Un **recovery manager** escanea periódicamente el ledger buscando transacciones que no avanzan.

```text
Recovery Manager
      │
      ▼
scan ledger
      │
      ├── tx actualizada recientemente ──▶ no hacer nada
      └── tx vieja/trabada ──▶ reasignar a otro coordinator
```

Si una transacción está en estado `preparing` o `committing` desde hace demasiado tiempo, el recovery manager puede asignarla a otro transaction coordinator.

El nuevo coordinador lee el estado del ledger y continúa desde donde quedó.

```text
TC original muere
Recovery Manager detecta tx trabada
Nuevo TC lee ledger
Nuevo TC continúa prepare/commit/cancel
```

Para que esto funcione, los storage nodes deben implementar operaciones idempotentes.

```text
prepare(tx_id) repetido ──▶ mismo efecto
commit(tx_id) repetido ──▶ mismo efecto
cancel(tx_id) repetido ──▶ mismo efecto
```

Puede ocurrir, en casos raros, que dos coordinadores intenten coordinar la misma transacción. El diseño debe tolerarlo.

## Optimistic concurrency control con timestamp ordering

DynamoDB usa una forma de control optimista basada en **timestamp ordering**.

La idea clave es:

```text
el orden serial de las transacciones se define a priori por timestamp
```

Cuando una transacción empieza, el transaction coordinator le asigna un timestamp.

```text
Tx1 ── timestamp 11
Tx2 ── timestamp 12
Tx3 ── timestamp 14
```

Ese timestamp define el orden serial conceptual:

```text
Tx1 antes que Tx2 antes que Tx3
```

No hace falta que exista un log global con todas las transacciones. Cada participante valida localmente si la operación que le llega respeta ese orden.

## Uso de relojes físicos

A diferencia de los relojes lógicos de Lamport o los vector clocks, acá se pueden usar timestamps físicos.

La razón es que no se busca inferir causalidad entre eventos distribuidos. El timestamp no intenta descubrir qué ocurrió antes en el mundo real. El timestamp asigna un orden serial que el sistema se compromete a respetar.

Hay un riesgo práctico: si una máquina tiene el reloj muy adelantado, puede asignar un timestamp enorme. Luego, muchas transacciones normales podrían parecer viejas y ser rechazadas.

Por eso igual conviene mantener relojes razonablemente sincronizados, pero el algoritmo no depende de usar relojes lógicos para capturar causalidad.

## Validación por timestamp en storage nodes

Cada storage node guarda, junto al item, el timestamp de la última escritura.

```text
key | value | write_ts
----|-------|---------
k1  | v1    | 10
```

Si llega un `prepare` para escribir el mismo item con timestamp 11:

```text
prepare(put(k1, v2), ts = 11)
```

Como `11 > 10`, la operación respeta el orden serial y puede aceptarse.

```text
OK
```

Si llega una operación con timestamp 9:

```text
prepare(put(k1, v3), ts = 9)
```

Como `9 < 10`, la operación es vieja respecto de la última escritura conocida. Se rechaza.

```text
NO
```

Con que un solo participante rechace el `prepare`, la transacción completa debe abortar.

```text
un participante responde NO
        │
        ▼
abort global
```

Las escrituras no transaccionales también actualizan el timestamp del item. Por eso un `put` común puede hacer que una transacción posterior sea rechazada si intenta escribir con un timestamp más viejo.

## Thomas write rule

Una optimización clásica es la **Thomas write rule**.

Supongamos este orden conceptual:

```text
T = 8    write viejo
T = 9    write que llega tarde
T = 10   put más nuevo
```

Si una escritura con timestamp 9 llega después de que ya existe un `put` con timestamp 10, normalmente parecería vieja y se rechazaría.

Pero si la operación de timestamp 10 pisa completamente el valor, entonces la escritura de timestamp 9 no afectaría el resultado final. Aunque se hubiera ejecutado en su posición serial correcta, luego habría sido sobrescrita por `T = 10`.

En ese caso puede responderse `OK` sin aplicar realmente la escritura vieja.

```text
prepare(write, ts = 9)
último put posterior: ts = 10
        │
        ▼
OK, pero la operación se descarta / no cambia el estado final
```

La regla evita abortar transacciones innecesariamente.

## Deletes y tombstones

Los deletes introducen una dificultad: si se borra físicamente un item, se pierde su timestamp.

```text
antes:
key | value | write_ts
k1  | v1    | 8

 delete(k1)

 después:
 no hay fila k1
```

Si luego llega una operación con timestamp 9 sobre `k1`, no hay una fila contra la cual comparar.

Una solución clásica es usar **tombstones** o borrado lógico.

```text
key | value | write_ts | deleted
k1  | v1    | 8        | true
```

También podría guardarse:

```text
deleted_at = timestamp del delete
```

Con tombstones, el sistema puede seguir comparando timestamps. El problema es que se acumulan datos borrados. En un servicio cloud, eso desperdicia espacio y además complica el cobro: el usuario borró algo y no debería seguir pagando por ese dato indefinidamente.

## Max delete por partición

DynamoDB evita guardar tombstones para cada item. En su lugar, guarda un valor agregado: el máximo timestamp de delete visto en la partición.

```text
max_delete_ts = máximo timestamp de delete
```

Ejemplo:

| key | value | write_ts |
|---|---|---|
| k1 | v1 | 8 |
| k2 | v2 | 10 |
| k3 | v3 | 7 |

Se borran los tres items:

```text
delete(k1) con ts = 8  ──▶ max_delete_ts = 8
delete(k2) con ts = 10 ──▶ max_delete_ts = 10
delete(k3) con ts = 7  ──▶ max_delete_ts sigue 10
```

Ahora `max_delete_ts` vale:

```text
10
```

Si llega:

```text
prepare(put(k1, v4), ts = 12)
```

Como `12 > max_delete_ts`, se acepta.

Si llega:

```text
prepare(put(k2, v5), ts = 9)
```

Como `9 < max_delete_ts`, se rechaza.

Caso interesante:

```text
prepare(put(k3, vN), ts = 9)
```

Si se hubiera guardado un tombstone individual para `k3`, se compararía contra `ts = 7` y la operación podría aceptarse.

Pero como solo se guarda `max_delete_ts = 10`, se rechaza.

Esto puede producir falsos positivos: se abortan algunas transacciones que quizá eran seguras. Es una decisión de diseño para evitar guardar tombstones por cada item.

## Reglas de prepare

El `prepare` en DynamoDB valida varias condiciones.

Conceptualmente:

```text
1. Las precondiciones de la operación deben cumplirse.
2. La operación no debe violar restricciones del sistema.
3. El timestamp debe ser válido respecto del último write/delete.
4. No debe haber otra transacción en preparación incompatible.
```

Las reglas pueden ser más restrictivas de lo estrictamente necesario. Eso es aceptable en control optimista: ante la duda se aborta y el cliente reintenta.

```text
conflicto real o posible
        │
        ▼
abort + retry
```

## TransactGetItems y snapshot reads

Las lecturas transaccionales buscan leer un **snapshot** consistente del sistema.

Conceptualmente, después de cada transacción existe un estado de la base:

```text
S0 ── Tx1 ──▶ S1 ── Tx2 ──▶ S2 ── Tx3 ──▶ S3
```

Un snapshot read debería leer todos los valores desde un mismo estado.

## Lectura no snapshot

Supongamos:

```text
Tx1:
  x = 1
  y = 1

Tx2:
  x = 2
  y = 2
```

Una lectura no transaccional podría hacer:

```text
get(x) ── lee antes de Tx2 ──▶ x = 1
get(y) ── lee después de Tx2 ─▶ y = 2
```

Resultado:

```text
x = 1
y = 2
```

Ese snapshot no existió nunca. El sistema tuvo `(1,1)` y luego `(2,2)`, pero no `(1,2)`.

## Snapshot read con timestamp ordering

Una forma clásica de implementar snapshot reads con timestamp ordering es asignar un timestamp también a la lectura.

```text
read_ts = 9
get(x, read_ts = 9)
get(y, read_ts = 9)
```

Si el item tiene `write_ts = 10`, una lectura con `read_ts = 9` no puede leerlo como si estuviera en el pasado.

```text
read_ts < write_ts
        │
        ▼
rechazar / reintentar
```

Además, hay otro caso: si una lectura ya leyó un valor con timestamp 12, una escritura posterior que pretende ubicarse antes de esa lectura, por ejemplo con timestamp 11, no puede modificar ese valor sin romper el orden serial.

Para detectar eso, el algoritmo clásico guarda también un timestamp de lectura:

```text
key | value | write_ts | read_ts
x   | 2     | 10       | 12
```

Si llega:

```text
put(x, 3, write_ts = 11)
```

Se rechaza porque `11 < read_ts`. La lectura de timestamp 12 ya observó el valor anterior; permitir una escritura con timestamp 11 cambiaría el pasado del snapshot.

El problema es que una lectura pasaría a escribir metadatos (`read_ts`). Es una lectura que modifica estado.

```text
get(x)
  └── actualiza read_ts
```

Eso encarece mucho las lecturas.

## Two-phase read en DynamoDB

DynamoDB evita convertir cada lectura transaccional en una escritura de metadatos. Para eso usa una estrategia de **two-phase read**.

La idea es leer los mismos items dos veces.

### Fase 1

```text
get(x) ──▶ x = 1
get(y) ──▶ y = 2
```

Puede haber leído una combinación inconsistente.

### Fase 2

```text
get(x) ──▶ x = 2
get(y) ──▶ y = 2
```

Se compara lo leído en ambas fases.

Si algún item cambió entre fase 1 y fase 2, la lectura no fue un snapshot estable y se rechaza.

```text
fase 1: x = 1, y = 2
fase 2: x = 2, y = 2
        │
        ▼
falló, retry
```

Si ambas fases observan las mismas versiones, la lectura se acepta.

```text
fase 1: x = 2, y = 2
fase 2: x = 2, y = 2
        │
        ▼
OK
```

En la implementación, no necesariamente se comparan los valores completos. Se puede comparar una posición de log o una versión interna asociada al item. Eso evita comparar payloads grandes.

Esta estrategia puede dar falsos positivos, pero evita que cada lectura tenga que actualizar un `read_ts` persistente.

## Diferencia entre 2PC y 2PL

Hay dos algoritmos con nombres parecidos:

```text
2PL = two-phase locking
2PC = two-phase commit
```

No son lo mismo.

| Algoritmo | Problema que resuelve |
|---|---|
| Two-phase locking | Aislamiento / serializabilidad mediante locks |
| Two-phase commit | Atomicidad distribuida entre participantes |

Una transacción distribuida completa puede necesitar ambos aspectos:

```text
control de concurrencia ──▶ aislamiento
atomic commit ──▶ todo o nada
```

DynamoDB usa 2PC para atomicidad y una forma optimista de control de concurrencia basada en timestamps.

## Resumen

Las transacciones distribuidas aparecen cada vez que una operación lógica toca más de un sistema.

Ejemplos típicos:

```text
reservar asiento + emitir ticket
escribir en DB + mandar evento a una cola
actualizar varios items ubicados en distintas particiones
```

El problema principal es evitar estados intermedios.

Para eso se separan dos preocupaciones:

```text
Atomicidad:
  que todos confirmen o todos aborten
  └── two-phase commit

Aislamiento:
  que las transacciones concurrentes no produzcan resultados imposibles
  └── control de concurrencia
```

El control de concurrencia puede ser pesimista u optimista.

```text
pesimista:
  usa locks
  puede bloquear
  puede tener deadlocks

optimista:
  no bloquea preventivamente
  valida antes de confirmar
  puede abortar y pedir retry
```

DynamoDB implementa transacciones con:

```text
transaction coordinator
ledger persistente
recovery manager
two-phase commit
optimistic concurrency control con timestamp ordering
two-phase read para lecturas transaccionales
```

El diseño prioriza latencia predecible y disponibilidad operativa. En vez de ofrecer transacciones SQL arbitrarias, ofrece un conjunto acotado de operaciones transaccionales sobre items. Esa restricción simplifica el protocolo y permite mantener el servicio dentro de objetivos de latencia y escalabilidad.
