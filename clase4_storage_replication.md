# Sistemas Distribuidos — Sistemas de storage, sharding y replicación

## 1. Sistemas de storage

Un sistema de storage es un sistema distribuido cuyo objetivo principal es almacenar y recuperar datos de forma confiable. Puede aparecer como:

- un sistema de archivos distribuido;
- una base de datos distribuida;
- un sistema de logs;
- una cola persistente;
- una plataforma de eventos;
- un almacenamiento interno para otros servicios.

Los dos objetivos principales son:

- **Escalabilidad**: poder crecer cuando aumentan los datos, la cantidad de usuarios o el volumen de operaciones.
- **Tolerancia a fallas**: poder seguir funcionando, o al menos preservar los datos, cuando fallan máquinas, discos, procesos o enlaces de red.

En sistemas grandes, las fallas no son casos excepcionales. Son parte normal del funcionamiento. Por eso el diseño del storage debe asumir que alguna máquina puede dejar de responder, reiniciarse, perder conectividad o quedar atrasada respecto del resto.

## 2. Relación con MapReduce

MapReduce resuelve un problema de procesamiento distribuido: dividir trabajo en muchas tareas para usar más CPU y escalar horizontalmente.

La idea central es:

```text
Problema grande -> particiones -> tareas map/reduce -> ejecución paralela
```

Esta estrategia mejora la escalabilidad porque permite repartir cómputo entre muchos nodos. También aporta tolerancia a fallas porque las tareas pueden reintentarse.

Para que el reintento sea correcto, las funciones `map` y `reduce` deben ser determinísticas. Si una tarea falla, se vuelve a ejecutar sobre la misma entrada y debería producir el mismo resultado.

En storage aparece una idea relacionada, pero aplicada a datos persistentes:

```text
Datos grandes -> particiones -> nodos de storage
```

En lugar de repartir solamente CPU, se reparten datos y responsabilidades de lectura/escritura.

## 3. Técnicas principales

Hay dos técnicas fundamentales para construir storage distribuido:

1. **Particionamiento / sharding**
2. **Replicación**

Ambas se pueden combinar.

El sharding reparte el dataset entre varios nodos. La replicación mantiene copias de los datos en varios nodos.

```text
                +-----------------------+
                | Sistema de storage    |
                +-----------------------+
                     /              \
                    /                \
          Sharding /                  \ Replicación
                  /                    \
       repartir datos             copiar datos
```

## 4. Sharding

Sharding consiste en dividir un conjunto de datos en fragmentos llamados **shards**. Cada shard vive en una ubicación física concreta: un nodo, una partición, un grupo de réplica o un conjunto de discos.

La idea es que no todos los datos estén en una única máquina.

```text
Dataset completo
+-----------------------------------+
| shard 1 | shard 2 | shard 3 | ... |
+-----------------------------------+
     |         |         |
     v         v         v
   Nodo A    Nodo B    Nodo C
```

Para ubicar un dato se usa un mecanismo de nombres o una función de particionamiento.

```text
Dato / clave -> función de partición -> ubicación física
```

Por ejemplo:

```text
hash(clave) mod N -> número de shard
```

La clave usada para particionar suele llamarse **shard key**. Elegirla mal puede generar desequilibrios: algunos shards reciben demasiado tráfico y otros quedan casi vacíos.

### Beneficios del sharding

El sharding permite:

- **Escalabilidad horizontal**: agregar más nodos para repartir más datos o más carga.
- **Fallas parciales**: si un shard falla, no necesariamente cae todo el sistema.
- **Paralelismo de acceso**: distintas consultas u operaciones pueden ejecutarse en paralelo sobre shards distintos.
- **Mejor performance**: se reduce la cantidad de datos que maneja cada nodo individual.

### Límite del sharding

El sharding no replica los datos por sí mismo. Si un shard vive solamente en un nodo y ese nodo se pierde, se pierde la disponibilidad de esa porción de datos. Por eso suele combinarse con replicación.

## 5. Replicación

Replicar significa mantener copias de los mismos datos en más de un nodo.

```text
        x = 1          x = 1          x = 1
      +-------+      +-------+      +-------+
      | Nodo1 | ---> | Nodo2 | ---> | Nodo3 |
      +-------+      +-------+      +-------+
```

La replicación apunta a mejorar disponibilidad, durabilidad y tolerancia a fallas.

### Tipos de fallas

