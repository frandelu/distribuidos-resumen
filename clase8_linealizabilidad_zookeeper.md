# Clase 8 — Linealizabilidad y ZooKeeper

## 1. Consistencia en sistemas replicados

En un sistema replicado hay varias copias de los datos. Esa replicación permite tolerar fallas y aumentar disponibilidad, pero introduce un problema central: no todas las réplicas ven las actualizaciones exactamente al mismo tiempo.

En Raft, las escrituras deben pasar por el líder. Si un cliente intenta escribir en un follower, ese nodo debería redirigir la operación al líder o responder un error. La escritura necesita pasar por el log del líder, replicarse en una mayoría y recién entonces considerarse confirmada.

Con las lecturas aparece una tentación distinta: leer desde los followers para aprovechar más máquinas y aumentar throughput. Esto es especialmente atractivo en sistemas donde hay muchas más lecturas que escrituras. El problema es que un follower puede estar atrasado respecto del líder.

## 2. Lecturas desde followers y consistencia eventual

Supongamos un cluster Raft con un líder y dos followers.

1. Un cliente escribe `x = 1` en el líder.
2. El líder agrega la operación al log.
3. El líder envía `AppendEntries(x = 1)` a los followers.
4. Cuando recibe confirmación de una mayoría, responde `OK` al cliente.
5. El líder ya puede ver `x = 1`.
6. Los followers todavía pueden no haber aplicado el commit.

Si inmediatamente después el cliente lee desde el líder, obtiene `x = 1`. Pero si lee desde un follower que todavía no aplicó el commit, puede obtener un valor viejo, por ejemplo `x = 0`.

Esto genera una historia como:

```text
Cliente 1:  write(x = 1)  -> OK
Cliente 1:  read(x)       -> 1
Cliente 1:  read(x)       -> 0
```

El cliente primero ve el valor nuevo y después ve un valor viejo. Eso no se comporta como una única copia de los datos.

Este comportamiento es una forma de **consistencia eventual**: si el sistema deja de recibir escrituras y la red funciona, eventualmente todas las réplicas convergen al mismo valor. Pero durante un período de tiempo, algunas lecturas pueden devolver información vieja.

La consistencia eventual no significa que la información se actualice “algún día”. En sistemas reales, se espera que la convergencia ocurra en segundos o milisegundos, salvo fallas o particiones de red. El problema no es tanto el tiempo en sí, sino que el orden observable de las operaciones puede ser impredecible.

## 3. Consistencia fuerte y linealizabilidad

La consistencia fuerte que interesa en este contexto se formaliza como **linealizabilidad**.

Un sistema linealizable se comporta como si hubiera una sola copia de los datos, aunque internamente esté replicado.

Una base de datos tradicional en una sola máquina se comporta así de manera natural porque realmente hay una única copia lógica. En un sistema distribuido hay que diseñar mecanismos adicionales para obtener esa propiedad.

La linealizabilidad exige dos cosas:

1. **Respetar el tiempo real**: si una operación termina antes de que otra empiece, el sistema debe observarlas en ese orden.
2. **Ser válida lógicamente**: las lecturas deben devolver valores compatibles con el orden elegido de escrituras.

La idea es que cada operación distribuida, aunque dure un intervalo de tiempo, puede pensarse como si hubiera ocurrido instantáneamente en algún punto entre su request y su response.

## 4. Operaciones no concurrentes

Si una escritura termina antes de que empiece una lectura, la lectura debe ver esa escritura.

```text
Cliente 1:  write(x = 1)  -> OK

Cliente 2:                    read(x) -> 1
```

Esto es linealizable.

En cambio:

```text
Cliente 1:  write(x = 1)  -> OK

Cliente 2:                    read(x) -> 0
```

No es linealizable, porque la lectura ocurre después de que la escritura terminó y aun así ve un valor viejo.

## 5. Operaciones concurrentes

Cuando las operaciones se solapan en el tiempo, puede haber más de un orden válido.

Ejemplo:

```text
Cliente 1:  write(x = 1) ------------- OK
Cliente 2:        read(x) ------> 1
```

Esto puede ser linealizable si se considera que la escritura se materializó antes de la lectura.

