# Clase 5 - Google File System (GFS)

## Idea general

Google File System (GFS) es un sistema de archivos distribuido pensado para almacenar archivos muy grandes sobre clusters de máquinas comunes (*commodity hardware*). El objetivo no es ofrecer una semántica POSIX tradicional, sino una abstracción útil para procesamiento distribuido de datos a gran escala.

GFS combina varias ideas centrales:

- **Sharding / particionado**: los archivos se dividen en partes grandes llamadas *chunks*.
- **Replicación**: cada chunk se guarda en varias máquinas.
- **Consistencia controlada**: se acepta una semántica más débil que la de un filesystem local para ganar escalabilidad y tolerancia a fallas.
- **Recuperación automática**: el sistema detecta fallas y vuelve a replicar datos cuando hace falta.
- **Optimización para batch**: está orientado a cargas donde se leen archivos grandes y se agregan datos al final de archivos existentes.

La decisión de diseño más importante es aceptar restricciones en la interfaz para que el sistema pueda escalar y tolerar fallas con una implementación relativamente simple.

---

## Requisitos de diseño

GFS fue diseñado para un entorno con estas características:

- Archivos muy grandes, de muchos GB o más.
- Muchos archivos distribuidos en muchas máquinas / Global.
- Fallas frecuentes de máquinas, discos o procesos.
- Hardware barato y reemplazable.
- Lecturas grandes y secuenciales.
- Escrituras mayormente por *append*.
- Aplicaciones batch, como MapReduce.

Los requisitos principales son:

| Requisito | Consecuencia en el diseño |
|---|---|
| Escalabilidad | Dividir archivos en chunks y distribuirlos. |
| Tolerancia a fallas | Replicar chunks y detectar fallas automáticamente. |
| Alto throughput | Evitar que los datos pasen por un nodo central. |
| Simplicidad operativa | Usar un coordinador central para metadatos. |
| Cargas batch | Priorizar lectura secuencial y append antes que baja latencia. |

---

## Abstracción fundamental: archivo dividido en chunks

La abstracción visible sigue siendo la de un **archivo**, por ejemplo:

```text
/dir/file.txt
```

Internamente, el archivo se divide en **chunks** grandes. En GFS, el tamaño típico de chunk es de **64 MB**.

```text
archivo /dir/file.txt

+----------+----------+----------+----------+
| chunk 0  | chunk 1  | chunk 2  | chunk 3  |
+----------+----------+----------+----------+
   64 MB      64 MB      64 MB      64 MB
```

Cada chunk tiene un identificador único llamado **chunk handle**. Ese identificador es asignado por el master cuando el chunk se crea.

Para leer una posición del archivo, el cliente traduce:

```text
read(filename, offset)
```

en:

```text
chunk_index = offset / 64 MB
chunk_offset = offset % 64 MB
```

Luego usa el nombre del archivo y el índice del chunk para consultar los metadatos necesarios.

---

## Arquitectura de GFS

La arquitectura tiene tres componentes principales:

1. **Cliente GFS**
2. **Master / coordinador**
3. **Chunkservers**

```text
Aplicación
   |
   v
Cliente GFS  ---- metadata ---->  Master
   |
   |---- datos ---------------->  Chunkserver
   |---- datos ---------------->  Chunkserver
   |---- datos ---------------->  Chunkserver
```

La separación clave es:

- El **master** maneja metadatos y coordinación.
- Los **chunkservers** almacenan datos reales.
- El **cliente** consulta al master solo para saber dónde están los chunks, pero lee y escribe datos directamente contra los chunkservers.

Esto evita que el master se convierta en cuello de botella para el tráfico de datos.

---

## Responsabilidades del master

El master mantiene los metadatos del sistema. Entre ellos:

- Namespace de archivos y directorios.
- Mapeo de archivo a lista de chunks.
- Mapeo de chunk a chunkservers que tienen réplicas.
- Versiones de chunks.
- Información de leases.
- Estado de los chunkservers.
- Decisiones de ubicación y replicación.
- Detección de fallas mediante heartbeats.