Una clasificación importante distingue entre:

- **Fallas fail-stop**: un nodo deja de funcionar o deja de responder. No produce respuestas incorrectas; simplemente se cae o queda inaccesible.
- **Fallas bizantinas o Bugs**: un nodo puede comportarse de manera arbitraria, responder cosas falsas, contradecirse o actuar maliciosamente.

Muchos sistemas de bases de datos y storage tradicionales se enfocan en fallas fail-stop. Las fallas bizantinas requieren protocolos más costosos y complejos.

## 6. Mecanismos de replicación

Hay dos enfoques principales:

1. **State transfer / transferencia de estado / Snapshot**
2. **Replicated State Machine / máquina de estados replicada**

## 7. State transfer

En state transfer, un nodo transfiere su estado a otro nodo.

```text
Cliente -> Nodo 1 -> Nodo 2
```

El nodo principal recibe escrituras y luego copia el estado hacia otro nodo. Esa copia puede ser una base completa, un snapshot, páginas modificadas o algún delta.

### Ventajas

- Es conceptualmente simple.
- Sirve para inicializar réplicas nuevas.
- Permite reconstruir un nodo atrasado copiando un estado conocido.

### Problemas

- Puede ser lento si el estado es grande.
- Puede requerir bloquear o coordinar cambios mientras se copia.
- Puede generar mucho tráfico de red.
- No siempre está claro qué versión exacta del estado se copió si hay escrituras concurrentes -> Deberia dejar de responder requests mientras se copia.

State transfer es útil, pero no alcanza por sí solo para resolver el orden fino de muchas operaciones concurrentes.

## 8. Replicated State Machine (RSM)

La idea de una **máquina de estados replicada** es que varias réplicas parten del mismo estado inicial y ejecutan la misma secuencia de operaciones en el mismo orden.

Si se cumplen estas condiciones:

1. todas las réplicas parten del mismo estado inicial;
2. todas reciben las mismas operaciones;
3. todas aplican esas operaciones en el mismo orden;
4. las operaciones son determinísticas;

entonces todas llegan al mismo estado final.

```text
Estado inicial S0

Réplica A: O1, O2, O3, O4 -> Estado final S
Réplica B: O1, O2, O3, O4 -> Estado final S
Réplica C: O1, O2, O3, O4 -> Estado final S
```

El problema central pasa a ser cómo garantizar el mismo orden de operaciones en todas las réplicas.

## 9. Orden total y consenso

Si dos clientes escriben al mismo tiempo, distintas réplicas podrían ver las operaciones en distinto orden.

```text
Cliente 1 escribe O1
Cliente 2 escribe O2

Réplica A ve: O1, O2
Réplica B ve: O2, O1
```

Si `O1` y `O2` afectan el mismo dato, el estado final puede ser distinto. Por eso se necesita una forma de acordar un orden único.

Garantizar el orden global de operaciones es un problema de consenso.

La abstracción fundamental para resolverlo suele ser un **log**.

## 10. El log como abstracción fundamental

Un log es una secuencia append-only de operaciones.

```text
+----+----+----+----+----+
| O1 | O2 | O3 | O4 | O5 |
+----+----+----+----+----+
  0    1    2    3    4
```

El log define:

- qué operaciones existen;
- en qué orden deben aplicarse;
- hasta dónde llegó cada réplica;
- qué operaciones están confirmadas.

Una vez que las réplicas acuerdan el contenido y el orden del log, cada una puede aplicar las operaciones localmente.

```text
Log replicado -> aplicar operaciones -> estado local
```

El log permite separar dos problemas:

1. acordar el orden de las operaciones;
2. ejecutar esas operaciones sobre el estado.

## 11. Ejemplo 1: bases de datos relacionales | WAL

En una base de datos relacional también aparece la idea de log.

La base puede pensarse como un conjunto de páginas o regiones persistidas en disco. Las modificaciones se registran en un log antes de quedar reflejadas definitivamente en las páginas de datos.

```text
Operación -> WAL / REDO log -> páginas de datos
```

El **WAL** (*Write-Ahead Log*) permite recuperación ante fallas. Antes de modificar definitivamente la estructura persistida, se registra qué cambio debe aplicarse. Si la base se cae, el log permite rehacer operaciones confirmadas o descartar operaciones incompletas.

Esta misma idea es útil para replicación: en vez de copiar toda la base constantemente, se puede propagar la secuencia de cambios.

