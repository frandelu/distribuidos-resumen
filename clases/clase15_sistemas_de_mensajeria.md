# Clase 15 — Sistemas de mensajería

## [Apuntes de Emma](https://drive.google.com/file/d/1PZ0Vv7UYi1SIUk_9dBGYNLgDjfCPGH0K/view)

## Sistemas de mensajería

Un sistema de mensajería es una infraestructura intermedia que permite que distintos componentes de un sistema distribuido se comuniquen de forma indirecta.

La idea central es evitar que un servicio tenga que llamar directamente a todos los servicios interesados mediante RPC. En vez de eso, un productor publica un mensaje en un sistema intermedio y ese sistema se encarga de almacenarlo, repartirlo, entregarlo o permitir que otros lo consuman.

Históricamente a estos sistemas se los llamó **Message Oriented Middleware** o **MOM**. El nombre quedó bastante asociado a tecnologías de los años noventa, pero el concepto sigue siendo central en sistemas distribuidos modernos. Hoy aparece bajo nombres como colas de mensajes, brokers, sistemas pub/sub, event streaming o logs distribuidos.

## Comunicación directa e indirecta

En una comunicación directa, un componente conoce al otro y lo invoca explícitamente.

Por ejemplo, un banco que necesita información del NASDAQ podría llamar por RPC a una API del NASDAQ cada cierto tiempo para consultar el precio de acciones. Ese esquema funciona, pero tiene problemas:

- hay muchos símbolos financieros para consultar;
- hay decenas de millones de transacciones por día;
- cada banco debería hacer polling constantemente;
- el servidor central recibe muchas consultas repetidas;
- la información se actualiza todo el tiempo.

La alternativa es invertir el flujo. En vez de que cada banco pregunte periódicamente, el NASDAQ publica actualizaciones en un sistema intermedio. Los bancos se suscriben a los temas que les interesan y reciben los mensajes cuando aparecen.

```text
NASDAQ ── publish ──► Sistema de mensajería ──► Banco 1
                                          ├──► Banco 2
                                          └──► Banco 3
```

El sistema de mensajería desacopla al productor de los consumidores.

## Desacople

Los sistemas de mensajería permiten desacoplar servicios en tres dimensiones.

### Desacople espacial

El productor no necesita conocer a todos los consumidores.

Un servicio publica un mensaje en un tópico, una cola o un log. Los consumidores interesados se registran en el sistema de mensajería. El productor no tiene que saber cuántos son, dónde están, qué equipo los mantiene ni qué lógica ejecutan.

Esto también desacopla equipos de desarrollo. Un equipo puede publicar eventos de negocio sin coordinar directamente con todos los equipos que puedan consumirlos.

### Desacople temporal

El productor y el consumidor no necesitan estar activos al mismo tiempo.

Con RPC, si el servicio destino está caído, la llamada falla. Con un sistema de mensajería persistente, el mensaje puede quedar guardado hasta que el consumidor vuelva a estar disponible.

Esto introduce una diferencia importante: los errores también aparecen desacoplados en el tiempo. El productor puede publicar correctamente un mensaje, pero el consumidor puede fallar mucho después al procesarlo.

### Desacople de sincronización

La comunicación ya no necesariamente es sincrónica.

Con RPC, el emisor suele quedar esperando una respuesta. Con mensajería, el productor puede publicar un mensaje y continuar. El procesamiento ocurre asincrónicamente.

Esto permite construir pipelines, workflows, procesamiento offline y sistemas con backpressure.

## Modelos básicos de integración

Hay tres modelos centrales:

| Modelo | Idea | Ejemplo típico |
|---|---|---|
| Producer-consumer / point-to-point | Un productor envía mensajes a una cola. Varios consumidores compiten por procesarlos. Cada mensaje lo procesa un consumidor. | Amazon SQS |
| Publisher-subscriber | Un productor publica un mensaje y el sistema lo distribuye a todos los suscriptores. | Amazon SNS |
| Event streaming / log particionado | Los mensajes se agregan a un log persistente. Muchos consumidores pueden leerlos desde distintas posiciones. | Kafka |

