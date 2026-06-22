# Clase 16 — Procesamiento de data streams

## Procesamiento de streams

El procesamiento de streams consiste en tomar datos que llegan continuamente y procesarlos a medida que aparecen.

A diferencia del procesamiento por lotes, donde se juntan muchos archivos y se ejecuta un trabajo sobre todo el conjunto, en streaming el input no está cerrado. Los eventos siguen llegando todo el tiempo y el sistema debe producir resultados con baja latencia.

Una forma de verlo es como una extensión de MapReduce hacia datos en tiempo real:

```text
Batch processing:
archivos completos ──► job ──► archivos de salida

Stream processing:
eventos continuos ──► procesamiento permanente ──► eventos / acciones / métricas
```

El objetivo no es esperar al final del día para procesar todo, sino ir actualizando resultados en segundos o incluso menos.

## Ejemplo motivador: anomalías de búsquedas

Un ejemplo típico es detectar anomalías en búsquedas. Supongamos que llegan eventos de búsquedas hechas por usuarios:

```text
8:00  búsqueda = "Britney"
8:02  búsqueda = "Carly"
8:03  búsqueda = "Britney"
```

Cada evento contiene al menos:

- una clave o término buscado;
- un timestamp que indica cuándo ocurrió el evento;
- posiblemente más datos asociados.

Con esos eventos se pueden armar ventanas temporales y contar cuántas veces apareció cada búsqueda por minuto, por segundo o por cualquier intervalo.

```text
Ventana 8:00 - 8:01  ──► Britney: 1
Ventana 8:01 - 8:02  ──► ...
Ventana 8:02 - 8:03  ──► Carly: 1
Ventana 8:03 - 8:04  ──► Britney: 1
```

A partir de esos conteos se puede detectar si una búsqueda sube o baja de manera anómala respecto de un modelo esperado.

La dificultad aparece cuando los eventos llegan desordenados. Por ejemplo, a las 8:10 puede llegar un evento cuyo timestamp real era 8:01.

```text
Processing time: 8:10
Event time:      8:01
Evento:          búsqueda = "Britney"
```

Ese evento pertenece a una ventana que, conceptualmente, ya parecía cerrada. El sistema debe decidir si corrige el resultado anterior, si ignora el evento tardío o si demora el cierre de las ventanas para esperar posibles atrasos.

## Características del problema

Los problemas de procesamiento de streams suelen tener varias propiedades que los hacen más difíciles que un procesamiento batch común.

### Datasets muy grandes

La fuente de datos suele ser actividad de usuarios o de sistemas: búsquedas, clics, visualizaciones, eventos de navegación, transacciones, logs, sensores o métricas.

El volumen puede ser enorme. No alcanza con una sola máquina procesando todo; hace falta distribuir el trabajo.

### Asociados a tiempo real

Muchos resultados tienen valor solo si se calculan rápido. Por ejemplo:

- gráficos de visualizaciones en vivo;
- detección de fraude;
- monitoreo de anomalías;
- clicks en anuncios;
- métricas operativas;
- features para modelos de inteligencia artificial.

Si se espera al final del día, el resultado puede seguir siendo correcto desde el punto de vista histórico, pero ya no sirve para reaccionar en el momento.

### Unbounded

El stream no tiene un final natural.

Las búsquedas de Google, los clicks en una aplicación o las visualizaciones de YouTube no terminan. Siempre puede llegar otro evento. Esto cambia la forma de pensar los algoritmos: ya no se procesa “todo el archivo”, sino una secuencia potencialmente infinita.

### Posiblemente desordenados

Los eventos pueden llegar fuera de orden.

El orden de llegada al sistema no necesariamente coincide con el orden en que los eventos ocurrieron. Esto puede pasar por latencia de red, buffers, retries, particiones, productores distribuidos o diferencias entre relojes.

### Posiblemente heterogéneos

Muchas veces no alcanza con un solo stream. Puede haber que combinar varios:

```text
Stream de push notifications
Stream de accesos a la app
Stream de compras
Stream de eventos de usuario
```

Cuando se combinan streams distintos, el problema se vuelve más difícil porque hay que alinear eventos por tiempo, por clave o por ambos.

## Objetivos de un sistema de stream processing

Un sistema de procesamiento de streams busca resolver varios objetivos al mismo tiempo.

### Procesamiento en tiempo real

Los eventos deben procesarse con baja latencia.