## 12. Ejemplo 2: Raft

Raft organiza la replicación alrededor de un log compartido. Un cliente envía una operación al líder. El líder la agrega a su log y la replica a los followers.

```text
Cliente
   |
   v
+---------+        +----------+
| Líder   | -----> | Follower |
| Log     |        | Log      |
+---------+        +----------+
      \              
       \----------> +----------+
                    | Follower |
                    | Log      |
                    +----------+
```

Cada entrada del log tiene una posición. Esa posición funciona como una referencia de orden. Si una operación está en el índice `3`, entonces debe aplicarse después de las entradas anteriores y antes de las posteriores.

```text
índice:  0   1   2   3   4   5
log:    [ ][ ][ ][X][ ][ ]
```

Cada réplica puede tener distinto progreso. Por ejemplo:

```text
Réplica A: commit hasta índice 3
Réplica B: commit hasta índice 11
Réplica C: commit hasta índice 10
```

La replicación debe manejar réplicas atrasadas, fallas, reintentos y cambios de líder sin romper el orden acordado.

## 13. Active-active y primary-backup

Hay dos familias de arquitectura frecuentes:

- **Active-active**: más de una réplica puede aceptar operaciones activamente.
- **Primary-backup**: una réplica primaria concentra las escrituras y las demás actúan como backups o read replicas.

### Active-active

En active-active, varios nodos pueden recibir operaciones. Esto mejora disponibilidad y puede reducir latencia, pero complica mucho el orden global.

Si dos nodos aceptan escrituras concurrentes, después hay que resolver conflictos, ordenar operaciones o usar algún protocolo de consenso.

### Primary-backup

En primary-backup, el nodo primario recibe las escrituras.

```text
          escritura
Cliente -----------> Primary
                       |
                       | WAL / cambios
                       v
                    Backup 1
                       |
                       v
                    Backup 2
```

El primary:

- decide el orden de las operaciones;
- aplica localmente la operación;
- registra los cambios en un log;
- envía los cambios a las réplicas.

Las réplicas suelen funcionar como backups o read replicas.

### Ventajas

- Es más simple que active-active.
- Hay un punto claro donde se ordenan las escrituras.
- Permite escalar lecturas si se habilitan read replicas.

### Problemas

- El primary puede convertirse en cuello de botella.
- Si la replicación es asincrónica, las réplicas pueden quedar atrasadas.
- Las lecturas desde réplicas pueden ver datos viejos.
- Si el primary confirma una escritura antes de replicarla y luego falla, puede haber pérdida de datos.

Por eso primary-backup suele ofrecer **consistencia eventual** cuando la replicación es asincrónica. No necesariamente ofrece linealizabilidad.

## 14. Configuración Interesante -> WAL, Debezium y Kafka

Una arquitectura frecuente usa el log de una base como fuente de eventos.

```text
Postgres -> WAL -> Debezium -> Kafka -> consumidores
                                      |-> Data warehouse
                                      |-> Búsqueda de texto
                                      |-> Microservicio
```

El WAL registra cambios de la base. Una herramienta como Debezium puede leer ese log y publicar eventos en Kafka. Luego distintos sistemas consumen esos eventos para construir vistas derivadas.

Ejemplos:

- un data warehouse para analítica;
- un índice de búsqueda de texto;
- un microservicio que mantiene una proyección propia;
- un sistema de auditoría;
- un pipeline de sincronización.

La idea importante es:

> El log es el dual de la base de datos.

Una base representa el estado actual. Un log representa la historia ordenada de cambios que lleva a ese estado.

```text
Log:     O1, O2, O3, O4
Estado:  resultado de aplicar O1..O4
```

Con el log se pueden reconstruir estados, alimentar otros sistemas y propagar cambios de manera ordenada.

## 15. Chain replication

Chain replication organiza las réplicas como una cadena.

```text
Cliente escribe
      |
      v
    Head ---> Nodo intermedio ---> Tail
                                  ^
                                  |
                           Cliente lee
```

- Las escrituras entran por el **head**.
- Cada nodo reenvía la operación al siguiente. 
- El último nodo es el **tail**.
- Las lecturas se responden desde el tail.
- Los acknowledgements vuelven hacia atrás.

```text
WRITE: Cliente -> Head -> ... -> Tail
ACK:   Tail -> ... -> Head -> Cliente
READ:  Cliente -> Tail
```

