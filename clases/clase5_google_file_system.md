# Clase 5 — Google File System (GFS)

## [Apuntes de Emma](https://drive.google.com/file/d/1FcaEAqVGvVee-Z4JJ6XRvsPyHnz8Rsvt/view)

## 1. Idea general

Google File System, o **GFS**, es un sistema de archivos distribuido diseñado para almacenar archivos muy grandes sobre clusters de máquinas comunes. No busca comportarse exactamente como un filesystem local tradicional, sino ofrecer una abstracción útil para procesamiento distribuido de datos a gran escala.

La motivación central es poder tener un filesystem compartido, global y escalable, donde distintas máquinas puedan leer y escribir archivos sin que la aplicación tenga que saber en qué disco físico está cada fragmento.

GFS combina varias ideas que aparecen repetidamente en sistemas de storage distribuidos:

- **Sharding / particionado**: los archivos se dividen en fragmentos grandes llamados *chunks*.
- **Replicación**: cada chunk se guarda en varias máquinas.
- **Consistencia débil o relajada**: se aceptan garantías más débiles que en un filesystem local para ganar escalabilidad y simplicidad.
- **Recuperación automática**: el sistema detecta fallas y vuelve a replicar datos cuando hace falta.
- **Optimización para procesamiento batch**: está pensado para leer grandes volúmenes de datos y agregar registros al final de archivos.

GFS fue importante porque no fue solo una idea teórica: fue parte de la infraestructura que permitió a Google ejecutar sistemas como MapReduce sobre grandes cantidades de datos.

---

## 2. Requisitos de diseño

GFS fue diseñado para un entorno con características muy concretas.

Primero, los archivos eran muy grandes. En el contexto original se hablaba de archivos de varios GB, que para principios de los 2000 era un tamaño considerable. El sistema debía poder guardar y procesar estos archivos sin depender de una única máquina.

Segundo, el sistema debía escalar horizontalmente. Si el volumen de datos crecía, la solución no podía ser únicamente comprar una máquina más grande. La idea era agregar más máquinas al cluster y distribuir los datos entre ellas.

Tercero, debía tolerar fallas. Al usar muchas máquinas comunes, las fallas de discos, procesos o nodos enteros pasan a ser normales. En un cluster grande siempre puede haber alguna máquina fallando. Por eso el sistema debía estar preparado para seguir funcionando y para no perder datos.

Cuarto, debía funcionar sobre **commodity hardware**. En lugar de comprar una solución de almacenamiento especializada y cara, se usaban máquinas comunes con discos locales, conectadas en red.

Quinto, estaba orientado a cargas **batch**. Es decir, procesos grandes que leen muchos datos, los procesan y generan nuevos archivos. Este patrón encaja muy bien con MapReduce: leer archivos grandes, producir datos intermedios y escribir resultados.

La operación más importante en este tipo de carga no es necesariamente modificar bytes arbitrarios en cualquier posición, sino **agregar datos al final de archivos**, operación conocida como `append`.

---

## 3. Abstracción fundamental: archivos divididos en chunks

La abstracción visible sigue siendo la de un archivo, por ejemplo:

```text
/dir/file.txt
```

Ese archivo se ve como una secuencia de bytes. Sin embargo, internamente no se guarda como una única unidad física, sino que se divide en fragmentos grandes llamados **chunks**.

En GFS, el tamaño típico de chunk es de **64 MB**.

```text
Archivo lógico: /dir/file.txt

+----------+----------+----------+----------+
| chunk 0  | chunk 1  | chunk 2  | chunk 3  |
+----------+----------+----------+----------+
   64 MB      64 MB      64 MB      64 MB
```

Este particionado es una forma de **sharding**. El archivo lógico se divide en partes y esas partes pueden ubicarse en distintas máquinas.

Cada chunk tiene un identificador único llamado **chunk handle**. Ese identificador no depende simplemente del nombre del archivo, sino que es asignado por el master/coordinador cuando el chunk se crea.