También puede pasar:

```text
Cliente 1:  write(x = 1) ------------- OK
Cliente 2:        read(x) ------> 0
```

Esto también puede ser linealizable si se considera que la lectura se materializó antes que la escritura. Aunque el request de lectura se haya solapado con la escritura, todavía no había garantía de que la escritura ya estuviera confirmada al momento exacto en que ocurrió la lectura.

La clave es encontrar un orden instantáneo posible dentro de los intervalos de ejecución.

## 6. Ejemplos de historias linealizables

### Ejemplo 1

```text
Cliente 1: write(x = 1)
Cliente 2: read(x) -> 2
Cliente 1: write(x = 2)
```

Es linealizable si el orden elegido es:

```text
write(x = 1)
write(x = 2)
read(x) -> 2
```

### Ejemplo 2

```text
Cliente 1: write(x = 1)
Cliente 1: write(x = 2)

Cliente 2: read(x) -> 2
Cliente 3: read(x) -> 1
```

Puede ser linealizable si las operaciones se solapan y el orden instantáneo elegido permite que una lectura ocurra antes de `write(x = 2)` y la otra después.

Un orden posible:

```text
write(x = 1)
read(x) -> 1
write(x = 2)
read(x) -> 2
```

### Ejemplo no linealizable

```text
Cliente 1: write(x = 1)
Cliente 1: write(x = 2)

Cliente 2: read(x) -> 2
Cliente 3: read(x) -> 1
```

Si por los intervalos temporales la lectura que devuelve `2` necesariamente ocurre antes que la lectura que devuelve `1`, no hay forma de ordenar las operaciones sin violar la lógica de los valores. No se puede leer `2` y después volver a leer `1` si no hubo una nueva escritura que restaure ese valor.

## 7. Raft y lecturas linealizables

En Raft, las escrituras son linealizables si pasan por el líder y se confirman en una mayoría.

Para las lecturas, leer del líder parece suficiente, pero no siempre lo es.

Puede ocurrir una partición de red:

```text
Partición A: líder viejo + minoría
Partición B: mayoría + nuevo líder
```

El líder viejo puede seguir creyendo que es líder. Si un cliente le pregunta por un valor, ese nodo puede responder con información vieja. Aunque el cliente “leyó del líder”, leyó de un líder obsoleto.

Por eso, para que una lectura sea estrictamente linealizable, el líder debe asegurarse de que sigue siendo líder.

La forma más segura es tratar la lectura como una operación en el log:

1. El cliente envía `read(x)` al líder.
2. El líder agrega una entrada de lectura o una operación `no-op` al log.
3. El líder replica esa entrada en una mayoría.
4. Cuando obtiene quórum, sabe que sigue siendo líder.
5. Recién entonces responde la lectura.

Esto garantiza linealizabilidad, pero es costoso: una lectura pasa a tener un costo parecido al de una escritura.

## 8. Optimización con leases

Para evitar confirmar cada lectura mediante un round-trip de quórum, se puede usar una idea similar a los **leases**.

La idea:

1. El líder verifica cada cierto tiempo que sigue teniendo mayoría.
2. Durante una ventana temporal corta, puede responder lecturas localmente.
3. Un nuevo líder espera esa misma ventana antes de aceptar escrituras.

Ejemplo conceptual:

```text
El líder chequea liderazgo cada N segundos.
Un nuevo líder no acepta escrituras durante N segundos.
```

Si una partición ocurre, el líder viejo puede seguir respondiendo lecturas por un tiempo, pero el nuevo líder todavía no acepta escrituras. Eso evita que las lecturas del líder viejo contradigan escrituras nuevas.

Esta técnica mejora performance, pero depende de supuestos temporales y de una configuración cuidadosa.

## 9. ZooKeeper como servicio de coordinación

Implementar un algoritmo de consenso es difícil. Una alternativa práctica es encapsularlo detrás de un servicio separado.

En vez de que cada aplicación implemente Raft o Paxos internamente, se usa un servicio de coordinación:

```text
Aplicación  --->  Servicio de coordinación
                    |
                    +-- nodos replicados con consenso
```