No se busca un resultado perfecto dentro de muchas horas, sino resultados actualizados mientras el stream sigue llegando.

### Replay

En algunos sistemas es importante poder reprocesar eventos viejos.

Esto depende de la fuente de datos. Si la fuente es Kafka, por ejemplo, los eventos pueden quedar persistidos durante cierto período, y un consumidor puede volver a leer desde una posición anterior.

El replay permite reconstruir estado, corregir bugs o recalcular métricas con una versión nueva del código.

### Escalabilidad

El sistema debe poder procesar volúmenes altos agregando más máquinas.

Para eso se particiona el stream y se reparte el trabajo entre nodos.

### Tolerancia a fallas

Si una máquina falla, el procesamiento debe continuar.

Esto requiere persistir estado, detectar nodos caídos, reasignar particiones y evitar que un mismo evento produzca efectos duplicados.

## Por qué MapReduce no alcanza

MapReduce está pensado para procesamiento por lotes.

Es escalable, porque se puede repartir trabajo entre muchas máquinas, pero no está diseñado para baja latencia. Levantar un job, procesar un lote y escribir la salida es relativamente pesado.

```text
MapReduce:
input completo ──► map ──► shuffle ──► reduce ──► output completo
```

Eso funciona bien cuando el dataset está cerrado, pero no cuando llegan eventos en tiempo real y se necesita actualizar resultados continuamente.

## Microbatching

Una forma intermedia es el **microbatching**.

En vez de procesar un lote enorme al final del día, se divide el tiempo en lotes pequeños:

```text
8:00:00 - 8:00:01 ──► batch 1
8:00:01 - 8:00:02 ──► batch 2
8:00:02 - 8:00:03 ──► batch 3
```

Spark Streaming usa una idea de este estilo: procesar pequeños lotes sucesivos.

Esto mejora la latencia respecto de un batch diario, pero sigue teniendo problemas con eventos tardíos. Si un evento de las 8:00:05 llega cuando ya se cerró y procesó el microbatch correspondiente, hay que decidir qué hacer.

Las opciones generales son:

- ignorarlo;
- emitir una corrección;
- reabrir o recalcular la ventana;
- usar un mecanismo más sofisticado para saber cuándo cerrar ventanas.

## Arquitectura general de streaming

Un sistema de stream processing puede pensarse así:

```text
Fuente de eventos ──► Procesamiento ──► Salida
     Kafka              Stream processor      Kafka / RPC / DB / alerta
```

La fuente puede ser Kafka u otro sistema de logs/eventos. El procesador consume eventos, aplica lógica y produce nuevos eventos o acciones.

La salida puede ser:

- otro stream;
- una base de datos;
- una llamada RPC;
- una alerta;
- una métrica;
- un cambio de estado en otro sistema.

## Tipos de problemas

Hay problemas simples y problemas difíciles.

La diferencia principal es si cada evento puede procesarse de forma independiente o si hay que agrupar eventos por clave, por tiempo o por ambos.

## Problemas fáciles: procesamiento por elemento

Los problemas fáciles son aquellos en los que cada evento puede procesarse individualmente.

### Filter

Un filtro recibe eventos y decide cuáles deja pasar.

```text
Stream original ──► filter ──► Stream filtrado
```

Ejemplos:

- dejar pasar solo eventos de cierto país;
- descartar eventos inválidos;
- quedarse solo con búsquedas de una categoría;
- filtrar clicks sospechosos.

Esto es fácil de distribuir porque cada evento se evalúa de manera independiente.

### Transform / map

Una transformación toma un evento y emite otro evento derivado.

```text
Evento original ──► transform ──► Evento transformado
```

Ejemplos:

- normalizar campos;
- agregar un campo calculado;
- cambiar el formato;
- transformar una unidad;
- emitir varios eventos a partir de uno.

También es fácil de paralelizar, porque no requiere mirar otros eventos.

### Enriquecer un stream

Un caso común de transformación es enriquecer eventos con datos de una tabla.

```text
Stream: user_id
        │
        ▼
  Consulta a DB
        │
        ▼
Stream enriquecido: user_id, nombre, dirección, país, etc.
```

Esto puede verse como un **join entre stream y tabla**.

```text
Stream + Table ──► Stream enriquecido
```

Conceptualmente es simple: por cada evento se consulta una base de datos y se agrega información.