## Producer-consumer

En el modelo **producer-consumer**, un productor envía mensajes a una cola y uno o más consumidores los toman para procesarlos.

```text
Producer ── send ──► Queue ── receive ──► Consumer
```

Si hay varios consumidores, todos compiten por los mensajes.

```text
Producer ──► Queue ──► Consumer 1
                 ├──► Consumer 2
                 └──► Consumer 3
```

La característica clave es que **cuando un mensaje es consumido correctamente, deja de estar disponible para los demás consumidores**.

Esto lo vuelve útil para distribuir trabajo. La cola funciona como un buffer de tareas pendientes y los consumidores como workers que compiten por tomarlas.

### Orden de los mensajes

Aunque se dibuje como una cola FIFO, no todos los sistemas garantizan FIFO estricto.

Algunos sistemas tienen colas FIFO explícitas. Otros ofrecen una especie de “bolsa de mensajes” donde los mensajes salen aproximadamente en orden, pero sin garantía fuerte.

En muchos casos de uso esto alcanza, porque lo importante es procesar trabajo pendiente, no preservar un orden total.

### Push, pull, polling y bloqueo

El productor suele hacer **push**: envía el mensaje a la cola mediante una llamada al broker.

Del lado del consumidor puede haber distintas estrategias:

- **pull / receive**: el consumidor pregunta si hay mensajes;
- **polling no bloqueante**: si no hay mensajes, recibe una respuesta vacía y vuelve a preguntar más tarde;
- **long polling / blocking receive**: la llamada queda bloqueada hasta que haya mensajes o expire un timeout;
- **push desde el broker**: el broker llama al consumidor cuando hay mensajes;
- **notify**: el broker notifica que hay mensajes y luego el consumidor los busca.

Amazon SQS se entiende bien como un sistema de `send`, `receive` y `delete`.

## SQS: ciclo de vida de un mensaje

En Amazon SQS hay tres operaciones fundamentales:

```text
SendMessage
ReceiveMessage
DeleteMessage
```

Un productor manda un mensaje a una cola:

```text
SendMessage(queue_url, body)
```

Un consumidor pide mensajes:

```text
ReceiveMessage(queue_url)
```

Cuando termina de procesar uno, lo borra:

```text
DeleteMessage(queue_url, receipt_handle)
```

El `receipt_handle` no es simplemente el ID del mensaje. Es un identificador opaco que el sistema devuelve cuando se recibe un mensaje y que luego debe pasarse para borrarlo. Internamente puede incluir metadata necesaria para encontrar la copia concreta del mensaje dentro del sistema distribuido.

### Visibility timeout

Cuando un consumidor recibe un mensaje, SQS **no lo borra inmediatamente**.

El mensaje queda invisible para otros consumidores durante un tiempo llamado **visibility timeout**.

```text
M1 visible ── receive ──► M1 invisible por 60s
```

Si el consumidor procesa correctamente el mensaje, llama a `DeleteMessage` y el mensaje desaparece.

Si el consumidor falla antes de borrar el mensaje, el timeout expira y el mensaje vuelve a estar disponible para otros consumidores.

```text
receive(M1)
procesar(M1)
delete(M1)
```

Este mecanismo permite tolerar fallas simples de consumidores.

### Semántica at-least-once

SQS y sistemas similares suelen dar una garantía **at-least-once**: el mensaje se entrega al menos una vez.

Eso significa que pueden aparecer duplicados.

Un mensaje puede ser procesado dos veces si:

- el consumidor tarda más que el visibility timeout;
- el consumidor falla después de producir efectos laterales pero antes de borrar el mensaje;
- hay fallas internas o condiciones de carrera;
- el broker replica mensajes y alguna copia vuelve a hacerse visible.

Por eso, los consumidores deben ser **idempotentes**. Procesar dos veces el mismo mensaje no debería romper el sistema ni duplicar efectos de negocio.

### Poison pills y dead-letter queues

Un **poison pill** es un mensaje que siempre hace fallar al consumidor.