Ejemplo conceptual:

```text
/foo/bar
  chunk 0 -> handle H0 -> CS1, CS2, CS3
  chunk 1 -> handle H1 -> CS2, CS5, CS8
  chunk 2 -> handle H2 -> CS1, CS4, CS7
```

El master no almacena los datos del archivo. Solo almacena información para encontrar y coordinar esos datos.

---

## Responsabilidades de los chunkservers

Los chunkservers guardan chunks en su disco local, normalmente usando el filesystem local de Linux.

Cada chunkserver puede:

- Leer un rango de bytes de un chunk.
- Escribir datos en un chunk.
- Participar en una operación de append.
- Reportar su estado al master.
- Detectar corrupción usando checksums.
- Enviar o recibir réplicas de chunks.

Los datos reales nunca pasan por el master. El cliente se comunica directamente con los chunkservers.

---

## Flujo de lectura

Una lectura se resuelve en dos etapas: primero metadatos, después datos.

### 1. Consulta al master

El cliente recibe una operación como:

```text
read(/foo/bar, offset)
```

Calcula qué chunk corresponde a ese offset y consulta al master:

```text
(file name, chunk index)
```

El master responde:

```text
(chunk handle, ubicaciones de réplicas)
```

El chunk handle es un identificador unico para cada chunk, es un int64.

### 2. Lectura directa desde un chunkserver

Con el chunk handle y las ubicaciones, el cliente elige una réplica, idealmente la más cercana, y pide:

```text
(chunk handle, byte range)
```

El chunkserver responde con los datos.

```text
Cliente -> Master: ¿dónde está chunk 2 de /foo/bar?
Master  -> Cliente: handle H2, réplicas CS1, CS4, CS7
Cliente -> CS4: leer H2 desde offset X durante N bytes
CS4     -> Cliente: chunk data
```

El cliente puede cachear los metadatos por un tiempo para no consultar al master en cada operación.

> El cliente siempre elije las replicas más cercanas para optimizar la latencia y el throughput.

---

## Replicación

Cada chunk se replica en varios chunkservers. El valor típico es **3 réplicas**.

```text
chunk H2
  réplica 1 -> CS1
  réplica 2 -> CS4
  réplica 3 -> CS7
```

La replicación permite:

- Seguir leyendo aunque falle una máquina.
- Recuperar datos copiando desde réplicas sanas.
- Distribuir carga de lectura.
- Tolerar fallas parciales.

El master monitorea los chunkservers. Si detecta que un chunk quedó con menos réplicas de las necesarias, ordena crear nuevas réplicas.

---

## Leases y primary replica

Para coordinar escrituras, GFS usa una variante de **primary-backup**.

Para cada chunk que se va a modificar, el master otorga un **lease** a una de las réplicas. Esa réplica pasa a ser la **primary replica** durante un tiempo limitado, por ejemplo 60 segundos.

Las demás réplicas actúan como **secondaries**.

```text
chunk H2

CS1 -> secondary
CS4 -> primary
CS7 -> secondary
```

La primary tiene una responsabilidad clave: **definir el orden de las operaciones** sobre ese chunk.

Esto evita que distintos clientes escriban en órdenes diferentes en distintas réplicas.

---

## Flujo de escritura

Una escritura separa el flujo de datos del flujo de control.

### 1. El cliente consulta al master

El cliente pregunta qué réplicas tienen el chunk y cuál es la primary.

```text
Cliente -> Master: quiero escribir en chunk H2
Master  -> Cliente: primary CS4, secondaries CS1 y CS7
```

### 2. El cliente envía los datos a las réplicas

El cliente envía los datos a las réplicas, usualmente en pipeline. Los datos pueden circular de una réplica a otra para aprovechar mejor la red.

```text
Cliente -> CS1 -> CS4 -> CS7
```