La dificultad práctica es la performance. Si llega un volumen muy alto, no se puede hacer una query individual por cada evento sin saturar la base. Hace falta batching, caching o algún mecanismo para amortiguar consultas.

## Problemas difíciles: agregados temporales

Los problemas difíciles aparecen cuando hay que agrupar eventos por ventanas de tiempo.

En términos de bases de datos, se parece a:

```sql
GROUP BY key, window(timestamp)
```

Ejemplo:

```text
Clicks por segundo
Búsquedas por minuto
Visualizaciones por cada 5 minutos
Fraudes por ventana temporal
```

El problema es que el stream no termina y los eventos pueden llegar desordenados.

## Event time y processing time

Para entender ventanas temporales hay que distinguir dos tiempos.

### Event time

Es el momento en que el evento ocurrió realmente.

Suele venir dentro del evento como metadata.

```text
Evento:
  key = "Britney"
  timestamp = 8:03
```

Ese timestamp representa el **event time**.

### Processing time

Es el momento en que el sistema procesa el evento.

Un evento ocurrido a las 8:03 puede procesarse a las 8:03:01, a las 8:05 o a las 8:20.

```text
Event time:      8:03
Processing time: 8:10
```

Para agregados temporales, normalmente interesa agrupar por **event time**, no por processing time. Si se agrupara por processing time, el resultado dependería de la latencia del sistema y no de cuándo ocurrió realmente el evento.

## Eventos fuera de orden

Los eventos pueden llegar fuera de orden.

```text
Llega primero:  event_time = 8:03
Llega después:  event_time = 8:01
```

Esto rompe la idea simple de “cierro la ventana cuando termina el minuto”.

Si la ventana 8:00 - 8:05 ya fue emitida y luego llega un evento con event time 8:02, hay dos posibilidades principales.

### Ignorar el evento tardío

Se descarta el evento porque llegó demasiado tarde.

Esto simplifica el sistema, pero introduce pérdida o imprecisión.

### Corregir la ventana

Se recalcula o corrige el resultado emitido previamente.

Esto mantiene mayor exactitud, pero obliga a que los consumidores de la salida soporten actualizaciones o correcciones.

## Stream-stream join

Un caso difícil es combinar dos streams.

Por ejemplo:

```text
Stream A: push notifications enviadas
Stream B: accesos a la app
```

Se quiere correlacionar si un usuario abrió la app dentro de los 5 minutos posteriores a recibir una notificación.

```text
Push notification:
  user_id, push_id, timestamp

Acceso a app:
  user_id, path, timestamp
```

La salida podría ser:

```text
user_id, push_id, path
```

Para hacer esto hay que alinear los dos streams por tiempo y por usuario.

```text
Push notifications ─┐
                    ├──► Stream processor ──► Correlaciones
Accesos a la app ───┘
```

El procesador debe esperar eventos de ambos lados, mantener estado temporal y cerrar ventanas cuando tiene suficiente seguridad de que ya no llegarán eventos relevantes.

## Cuándo cerrar una ventana

El problema central es decidir hasta cuándo esperar eventos viejos.

Supongamos una ventana:

```text
8:00 ───────────── 8:05
```

Al llegar a las 8:05 en processing time, no necesariamente se puede cerrar la ventana, porque todavía pueden llegar eventos con event time dentro de esa ventana.

Se necesita un **punto de corte**: una garantía o estimación de que ya no van a llegar eventos anteriores a cierto timestamp.

```text
Eventos procesados hasta ahora
        │
        ▼
8:00 ───────────── 8:05 ───────────── 8:09
                         ▲
                  punto de corte
```

Cuando el punto de corte supera el final de la ventana, la ventana puede cerrarse.

## Idea 1: delay fijo

La solución más simple es esperar un tiempo fijo.

Por ejemplo, si la ventana termina a las 8:05, se espera 10 minutos antes de cerrarla.

```text
Ventana: 8:00 - 8:05
Delay:   10 minutos
Cierre:  8:15
```

Esto es fácil de implementar, pero tiene problemas:

- el delay es arbitrario;
- si es muy corto, llegan eventos tardíos después del cierre;
- si es muy largo, aumenta la latencia;
- en el extremo, si se espera demasiado, el sistema se parece otra vez a batch.

Cuanto más seguro se quiere estar, más lento se vuelve el sistema.

## Idea 2: watermark

La solución más interesante es que la fuente informe un **watermark**.

Un watermark es una marca temporal que indica:

```text
Ya no debería llegar ningún evento con event_time menor que este valor.
```

Si el watermark llega a 8:05, entonces se puede cerrar la ventana que termina en 8:05.

```text
Ventana:   8:00 - 8:05
Watermark: 8:05
Resultado: cerrar ventana
```

La metáfora es la marca de agua que va subiendo: el watermark avanza de manera monótona y permite saber hasta dónde el stream está “completo”.

## Watermark en Kafka

Kafka ayuda a entender una versión simple del watermark.

Dentro de una partición, los mensajes tienen un orden. Si los timestamps son asignados de manera monótona dentro de esa partición, entonces al ver un timestamp `T`, se sabe que los siguientes eventos de esa partición tendrán timestamp mayor o igual.

```text
Partición 1:  9  ── 11 ── 12 ── 15
Partición 2:  8  ── 10 ── 13 ── 20
Partición 3:  7  ── 14 ── 18 ── 22
```

Por partición:

```text
si vi T, los siguientes eventos tendrán timestamp >= T
```

Para un stream compuesto por varias particiones, el watermark global es el mínimo de los watermarks de cada partición.

```text
watermark(stream) = min(watermark(partición_1), watermark(partición_2), ...)
```

La partición más atrasada define hasta dónde se puede garantizar completitud.

## Watermarks exactos y heurísticos

Hay dos tipos generales de watermarks.

### Watermark exacto

El sistema puede garantizar con precisión que no vendrán eventos más viejos.

Kafka puede aproximar este caso cuando el timestamp relevante es el timestamp asignado por Kafka en una partición ordenada.

### Watermark heurístico

El sistema estima el watermark.

Por ejemplo, si se sabe que los productores están sincronizados con un servidor de tiempo y que el atraso máximo esperado es bajo, se puede usar:

```text
watermark ≈ tiempo_actual - delta
```

Ese delta puede ser 1 segundo, algunos milisegundos o más, según la confianza que se tenga en la fuente.

Este watermark no es una garantía perfecta, pero permite operar con baja latencia.

## Visualización: event time vs processing time

Una forma de visualizar streams es usar dos ejes:

```text
processing time ↑
                │
                │
                │
                └────────────────► event time
```

La diagonal representa eventos que se procesan exactamente cuando ocurren.

```text
processing time = event time
```

Los eventos reales suelen caer a la izquierda de esa diagonal: ocurrieron antes de ser procesados.

El watermark marca una frontera. En teoría, los eventos deberían aparecer entre la diagonal perfecta y el watermark. Si aparece un evento más viejo que el watermark, es un evento tardío.

```text
processing time ↑
                │        diagonal ideal
                │       /
                │      /   eventos esperados
                │     /  • • •
                │    / •
                │   /________ watermark
                └────────────────► event time
```

Cuando el watermark cruza el final de una ventana, esa ventana puede emitirse.

## Modelo de programación de MillWheel

MillWheel propone un modelo de programación para procesar streams distribuidos.

La idea es similar en espíritu a MapReduce: el programador implementa una interfaz relativamente chica y el sistema se encarga de distribuir, reintentar, persistir estado y coordinar.

Pero el modelo no es solo `map` y `reduce`. Es un grafo dirigido acíclico de cómputos.

```text
Stream de entrada ──► Compute A ──► Compute B ──► Stream de salida
                         │
                         └────────► Compute C
```

Cada nodo del grafo es un **cómputo**. Cada arista representa un stream lógico entre cómputos.

Internamente, cada cómputo puede ejecutarse en muchas máquinas y cada máquina procesa una partición del espacio de claves.

## Records

El sistema trabaja con records.

Un record contiene:

```text
Record(key, data, timestamp)
```

- `key`: clave usada para particionar y agrupar;
- `data`: payload del evento;
- `timestamp`: event time del evento.

La clave es fundamental porque el sistema distribuye el trabajo por clave.

## Hooks principales

El programador implementa principalmente dos funciones.

### process_record

Se ejecuta cada vez que llega un record.

```python
def process_record(record):
    ...
```

Sirve para procesar eventos, actualizar estado, producir nuevos records o registrar timers.

### process_timer

Se ejecuta cuando dispara un timer.

```python
def process_timer(timer):
    ...
```

Sirve para cerrar ventanas, emitir agregados o ejecutar lógica diferida.

## Funciones provistas por el framework

Además de los hooks, el framework provee primitivas.

### set_timer