El servicio expone una API simple y esconde la complejidad de consenso, replicación, fallas y elección de líder.

ZooKeeper cumple ese rol.

## 10. Antecedentes: Chubby, ALF y ZooKeeper

Distintas empresas construyeron sistemas de coordinación similares:

- **Chubby**: sistema de Google, basado en Paxos, orientado a locks distribuidos.
- **ALF**: sistema interno de Amazon, también basado en Paxos.
- **ZooKeeper**: sistema de Yahoo, basado en ZAB, un algoritmo de consenso parecido a Raft.

ZooKeeper no usa Raft, pero ZAB tiene ideas similares: líder, log replicado, quórums y aplicación ordenada de operaciones.

## 11. Modelo de datos de ZooKeeper

ZooKeeper expone una estructura jerárquica parecida a un file system.

Los nodos se organizan por paths:

```text
/
├── config
│   ├── db
│   └── cache
└── locks
    └── lock-0001
```

Cada nodo puede tener:

- nombre,
- datos,
- hijos,
- versión,
- flags especiales.

La estructura parece un file system, pero está pensada para coordinación, no para almacenar grandes volúmenes de datos.

## 12. API básica de ZooKeeper

Operaciones principales:

```text
create(path, data, flags)
delete(path, version)
exists(path, watch)
getData(path, watch)
setData(path, data, version)
getChildren(path, watch)
sync(path)
```

No hay una operación directa de “crear lock” o “elegir líder”. Esas primitivas se construyen usando estas operaciones básicas.

## 13. Objetivo de diseño: muchas lecturas, pocas escrituras

ZooKeeper está pensado para workloads dominados por lecturas.

Un caso típico puede tener muchas más lecturas que escrituras: desde el doble hasta cien veces más lecturas que escrituras.

Para lograr alto throughput de lectura, ZooKeeper permite leer desde followers. Eso aumenta performance, pero implica descartar linealizabilidad completa en las lecturas.

ZooKeeper mantiene un modelo intermedio:

- escrituras linealizables,
- lecturas no necesariamente linealizables,
- garantías por cliente mediante FIFO client order.

## 14. Escrituras linealizables

Las escrituras en ZooKeeper pasan por el líder y se aplican en el mismo orden en todas las réplicas.

Esto permite usar replicación de máquina de estados: todos los nodos aplican las mismas mutaciones en el mismo orden y llegan al mismo estado.

Las escrituras también permiten mecanismos como optimistic locking, porque una escritura puede depender de una versión previa del dato.

## 15. FIFO client order

ZooKeeper no garantiza que todos los clientes vean siempre la versión más nueva, pero sí garantiza propiedades útiles para cada cliente individual.

### Read your writes

Si un cliente escribe un valor y luego lee ese mismo dato, debe ver su propia escritura o una versión posterior.

Ejemplo válido:

```text
Cliente 1: write(x = 1) -> OK
Cliente 1: read(x)      -> 1
```

No debería ocurrir:

```text
Cliente 1: write(x = 1) -> OK
Cliente 1: read(x)      -> 0
```

Desde la perspectiva del mismo cliente, no se vuelve al pasado.

### Monotonic reads

Si un cliente leyó una versión de un dato, sus lecturas posteriores no pueden devolver versiones más viejas.

Ejemplo:

```text
Cliente 1: read(x) -> 1
Cliente 1: read(x) -> 1
Cliente 1: read(x) -> 2
```

Eso es válido.

No debería pasar:

```text
Cliente 1: read(x) -> 1
Cliente 1: read(x) -> 0
```

El cliente puede ver información vieja respecto de otros clientes, pero no puede retroceder respecto de lo que él mismo ya observó.

## 16. ZXID

ZooKeeper expone una noción clave: el **ZXID**.

El ZXID identifica la posición de una operación en el log. Es una forma de versionar el estado global del sistema.

Cuando un cliente escribe:

```text
write(x = 1)
```

ZooKeeper puede responder:

```text
OK, zxid = 100
```

El cliente recuerda ese `zxid`.

Si luego quiere leer desde un follower, envía también el último `zxid` que conoce. El follower solo puede responder cuando está actualizado al menos hasta ese punto.