Si una aplicación quiere leer desde un offset determinado, el cliente GFS calcula qué chunk corresponde a ese offset:

```text
chunk_index  = offset / 64 MB
chunk_offset = offset % 64 MB
```

De esta forma, una operación como:

```text
read(filename, offset)
```

se transforma internamente en una consulta sobre un chunk específico.

---

## 4. Arquitectura general

GFS tiene tres componentes principales:

1. **Cliente GFS**.
2. **Master o coordinador**.
3. **Chunkservers**.

```text
Aplicación
   |
   v
Cliente GFS  ---- metadata ---->  Master / Coordinador
   |
   |---- datos ---------------->  Chunkserver
   |---- datos ---------------->  Chunkserver
   |---- datos ---------------->  Chunkserver
```

La separación importante es que el **master maneja metadatos**, pero los **datos reales no pasan por el master**. Los datos se leen y se escriben directamente entre el cliente y los chunkservers.

Esto evita que el master se convierta en un cuello de botella para el tráfico pesado de datos.

---

## 5. Master o coordinador

El master es una máquina central que administra la metadata del filesystem. Su rol no es almacenar el contenido de los archivos, sino saber dónde está cada cosa y coordinar algunas decisiones.

Sus responsabilidades principales son:

- Mantener el namespace de archivos y directorios.
- Mapear cada archivo a sus chunks.
- Mapear cada chunk a las máquinas que tienen réplicas.
- Asignar identificadores únicos de chunks.
- Elegir la réplica primaria para escrituras.
- Monitorear chunkservers.
- Detectar fallas.
- Ordenar nuevas réplicas cuando un chunk queda subreplicado.
- Mantener números de versión para detectar réplicas viejas.

Ejemplo conceptual de metadata:

```text
/foo/bar
  chunk 0 -> handle H0 -> CS1, CS2, CS3
  chunk 1 -> handle H1 -> CS2, CS5, CS8
  chunk 2 -> handle H2 -> CS1, CS4, CS7
```

El master mantiene muchas de estas estructuras en memoria para responder rápido. Para no perder información crítica, persiste cambios mediante un **log** y puede crear **snapshots**.

---

## 6. Chunkservers

Los **chunkservers** son las máquinas que almacenan los datos reales. Cada chunkserver usa su disco local y el filesystem local de Linux para guardar chunks.

Cada chunkserver puede:

- Guardar chunks en disco local.
- Leer rangos de bytes de un chunk.
- Recibir datos para una escritura.
- Aplicar operaciones ordenadas por una primary replica.
- Reportar su estado al master mediante heartbeats.
- Participar en recuperación y replicación de chunks.

Un chunk en GFS termina siendo, en la práctica, un archivo local dentro de un chunkserver. La capa distribuida aparece por encima: GFS hace que muchos archivos locales distribuidos parezcan un único filesystem lógico.

---

## 7. Flujo de lectura

Una lectura se divide en dos etapas: primero se consultan metadatos y luego se leen datos.

### 7.1 Consulta al master

La aplicación invoca una lectura sobre un archivo y un offset:

```text
read(/foo/bar, offset)
```

El cliente GFS calcula el índice del chunk y consulta al master:

```text
Cliente -> Master: file name + chunk index
```

El master responde:

```text
Master -> Cliente: chunk handle + ubicaciones de réplicas
```

### 7.2 Lectura directa desde un chunkserver

Con esa información, el cliente elige una réplica. Normalmente intenta elegir la más cercana: una réplica en la misma máquina, en el mismo rack, en el mismo datacenter o con menor costo de red.

Luego le pide directamente al chunkserver el rango de bytes:

```text
Cliente -> Chunkserver: chunk handle + byte range
Chunkserver -> Cliente: datos
```

El master no participa en el envío de datos. Solo ayuda a encontrar dónde están.

El cliente puede cachear la metadata que recibe del master. Esto reduce la cantidad de consultas al master y evita sobrecargarlo.

---

## 8. Replicación

Cada chunk se guarda en varias réplicas. El factor típico de replicación es **3**.

