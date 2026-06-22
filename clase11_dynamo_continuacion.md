# Clase 11 — Dynamo, DynamoDB y servicios cloud

## [Apuntes de Emma](https://drive.google.com/file/d/1jCvDa2nqLOKb0TSuGFuTpEDaV8D3YckR/view)

## Objetivo de Dynamo         

Dynamo fue diseñado para servicios de Amazon donde la disponibilidad era más importante que la consistencia fuerte. El caso típico era el carrito de compras: si una base de datos del carrito deja de aceptar escrituras, la pérdida económica puede ser enorme.

El objetivo operativo era muy estricto:

```text
99,9% de las operaciones con latencia < 300 ms
```

Eso no significa que el promedio tenga que ser menor a 300 ms, sino que prácticamente todos los requests deben responder rápido. Importa especialmente la cola de la distribución de latencias: no alcanza con que la mayoría sea rápida si un porcentaje relevante queda muy lento.

La otra idea central es:

```text
always writable
```

El sistema debe seguir aceptando escrituras incluso con fallas, particiones o nodos temporalmente inaccesibles. La consistencia fuerte se resigna a cambio de disponibilidad.

## Técnicas usadas en Dynamo

El paper de Dynamo resume varias técnicas distribuidas:

| Problema | Técnica | Ventaja |
|---|---|---|
| Particionado | Consistent hashing | Escalabilidad incremental |
| Alta disponibilidad para escrituras | Vector clocks con reconciliación durante lecturas | El tamaño de las versiones se desacopla del ritmo de escritura |
| Fallas temporales | Sloppy quorum y hinted handoff | Alta disponibilidad y durabilidad aunque algunas réplicas no estén disponibles |
| Fallas permanentes | Anti-entropy con Merkle trees | Sincronización eficiente entre réplicas divergentes |
| Membresía y detección de fallas | Gossip protocol | Simetría, descentralización y propagación de cambios de membresía |

La clase anterior se concentró en consistent hashing y vector clocks. Esta parte completa el diseño con sloppy quorum, hinted handoff, Merkle trees y gossip.

## Quórums clásicos

En un esquema de quórums clásico, se eligen tres parámetros:

```text
N = cantidad de réplicas
W = cantidad de réplicas necesarias para confirmar una escritura
R = cantidad de réplicas necesarias para confirmar una lectura
```

La regla usual es:

```text
R + W > N
```

Si se cumple, todo quórum de lectura se solapa con todo quórum de escritura. Ese solapamiento permite que una lectura encuentre al menos una réplica que vio la escritura más reciente.

Ejemplo típico de Dynamo:

```text
N = 3
W = 2
R = 2
```

Con `W = 2`, una escritura debería llegar a dos de las tres réplicas. Con `R = 2`, una lectura consulta dos réplicas. Como `2 + 2 > 3`, hay solapamiento.

En Raft, el quórum es estricto: si no se consigue mayoría, la operación no se considera confirmada. Dynamo relaja esa idea.

## Sloppy quorum

Dynamo usa **sloppy quorum**. La idea es intentar conseguir un quórum, pero sin exigir que las réplicas que responden sean exactamente las de la preference list original.

Una clave tiene una **preference list**, por ejemplo:

```text
preference list(k) = [S1, S2, S3]
```

Si `S2` o `S3` están caídos, el coordinador no bloquea la escritura esperando que vuelvan. Busca otros nodos disponibles del anillo y guarda temporalmente la escritura allí.

```text
Preference list original:
S1, S2, S3

S2 y S3 caídos:
S1 guarda la escritura
S4 guarda una copia temporal
```

Esto permite responder al cliente aunque no se haya escrito en las réplicas ideales. El quórum es “desprolijo”: se alcanza con nodos disponibles, no necesariamente con los nodos preferidos.

La prioridad vuelve a ser disponibilidad:

```text
mejor guardar la escritura en algún lugar durable
que rechazarla por no poder escribir justo en las réplicas ideales
```

## Coordinador de escritura

Cuando un cliente quiere escribir una clave, elige uno de los nodos de la preference list como coordinador. Ese coordinador:

