# Clase 10 — Dynamo y relojes lógicos

## [Apuntes de Emma](https://drive.google.com/file/d/1RCF1AN8yZkZp08Fi8F1DB2khwP3Gpir-/view)

## Dynamo original y DynamoDB

Dynamo es el sistema descripto por Amazon en el paper de 2007. No debe confundirse con DynamoDB. DynamoDB toma el nombre por razones de marketing y porque el paper original fue muy influyente, pero el producto administrado de AWS terminó teniendo diferencias importantes respecto del diseño original.

Dynamo aparece como respuesta a un problema concreto de Amazon: necesitaban una base de datos distribuida para sistemas críticos como el carrito de compras, sesiones de usuario y catálogo de productos. En esos casos, una caída o una latencia impredecible impacta directamente en ventas. Por eso el objetivo central no era tener la consistencia más fuerte posible, sino mantener el sistema siempre disponible y con latencia baja y predecible.

## Problema con el esquema relacional clásico

Una forma tradicional de escalar una base relacional es usar un esquema `writer / reader`, también conocido como `primary / secondary`.

```text
             writes
               │
            Writer
           /   |   \
      Reader Reader Reader
          reads / replicas
```

Las escrituras van al `writer` y las lecturas pueden repartirse entre réplicas `reader`. Esto es simple y funciona bien para muchos escenarios, pero tiene dos problemas fuertes:

- Si falla el `writer`, se dejan de aceptar escrituras hasta detectar la falla, promover un nuevo writer y reconectar clientes.
- El `writer` es difícil de escalar si aumentan mucho las escrituras.

Para un carrito de compras, dejar de aceptar escrituras es especialmente problemático. La decisión de diseño de Dynamo es priorizar que las escrituras siempre sean aceptadas, incluso si eso implica resignar consistencia fuerte.

## Objetivo principal: disponibilidad antes que consistencia

El objetivo principal puede resumirse así:

```text
writes > consistencia
```

Es más importante aceptar una escritura que bloquear el sistema para mantener una única visión linealizable de los datos.

Esto lleva a tres ideas centrales:

1. **Consistencia eventual**: el sistema no es linealizable. Distintas réplicas pueden ver valores diferentes durante un tiempo.
2. **Siempre writable**: cualquier nodo debe poder aceptar escrituras.
3. **Conflictos de escritura permitidos**: si dos partes del sistema escriben concurrentemente sobre la misma clave, el sistema no descarta automáticamente una de las versiones.

La consecuencia más llamativa es que Dynamo admite situaciones parecidas a un *split brain*. Si una partición de red divide el sistema en dos mitades, ambas mitades pueden seguir aceptando escrituras. Luego, cuando la partición se repara, el sistema debe reconciliar las versiones divergentes.

## CAP

El teorema CAP plantea tres propiedades:

- **Consistency**: todos ven una visión consistente de los datos.
- **Availability**: el sistema responde a las operaciones.
- **Partition tolerance**: el sistema tolera particiones de red.

En sistemas distribuidos reales, la tolerancia a particiones no es opcional: las redes fallan y las particiones pueden ocurrir. Por eso, ante una partición, el sistema debe elegir entre mantener consistencia fuerte o mantener disponibilidad.

Raft elige consistencia: la partición minoritaria no puede avanzar. Dynamo elige disponibilidad: las particiones pueden seguir aceptando escrituras, pero eso introduce conflictos que después deben resolverse.

```text
Raft:    CP → consistencia + tolerancia a particiones
Dynamo:  AP → disponibilidad + tolerancia a particiones
```

## NoSQL: qué se gana y qué se pierde

Dynamo también populariza la idea de sistemas NoSQL. NoSQL no significa simplemente “no usar SQL” por capricho, sino reducir funcionalidades difíciles de distribuir.

Una base relacional con SQL es muy versátil: permite joins, filtros complejos, agregaciones, transacciones y consultas ad hoc. Esa versatilidad tiene costo. Distribuir una base con ese nivel de expresividad es mucho más difícil.

Dynamo simplifica la interfaz:

```text
put(key, value)
get(key)
```

Es un **key-value store**. La base funciona como un gran diccionario distribuido: una clave identifica un valor opaco, normalmente una secuencia de bytes.

Lo que se pierde:

- No hay lenguaje SQL.
- No hay joins.
- No hay transacciones ACID generales.
- La atomicidad fuerte se limita a cada clave individual.
- La aplicación debe encargarse de más lógica.

Lo que se gana:

- Es mucho más fácil particionar los datos.
- Cada clave puede tratarse de forma independiente.
- No hace falta coordinar operaciones complejas entre múltiples shards.
- El sistema puede seguir aceptando escrituras aun ante fallas o particiones.

## Por qué key-value simplifica la distribución

Si cada clave es independiente, no hace falta mantener un orden global de todas las operaciones del sistema. Lo importante es preservar el orden de las operaciones que afectan a una misma clave.

Por ejemplo, para una clave `k` importa que ocurra:

```text
add(k) → update(k) → delete(k)
```

Pero no necesariamente importa cómo se intercalan esas operaciones con operaciones sobre otras claves independientes.

Esto relaja el problema. En Raft se mantiene un log global: todas las operaciones se ordenan en una única secuencia. Dynamo evita ese log global y trabaja con orden parcial por clave.

## Particionado con hashing común

Una forma simple de repartir claves entre servidores es aplicar un hash y hacer módulo por la cantidad de servidores:

```text
server = md5(key) % N
```

Si hay tres servidores:

```text
S0, S1, S2
```

entonces cada clave cae en uno de ellos.

El problema aparece al agregar un nuevo servidor. Si `N` pasa de 3 a 4, cambia el resultado de `md5(key) % N` para muchísimas claves. Eso obliga a mover gran parte de los datos entre servidores, proceso conocido como *rehashing*.

En sistemas grandes, agregar una máquina no puede implicar redistribuir casi todo el sistema.

## Consistent hashing

Dynamo usa **consistent hashing**. La idea es imaginar un anillo de hashes:

```text
0 ───────────────────────────── 2^128 - 1
└──────────────────────────────────────┘
```

Cada servidor ocupa una o más posiciones del anillo. Una clave también cae en una posición del anillo. Para saber qué servidor la aloja, se avanza en sentido horario hasta encontrar el primer nodo.

```text
        S0
     /      \
 key         S1
     \      /
        S2
```

Cuando se agrega un nuevo nodo, solo se redistribuye una porción del anillo, no todas las claves del sistema.

```text
Antes:
S0 ───── S1 ───── S2

Después:
S0 ── S3 ── S1 ── S2
```

El nodo nuevo recibe solo parte del rango que antes pertenecía al siguiente nodo del anillo.

## Nodos virtuales

En la práctica, cada servidor físico no aparece una sola vez en el anillo. Aparece muchas veces como varios **nodos virtuales**.

Esto mejora la distribución de carga porque:

- reduce la varianza de cuántas claves recibe cada servidor;
- permite que al agregar un servidor nuevo, varios servidores existentes le transfieran partes pequeñas en paralelo;
- permite asignar más nodos virtuales a máquinas más grandes y menos a máquinas más chicas.

## Replicación y preference list

Dynamo no guarda cada clave en un único nodo. Para cada clave se define una lista de preferencia (*preference list*) con `N` nodos sucesivos del anillo.

Si `N = 3`, una clave se guarda en tres nodos:

```text
key cae acá
    ↓
   S0 → S1 → S2

preference list = [S0, S1, S2]
```

El primer nodo de la preference list no es un líder fijo. Cualquier nodo de esa lista puede coordinar una escritura. Se elige un coordinador para esa operación y ese coordinador replica la escritura a los demás nodos de la lista.

Esto es una diferencia importante con Raft:

- No hay un líder único.
- No hay log global.
- No hay primary fijo.
- No se usa Raft ni Paxos.
- Se tolera que distintas partes acepten escrituras concurrentes.

## Replicación en Dynamo

En Dynamo, una escritura puede llegar a cualquier nodo de la preference list. Ese nodo actúa como coordinador para esa operación.

```text
put(k, v)
   │
   ▼
coordinador
   ├── replica 1
   ├── replica 2
   └── replica 3
```

Si hay una partición, puede ocurrir que dos coordinadores distintos acepten escrituras sobre la misma clave al mismo tiempo. El sistema no intenta impedirlo. En cambio, detecta luego que esas escrituras son concurrentes y conserva ambas versiones.

La pregunta central pasa a ser:

```text
¿Cómo sabemos si una escritura ocurrió antes que otra
o si dos escrituras son concurrentes?
```