Por ejemplo, un mensaje con un formato inesperado puede causar que cada worker que lo tome falle. Si el mensaje vuelve a la cola una y otra vez, puede consumir recursos indefinidamente.

Para evitarlo, los sistemas suelen contar intentos de entrega. Si un mensaje supera cierta cantidad de intentos, se lo envía a una **dead-letter queue** o se lo descarta según configuración.

```text
Queue principal ── fallas repetidas ──► Dead-letter queue
```

La dead-letter queue permite inspeccionar manualmente mensajes problemáticos y corregir bugs.

## Ejemplo: SQS como balanceador de carga

Un caso típico es el procesamiento asincrónico de videos.

El usuario sube un video. El frontend guarda el archivo en un storage de objetos, como S3, y luego publica un mensaje en SQS indicando que ese video debe ser procesado.

```text
Usuario ── upload ──► Frontend
                       ├── guarda video ──► S3
                       └── envía job ─────► SQS
                                              ├── Worker 1
                                              ├── Worker 2
                                              └── Worker 3
```

El mensaje no contiene el video entero. Contiene metadata, por ejemplo:

```json
{
  "task": "transcode",
  "path": "s3://bucket/videos/123.mp4"
}
```

Los workers consumen mensajes de la cola y ejecutan la transcodificación, por ejemplo usando FFmpeg. Cuando terminan, guardan el resultado en S3 y borran el mensaje de la cola.

La cola permite escalar el sistema de forma elástica. Si hay muchos mensajes pendientes, se agregan más workers. Si hay pocos, se reduce la cantidad de workers.

La cola funciona como:

- buffer de trabajo pendiente;
- mecanismo de balanceo de carga;
- desacople temporal entre el frontend y el procesamiento pesado;
- mecanismo simple de tolerancia a fallas mediante visibility timeout.

## Message brokers

Un **message broker** es el sistema intermedio que recibe, almacena y entrega mensajes.

Ejemplos:

- Amazon SQS;
- Amazon SNS;
- Google Pub/Sub;
- RabbitMQ;
- ActiveMQ;
- Kafka.

Conceptualmente, un broker es un sistema de storage especializado. Guarda mensajes, pero no como una base de datos tradicional.

En colas y pub/sub clásicos, los mensajes suelen ser efímeros: se guardan por poco tiempo y desaparecen al ser entregados o confirmados. No suelen tener índices generales ni consultas complejas. Están optimizados para mover mensajes, no para hacer queries arbitrarias.

## Publisher-subscriber

En el modelo **publisher-subscriber**, un productor publica un mensaje en un tópico y el sistema lo distribuye a todos los suscriptores.

```text
Publisher ── publish ──► Topic
                           ├──► Subscriber 1
                           ├──► Subscriber 2
                           └──► Subscriber 3
```

A diferencia de producer-consumer, acá los consumidores no compiten. Todos reciben una copia del mensaje.

Esto sirve cuando los consumidores son heterogéneos, es decir, cuando cada uno tiene una lógica de negocio diferente.

Por ejemplo, ante un evento `UsuarioEliminado`:

- un servicio borra datos de perfil;
- otro invalida sesiones;
- otro actualiza métricas;
- otro envía un email;
- otro limpia permisos.

Todos necesitan enterarse del evento.

### SNS

Amazon SNS es un servicio pub/sub.

Tiene operaciones como:

```text
Subscribe(topic, protocol, endpoint)
Publish(topic, message)
```

El productor publica en un tópico. Los suscriptores reciben el mensaje según el protocolo configurado.

SNS puede usarse en dos escenarios:

### App-to-app

Una aplicación le publica eventos a otras aplicaciones.

Por ejemplo:

```text
Servicio de usuarios ── UsuarioEliminado ──► SNS
                                            ├──► Servicio de sesiones
                                            ├──► Servicio de facturación
                                            └──► Servicio de auditoría
```

### App-to-person

SNS también permite entregar mensajes a personas mediante:

- email;
- SMS;
- push notifications.

### Push y retries