```text
chunk H2
  réplica 1 -> CS1
  réplica 2 -> CS4
  réplica 3 -> CS7
```

La replicación sirve para:

- Seguir leyendo aunque falle una máquina.
- Evitar pérdida de datos si se rompe un disco.
- Permitir recuperación automática.
- Distribuir carga de lectura.

Si un chunkserver falla y algunos chunks quedan con menos réplicas de las necesarias, el master ordena copiar esos chunks desde réplicas sanas hacia otros chunkservers.

---

## 9. Escrituras y problema del orden

La escritura es más complicada que la lectura porque las réplicas deben aplicar las modificaciones en el mismo orden.

Una estrategia ingenua sería que el cliente escriba directamente en todas las réplicas. El problema es que, si hay varios clientes escribiendo concurrentemente, cada réplica podría recibir las operaciones en distinto orden.

Ejemplo:

```text
Cliente 1 quiere escribir x = 1
Cliente 2 quiere escribir x = 2
```

Una réplica podría recibir primero `x = 1` y luego `x = 2`, mientras otra podría recibir primero `x = 2` y luego `x = 1`. El resultado final sería distinto en cada réplica.

Para evitar esto, GFS usa una variante de **primary-backup**.

---

## 10. Primary replica, secondaries y leases

Para cada chunk que se va a modificar, el master elige una réplica como **primary**. Las demás réplicas actúan como **secondaries**.

```text
chunk H2

CS1 -> secondary
CS4 -> primary
CS7 -> secondary
```

La primary tiene una función central: **definir el orden de las escrituras** sobre ese chunk.

El master no convierte a una réplica en primary para siempre. Le otorga un **lease**, es decir, un permiso temporal para actuar como primary. En GFS, el lease dura típicamente **60 segundos** y puede renovarse.

El lease ayuda a evitar situaciones de **split brain**, donde dos réplicas creen simultáneamente que son la primary. Si el master pierde contacto con una primary, puede esperar a que venza el lease antes de elegir otra. De esa forma, aunque la primary vieja siga viva pero aislada, deja de tener autoridad cuando expira su lease.

---

## 11. Flujo de escritura

Una escritura separa el flujo de datos del flujo de control.

### 11.1 Consulta inicial al master

El cliente pregunta al master cuál es la primary y cuáles son las secondaries para el chunk que quiere modificar.

```text
Cliente -> Master: quiero escribir en chunk H2
Master -> Cliente: primary CS4, secondaries CS1 y CS7
```

### 11.2 Envío de datos a las réplicas

Antes de ordenar la escritura, el cliente envía los datos a las réplicas. Este envío puede hacerse en pipeline: el cliente manda los datos a una réplica cercana, esa réplica los reenvía a otra y así sucesivamente.

```text
Cliente -> CS1 -> CS4 -> CS7
```

En esta etapa los datos quedan disponibles en buffers, pero todavía no necesariamente se aplicaron como escritura definitiva.

### 11.3 Orden de escritura

Una vez que las réplicas tienen los datos, el cliente envía la orden de escritura a la primary.

```text
Cliente -> Primary: aplicar write
```

La primary:

1. Asigna un orden a la operación.
2. Aplica la operación localmente.
3. Reenvía la operación a las secondaries en ese mismo orden.
4. Espera respuestas.
5. Responde al cliente.

```text
Cliente
   |
   v
Primary replica ---- orden ----> Secondary A
        |
        +---------- orden ----> Secondary B
```

Si todas las réplicas responden correctamente, el cliente recibe `OK`. Si alguna falla, el cliente recibe error.

---

## 12. Write y append

GFS distingue dos tipos de modificación:

```text
write(file, data, offset)
append(file, data) -> offset
```

### 12.1 Write

En un `write`, el cliente indica explícitamente el offset donde quiere escribir.

```text
write(file, data, offset)
```

Este tipo de operación es más delicado cuando hay concurrencia o fallas, porque puede haber escrituras superpuestas o réplicas parcialmente actualizadas.

### 12.2 Append