1. Guarda la escritura localmente.
2. Intenta replicarla en los otros nodos de la preference list.
3. Si alguno no está disponible, usa otros nodos disponibles.
4. Responde al cliente cuando considera que logró suficiente durabilidad.

```text
put(k, v)
   │
   ▼
coordinador
   ├── replica preferida
   ├── replica preferida caída
   └── nodo alternativo disponible
```

No hay un líder fijo para una clave. Distintas operaciones pueden tener coordinadores distintos.

## Lecturas con quórum

Las lecturas también usan un coordinador. El cliente manda el pedido a un nodo, y ese nodo consulta varias réplicas.

Con `R = 2`, el coordinador espera respuestas de dos réplicas. Luego compara las versiones usando los vector clocks y devuelve la más actualizada, o varias versiones si hay conflicto concurrente.

En funcionamiento normal, esto da una consistencia eventual bastante buena. Si no hay fallas graves ni particiones grandes, el quórum de lectura suele encontrar una versión reciente.

Con fallas raras, puede devolver información vieja. Por ejemplo, si las réplicas correctas están inaccesibles y el coordinador consulta nodos alternativos que tienen una versión atrasada, la lectura puede responder un valor desactualizado. Eso es parte del contrato de consistencia eventual.

## Hinted handoff

El sloppy quorum necesita un mecanismo para reparar después lo que se escribió en un nodo alternativo. Ese mecanismo es **hinted handoff**.

Supongamos:

```text
preference list(k) = [S1, S2, S3]
S3 está caído
S4 recibe una copia temporal
```

Cuando `S1` le manda la escritura a `S4`, no solo le manda:

```text
(k, v)
```

También le manda un hint:

```text
(k, v, destino_original = S3)
```

Ese hint significa:

```text
Esta escritura no era realmente para vos.
Guardala por ahora.
Cuando S3 vuelva, entregásela.
```

Luego `S4` monitorea a `S3`. Cuando `S3` vuelve a estar disponible, `S4` le transfiere la escritura pendiente.

```text
S4 ─── entrega (k, v) ───▶ S3
```

Al recibirla, `S3` no la acepta ciegamente. Compara versiones con vector clocks. Si el dato que trae `S4` es más nuevo, lo incorpora. Si `S3` ya tiene una versión más reciente, la rechaza o reconcilia según corresponda.

## Fallas temporales

El diseño parte de una hipótesis práctica: la mayoría de las fallas son temporales.

No siempre falla una máquina porque se quemó el disco, explotó el servidor o desapareció un datacenter. Muchas fallas reales son más simples:

- un router pierde paquetes durante unos segundos;
- una máquina está saturada;
- una base local está llena y empieza a rechazar requests;
- un proceso queda lento y responde con timeout;
- una partición de red aparece y luego desaparece.

Para esas fallas, no conviene reemplazar inmediatamente una máquina. Tiene más sentido asumir que puede volver y guardar temporalmente los datos en otro nodo.

El hinted handoff está diseñado para ese escenario.

## Fallas permanentes y anti-entropy

El hinted handoff ayuda con fallas temporales, pero no alcanza para todo. Si una máquina estuvo caída mucho tiempo, si se perdió información o si las réplicas quedaron divergentes por varias fallas encadenadas, hace falta un mecanismo adicional.

Dynamo usa **anti-entropy**: las réplicas se comparan periódicamente entre sí y reparan diferencias.

La forma ingenua de comparar dos réplicas sería transferir toda la tabla:

```text
S1: todas las claves y valores
S2: todas las claves y valores

comparar fila por fila
```

Eso es demasiado costoso. En la mayoría de los casos, las réplicas van a estar iguales o van a diferir en pocas claves. Transferir todo sería desperdiciar red y CPU.

## Merkle trees

Para comparar réplicas eficientemente se usan **Merkle trees**.

La idea básica es calcular hashes de los datos y combinarlos en forma de árbol.

Primero, cada fila o rango de claves tiene un hash:

```text
k1, v1 ──▶ h1
k2, v2 ──▶ h2
k3, v3 ──▶ h3
k4, v4 ──▶ h4
```

Luego se combinan hashes de a pares:

```text
h12 = hash(h1, h2)
h34 = hash(h3, h4)
```

Y finalmente se calcula la raíz:

```text
hroot = hash(h12, h34)
```

Representación:

```text
              hroot
             /     \
          h12       h34
         /   \     /   \
       h1    h2  h3    h4
```

Para comparar dos réplicas, primero se compara la raíz.

- Si las raíces son iguales, las réplicas son iguales para ese rango.
- Si las raíces difieren, se baja un nivel.
- Si una mitad coincide, se descarta esa mitad.
- Se sigue bajando hasta encontrar las claves exactas que difieren.

Esto permite localizar diferencias sin transferir toda la tabla.

Una vez detectada la clave divergente, se comparan las versiones con vector clocks y se aplica reconciliación sintáctica o se conservan siblings si hay concurrencia real.

## Gossip protocols

Para que Dynamo funcione, cada nodo necesita conocer la membresía del cluster:

```text
qué nodos existen
qué nodos están vivos
qué nodos virtuales ocupa cada máquina en el anillo
qué rangos le corresponden a cada uno
```

Una alternativa sería centralizar esa información en un servicio de configuración:

```text
nodos ───▶ config service
nodos ◀─── config service
```

Ese servicio podría ser ZooKeeper, una base de datos o un host con archivos de configuración. El problema es que introduce un componente central que hay que mantener disponible.

Dynamo usa una estrategia más descentralizada: **gossip**.

## Idea de gossip

Cada nodo mantiene una visión local del estado del cluster. Periódicamente elige otro nodo al azar e intercambia información.

Algoritmo básico:

```text
cada T segundos:
    peer = elegir_peer_al_azar()
    estado_peer = intercambiar_estado(peer)
    self.estado = merge(self.estado, estado_peer)
```

El estado no contiene solo información del peer que responde. Contiene todo lo que ese peer sabe del cluster.

Así, si un nodo aprende algo nuevo, lo empieza a propagar. Otros nodos lo reciben y lo propagan a su vez. La información se disemina como un rumor o como una epidemia.

```text
1 nodo sabe algo
2 nodos lo saben
4 nodos lo saben
8 nodos lo saben
...
```

La convergencia es logarítmica en la cantidad de nodos. Para `1024` nodos, `log2(1024) = 10`, por lo que en unas pocas rondas la información puede haberse propagado a todo el cluster.

## Push, pull y push-pull

Hay distintas formas de intercambio:

### Push

Un nodo le envía su estado a otro.

```text
A ─── estado(A) ───▶ B
```

### Pull

Un nodo le pide el estado a otro.

```text
A ─── request ───▶ B
A ◀── estado(B) ─ B
```

### Push-pull

Ambos intercambian estado en la misma interacción.

```text
A ─── estado(A) ───▶ B
A ◀── estado(B) ─── B
```

Push-pull suele converger más rápido porque en un ida y vuelta ambos nodos aprenden algo.

## Gossip en Dynamo

En Dynamo, el gossip propaga principalmente:

- miembros del cluster;
- estado de disponibilidad de nodos;
- tokens o virtual nodes dentro del anillo;
- información necesaria para que cada nodo sepa cómo construir preference lists.

Como la membresía no cambia miles de veces por segundo, este enfoque funciona bien. Cuando aparece un nodo nuevo, durante un tiempo algunos nodos lo conocen y otros todavía no. Eso genera una ventana de inconsistencia en la visión del cluster.

Dynamo tolera esa ventana porque el resto de los mecanismos absorben la transición:

- sloppy quorum permite escribir en nodos alternativos;
- hinted handoff reubica datos luego;
- Merkle trees reparan divergencias;
- la consistencia eventual termina estabilizando el sistema.

## Sistemas derivados de Dynamo

El diseño de Dynamo fue muy influyente. Varias empresas implementaron sistemas inspirados en ese paper.

Ejemplos:

- **Cassandra**: combina ideas de Dynamo con el modelo de datos de Bigtable.
- **Riak**: más cercano al diseño original de Dynamo.

Muchas de estas ideas siguen apareciendo en bases distribuidas modernas: particionado, replicación, anti-entropy, gossip y reconciliación de conflictos.

## DynamoDB como servicio cloud

DynamoDB no es simplemente “Dynamo como producto”. Toma el nombre por la fama del paper, pero su arquitectura es bastante distinta.

Dynamo era un sistema que equipos internos de Amazon podían desplegar y operar para sus aplicaciones. DynamoDB es un **servicio cloud**: una base de datos ofrecida como producto administrado.

La diferencia central es operativa:

```text
Dynamo:    cada equipo opera su propio sistema
DynamoDB:  AWS opera el sistema para todos los clientes
```

## Qué significa fully managed

Un servicio fully managed es operado por el proveedor cloud. El usuario crea recursos y los usa, pero no administra directamente los servidores físicos ni la replicación interna.

En DynamoDB, al crear una tabla, el usuario no decide:

- en qué máquina física se guarda;
- qué disco se usa;
- cómo se replica;
- qué proceso corre el storage;
- cómo se actualiza el software;
- cómo se reemplaza una máquina caída;
- cómo se balancea la carga.

Todo eso lo resuelve AWS.

Propiedades típicas de un servicio fully managed:

- aprovisionamiento automático;
- escalado;
- actualización de software;
- tolerancia a fallas;
- replicación;
- monitoreo;
- reemplazo de nodos;
- backups.

El usuario paga para no tener que operar esa complejidad.

## Multi-tenant

Los servicios cloud suelen ser **multi-tenant**: muchos clientes comparten la misma infraestructura física y lógica.

No se crea un cluster separado para cada cliente. Sería demasiado caro. En cambio, todos usan el mismo servicio, con mecanismos de aislamiento.

```text
cliente A ┐
cliente B ├──▶ infraestructura compartida
cliente C ┘
```

La plataforma debe evitar que un cliente afecte a otros. Esto se conoce como problema de *noisy neighbor*: un usuario que consume demasiados recursos puede degradar la experiencia de otros si el sistema no lo controla.

El multi-tenancy permite economía de escala. El proveedor invierte en infraestructura grande y reparte el costo entre miles o millones de clientes.

## SLA, disponibilidad y latencia

Un servicio cloud no solo ofrece una API. También ofrece garantías contractuales.

Un **SLA** define qué promete el proveedor. Por ejemplo:

```text
disponibilidad mínima
latencia esperada
porcentaje de operaciones exitosas
```

Si el proveedor no cumple, puede tener que devolver dinero o créditos al cliente.

Para poder ofrecer esas garantías, el sistema limita qué puede hacer el usuario. DynamoDB no permite consultas arbitrarias como una base relacional tradicional. La API está diseñada para que el proveedor pueda controlar latencia, capacidad y escalabilidad.

## Escala y elasticidad

Otra promesa central de los servicios cloud es la elasticidad.

El usuario no compra una máquina física ni calcula de antemano toda la capacidad necesaria. Pide más capacidad cuando la necesita y paga más mientras la usa.

```text
más tráfico  → más capacidad
menos tráfico → menos costo
```

La escala no es realmente infinita, pero para el usuario se presenta como “ilimitada” dentro de ciertos límites operativos y comerciales.

## Modelo de cobro

Los servicios cloud se cobran por uso. El modelo exacto depende del servicio.

Ejemplos:

| Servicio | Unidad de cobro típica |
|---|---|
| Máquina virtual | horas de uso |
| CDN | tráfico transferido |
| DynamoDB | storage + requests por segundo |

Para cobrar por uso, el proveedor necesita medirlo. Eso introduce sistemas internos adicionales.

```text
usuario ── usa ──▶ servicio cloud
                    │
                    ▼
                 metering
                    │
                    ▼
                  billing
                    │
                    ▼
                 factura
```

El servicio reporta métricas de uso a un sistema de **metering**. Luego el sistema de **billing** combina esas métricas con precios, descuentos y reglas comerciales para generar la factura.