Registra un timer para que el sistema llame luego a `process_timer`.

```python
set_timer(tag, timestamp)
```

El timer dispara cuando el watermark supera el timestamp indicado.

### produce_record

Emite un record hacia otro stream del grafo.

```python
produce_record(record, "stream-de-salida")
```

### mutable_persistent_state

Permite acceder a estado mutable persistente asociado a la clave.

```python
state = mutable_persistent_state()
```

Esto reemplaza a una variable global en memoria. Como el sistema es distribuido y tolerante a fallas, el estado debe persistirse fuera del proceso local.

## Ejemplo: Windower

Un `Windower` cuenta eventos por ventana.

Primera versión conceptual, usando una variable global:

```python
window_counts = {}

class Windower:
    def process_record(self, record):
        bucket = window_id(record.timestamp)
        tag = (record.key, bucket)

        window_counts[tag] = window_counts.get(tag, 0) + 1

        set_timer(tag, window_boundary(record.timestamp))

    def process_timer(self, timer):
        key, bucket = timer.tag
        count = window_counts.get(timer.tag, 0)

        record = Record(key, count, timer.timestamp)
        produce_record(record, "windows")
```

La función `window_id` calcula a qué ventana pertenece el timestamp.

Por ejemplo, con ventanas de 5 minutos:

```text
8:03:01 ──► ventana 8:00:00 - 8:05:00
8:06:01 ──► ventana 8:05:00 - 8:10:00
```

La función `window_boundary` devuelve el final de la ventana. Al registrar un timer en ese boundary, el sistema avisa cuando el watermark ya permite cerrar la ventana.

## Estado persistente mutable

La versión con variable global no sirve en un sistema distribuido.

Si el cómputo se ejecuta en varias máquinas, cada una tendría su propia memoria. Además, si una máquina falla, se pierde el estado local.

Por eso MillWheel ofrece estado persistente mutable.

```python
class Windower:
    def process_record(self, record):
        state = WindowState(mutable_persistent_state())

        state.update_bucket_count(record.timestamp)

        window = window_id(record.timestamp)
        set_timer(window, window_boundary(record.timestamp))

    def process_timer(self, timer):
        state = WindowState(mutable_persistent_state())

        count = state.window_count(timer.tag)
        record = Record(key(), count, timer.timestamp)
        produce_record(record, "windows")
```

`WindowState` es una abstracción definida por el usuario que interpreta los bytes persistidos por el framework.

El framework se encarga de persistir el estado modificado al finalizar cada hook.

## Ejemplo: DipDetector

Un segundo cómputo puede consumir las ventanas producidas por `Windower` y detectar caídas o anomalías.

```python
class DipDetector:
    def process_record(self, record):
        state = DipState(mutable_persistent_state())

        prediction = state.get_prediction(record.timestamp)
        actual = record.data

        state.update_confidence(prediction, actual)

        if state.confidence() > CONFIDENCE_THRESHOLD:
            dip = Record(key(), state.confidence(), record.timestamp)
            produce_record(dip, "dip-stream")
```

La idea es comparar el valor real de una ventana con una predicción del modelo. Si la confianza de anomalía supera un umbral, se emite un evento al stream de salida.

## Escalabilidad: particionado por clave

Para escalar, MillWheel particiona el trabajo por clave.

Cada record tiene una `key`. Esa key determina qué nodo procesa ese record.

```text
Record(key, bytes, timestamp)
```

Entre cómputos ocurre una especie de shuffle continuo.

```text
Compute A                    Compute B
partición A1 ──────────────► partición B1
partición A1 ──────────────► partición B2
partición A2 ──────────────► partición B1
partición A2 ──────────────► partición B3
```

En MapReduce, el shuffle ocurre una vez entre map y reduce. En streaming, el shuffle es permanente: los nodos siguen enviándose records mientras el sistema está vivo.

## Rangos dinámicos

En MapReduce se podía usar algo como:

```text
hash(key) % N
```

Eso funciona si `N` es fijo.

En MillWheel, el número de nodos y particiones puede cambiar dinámicamente. Por eso se usan rangos de claves.

```text
[A - H)  ──► nodo 1
[H - P)  ──► nodo 2
[P - Z]  ──► nodo 3
```

Si una partición está muy cargada, se puede partir:

```text
[A - H)  ──► [A - D) + [D - H)
```

Esto permite redistribuir trabajo sin rehacer toda la asignación.