En un `append`, el cliente no elige el offset exacto. El sistema decide dónde agregar el registro y devuelve el offset final.

```text
append(file, data) -> offset
```

El `append` es la operación más importante para las cargas batch de GFS. Muchos procesos no necesitan modificar bytes arbitrarios, sino agregar nuevos registros al final de archivos compartidos.

---

## 13. Record append

El **record append** permite que muchos clientes agreguen registros concurrentemente al mismo archivo.

La primary decide dónde ubicar cada registro. Luego ordena a las secondaries que escriban el mismo registro en el mismo offset.

Ejemplo de caso exitoso:

```text
Cliente 1 append(A)
Cliente 2 append(B)
Cliente 3 append(C)

Resultado lógico:

+---+---+---+
| A | B | C |
+---+---+---+
```

El orden exacto puede depender del orden en que la primary reciba las operaciones, pero todas las réplicas deben aplicar ese mismo orden.

---

## 14. Consistencia débil: duplicados, padding e inconsistencias

GFS no ofrece un `atomic write` fuerte como el que uno esperaría en otros sistemas de storage más estrictos.

Puede pasar que una escritura se aplique en algunas réplicas y falle en otras. En ese caso, la región queda **inconsistente**: distintas réplicas pueden tener datos diferentes.

También puede pasar que un cliente reintente un `append` después de un error. Si la primera operación llegó a algunas réplicas y luego el cliente reintenta, el mismo registro puede aparecer duplicado.

Ejemplo conceptual:

```text
Replica primaria:     A B C B
Replica secundaria 1: A B C B
Replica secundaria 2: A padding C B
```

En este ejemplo:

- `B` puede aparecer duplicado por un retry.
- `padding` representa una zona inválida o de relleno.
- Algunas réplicas pueden haber quedado con regiones diferentes.

GFS acepta esta semántica porque está orientado a cargas donde las aplicaciones pueden manejar estas situaciones.

---

## 15. Cómo resolver inconsistencias desde la aplicación

Parte de la responsabilidad se traslada al cliente o a la aplicación que usa GFS.

### 15.1 IDs únicos

Cada registro puede incluir un identificador único.

```text
record_id = 123
payload = ...
```

Si el mismo registro aparece dos veces por un retry, la aplicación puede detectar el duplicado y procesarlo una sola vez.

### 15.2 Checksums

Cada registro puede incluir un checksum o algún mecanismo de validación.

```text
[record_id][payload][checksum]
```

Al leer, si una región no tiene formato válido o el checksum no coincide, se descarta. Esto permite ignorar padding, basura o registros parciales.

Estas estrategias son necesarias porque GFS no intenta esconder completamente todos los problemas de consistencia. La aplicación se diseña sabiendo que puede encontrar duplicados o regiones inválidas.

---

## 16. Relación entre GFS y MapReduce

GFS y MapReduce encajan muy bien porque ambos están diseñados para procesamiento batch sobre grandes volúmenes de datos.

MapReduce necesita leer archivos enormes, dividirlos en partes, procesarlos en paralelo y escribir resultados. GFS provee justamente una forma de almacenar esos archivos grandes y partirlos en chunks.

Una optimización clave es la **localidad de datos**.

```text
Host físico

+--------------------+
| MapReduce worker   |
| GFS chunkserver    |
| Disco local        |
|  - chunk 0         |
|  - chunk 7         |
+--------------------+
```

Si un worker de MapReduce corre en la misma máquina donde está el chunk que necesita leer, puede leer localmente o con menor costo de red.

El master de MapReduce puede preguntarle al coordinador de GFS dónde están los chunks y asignar tareas intentando colocar el cómputo cerca de los datos.

```text
Archivo grande en GFS

+----------+----------+----------+----------+
| chunk 0  | chunk 1  | chunk 2  | chunk 3  |
+----------+----------+----------+----------+
    |          |          |          |
    v          v          v          v
  mapper     mapper     mapper     mapper
```

La idea no es mover todos los datos hacia el cómputo, sino mover el cómputo hacia donde ya están los datos.