```text
read(x, zxid >= 100)
```

Si el follower todavía está atrasado, debe esperar a ponerse al día o forzar algún mecanismo de sincronización.

Esto permite cumplir read-your-writes y monotonic reads sin exigir que todas las lecturas pasen por el líder.

## 17. Lectura consistente de una configuración

Supongamos que un cliente escribe una configuración compuesta por varios valores:

```text
write(a = b1)
write(b = b2)
write(a = b3)
create(ready)
```

`ready` funciona como marcador de finalización. No importa tanto el contenido del nodo; importa que su creación ocurre después de todas las escrituras relevantes.

Otro cliente que quiere leer la configuración hace:

```text
exists(ready)
read(a)
read(b)
```

Si `exists(ready)` devuelve verdadero, entonces el cliente sabe que está leyendo desde una réplica que avanzó al menos hasta la creación de `ready`. Por lo tanto, también debe haber visto las escrituras anteriores.

Esto evita leer una configuración incompleta.

La idea es similar a crear un archivo marcador de “terminé de escribir”. Si el marcador existe, todo lo anterior ya debería estar disponible.

## 18. Problema con actualizaciones repetidas

Si la configuración se actualiza varias veces, aparece un problema.

Secuencia posible:

```text
delete(ready)
write(a = a1)
write(b = b1)
create(ready)

delete(ready)
write(a = a2)
write(b = b2)
create(ready)
```

Un cliente podría hacer:

```text
exists(ready)
read(a)
read(b)
```

Pero si `ready` correspondía a la primera configuración y luego fue borrado y recreado, el cliente puede mezclar valores:

```text
a = a1
b = b2
```

Eso es una configuración inconsistente: mitad vieja, mitad nueva.

## 19. Watches

Para resolver este tipo de problema, ZooKeeper ofrece **watches**.

Un watch permite pedir notificación cuando cambia un nodo.

Ejemplo:

```text
exists(ready, watch = true)
```

Si `ready` cambia, ZooKeeper notifica al cliente.

Entonces, si el cliente estaba leyendo una configuración y recibe una notificación de que `ready` cambió, debe reiniciar la lectura:

```text
start:
  exists(ready, watch = true)
  read(a)
  read(b)

  si ready cambió:
    volver a start
```

Los watches permiten detectar que el estado usado como punto de sincronización dejó de ser válido.

## 20. Optimistic locking

ZooKeeper permite implementar optimistic locking usando versiones.

Problema clásico:

```text
Cliente 1: read(c) -> 1
Cliente 2: read(c) -> 1

Cliente 1: write(c = 2) -> OK
Cliente 2: write(c = 2) -> OK
```

Dos clientes incrementaron el contador, pero el resultado final quedó en `2` en vez de `3`.

Con optimistic locking, la lectura devuelve también una versión:

```text
Cliente 1: read(c) -> valor = 1, version = 1
Cliente 2: read(c) -> valor = 1, version = 1
```

La escritura se hace condicionada a esa versión:

```text
Cliente 1: write(c = 2, expectedVersion = 1) -> OK
Cliente 2: write(c = 2, expectedVersion = 1) -> ERROR
```

El segundo cliente falla porque la versión actual ya no es `1`.

Entonces debe reintentar:

```text
Cliente 2: read(c) -> valor = 2, version = 2
Cliente 2: write(c = 3, expectedVersion = 2) -> OK
```

Se llama optimistic locking porque no se bloquea preventivamente. Se intenta escribir y solo se falla si hubo una modificación concurrente.

## 21. Cuándo se chequea la versión

El chequeo de versión debe hacerse en el orden del log.

No alcanza con verificar la versión apenas llega el request, porque la aplicación local puede estar atrasada respecto del log.

El orden correcto es:

1. Llega `setData(path, value, expectedVersion)`.
2. La operación entra al log del líder.
3. Se replica y se ordena con las demás operaciones.
4. Cuando se aplica en la capa de datos, se verifica la versión.
5. Si coincide, se modifica el dato.
6. Si no coincide, la operación falla.

Esto preserva la linealizabilidad de las escrituras.