## De SimpleDB a DynamoDB

DynamoDB surge como evolución de **SimpleDB**, un servicio previo de AWS. El nombre DynamoDB capitaliza la fama del paper de Dynamo, pero la arquitectura final no replica el diseño original.

La idea general sigue siendo una base no relacional administrada, pero aparecen abstracciones más cercanas a una base como servicio.

## Modelo de datos de DynamoDB

DynamoDB introduce tablas.

Una tabla contiene **items**. Cada item tiene:

- atributos;
- primary key.

La primary key puede ser:

```text
partition key
```

o:

```text
partition key + sort key
```

La partition key es la clave que se usa para distribuir los datos entre particiones.

## Particionado en DynamoDB

DynamoDB no usa consistent hashing como Dynamo.

En vez de un anillo, usa un espacio de hashes dividido en rangos. Cada rango corresponde a una partición, y cada partición está asignada a un grupo de storage nodes.

```text
hash(partition_key) ──▶ posición en el espacio de hashes
```

Ejemplo conceptual:

```text
rango hash      storage nodes
0 - 1000        S1, S2, S3
1001 - 5000     S4, S8, S10
...
```

Esa información no se infiere recorriendo un anillo. Está guardada en un servicio aparte: el **partition metadata service**.

## Request router

Los requests entran por un **request router**.

El request router se encarga de:

- recibir requests de clientes;
- autenticar;
- medir uso para metering;
- calcular el hash de la partition key;
- consultar el partition metadata service;
- enviar la operación a los storage nodes correctos.

Flujo conceptual:

```text
cliente
   │
   ▼
request router
   │
   ├── consulta partition metadata
   │
   ▼
storage nodes
```

El request router no decide por consistent hashing. Consulta metadata.

## Storage nodes

Cada partición se replica en varios storage nodes. En DynamoDB, esos storage nodes usan un protocolo de consenso, en el paper indicado como Multi-Paxos.

Para razonar sobre la materia, puede pensarse como un grupo Raft:

```text
S1 ─┐
S2 ─┼── grupo de replicación
S3 ─┘
```

Dentro de cada storage node aparecen dos componentes familiares:

```text
write-ahead log
B-tree / estructura de storage
```

El log se replica con el protocolo de consenso. La estructura de storage mantiene los datos consultables.

## Lecturas y escrituras en DynamoDB

Para una lectura:

1. El cliente manda `get` al request router.
2. El request router calcula el hash de la partition key.
3. Consulta el partition metadata service.
4. Envía el request a uno de los storage nodes del grupo.
5. El storage node responde.

Por defecto, una lectura puede no ser fuertemente consistente. Puede responder desde una réplica. DynamoDB también permite pedir lecturas fuertemente consistentes, con mayor costo o menor disponibilidad dependiendo del caso.

Para una escritura:

1. El cliente manda `put` al request router.
2. El request router ubica el grupo de replicación.
3. Envía la escritura a un storage node.
4. Si ese nodo no es líder, la reenvía al líder.
5. El líder replica con el consenso del grupo.
6. Se confirma cuando el protocolo alcanza quórum.

Esto se parece mucho más a Raft/Paxos que al Dynamo original.

## Diferencias fuertes entre Dynamo y DynamoDB

Dynamo original:

- usa consistent hashing;
- usa preference lists;
- permite sloppy quorum;
- usa hinted handoff;
- usa vector clocks;
- admite siblings;
- permite escrituras conflictivas;
- resuelve conflictos con reconciliación sintáctica o semántica;
- prioriza always writable.

DynamoDB:

- no usa consistent hashing como mecanismo principal visible;
- usa metadata de particiones;
- usa grupos de replicación con consenso;
- no expone vector clocks al usuario;
- no conserva siblings al estilo Dynamo;
- separa fuertemente control plane y data plane;
- está pensado como servicio cloud multi-tenant.

DynamoDB termina siendo mucho más parecido a una gran colección de grupos Raft/Paxos administrados que a la arquitectura original de Dynamo.