---

## 17. Datos que mantiene el coordinador

El coordinador mantiene metadata en memoria, pero persiste cambios críticos usando log y snapshots.

Entre los datos importantes están:

| Metadata | Función |
|---|---|
| Filename -> chunk handles | Saber qué chunks forman cada archivo. |
| Chunk handle -> chunkservers | Saber dónde están las réplicas de cada chunk. |
| Chunk handle -> version number | Detectar réplicas viejas. |
| Leases activos | Saber qué réplica puede actuar como primary. |
| Estado de chunkservers | Detectar fallas y re-replicar datos. |

Hay una diferencia importante: el mapeo de archivos a chunks debe persistirse porque si se pierde, se pierde la estructura lógica del filesystem. En cambio, el master puede reconstruir parte de la información de ubicación preguntándoles a los chunkservers qué chunks tienen.

---

## 18. Versiones de chunks

Cada chunk tiene un número de versión. Este número sirve para detectar réplicas obsoletas.

Ejemplo:

```text
chunk H2
  CS1 -> versión 10
  CS4 -> versión 11
  CS7 -> versión 11
```

En este caso, CS1 tiene una versión vieja. Puede haber estado caído o desconectado mientras el sistema avanzó.

El número de versión ayuda a que el master, los clientes y los chunkservers distingan réplicas válidas de réplicas desactualizadas.

---

## 19. Leases y prevención de split brain

El lease es un permiso temporal para que una réplica actúe como primary.

```text
Master -> CS4: sos primary de H2 por 60 segundos
```

Mientras el lease está vigente, CS4 puede ordenar escrituras para ese chunk. Si quiere seguir siendo primary, debe renovar el lease.

Esto evita un problema típico de sistemas distribuidos: que el master crea que una primary murió, elija otra, pero la primary vieja siga aceptando escrituras porque algunos clientes todavía la tienen cacheada.

Con leases, la primary vieja deja de tener autoridad cuando vence su permiso. Entonces, antes de elegir una nueva primary, el master puede esperar a que expire el lease anterior.

```text
1. Master pierde contacto con primary vieja.
2. Espera a que venza el lease.
3. Elige una nueva primary.
```

Esta espera reduce el riesgo de tener dos primaries activas para el mismo chunk.

---

## 20. Fallas del coordinador

El master/coordinador es centralizado, por lo que es un punto sensible del diseño.

Para recuperarse, usa dos mecanismos:

1. **Persistencia local**: escribe metadata crítica en disco mediante log y snapshots.
2. **Primary-backup fijo**: puede haber un coordinador backup en standby que recibe el log del coordinador principal.

```text
Coordinador principal ---- log ----> Coordinador backup
```

El backup no participa activamente mientras el principal está vivo. Queda en espera para tomar el rol si el coordinador principal falla.

Este diseño simplifica mucho el sistema, pero limita la escalabilidad y la tolerancia a fallas del coordinador.

---

## 21. Problemas y limitaciones de GFS

GFS funcionó muy bien para el tipo de carga para el que fue diseñado, pero tiene limitaciones importantes.

### 21.1 Coordinador centralizado

El master centralizado simplifica el diseño porque hay una autoridad única para metadata, leases y detección de fallas. Pero también implica:

- Menor tolerancia a fallas.
- Menor escalabilidad de metadata.
- Riesgo de quedarse sin memoria si la metadata crece demasiado.

Una evolución posterior de GFS, conocida como **Colossus**, distribuyó mejor esta función de metadata.

### 21.2 Consistencia débil

GFS no ofrece escrituras atómicas fuertes en todos los casos. Una escritura puede quedar parcialmente aplicada y generar regiones inconsistentes.

Esto obliga a que la aplicación tolere:

- Duplicados.
- Padding.
- Registros inválidos.
- Lecturas de réplicas desactualizadas.

### 21.3 Stale secondaries

Si un cliente tiene cacheada la ubicación de una secondary vieja y la red se particiona, puede seguir leyendo datos desactualizados.