Para responder eso aparecen los relojes lógicos.

## Por qué no alcanza con relojes físicos

Una idea inicial sería usar el timestamp de cada máquina. Pero eso no funciona bien en sistemas distribuidos porque los relojes físicos no están perfectamente sincronizados.

Ejemplo:

```text
P1: evento e1 a las 10:03
P2: evento e2 a las 10:00
```

Desde afuera podría ser claro que `e1` ocurrió antes que `e2`, pero si los relojes están desfasados, ordenar por timestamp físico daría el resultado contrario.

El problema es el **clock skew**, la diferencia entre relojes físicos.

```text
ε = T2 - T1
```

Para estimar esa diferencia, una máquina puede pedirle la hora a otra:

```text
P1 ─── get_time ───▶ P2
P1 ◀──── time ───── P2
```

Pero no se sabe exactamente cuánto tardó el mensaje de ida ni cuánto tardó la respuesta. La red puede tener colas, routers congestionados, caminos distintos para ida y vuelta, etc.

NTP permite sincronizar relojes de forma aproximada, pero no brinda una garantía matemática perfecta. Para razonar sobre algoritmos distribuidos no alcanza con “relojes más o menos sincronizados”.

## Secuenciador centralizado

Una alternativa es usar un servidor centralizado que entregue números de secuencia.

```text
P1 ── pide timestamp ─▶ T
P1 ◀────── 1 ───────── T

P2 ── pide timestamp ─▶ T
P2 ◀────── 2 ───────── T
```

Esto evita depender del reloj físico. El número no representa tiempo real, sino orden lógico.

La desventaja es clara: el secuenciador central es un punto único de falla y puede convertirse en cuello de botella.

Google File System usa ideas de este estilo cuando el primary asigna números de secuencia a las mutaciones. No siempre está mal centralizar; simplemente es una decisión de diseño con costos.

## Happens-before

Lamport propone pensar el orden no como tiempo físico, sino como causalidad.

La relación **happens-before** se escribe:

```text
a → b
```

y significa que el evento `a` ocurrió antes que `b` en un sentido causal.

Se define con tres reglas:

1. Si dos eventos ocurren en el mismo proceso y `a` aparece antes que `b`, entonces `a → b`.
2. Si un proceso envía un mensaje y otro lo recibe, el envío ocurre antes que la recepción.
3. La relación es transitiva: si `a → b` y `b → c`, entonces `a → c`.

Ejemplo:

```text
P1:  a ─────── send ─────────────
                  │
                  ▼
P2:              receive ─── b
```

El envío ocurre antes que la recepción, y todo lo que sucede después de la recepción puede estar causalmente influenciado por el mensaje.

Si entre dos eventos no puede construirse una cadena de causalidad, se consideran **concurrentes**:

```text
a || b
```

Concurrente no significa necesariamente “al mismo tiempo físico”. Significa que no hay relación causal conocida entre ambos eventos.

## Relojes lógicos de Lamport

Un reloj lógico no cuenta segundos. Cuenta eventos.

Cada proceso mantiene un contador. Inicialmente vale `0`.

Reglas:

1. Antes de cada evento local, el proceso incrementa su contador.
2. Al enviar un mensaje, incluye el valor actual del contador.
3. Al recibir un mensaje con timestamp `t`, actualiza su reloj así:

```text
C_local = max(C_local, t) + 1
```

Ejemplo:

```text
P1: C=0
evento local → C=1
envía mensaje con C=1

P2: C=0
recibe mensaje con 1
C=max(0,1)+1 = 2
```

La condición que garantizan los relojes de Lamport es:

```text
si a → b, entonces C(a) < C(b)
```

Es decir: si un evento ocurrió causalmente antes que otro, su reloj lógico será menor.

Pero no vale la inversa:

```text
C(a) < C(b) no implica necesariamente a → b
```

Dos eventos concurrentes pueden terminar con valores de reloj distintos. Por eso los relojes de Lamport sirven para construir un orden compatible con la causalidad, pero no permiten distinguir perfectamente causalidad de concurrencia.

## Orden parcial y orden total

La relación happens-before define un **orden parcial**.

En un orden total, todos los elementos se pueden comparar entre sí. Los números naturales son un ejemplo: dados dos números, siempre se puede decir cuál es menor, mayor o si son iguales.