## Comunicación entre nodos

Las aristas del grafo lógico se implementan con comunicación directa entre nodos.

No hay necesariamente un Kafka intermedio entre cada cómputo. Los nodos se envían records por RPC.

```text
Compute A ── RPC ──► Compute B
```

Para eficiencia, es esperable que el sistema agrupe varios records y los mande en batches.

## Master y asignación de trabajo

Hay un master que asigna trabajo a los nodos.

El master no es una única máquina frágil: internamente puede estar replicado con Paxos o un mecanismo similar a Raft.

El master decide:

- qué cómputo ejecuta cada nodo;
- qué rango de claves le corresponde;
- qué epoch tiene esa asignación.

Una asignación puede verse así:

```text
assign(compute_id, key_range, epoch)
```

Ejemplo:

```text
compute_id = WindowCounter
key_range  = [A - H)
epoch      = 4
```

El nodo asignado empieza a procesar records de ese cómputo y ese rango.

## Backing store y recuperación de estado

El estado persistente se guarda en un backing store, que puede estar implementado sobre Bigtable o Spanner.

Cuando un nodo recibe una asignación, primero carga el estado persistido:

```text
reload_state(compute_id, key_range, epoch)
```

Si es la primera vez que se procesa ese rango, no hay estado previo. Si el nodo está reemplazando a otro que falló, carga el estado que el anterior había guardado.

Esto permite tolerar fallas: el procesamiento puede continuar desde el último estado persistido.

## Epochs y zombies

Puede pasar que un nodo parezca muerto porque falla un heartbeat o se corta la red, pero en realidad siga vivo. Ese nodo viejo puede intentar seguir procesando y escribiendo estado.

A ese caso se lo puede pensar como un **zombie**.

Para evitar que un zombie pise el estado nuevo, cada asignación tiene un epoch.

```text
Nodo viejo: epoch 3
Nodo nuevo: epoch 4
```

El backing store acepta escrituras solo si el epoch coincide con el epoch vigente.

```text
set_state(state, epoch=3) ──► rechazado
set_state(state, epoch=4) ──► aceptado
```

Así, aunque el nodo viejo siga vivo, no puede corromper el estado del rango reasignado.

## Semántica exactly-once

El sistema busca que cada record tenga efecto una sola vez.

En sistemas distribuidos, `exactly once` puro es imposible como garantía absoluta de comunicación. Lo que se puede lograr es un efecto equivalente combinando retries con idempotencia.

Hay tres semánticas clásicas:

| Semántica | Idea | Problema |
|---|---|---|
| At-most-once | Se intenta una vez, sin retry | El mensaje puede perderse |
| At-least-once | Se reintenta hasta recibir ACK | El mensaje puede procesarse más de una vez |
| Exactly-once | Se procesa exactamente una vez | No es posible de forma directa en sistemas distribuidos |

La aproximación práctica es **effectively-once**:

```text
at-least-once + idempotencia = effectively-once
```

El sistema puede enviar varias veces, pero garantiza que los efectos observables se materialicen una sola vez.

## Idempotencia mediante IDs

Cada record se marca con un ID de idempotencia.

```text
send(record, id)
```

El receptor consulta el backing store para ver si ese ID ya fue procesado.

```text
check(id)
```

Si el ID ya existe, el record se descarta porque ya fue procesado.

Si no existe, se ejecuta `process_record`.

## Buffer de efectos colaterales

`process_record` puede tener efectos colaterales lógicos:

- modificar estado mutable;
- crear timers;
- producir records de salida.

Pero esos efectos no se publican inmediatamente. Primero se acumulan en memoria.

Al terminar el hook, el sistema guarda atómicamente en el backing store:

```text
estado mutable actualizado
producciones pendientes
timers nuevos
ID de idempotencia procesado
```

Esto es clave. Si falla antes de guardar, no quedó ningún efecto persistido y el record puede reintentarse.

Si guarda correctamente, el ID queda marcado como procesado junto con todos sus efectos.

Después de ese update atómico, se puede responder ACK al emisor.

```text
A ── send(record, id) ──► B
B ── check(id) ────────► Store
B ── process_record ───► memoria
B ── update(...) ──────► Store
B ── ack ──────────────► A
```

Luego, las producciones pendientes se envían al siguiente cómputo.

Este esquema evita que un record sea contado dos veces aunque haya retries, fallas o zombies.

## Watermarks dentro del grafo