En esta etapa los datos llegan a las réplicas, pero todavía no necesariamente quedaron aplicados como escritura definitiva.

### 3. El cliente envía la orden de escritura a la primary

Cuando las réplicas ya tienen los datos, el cliente le pide a la primary que ejecute la escritura.

```text
Cliente -> Primary: aplicar write
```

La primary:

1. Asigna un orden a la operación.
2. Aplica la operación localmente.
3. Reenvía la operación en ese mismo orden a las secondaries.
4. Espera respuestas.
5. Devuelve resultado al cliente.

```text
Cliente
   |
   v
Primary replica ---- orden ----> Secondary A
        |
        +---------- orden ----> Secondary B
```

Si todas las réplicas responden correctamente, la operación se considera exitosa. Si alguna falla, se informa error al cliente.

---

## Write vs append

GFS distingue dos tipos de modificación:

```text
write(file, data, offset)
append(file, data)
```

### Write

En un `write`, el cliente indica explícitamente el offset donde quiere escribir.

```text
write(file, data, offset)
```

Esto puede ser problemático en presencia de concurrencia y fallas, porque varios clientes podrían intentar escribir regiones superpuestas o porque algunas réplicas podrían aplicar una escritura y otras no.

### Append

En un `append`, el cliente no elige el offset exacto. El sistema decide dónde agregar el registro.

```text
append(file, data) -> offset
```

GFS está especialmente optimizado para `append`, porque muchas aplicaciones de Google agregaban registros al final de archivos compartidos.

El append tiene una propiedad importante: el registro se agrega de forma atómica al menos una vez. Sin embargo, puede aparecer duplicado si hay reintentos.

---

## Record append

El **record append** permite que muchos clientes agreguen registros concurrentemente al mismo archivo.

La primary replica decide el offset final donde se va a insertar cada registro. Luego ordena a las secondaries que escriban el mismo registro en el mismo lugar.

```text
Cliente 1 append(B)
Cliente 2 append(C)
Cliente 3 append(B)
Cliente 4 append(A)

Resultado posible:

+---+---+---+---+
| B | C | B | A |
+---+---+---+---+
```

Si un registro no entra en el espacio restante del chunk, GFS puede insertar **padding** y continuar en otro chunk.

```text
+---+---+---------+---+
| B | C | padding | A |
+---+---+---------+---+
```

El padding no representa un registro válido de la aplicación. Por eso las aplicaciones suelen usar checksums o delimitadores para detectar y descartar regiones inválidas.

---

## Consistencia

GFS no promete la misma consistencia que un filesystem local tradicional. La semántica está adaptada a un sistema distribuido con fallas frecuentes.

Una región de un archivo puede estar en distintos estados:

| Estado | Significado |
|---|---|
| Consistente | Todas las réplicas devuelven los mismos datos. |
| Definida | Además de ser consistente, contiene exactamente la escritura esperada. |
| Indefinida | Las réplicas coinciden, pero el contenido puede ser una mezcla resultante de escrituras concurrentes. |
| Inconsistente | Distintas réplicas pueden devolver datos distintos. |

En operaciones concurrentes o fallidas, GFS puede dejar regiones indefinidas o inconsistentes. Por eso muchas aplicaciones se diseñan para tolerar:

- Duplicados.
- Padding.
- Registros parcialmente escritos.
- Reintentos.

La responsabilidad de resolver ciertos problemas se traslada a la capa de aplicación.

---

## Cómo resolver inconsistencias desde la aplicación

Dos estrategias típicas son:

### 1. Identificadores únicos

Cada registro puede incluir un ID único. Si por un reintento aparece dos veces, la aplicación puede detectar el duplicado.

```text
record_id = 123
payload = ...
```

Al procesar el archivo, si aparece dos veces el mismo `record_id`, se conserva uno solo.

### 2. Checksums

Cada registro puede incluir un checksum. Si el registro está corrupto, truncado o corresponde a padding, se descarta.