## Control plane y data plane

En servicios cloud aparece una separación importante:

```text
control plane
```

 y:

```text
data plane
```

El **data plane** maneja las operaciones en tiempo real sobre los datos:

```text
put item
get item
delete item
update item
```

El **control plane** administra el sistema:

```text
create table
create index
update table
delete table
scaling
backups
monitoreo
```

La separación existe porque ambos planos tienen requisitos distintos.

## Disponibilidad no uniforme

No todas las operaciones necesitan la misma disponibilidad.

Si el data plane falla, los usuarios no pueden leer ni escribir datos. Eso es crítico.

Si el control plane falla durante unos minutos, quizá no se puedan crear tablas o cambiar configuraciones, pero las tablas existentes deberían seguir funcionando.

Objetivo típico:

```text
si se cae el control plane,
el data plane debe seguir operando
```

Esto evita que una falla en un subsistema administrativo afecte al camino crítico de lectura y escritura.

## Auto-admin

En DynamoDB, el paper menciona un componente de administración automática: **auto-admin**.

El auto-admin implementa gran parte del control plane. No atiende los `get` y `put` comunes, sino que gestiona el ciclo de vida y operación del sistema.

Ejemplo: `create table`.

```text
cliente ── create table ──▶ auto-admin
                              │
                              ├── actualiza partition metadata
                              ├── elige storage nodes
                              ├── inicializa particiones
                              └── configura grupos de replicación
```

Crear una tabla no tiene por qué ser una operación instantánea. Puede disparar un workflow interno y dejar la tabla en estado `creating` hasta que todo quede listo.

## Responsabilidades del control plane

El control plane suele encargarse de:

### Ciclo de vida de tablas

Crear, modificar y eliminar tablas. Muchas de estas operaciones son asincrónicas porque implican coordinar múltiples nodos.

### Monitoreo de flota

Observar storage nodes, detectar fallas, reemplazar nodos y restaurar réplicas.

```text
falla un nodo → se detecta → se reemplaza → se re-replica
```

### Scaling y rebalanceo

Detectar particiones calientes o desbalanceadas y dividirlas.

```text
partición hot ── split ──▶ dos particiones más chicas
```

Esto no siempre resuelve hot keys, pero sí ayuda cuando la carga está distribuida dentro de un rango.

### Backups

Gestionar backups periódicos y restauraciones. En AWS, los backups pueden terminar almacenados en otros servicios internos, como S3.

## Degradación gradual

Los sistemas cloud están diseñados para degradarse de forma gradual.

No se busca que una falla parcial tire abajo todo el servicio. Si falla un storage node o una partición, el impacto debería limitarse a una parte de las claves.

```text
falla parcial → algunas claves lentas o temporalmente afectadas
resto del sistema → sigue funcionando
```

Eso requiere particionado, replicación, monitoreo, failover y aislamiento entre componentes.

## Availability Zones

DynamoDB replica datos en distintas Availability Zones. Una Availability Zone puede pensarse como un datacenter o conjunto de datacenters aislados físicamente dentro de una región.

El objetivo es que una falla física grande no afecte todas las réplicas al mismo tiempo.

```text
réplica 1 → AZ A
réplica 2 → AZ B
réplica 3 → AZ C
```

Las Availability Zones están conectadas con redes de baja latencia, pero separadas para tolerar fallas de energía, red, incendios u otros problemas locales.

## Cierre

Dynamo muestra un diseño orientado a disponibilidad extrema: sloppy quorum, hinted handoff, Merkle trees y gossip se combinan para que el sistema siga aceptando escrituras y se repare con el tiempo.

DynamoDB toma otro camino. Conserva la motivación de una base no relacional escalable, pero la rediseña como servicio cloud administrado, multi-tenant, con particiones explícitas, metadata centralizada, grupos de replicación con consenso y una separación fuerte entre control plane y data plane.

La diferencia más importante es conceptual:

```text
Dynamo:   aceptar divergencia y reconciliar después
DynamoDB: administrar particiones replicadas con consenso como servicio cloud
```