### Beneficios

Chain replication permite separar escrituras y lecturas de forma clara.
 
El tail solamente responde lecturas cuando recibió las operaciones anteriores de la cadena. Por eso las lecturas desde el tail pueden ser fuertemente consistentes respecto del orden confirmado.

También permite razonar con una secuencia clara: una escritura confirmada ya atravesó toda la cadena.

## 16. Fallas en chain replication

### Falla del head

Si falla el head, el siguiente nodo puede ser promovido como nuevo head.

```text
Head falla -> nuevo head = siguiente nodo vivo
```

Los clientes deben ser avisados o reconfigurados para enviar escrituras al nuevo head.

### Falla del tail

Si falla el tail, el nodo anterior puede convertirse en nuevo tail.

```text
Tail falla -> nuevo tail = nodo anterior vivo
```

Los clientes deben ser reconfigurados para leer desde el nuevo tail.

### Falla de un nodo intermedio

Si falla un nodo intermedio, la cadena debe reconectarse.

```text
A -> B -> C

B falla

A -> C
```

Puede ser necesario reenviar operaciones que estaban en tránsito o revisar qué operaciones llegaron a cada nodo.

## 17. Reconfiguración y detección de fallas

Un punto crítico es decidir quién detecta fallas y quién tiene autoridad para reconfigurar la cadena.

Si cada lado de una partición de red cree que puede continuar de forma independiente, aparece el problema de **split brain**.

```text
DC1        partición        DC2
C1 -> cadena parcial    C2 -> cadena parcial
```

En un split brain, dos subconjuntos del sistema pueden aceptar operaciones incompatibles. Después no hay una única historia de cambios.

Para evitarlo se necesita un servicio de configuración o coordinación que diga cuál es la configuración válida.

```text
Servicio de configuración / consenso
        |
        v
configuración actual de la cadena
```

Ese servicio debe ser tolerante a fallas y evitar que dos configuraciones incompatibles estén activas al mismo tiempo. Un ejemplo típico es usar un sistema basado en consenso como Raft, o servicios como Zookeper o Kubernetes, para decidir la configuración.

## 18. Sharding + replicación

En sistemas reales, sharding y replicación suelen combinarse.

```text
Shard 1: A1 -> A2 -> A3
Shard 2: B1 -> B2 -> B3
Shard 3: C1 -> C2 -> C3
```

Cada shard tiene sus propias réplicas. Esto permite:

- repartir el dataset completo;
- tolerar fallas dentro de cada shard;
- escalar lecturas y escrituras;
- mover shards entre nodos;
- reconfigurar réplicas cuando hay fallas.

El costo es mayor complejidad. El sistema debe resolver:

- dónde vive cada dato;
- qué réplica es líder, primary, head o tail;
- cómo se detectan fallas;
- cómo se reconfigura sin split brain;
- cómo se mantiene el orden de operaciones;
- qué garantías de consistencia se ofrecen.

## 19. Ideas principales

Un sistema de storage distribuido necesita resolver simultáneamente ubicación, orden, persistencia y fallas.

- **Sharding** reparte datos.
- **Replicación** copia datos.
- **State transfer** copia estado.
- **Replicated State Machine** replica operaciones ordenadas.
- **El log** permite representar una historia ordenada de cambios.
- **WAL/REDO** permite recuperación y también puede alimentar replicación.
- **Primary-backup** simplifica el orden, pero puede tener consistencia eventual.
- **Active-active** mejora disponibilidad, pero complica la resolución de conflictos.
- **Chain replication** ofrece una estructura clara para lecturas y escrituras, pero requiere reconfiguración segura ante fallas.
- **El consenso** aparece cuando hay que acordar orden, líder o configuración.
- **Split brain** es uno de los problemas más peligrosos: dos partes del sistema creen ser válidas al mismo tiempo.

## 20. Resumen conceptual

```text
Storage distribuido
    |
    +-- Escalabilidad
    |      |
    |      +-- Sharding
    |      +-- Paralelismo
    |      +-- Reparto de carga
    |
    +-- Tolerancia a fallas
           |
           +-- Replicación
           +-- Logs
           +-- Reintentos
           +-- Reconfiguración
           +-- Consenso
```

El diseño de sistemas de storage es una búsqueda de equilibrio entre performance, disponibilidad, consistencia, simplicidad operativa y tolerancia a fallas.