En un orden parcial, no todos los pares son comparables. Algunos eventos tienen relación causal y otros son concurrentes.

```text
a → b    comparables
a || c   no comparables causalmente
```

Con relojes de Lamport se puede construir un orden total artificial: se ordena por reloj lógico y, si hay empate, por ID de proceso. Ese orden total respeta las relaciones causales reales, pero también impone un orden arbitrario entre eventos concurrentes.

## Ejemplo: chat distribuido

Supongamos un chat con tres participantes:

```text
A: ¿Quién viene a comer?
B: Yo voy
C: Qué calor
```

El mensaje de B responde al de A, por lo tanto:

```text
A → B
```

El mensaje de C no tiene relación causal con esos mensajes. Es concurrente.

```text
C || A
C || B
```

Si los mensajes llegan en distinto orden a cada cliente por demoras de red, cada uno podría ver una historia distinta. Usando relojes de Lamport, cada mensaje se etiqueta con un timestamp lógico y luego se ordena de manera determinista.

La garantía importante es que A aparecerá antes que B. El mensaje de C puede aparecer antes, entre medio o después, porque no tiene relación causal con los otros.

## Limitación de Lamport

Los relojes de Lamport garantizan:

```text
a → b  =>  C(a) < C(b)
```

Pero no permiten concluir:

```text
C(a) < C(b)  =>  a → b
```

Esto limita su utilidad para detectar conflictos. Si dos eventos tienen timestamps distintos, no se sabe si uno causó al otro o si simplemente son concurrentes y quedaron ordenados artificialmente.

Dynamo necesita algo más potente: debe poder saber si una versión de una clave reemplaza causalmente a otra o si ambas son concurrentes. Para eso usa relojes vectoriales.

## Vector clocks

Un **vector clock** reemplaza el contador único por un vector de contadores. Si hay `N` nodos, cada vector tiene `N` posiciones.

Ejemplo con tres nodos:

```text
[0, 0, 0]
```

Cada nodo incrementa su propia posición.

Reglas:

1. Inicialmente todos los valores son `0`.
2. Antes de un evento local, el nodo incrementa su componente.
3. Cada mensaje incluye el vector completo.
4. Al recibir un mensaje, se hace un merge componente a componente, tomando el máximo de cada posición, y luego se incrementa la posición propia si corresponde.

Ejemplo:

```text
P1 empieza con [0,0,0]
evento local en P1 → [1,0,0]
otro evento en P1  → [2,0,0]

P1 envía [2,0,0] a P2
P2 tenía [0,0,0]
P2 recibe:
max([0,0,0], [2,0,0]) = [2,0,0]
incrementa P2 → [2,1,0]
```

## Comparación de vector clocks

Para comparar dos vectores `A` y `B`:

```text
A <= B si para todo i: A[i] <= B[i]
A < B si A <= B y A != B
```

Entonces:

```text
A < B  ⇔  A ocurrió antes que B
```

Si no se cumple `A <= B` ni `B <= A`, entonces los eventos son concurrentes.

Ejemplo causal:

```text
A = [1,0,0]
B = [2,1,0]

A < B porque:
1 <= 2
0 <= 1
0 <= 0
y al menos una posición es estrictamente menor
```

Ejemplo concurrente:

```text
A = [1,1,0]
B = [0,1,1]
```

Comparación:

```text
A[0] > B[0]
A[2] < B[2]
```

Ninguno domina al otro. Por lo tanto:

```text
A || B
```

Esta es la propiedad que Lamport no daba y que Dynamo necesita.

## Vector clocks en Dynamo

En Dynamo, cada clave tiene asociado un vector clock.

Para un ítem nuevo:

```text
put(k1, v1)
```

Si el coordinador es el nodo 2, inicializa o actualiza el vector incrementando su posición:

```text
k1 → (v1, [0,1,0])
```

Luego replica ese valor y ese vector a los otros nodos de la preference list.

## Updates en Dynamo

Una actualización no debería hacerse “a ciegas”. Primero se lee la clave:

```text
get(k1)
```

La respuesta incluye:

```text
valor actual
contexto opaco
```

Ese contexto opaco contiene el vector clock asociado. El cliente no necesita interpretarlo; simplemente lo devuelve cuando hace el `put` siguiente.

```text
get(k1) → (v1, contexto=[0,1,0])

put(k1, v2, contexto=[0,1,0])
```