SNS hace push a los suscriptores. Si un endpoint no está disponible, puede reintentar algunas veces y luego descartar el mensaje o enviarlo a algún mecanismo alternativo, según configuración.

Este modelo tiene un problema natural: el suscriptor tiene que estar disponible cuando SNS intenta entregarle el mensaje.

Por eso suele usarse un patrón más robusto.

## Fan-out + buffer

Un patrón común es combinar SNS con SQS.

```text
Publisher ──► SNS
              ├──► SQS A ──► Consumer A
              ├──► SQS B ──► Consumer B
              └──► SQS C ──► Consumer C
```

SNS hace el fan-out y cada suscriptor recibe su propia cola SQS.

Esto tiene varias ventajas:

- cada consumidor procesa a su ritmo;
- si un consumidor cae, sus mensajes quedan acumulados en su cola;
- se evita perder mensajes por caídas temporales;
- cada consumidor puede escalar sus propios workers;
- se desacopla el fan-out de la disponibilidad de cada servicio.

Google Pub/Sub integra más directamente este patrón: la suscripción suele tener internamente una cola/buffer desde donde los consumidores leen.

## Implementación posible de SNS usando SQS

Un sistema pub/sub puede implementarse combinando una cola interna, workers y una base de datos de suscripciones.

```text
Publish ──► Frontend ──► SQS interna ──► Notifier workers
                                      └── consulta DB de suscripciones
                                                   ├──► Subscriber 1
                                                   ├──► Subscriber 2
                                                   └──► Subscriber 3
```

El flujo sería:

1. El frontend recibe `Publish(topic, message)`.
2. Guarda un mensaje interno con el tópico y el payload.
3. Un notifier worker consume ese mensaje.
4. Consulta la base de datos de suscripciones para ese tópico.
5. Envía el mensaje a todos los endpoints registrados.

Si un suscriptor está caído, el worker puede reintentar, enviar el trabajo a una cola de retries o descartarlo luego de cierto límite.

Una optimización posible es separar una cola rápida de una cola lenta:

```text
SQS principal ──► Notifier workers ──► suscriptores rápidos
       │
       └──► Cola de retries / lentos ──► workers especializados
```

Así un suscriptor problemático no bloquea el procesamiento normal de los demás.

## Implementar una cola con Postgres

Una cola producer-consumer puede implementarse de forma simple usando una tabla en Postgres.

La tabla puede tener, por ejemplo:

```sql
CREATE TABLE queue (
  id BIGSERIAL PRIMARY KEY,
  message JSONB NOT NULL,
  visibility_timeout TIMESTAMP NOT NULL
);
```

Enviar un mensaje es insertar una fila:

```sql
INSERT INTO queue (message, visibility_timeout)
VALUES (:message, now());
```

Recibir un mensaje implica buscar uno visible y actualizar su visibility timeout:

```sql
WITH picked AS (
  SELECT id
  FROM queue
  WHERE visibility_timeout < now()
  ORDER BY id
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE queue q
SET visibility_timeout = now() + interval '60 seconds'
FROM picked
WHERE q.id = picked.id
RETURNING q.id, q.message;
```

Borrar el mensaje es eliminar la fila:

```sql
DELETE FROM queue
WHERE id = :id;
```

Este diseño tiene problemas claros:

- no escala como un sistema dedicado;
- depende de una única base si no se replica;
- puede generar contención;
- no está pensado para volúmenes enormes.

Pero es útil en un caso muy común: evitar una mini transacción distribuida.

## Outbox pattern

Un problema clásico aparece cuando un servicio necesita escribir en su base de datos y además publicar un mensaje.

```text
Frontend ──► DB
        └──► SQS
```

Si escribe en la DB y luego falla el envío a SQS, el mensaje no sale. Si publica en SQS y luego falla la DB, otros servicios pueden recibir un evento sobre algo que no existe.

Una solución práctica es guardar el mensaje en la misma base de datos, dentro de la misma transacción que actualiza el estado de negocio.

```sql
BEGIN;

INSERT INTO orders (...);
INSERT INTO outbox (event_type, payload, status)
VALUES ('OrderCreated', '{...}', 'pending');

COMMIT;
```