```text
[record_id][payload][checksum]
```

Esto permite distinguir datos válidos de basura o zonas incompletas.

---

## GFS y MapReduce

GFS funciona muy bien con MapReduce porque ambos están pensados para procesamiento batch distribuido.

La idea clave es mover el cómputo cerca de los datos.

```text
MapReduce Master -> pregunta a GFS dónde están los chunks
MapReduce Worker -> se ejecuta cerca del chunkserver que tiene el dato
Worker -> lee el chunk local o cercano
```

Como los archivos están divididos en chunks, cada chunk puede usarse como una unidad natural de trabajo.

```text
archivo grande

+----------+----------+----------+----------+
| chunk 0  | chunk 1  | chunk 2  | chunk 3  |
+----------+----------+----------+----------+
    |          |          |          |
    v          v          v          v
  mapper     mapper     mapper     mapper
```

Esto mejora el rendimiento porque evita mover grandes volúmenes de datos por la red. En vez de traer todos los datos al cómputo, se intenta llevar el cómputo a las máquinas que ya tienen esos datos.

---

## Datos que mantiene el coordinador

El coordinador mantiene estructuras en memoria para responder rápido. Además, persiste cambios importantes mediante un log y snapshots.

Metadatos importantes:

| Dato | Ejemplo |
|---|---|
| Filename -> chunk handles | `/foo/bar -> H0, H1, H2` |
| Chunk handle -> chunkservers | `H2 -> CS1, CS4, CS7` |
| Chunk handle -> version number | `H2 -> versión 17` |
| Leases activos | `H2 -> primary CS4 hasta T` |
| Estado de chunkservers | vivos, caídos, atrasados |

La metadata se mantiene en memoria para que las operaciones sean rápidas. Para no perder estado, el master escribe cambios en un log persistente y puede crear snapshots.

---

## Versiones de chunks

Cada chunk tiene un número de versión. Esto ayuda a detectar réplicas obsoletas.

Ejemplo:

```text
H2 versión 10 en CS1
H2 versión 11 en CS4
H2 versión 11 en CS7
```

En este caso, CS1 tiene una réplica vieja. El master puede detectar que esa réplica está desactualizada y no usarla como válida.

Las versiones son importantes cuando un chunkserver estuvo caído durante una modificación y luego vuelve al sistema.

---

## Manejo de fallas

GFS asume que las fallas son normales. Por eso el sistema debe detectarlas y recuperarse.

### Fallas de chunkservers

El master usa heartbeats para saber si un chunkserver sigue vivo.

Si un chunkserver falla:

1. El master deja de considerarlo para lecturas y escrituras.
2. Detecta qué chunks quedaron con pocas réplicas.
3. Ordena copiar esos chunks desde réplicas sanas.
4. Restaura el factor de replicación esperado.

```text
CS3 falla
chunk H2 queda con 2 réplicas
master ordena crear una nueva réplica en CS8
```

### Fallas del master

El master es centralizado, lo cual simplifica el diseño pero introduce un punto crítico.

Para hacerlo recuperable:

- Escribe cambios en disco.
- Mantiene un log persistente.
- Crea snapshots.
- Puede tener backups en standby.

La idea es que si el master falla, otro proceso pueda reconstruir su estado a partir del log y los snapshots.

---

## Split brain

Un problema peligroso en sistemas con coordinadores, primaries o leases es el **split brain**.

Ocurre cuando dos partes del sistema creen al mismo tiempo que tienen autoridad para aceptar escrituras.

Ejemplo:

```text
DC1 cree que primary = P1
DC2 cree que primary = P2
```

Si ambos aceptan escrituras, el sistema puede divergir.

GFS reduce este riesgo usando leases con vencimiento. Una primary solo puede actuar mientras su lease siga vigente. Cuando el lease vence, el master puede elegir otra primary.

Aun así, los sistemas distribuidos deben ser cuidadosos con particiones de red, relojes, reintentos y fallas parciales.