El sistema tiene mecanismos de versiones y leases para reducir estos problemas, pero no ofrece una semántica fuerte como la que luego se busca en sistemas basados en algoritmos como Raft.

---

## 22. Comparación conceptual con Raft

GFS usa una coordinación relativamente simple basada en master, primary replica, secondaries y leases. Esto le permite ser eficiente para batch, pero deja algunas responsabilidades al cliente.

Raft, en cambio, apunta a resolver replicación con garantías más fuertes:

- Escrituras atómicas.
- Orden de operaciones bien definido.
- Evitar divergencia entre réplicas.
- Elección de líder integrada al protocolo.
- Lecturas consistentes si se hacen correctamente.

La contracara es que Raft es más complejo de implementar y entender.

GFS elige un punto distinto del espacio de diseño: acepta semánticas más débiles porque su carga principal puede convivir con ellas.

---

## 23. Resumen de flujo completo

### Lectura

```text
1. La aplicación pide read(filename, offset).
2. El cliente calcula el chunk index.
3. El cliente pregunta al master por chunk handle y réplicas.
4. El master devuelve ubicaciones.
5. El cliente elige una réplica cercana.
6. El cliente lee directamente desde el chunkserver.
```

### Escritura

```text
1. El cliente pregunta al master por primary y secondaries.
2. El cliente envía los datos a las réplicas.
3. El cliente manda la orden de escritura a la primary.
4. La primary define el orden.
5. La primary aplica localmente.
6. La primary ordena a las secondaries aplicar la operación.
7. Si todas responden OK, el cliente recibe OK.
8. Si alguna falla, el cliente recibe error y puede reintentar.
```

### Append

```text
1. El cliente pide append(file, data).
2. La primary elige el offset final.
3. La primary ordena a las secondaries escribir en ese offset.
4. Si hay error, el cliente puede reintentar.
5. El reintento puede generar duplicados.
6. La aplicación usa IDs únicos y checksums para limpiar el resultado.
```

---

## 24. Conceptos clave

| Concepto | Definición |
|---|---|
| GFS | Sistema de archivos distribuido de Google. |
| Chunk | Fragmento grande de un archivo, típicamente de 64 MB. |
| Chunk handle | Identificador único de un chunk. |
| Chunkserver | Servidor que almacena chunks en disco local. |
| Master / coordinador | Nodo que administra metadata y coordinación. |
| Metadata | Información sobre archivos, chunks, ubicaciones y versiones. |
| Sharding | División de archivos en chunks distribuidos. |
| Replicación | Copias de un chunk en varios chunkservers. |
| Primary | Réplica que ordena escrituras para un chunk. |
| Secondary | Réplica que sigue el orden definido por la primary. |
| Lease | Permiso temporal para actuar como primary. |
| Record append | Operación optimizada para agregar registros al final de un archivo. |
| Padding | Región inválida o de relleno insertada por el sistema. |
| Checksum | Valor usado para validar registros o detectar corrupción. |
| Stale replica | Réplica vieja o desactualizada. |
| Split brain | Situación donde dos nodos creen tener autoridad al mismo tiempo. |

---

## 25. Ideas para recordar

- GFS está pensado para archivos enormes y procesamiento batch.
- Los archivos se dividen en chunks de 64 MB.
- Cada chunk se replica, típicamente tres veces.
- El master administra metadata, no datos.
- Los clientes leen y escriben directamente contra chunkservers.
- Las lecturas son simples: consultar metadata y leer desde una réplica.
- Las escrituras requieren una primary que ordene operaciones.
- El lease define temporalmente qué réplica puede actuar como primary.
- El `append` es la operación central para las cargas de GFS.
- El sistema puede producir duplicados, padding o regiones inconsistentes.
- La aplicación debe usar IDs únicos y checksums para tolerar esas situaciones.
- GFS y MapReduce se complementan porque MapReduce puede ejecutar cómputo cerca de los chunks.
- El diseño prioriza throughput, escalabilidad y tolerancia a fallas sobre consistencia fuerte.