Luego un worker lee la tabla `outbox` y publica esos eventos a SQS, SNS o Kafka.

```text
Frontend ── tx ──► DB + Outbox
                    │
                    └──► Worker ──► Broker externo
```

Así se evita una transacción distribuida entre la base de datos y el sistema de mensajería. La atomicidad queda delegada a la base de datos local.

## Cómo escalar un broker producer-consumer

### Opción 1: broker único con storage único

```text
Broker ──► Postgres
```

Es simple, pero no escala y no tolera fallas.

### Opción 2: sharding

Se pueden tener varios brokers o shards.

```text
Frontend
   ├──► Shard 1
   ├──► Shard 2
   └──► Shard 3
```

Para enviar un mensaje, el frontend elige un shard al azar.

Para recibir, puede elegir uno al azar o intentar en varios.

El problema aparece al borrar: hay que saber en qué shard quedó el mensaje. Por eso el `receive` puede devolver metadata opaca, como un `receipt_handle`, que luego se pasa al `delete`.

```text
ReceiveMessage ──► message + receipt_handle
DeleteMessage(receipt_handle)
```

Ese handle puede contener información interna del shard, encriptada u opaca para el usuario.

### Opción 3: sharding + replicación

Para tolerar fallas, el mensaje se escribe en más de un nodo.

```text
Mensaje M
   ├──► Nodo A
   ├──► Nodo B
   └──► Nodo C
```

Una implementación simple puede elegir:

- un primary;
- un secondary;
- un tertiary.

Solo el primary responde lecturas. Si el primary cae, el secondary puede empezar a responder.

```text
Primary ── replica ──► Secondary
       └─ replica ──► Tertiary
```

Este diseño mejora la disponibilidad, pero puede producir duplicados. Por eso vuelve a aparecer la semántica **at-least-once** y la necesidad de consumidores idempotentes.

## Event streaming

El tercer modelo es **event streaming**, representado por Kafka.

La diferencia central con SQS/SNS es que los mensajes no desaparecen al ser consumidos. Se agregan a un log persistente.

```text
Topic log:
[ e1 ][ e2 ][ e3 ][ e4 ][ e5 ][ e6 ]
              ▲
              c A
                          ▲
                          c B
```

Cada consumidor mantiene su posición de lectura. Puede leer desde el final, desde el principio o desde una posición histórica.

Esto habilita:

- replay de eventos;
- reconstrucción de estados;
- auditoría;
- fan-out natural;
- pipelines de procesamiento;
- integración entre muchos servicios;
- event sourcing;
- análisis de streams de comportamiento de usuarios.

## Kafka: tópicos, particiones y orden

Kafka organiza los mensajes en **topics**.

Un topic se divide en **particiones**. Cada partición es un log ordenado.

Cuando un productor publica, envía:

```text
send(topic, key, payload)
```

La key se usa para elegir partición:

```text
partition = hash(key) % N
```

Donde `N` es la cantidad de particiones del topic.

Esto garantiza orden **por partición**, no orden global.

Si se usa `user_id` como key, todos los eventos de un mismo usuario caen en la misma partición y se preserva el orden de esos eventos.

```text
Topic: user-events

Partition 0: user 1 event A, user 1 event B, ...
Partition 1: user 2 event A, user 2 event B, ...
Partition 2: user 3 event A, user 3 event B, ...
```

Entre particiones no hay un orden total confiable.

## Consumir en Kafka

Los consumidores se organizan en **consumer groups**.

Dentro de un consumer group, cada partición se asigna a un único consumidor.

```text
Partition 0 ──► Consumer A
Partition 1 ──► Consumer A
Partition 2 ──► Consumer B
```

No debería haber dos consumidores del mismo grupo leyendo la misma partición al mismo tiempo, porque procesarían los mismos mensajes duplicados.

Si hay más consumidores que particiones, algunos consumidores quedan sin trabajo.

```text
3 particiones, 5 consumidores
=> 2 consumidores quedan ociosos
```