---

## Ventajas de GFS

- Escala a archivos enormes y muchos servidores.
- Tolera fallas de máquinas comunes.
- Aprovecha la localidad de datos.
- Funciona muy bien con MapReduce.
- Evita que el master transporte datos.
- Simplifica la coordinación usando un master central.
- Permite append concurrente de muchos clientes.

---

## Limitaciones de GFS

- El **master centralizado** simplifica, pero limita escalabilidad y tolerancia a fallas.
- La **consistencia es débil** frente a escrituras concurrentes o fallidas.
- Se pueden leer stale secondaries.
- No está pensado para muchas escrituras aleatorias pequeñas.
- El record append puede producir duplicados.
- La aplicación debe manejar duplicados, padding y registros inválidos.

---

## Trade-off principal

GFS elige un punto de diseño muy concreto:

```text
menos semántica de filesystem tradicional
+ más escalabilidad
+ más tolerancia a fallas
+ más throughput para batch
```

No intenta ser el filesystem perfecto para cualquier carga. Está optimizado para el tipo de trabajo que Google necesitaba: procesar grandes volúmenes de datos en clusters con fallas frecuentes.

---

## Relación con primary-backup

GFS usa una idea similar a primary-backup para coordinar la escritura de un chunk:

- El master elige una primary mediante un lease.
- La primary ordena las operaciones.
- Las secondaries aplican el mismo orden.
- El cliente recibe éxito si las réplicas necesarias confirman.

Pero no es primary-backup de una base de datos completa. La primary es por chunk y por un tiempo limitado.

---

## Resumen de flujo completo

### Lectura

```text
1. Cliente calcula chunk index a partir del offset.
2. Cliente pregunta al master por chunk handle y réplicas.
3. Master responde ubicaciones.
4. Cliente lee directamente desde un chunkserver.
```

### Escritura

```text
1. Cliente pregunta al master por primary y secondaries.
2. Cliente envía datos a las réplicas.
3. Cliente pide a la primary aplicar la operación.
4. Primary define el orden.
5. Primary reenvía a secondaries.
6. Secondaries aplican y responden.
7. Primary responde al cliente.
```

### Append

```text
1. Cliente pide append.
2. Primary elige offset.
3. Réplicas aplican en el mismo offset.
4. Puede haber duplicados si hubo reintentos.
5. La aplicación limpia duplicados usando IDs y checksums.
```

---

## Conceptos clave

| Concepto | Definición |
|---|---|
| Chunk | Fragmento grande de un archivo, típicamente 64 MB. |
| Chunk handle | Identificador único de un chunk. |
| Chunkserver | Servidor que almacena chunks en disco local. |
| Master | Coordinador que administra metadatos. |
| Lease | Permiso temporal para que una réplica actúe como primary. |
| Primary | Réplica que define el orden de escrituras para un chunk. |
| Secondary | Réplica que aplica el orden definido por la primary. |
| Record append | Operación para agregar registros al final de un archivo de forma concurrente. |
| Padding | Espacio inválido agregado para completar o saltar una región. |
| Checksum | Valor usado para detectar corrupción o registros inválidos. |
| Split brain | Situación donde dos nodos creen simultáneamente tener autoridad. |

---

## Ideas para recordar

- GFS divide archivos en chunks grandes de 64 MB.
- Cada chunk se replica, típicamente tres veces.
- El master maneja metadatos; los datos van directo entre cliente y chunkservers.
- Las lecturas consultan metadatos y luego leen desde una réplica.
- Las escrituras usan primary y secondaries.
- La primary define el orden de las mutaciones.
- El append es la operación más natural para las cargas de GFS.
- El sistema puede generar duplicados o padding, y la aplicación debe tolerarlo.
- MapReduce aprovecha GFS porque puede ejecutar mappers cerca de los chunks.
- El diseño prioriza throughput, escalabilidad y tolerancia a fallas sobre una semántica fuerte de filesystem tradicional.