Los watermarks nacen en las fuentes del sistema, por ejemplo en inyectores que consumen desde Kafka.

Luego se propagan por el grafo.

```text
Fuente ──► A ──► B ──► C
```

Si `A` está más cerca de la fuente que `B`, y `B` más cerca que `C`, entonces normalmente:

```text
W(A) >= W(B) >= W(C)
```

Cuanto más lejos está una etapa de la fuente, más atrasado puede estar su watermark.

## Cálculo del watermark de una etapa

El watermark de una etapa depende de dos cosas:

1. el watermark de sus fuentes;
2. el trabajo propio pendiente más viejo.

Para una etapa `B`:

```text
W(B) = min(W(fuentes_de_B), timestamp_del_trabajo_más_viejo_en_B)
```

Si una etapa tiene records pendientes viejos, su watermark no puede avanzar aunque la fuente ya haya avanzado.

Esto también sirve como métrica de performance. Si el watermark de una etapa se queda muy atrás respecto de la etapa anterior, esa etapa está procesando lento o está saturada.

## Watermark con particiones

Si un cómputo está particionado, cada partición puede tener su propio watermark.

```text
A partición [A-M): watermark 100
A partición [M-Z): watermark 200
```

El watermark del cómputo completo es el mínimo:

```text
W(A) = min(100, 200) = 100
```

La partición más atrasada determina el avance global.

## Autoridad de watermarks

MillWheel usa una autoridad de watermarks.

Cada nodo informa su watermark a esta autoridad. La autoridad mantiene información sobre particiones, rangos y cómputos, y calcula el watermark global correspondiente.

Esto simplifica la propagación cuando los rangos cambian dinámicamente, se dividen, se reasignan o aparecen nodos zombies.

```text
Nodo A1 ── watermark ──► Autoridad
Nodo A2 ── watermark ──► Autoridad
                          │
                          ▼
                      W(A) global
```

La autoridad permite que los consumidores consulten un watermark consistente para el cómputo fuente.

## Watermarks como métrica del pipeline

Además de cerrar ventanas, los watermarks permiten medir retrasos.

En un pipeline:

```text
A ──► B ──► C
```

puede pasar que:

```text
W(A) = 8:30
W(B) = 8:20
W(C) = 8:10
```

Eso muestra que cada etapa introduce delay. Si una etapa queda demasiado atrás, indica que está saturada o que su procesamiento es lento.

El watermark mide cuánto se aleja cada etapa del tiempo real y ayuda a detectar cuellos de botella.

## Estado actual

MillWheel evolucionó hacia sistemas más modernos de procesamiento de streams.

En Google Cloud, la idea aparece como **Dataflow**. El modelo de programación de más alto nivel se relaciona con **Apache Beam**.

Apache Beam permite expresar pipelines de streaming con abstracciones más cómodas que `process_record` y `process_timer`. Luego esos pipelines pueden ejecutarse sobre distintos runners, como Dataflow, Flink o Spark.

```text
MillWheel ──► Windmill ──► Dataflow
                         └──► Apache Beam como modelo de programación
```

La idea central se mantiene: procesar streams distribuidos, con estado persistente, timers, watermarks, tolerancia a fallas y semántica effectively-once.

## Resumen

El procesamiento de streams extiende el procesamiento distribuido hacia datos continuos y en tiempo real.

Los problemas simples procesan evento por evento: filtros, transformaciones y joins con tablas. Los problemas difíciles agrupan por tiempo, combinan streams o necesitan cerrar ventanas con eventos desordenados.

La distinción entre **event time** y **processing time** es central. Los agregados deben hacerse por event time, pero el sistema recibe los eventos en processing time.

Los **watermarks** resuelven cuándo cerrar ventanas: indican hasta qué timestamp se considera que el stream está completo.

MillWheel propone un modelo basado en un DAG de cómputos, records con clave y timestamp, hooks `process_record` y `process_timer`, timers, estado persistente y producción de nuevos records.

La escalabilidad se logra particionando por clave y reasignando rangos dinámicamente. La tolerancia a fallas se apoya en estado persistente, epochs y recuperación de asignaciones.

La semántica **effectively-once** se logra combinando at-least-once, IDs de idempotencia y updates atómicos del estado, timers, producciones pendientes y marcas de procesamiento.

Los watermarks no solo cierran ventanas: también permiten observar el retraso de cada etapa y detectar cuellos de botella en el pipeline.