El paralelismo máximo de un consumer group está limitado por la cantidad de particiones.

### Coordinador y rebalanceo

Hace falta un coordinador para decidir qué consumidor lee qué particiones.

Ese coordinador se encarga de:

- asignar particiones;
- detectar consumidores caídos;
- liberar particiones;
- reasignarlas a otros consumidores;
- rebalancear el grupo.

En versiones antiguas de Kafka, esta coordinación usaba ZooKeeper. En versiones modernas, la coordinación está integrada más directamente en Kafka.

## Kafka y replay

Como los mensajes quedan en el log, un consumidor puede volver atrás y reprocesar.

Esto sirve para reconstruir una base derivada.

```text
Log de eventos ── replay ──► reconstrucción de vista materializada
```

Por ejemplo, si una base de analytics se corrompe, se puede volver a leer el topic desde el principio o desde cierto offset y reconstruir el estado.

También permite agregar nuevos consumidores en el futuro. Un nuevo servicio puede empezar a leer eventos históricos que fueron publicados antes de que el servicio existiera.

## Kafka como alternativa a fan-out + buffer

Con SNS + SQS, para que varios servicios consuman el mismo evento se suele crear una cola por consumidor.

```text
SNS
 ├──► SQS Servicio A
 ├──► SQS Servicio B
 └──► SQS Servicio C
```

Con Kafka, el productor publica en un topic y cada consumidor lee del log.

```text
Producer ──► Topic Kafka
              ├──► Consumer group A
              ├──► Consumer group B
              └──► Consumer group C
```

Cada consumer group tiene sus propios offsets. Consumir desde un grupo no borra mensajes ni afecta a los demás.

## Replicación en Kafka

Kafka replica particiones para tolerar fallas.

Una partición tiene un líder o primary y réplicas followers/backups.

```text
Partition leader ──► Replica 1
                 └──► Replica 2
```

El productor escribe en el líder. El líder replica a los backups. Cuando las réplicas necesarias confirman, el mensaje puede considerarse comprometido y visible.

Este modelo se parece a primary-backup y comparte algunas ideas con logs replicados, aunque los detalles de Kafka son específicos.

## Comparación rápida

| Necesidad | Modelo conveniente |
|---|---|
| Distribuir tareas entre workers homogéneos | SQS / producer-consumer |
| Avisar el mismo evento a varios sistemas distintos | SNS / pub-sub |
| Fan-out robusto con buffering por consumidor | SNS + SQS |
| Guardar historial de eventos y permitir replay | Kafka |
| Orden por entidad de negocio | Kafka con key adecuada |
| Atomicidad entre DB local y publicación de evento | Outbox pattern |
| Procesamiento offline escalable | SQS + workers |
| Analytics, tracking, engagement, event sourcing | Kafka |

## Ideas importantes

Los sistemas de mensajería son una forma de comunicación indirecta. Permiten desacoplar servicios en espacio, tiempo y sincronización.

Las colas producer-consumer sirven para distribuir trabajo. Un mensaje lo procesa un consumidor. SQS es un ejemplo típico.

El visibility timeout permite tolerar fallas de consumidores sin borrar mensajes inmediatamente. Como consecuencia, puede haber duplicados y los consumidores deben ser idempotentes.

SNS implementa pub/sub. Un mensaje se distribuye a todos los suscriptores. Es útil cuando varios sistemas heterogéneos necesitan enterarse del mismo evento.

El patrón fan-out + buffer combina SNS y SQS para desacoplar la entrega del procesamiento de cada consumidor.

Postgres puede usarse como cola simple y permite resolver problemas de atomicidad local mediante una tabla outbox.

Kafka modela los mensajes como logs persistentes particionados. Los mensajes no se borran al ser consumidos. Esto habilita replay, fan-out natural, consumer groups y procesamiento de streams.

Kafka garantiza orden por partición, no orden global. La key de particionamiento debe elegirse según la entidad de negocio cuyo orden se quiere preservar.

Los brokers son sistemas de storage especializados. Guardan mensajes, pero con semánticas muy distintas a las de una base de datos relacional tradicional.