El coordinador usa ese contexto para saber si la nueva escritura desciende causalmente de la versión anterior.

Si la versión nueva domina a la vieja, Dynamo puede reemplazar la vieja. Eso es reconciliación sintáctica.

## Escrituras concurrentes

Supongamos que la clave `k1` está en tres nodos y la versión conocida es:

```text
k1 → v1, [0,1,0]
```

Ahora ocurren dos escrituras concurrentes, una coordinada por `S1` y otra por `S3`.

```text
S1 escribe v2 → [1,1,0]
S3 escribe v3 → [0,1,1]
```

Comparación:

```text
[1,1,0] y [0,1,1] son concurrentes
```

Ninguna versión domina a la otra. Dynamo no puede decidir automáticamente cuál es la correcta. Entonces conserva ambas versiones.

```text
k1 →
  v2, [1,1,0]
  v3, [0,1,1]
```

Estas versiones paralelas suelen llamarse *siblings*.

## Reconciliación sintáctica

La reconciliación sintáctica ocurre cuando Dynamo puede resolver el conflicto solo mirando los vector clocks.

Si una versión domina a otra:

```text
[1,1,0] < [2,1,0]
```

entonces la segunda viene causalmente después de la primera y puede reemplazarla.

```text
vieja: v1, [1,1,0]
nueva: v2, [2,1,0]

resultado:
v2, [2,1,0]
```

No hace falta entender el contenido del valor.

## Reconciliación semántica

Cuando hay versiones concurrentes, los vector clocks detectan el conflicto, pero no dicen cómo resolverlo.

Ahí Dynamo devuelve todas las versiones al cliente:

```text
get(k1) →
  v2, [1,1,0]
  v3, [0,1,1]
```

El cliente debe usar lógica de negocio para combinarlas. Esto es reconciliación semántica.

Dynamo no interpreta los valores. Para Dynamo son bytes. Solo la aplicación sabe qué significan.

## Ejemplo: carrito de compras

Un caso clásico es el carrito de compras.

Estado inicial:

```text
carrito = [pan]
```

Dos operaciones concurrentes:

```text
S1: put(user1, [pan, queso], [1,1,0])
S3: put(user1, [pan, leche], [0,1,1])
```

Dynamo detecta que los vectores son concurrentes y guarda ambas versiones:

```text
user1 →
  [pan, queso], [1,1,0]
  [pan, leche], [0,1,1]
```

Cuando el cliente hace:

```text
get(user1)
```

recibe las dos versiones. La aplicación decide combinarlas con una unión de listas:

```text
[pan, queso] ∪ [pan, leche] = [pan, queso, leche]
```

Luego escribe la versión reconciliada:

```text
put(user1, [pan, queso, leche], [1,1,1])
```

El vector `[1,1,1]` subsume a los anteriores porque toma el máximo componente a componente:

```text
max([1,1,0], [0,1,1]) = [1,1,1]
```

Así se cierra el conflicto.

## Qué problema se resolvió

El sistema permite que el split brain avance. Durante una partición, distintas mitades pueden aceptar escrituras sobre la misma clave. Luego, al repararse la red, los vector clocks permiten distinguir dos casos:

1. Una versión viene causalmente después de otra.
2. Las versiones son concurrentes y hay conflicto real.

En el primer caso, Dynamo resuelve solo. En el segundo, delega la reconciliación a la aplicación.

```text
causalidad      → reconciliación sintáctica
concurrencia    → reconciliación semántica
```

## Conclusión

Dynamo cambia el objetivo respecto de sistemas como Raft. En vez de impedir divergencias mediante un líder, un log global y quórums fuertes, acepta que las divergencias ocurran y diseña mecanismos para detectarlas y reconciliarlas.

Las ideas principales son:

- consistencia eventual;
- alta disponibilidad;
- escrituras siempre aceptadas;
- particionado con consistent hashing;
- replicación mediante preference lists;
- ausencia de líder fijo;
- conflictos permitidos;
- detección de causalidad con vector clocks;
- reconciliación sintáctica cuando hay dominancia causal;
- reconciliación semántica cuando hay concurrencia real.

La clase siguiente continúa con Dynamo: quórums relajados, gossip para que los nodos conozcan el estado del anillo y comparación con DynamoDB, que terminó usando un diseño bastante diferente al Dynamo original.