## 22. Nodos efímeros

Un nodo efímero existe mientras la sesión del cliente que lo creó siga viva.

Si el cliente se desconecta, deja de enviar heartbeats o se considera muerto, ZooKeeper elimina automáticamente sus nodos efímeros.

Esto permite representar recursos asociados a clientes vivos.

Ejemplo:

```text
/clients/client-1
```

Si `client-1` muere, el nodo desaparece.

## 23. Nodos secuenciales

Un nodo secuencial recibe automáticamente un sufijo creciente.

Si varios clientes crean nodos con el mismo prefijo:

```text
create("/locks/lock-", sequential = true)
```

ZooKeeper puede crear:

```text
/locks/lock-0001
/locks/lock-0002
/locks/lock-0003
```

El número lo asigna ZooKeeper de manera ordenada.

Esto permite construir colas distribuidas.

## 24. Locks distribuidos con ZooKeeper

Para implementar un lock distribuido:

1. Todos los clientes crean un nodo efímero y secuencial dentro de un directorio de locks.

```text
create("/locks/lock-", ephemeral = true, sequential = true)
```

2. ZooKeeper devuelve un nombre concreto:

```text
/locks/lock-0004
```

3. El cliente lista los hijos de `/locks`.

```text
getChildren("/locks")
```

4. Si su nodo tiene el número más bajo, obtuvo el lock.

```text
/locks/lock-0001  <- dueño del lock
/locks/lock-0002
/locks/lock-0003
/locks/lock-0004
```

5. Si no tiene el número más bajo, espera al nodo inmediatamente anterior.

Si el cliente tiene:

```text
/locks/lock-0004
```

debe poner un watch sobre:

```text
/locks/lock-0003
```

Cuando `/locks/lock-0003` desaparece, vuelve a listar los hijos y chequea si ahora tiene el número más bajo.

## 25. Liberación del lock

El lock se libera borrando el nodo propio:

```text
delete("/locks/lock-0001")
```

Si el cliente muere, como el nodo es efímero, ZooKeeper lo borra automáticamente. Eso evita locks abandonados para siempre.

## 26. Elección de líder con ZooKeeper

La elección de líder se puede implementar casi igual que un lock distribuido.

Cada candidato crea un nodo efímero y secuencial:

```text
create("/election/candidate-", ephemeral = true, sequential = true)
```

El candidato con el número más bajo es el líder.

Los demás observan al candidato inmediatamente anterior. Si el líder muere, su nodo efímero desaparece y el siguiente candidato puede convertirse en líder.

Así, la elección de líder queda implementada usando primitivas simples de ZooKeeper, sin que la aplicación tenga que implementar Raft o Paxos.

## 27. Uso de ZooKeeper en sistemas distribuidos

ZooKeeper sirve para guardar información de coordinación, no datos grandes.

Casos típicos:

- configuración del sistema,
- membership de nodos,
- elección de líder,
- locks distribuidos,
- barreras,
- colas,
- detección de nodos vivos mediante nodos efímeros.

En un sistema tipo DynamoDB o MiniDynamoDB, ZooKeeper puede guardar qué nodos existen, qué shards tiene cada uno y cuál es la configuración actual del cluster.

## 28. Ideas principales

- Leer desde followers mejora throughput, pero puede devolver datos viejos.
- La consistencia eventual aparece naturalmente cuando se relajan lecturas sobre réplicas.
- La linealizabilidad formaliza la idea de consistencia fuerte.
- Para que Raft sea linealizable en lecturas, no alcanza con leer del líder: hay que verificar que sigue siendo líder.
- Confirmar lecturas mediante el log es seguro, pero costoso.
- Los leases son una optimización temporal para reducir el costo de las lecturas.
- ZooKeeper encapsula consenso en un servicio de coordinación.
- ZooKeeper no es completamente linealizable para lecturas, pero ofrece garantías por cliente.
- El ZXID permite que un cliente no retroceda en el estado observado.
- Los watches permiten reaccionar ante cambios.
- Las versiones permiten optimistic locking.
- Los nodos efímeros y secuenciales permiten construir locks y elección de líder.
