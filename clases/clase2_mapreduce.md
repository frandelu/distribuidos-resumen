# Clase 2 — MapReduce

## [Apuntes de Emma](https://drive.google.com/file/d/1kLjeiW77cAGTCqIfxOtwWB5KTHvVQu68/view)

## Contexto histórico de los sistemas distribuidos

El estudio de los sistemas distribuidos surge de una combinación de problemas teóricos y problemas prácticos de ingeniería. A lo largo del tiempo se pueden distinguir varias etapas, cada una asociada a una forma distinta de entender qué significa distribuir un sistema.

### Fundamentos teóricos: años 70 y 80

En los primeros trabajos sobre sistemas distribuidos aparecen problemas fundamentales vinculados con la coordinación entre procesos. En un sistema distribuido no existe una memoria compartida global ni un reloj global perfectamente sincronizado. Cada nodo tiene su propia memoria, su propio reloj y se comunica con otros nodos mediante mensajes.

Esta separación genera problemas que no aparecen de la misma manera en una computadora con memoria compartida. Coordinar procesos distribuidos implica razonar sobre qué ocurrió antes, qué ocurrió después y cómo ponerse de acuerdo cuando no hay una fuente única de verdad.

Leslie Lamport es una figura central de esta etapa. Sus trabajos ayudan a formalizar ideas como:

- relojes de Lamport;
- relojes vectoriales;
- causalidad entre eventos;
- consenso distribuido;
- máquinas de estados replicadas.

La idea de causalidad es clave: en muchos sistemas no interesa tanto conocer el tiempo absoluto exacto en que ocurrió un evento, sino poder determinar si un evento ocurrió antes que otro o si ambos son concurrentes.

### Máquina de estados replicada

Una máquina de estados replicada es una forma de lograr tolerancia a fallas mediante réplicas. Si dos réplicas parten del mismo estado inicial y reciben exactamente las mismas operaciones en el mismo orden, terminan en el mismo estado final.

Por ejemplo, una cuenta bancaria puede pensarse como una máquina de estados. Si se parte de un saldo inicial y se aplican todas las transacciones del mes en el mismo orden, se llega al saldo actual. Si dos réplicas aplican las mismas transacciones en el mismo orden, ambas deberían llegar al mismo resultado.

El problema se traslada entonces a cómo lograr que todas las réplicas acuerden el mismo orden de operaciones. De ahí aparece el problema de consenso: varias máquinas tienen que ponerse de acuerdo sobre un valor o sobre la próxima operación a ejecutar. Paxos fue una de las primeras soluciones importantes a este problema, aunque es conocido por ser difícil de entender e implementar. Más adelante aparecen algoritmos como Raft, diseñados para resolver un problema similar pero con una presentación más comprensible.

## Intentos de transparencia: años 80 y 90

En una segunda etapa se intentó construir sistemas distribuidos que ocultaran casi por completo la existencia de la red. La idea era que usar un recurso remoto se sintiera igual que usar un recurso local.

Algunos ejemplos de esta etapa son:

- **NFS**, como sistema de archivos remoto;
- **RPC**, para llamadas a procedimientos remotos;
- sistemas operativos distribuidos como **Amoeba** y **Plan 9**.

El objetivo era lograr transparencia en la distribución. Sin embargo, esa transparencia tenía límites. La red introduce latencia, fallas parciales, pérdida de mensajes, timeouts y problemas de concurrencia que no pueden tratarse como si fueran detalles invisibles.

Un sistema como NFS permite montar un file system remoto como si fuera local. Mientras todo funciona bien, la abstracción es cómoda. Pero si la red se corta, el servidor remoto no responde o la latencia aumenta, una aplicación que esperaba el comportamiento de un disco local puede bloquearse o fallar de formas inesperadas.

La conclusión es que no conviene esconder totalmente la red. La abstracción es útil, pero el programador y el diseño del sistema tienen que reconocer que hay comunicación remota y que esa comunicación puede fallar.

## Middleware

En los años 90 toma fuerza la idea de **middleware**: una capa intermedia entre la aplicación y la red que resuelve problemas comunes de comunicación distribuida.

Un middleware puede encargarse de:

- comunicación entre sistemas;
- descubrimiento de servicios;
- transmisión de datos;
- manejo de errores;
- serialización y deserialización;
- persistencia de mensajes;
- abstracción de protocolos.

### CORBA

CORBA aparece como una arquitectura para objetos distribuidos. Buscaba permitir que objetos en distintos procesos o máquinas se invocaran entre sí usando una abstracción común. Fue importante históricamente, aunque hoy no es una tecnología habitual en sistemas modernos.

### Message-Oriented Middleware

El **Message-Oriented Middleware** propone una forma de comunicación asincrónica basada en mensajes. A diferencia de RPC, donde un cliente llama a un servidor y espera una respuesta, en la comunicación por mensajes el emisor puede enviar un mensaje y continuar.

La idea central es:

> mando un mensaje y me olvido.

En una versión robusta, el middleware persiste el mensaje y se ocupa de entregarlo eventualmente al destinatario. Esto permite desacoplar al emisor y al receptor: el receptor puede estar temporalmente caído y el mensaje puede quedar almacenado hasta que pueda entregarse.

Este modelo sigue existiendo en sistemas actuales, aunque muchas veces se habla directamente de colas de mensajes, sistemas de streaming o brokers. Ejemplos modernos son SQS, Pub/Sub, RabbitMQ y Kafka.

## La era web y el problema de escala

A partir de los años 2000, con la expansión de la web y el crecimiento de los datos, aparecen nuevos problemas. Ya no alcanza con comunicar sistemas corporativos entre sí. Ahora hay que procesar volúmenes enormes de información generada por usuarios, crawlers, logs, búsquedas y páginas web.

En esta etapa aparecen varios sistemas fundamentales:

- **Google File System (GFS)**, publicado en 2003;
- **MapReduce**, publicado en 2004;
- **Bigtable**, publicado en 2006;
- **Dynamo**, publicado en 2007.

Estos sistemas surgen de problemas reales de empresas como Google y Amazon. La pregunta central pasa a ser cómo procesar, almacenar y consultar cantidades masivas de datos usando clusters de máquinas comunes, tolerando fallas y manteniendo buena performance.

## Motivación de MapReduce

MapReduce nace en Google para resolver el problema de ejecutar grandes pipelines de procesamiento de datos sobre clusters de muchas máquinas.

Google necesitaba procesar enormes cantidades de datos, por ejemplo para reescribir su pipeline de indexación de la web. Muchas etapas de ese pipeline tenían una estructura similar:

1. transformar registros de entrada en pares intermedios;
2. agrupar esos pares por clave;
3. combinar los valores asociados a cada clave.

La parte difícil no era necesariamente escribir la lógica de negocio de cada transformación, sino distribuirla correctamente: partir los datos, asignar trabajo a muchas máquinas, mover datos entre nodos, tolerar fallas, balancear carga y recolectar resultados.

MapReduce propone una abstracción simple: el programador escribe dos funciones, `map` y `reduce`, y el framework se ocupa de paralelizar, distribuir y tolerar fallas.

## Modelo de programación

Un cómputo MapReduce toma un conjunto de pares clave/valor de entrada y produce otro conjunto de pares clave/valor de salida.

El programador define dos funciones:

```text
map(k1, v1) -> list(k2, v2)

reduce(k2, list(v2)) -> list(v2)
```

La función `map` recibe un par de entrada y emite cero, uno o muchos pares intermedios. La función `reduce` recibe una clave intermedia y todos los valores asociados a esa clave, y produce el resultado final para esa clave.

La clave intermedia `k2` es fundamental porque determina cómo se agrupan los datos.

## Ejemplo: contar palabras

El ejemplo clásico es contar cuántas veces aparece cada palabra en un conjunto grande de documentos.

La función `map` procesa cada documento y emite un par por cada palabra:

```text
map(document_name, document_content):
    for each word w in document_content:
        emit(w, 1)
```

Si el documento contiene:

```text
hola mundo hola
```

el mapper emite:

```text
(hola, 1)
(mundo, 1)
(hola, 1)
```

Luego el sistema agrupa por clave intermedia:

```text
hola  -> [1, 1]
mundo -> [1]
```

La función `reduce` suma los valores:

```text
reduce(word, values):
    total = 0
    for v in values:
        total += v
    emit(word, total)
```

El resultado sería:

```text
(hola, 2)
(mundo, 1)
```

Lo importante es que el programador no escribe el código que distribuye los documentos, junta todas las apariciones de una palabra, mueve datos por la red o reintenta tareas fallidas. Esa lógica queda en el framework.

## Splits de entrada

Los datos de entrada pueden estar formados por muchos archivos o por archivos enormes. Para paralelizar el procesamiento, el sistema divide la entrada en **splits**.

Un split es una porción lógica del input que puede procesarse de manera independiente. Cada split se asigna a una tarea `map`.

Si hay `M` splits, hay `M` tareas map lógicas. Esto no significa que haya `M` máquinas físicas. Puede haber muchos más mappers lógicos que nodos disponibles. El sistema va asignando tareas a medida que los workers quedan libres.

## Mappers, reducers y nodos físicos

Conviene distinguir tres cantidades:

- `M`: cantidad de tareas map;
- `R`: cantidad de tareas reduce;
- `N`: cantidad de nodos físicos o workers disponibles.

Normalmente `M` puede ser mucho mayor que `N`. Esto permite mejor balanceo dinámico de carga: cuando un worker termina una tarea, puede recibir otra.

Los workers no son inherentemente mappers o reducers. Son nodos genéricos que primero pueden ejecutar tareas map y luego tareas reduce. El mismo cluster se reutiliza para ambas fases.

## Fase Map

Cada mapper:

1. recibe un split de entrada;
2. ejecuta la función `map` definida por el usuario;
3. genera pares intermedios `(k2, v2)`;
4. particiona esos pares según la cantidad de reducers;
5. escribe archivos intermedios en disco local.

Si hay `R` reducers, cada mapper genera datos para cada una de esas `R` particiones. Por eso, un mapper puede producir archivos intermedios del estilo:

```text
M1_R1
M1_R2
...
M1_RR
```

El archivo `M1_R2` contiene los datos producidos por el mapper 1 que deberán ser procesados por el reducer 2.

## Shuffle

El **shuffle** es la fase que conecta mappers con reducers. Su objetivo es garantizar que todos los valores asociados a una misma clave intermedia lleguen al mismo reducer.

Para decidir a qué reducer va cada clave, se usa una función de particionado. La forma típica es:

```text
hash(key) mod R
```

Donde `R` es la cantidad de reducers.

La función debe ser determinista. Si la clave `"hola"` aparece en varios mappers, todos tienen que enviarla al mismo reducer. Si distintos mappers tomaran decisiones distintas para la misma clave, el reducer no podría reunir todos los valores necesarios.

El shuffle genera un patrón de comunicación muchos-a-muchos: cada mapper puede enviar datos a cada reducer.

## Fase Reduce

Cada reducer recibe los archivos intermedios correspondientes a su partición. Por ejemplo, el reducer 1 recibe:

```text
M1_R1
M2_R1
M3_R1
...
MM_R1
```

Luego:

1. lee los datos intermedios;
2. ordena por clave;
3. agrupa todas las ocurrencias de la misma clave;
4. invoca la función `reduce` para cada clave;
5. escribe el resultado final.

El sort es importante porque permite que todos los valores de una clave queden juntos. Así el reducer puede recorrer los datos ordenados e ir llamando a `reduce` con una clave y su colección de valores.

Si los mappers ya generan salidas parcialmente ordenadas, el reducer puede hacer un merge más eficiente en lugar de ordenar todo desde cero.

## Archivos intermedios

Si hay `M` tareas map y `R` tareas reduce, se generan conceptualmente `M × R` archivos intermedios.

Esto aparece porque cada mapper produce una partición para cada reducer. Por ejemplo, con 3 mappers y 2 reducers:

```text
M1_R1  M1_R2
M2_R1  M2_R2
M3_R1  M3_R2
```

Los archivos intermedios suelen escribirse en discos locales de los workers que ejecutaron los mappers. Luego los reducers los leen remotamente.

## Arquitectura de ejecución

La arquitectura incluye un coordinador, llamado master en el paper, y muchos workers.

El coordinador se encarga de:

- dividir el input en splits;
- mantener el estado de las tareas;
- asignar tareas map y reduce a workers disponibles;
- recibir información sobre archivos intermedios;
- informar a los reducers dónde leer esos archivos;
- detectar workers fallidos;
- replanificar tareas;
- saber cuándo el job completo terminó.

Cada tarea puede estar en alguno de estos estados:

```text
idle
in-progress
completed
```

Cuando un worker queda disponible, pide trabajo. El coordinador le asigna una tarea map o reduce según la fase del job.

La ejecución tiene dos fases claramente separadas. Primero deben terminar todas las tareas map. Recién después pueden ejecutarse correctamente las tareas reduce, porque cada reducer necesita todos los datos intermedios asociados a su partición.

## Localidad de datos

Mover datos por red es caro. Por eso MapReduce se combina con un file system distribuido como GFS.

GFS divide los archivos en bloques grandes, típicamente de 64 MB, y replica esos bloques en varias máquinas. Cuando el coordinador asigna una tarea map, intenta asignarla a un worker que ya tenga localmente el bloque de datos que debe procesar. Si no puede, intenta asignarla cerca, por ejemplo en la misma red o rack.

La idea es llevar el cómputo hacia los datos, en vez de mover grandes volúmenes de datos hacia el cómputo.

Esto reduce el uso de red en la fase de lectura del input. La red sigue siendo importante durante el shuffle, porque ahí los datos intermedios sí deben moverse entre mappers y reducers.

## Tolerancia a fallas

MapReduce está pensado para ejecutarse sobre cientos o miles de máquinas comunes, donde las fallas son esperables. La estrategia principal de tolerancia a fallas es reejecutar tareas.

Si un worker falla o deja de responder, el coordinador marca sus tareas como pendientes y las reasigna a otros workers.

### Fallas en tareas map

Si una tarea map ya había terminado pero el worker que la ejecutó falla, esa tarea debe reejecutarse. La razón es que su salida intermedia estaba en el disco local de ese worker y puede haberse perdido o quedar inaccesible.

### Fallas en tareas reduce

Si una tarea reduce terminó correctamente, normalmente no hace falta reejecutarla, porque su salida se escribe en un file system global o distribuido. Si falla mientras estaba en progreso, se reasigna.

### Fallas del coordinador

El coordinador es un punto débil. El paper menciona que sería posible persistir checkpoints del estado del master, pero la implementación original abortaba el cómputo si el master fallaba. En ese caso, el cliente podía volver a ejecutar el job.

## Determinismo

Para que la reejecución funcione correctamente, las funciones `map` y `reduce` deben ser deterministas.

Si una tarea se ejecuta dos veces con el mismo input, debe producir el mismo output. Esto permite que el sistema pueda reintentar tareas sin cambiar el resultado final.

Una función map o reduce no debería depender de:

- números aleatorios no controlados;
- hora actual;
- estado externo mutable;
- entrada por consola;
- efectos colaterales no idempotentes;
- archivos externos que cambian durante la ejecución.

Si las funciones son deterministas, el resultado distribuido con fallas y reintentos puede razonarse como si hubiera sido una ejecución secuencial sin fallas.

## Commits atómicos

Las tareas en progreso escriben resultados en archivos temporales privados.

Cuando una tarea termina:

- un mapper informa al coordinador los archivos intermedios producidos;
- un reducer renombra atómicamente su archivo temporal al nombre final.

Si una misma tarea reduce se ejecuta dos veces, ambas pueden producir archivos temporales. Solo una versión queda como salida final mediante una operación atómica de rename. Si las funciones son deterministas, cualquiera de las dos versiones es equivalente.

## Stragglers y backup tasks

Un **straggler** es una tarea que tarda mucho más que las demás. Puede ocurrir por discos lentos, competencia por CPU, problemas de red, errores de hardware o una máquina mal configurada.

Los stragglers son importantes porque un job completo no termina hasta que termina la última tarea. Aunque el 99% del trabajo haya finalizado, unas pocas tareas lentas pueden alargar mucho el tiempo total.

MapReduce usa **backup tasks** para reducir este problema. Cuando el job está cerca de terminar, el coordinador puede lanzar ejecuciones redundantes de las tareas pendientes. La tarea se considera completada cuando termina la ejecución original o la backup, la que llegue primero.

Esto consume un poco más de recursos, pero puede reducir significativamente la latencia total del job.

## Granularidad de tareas

Es conveniente que haya más tareas que máquinas físicas. Si las tareas son pequeñas o medianas, el sistema puede balancear mejor la carga.

Por ejemplo, si un worker rápido termina muchas tareas, puede recibir más. Si otro worker es lento, afecta menos al job total porque solo retiene una parte pequeña del trabajo.

Sin embargo, tampoco conviene crear una cantidad infinita de tareas. El coordinador debe mantener estado sobre ellas. En particular, necesita estado proporcional a `M + R` para scheduling y también información relacionada con `M × R` archivos intermedios.

## Función combiner

En muchos casos, los mappers generan muchas claves repetidas localmente. En el conteo de palabras, un mapper puede emitir miles de pares como:

```text
(the, 1)
(the, 1)
(the, 1)
```

Enviar todo eso por red es ineficiente.

Un **combiner** permite hacer una reducción parcial en el mismo nodo que ejecuta el mapper. Por ejemplo, en vez de enviar mil pares `(the, 1)`, puede enviar:

```text
(the, 1000)
```

El combiner es útil cuando la operación es asociativa y conmutativa, como la suma. En muchos casos puede usar el mismo código que el reducer, pero su salida sigue siendo intermedia, no final.

## Tipos de input y output

MapReduce puede trabajar con distintos formatos de entrada y salida.

Un formato común es tratar cada línea de un archivo como un registro. También se pueden definir readers personalizados que lean desde otros formatos, bases de datos o estructuras específicas.

El input debe poder dividirse en splits significativos. Por ejemplo, si se procesa texto por líneas, los splits deben respetar límites de línea para no cortar registros por la mitad.

## Contadores y observabilidad

El sistema permite definir contadores para observar el comportamiento del job. Por ejemplo:

- cantidad de palabras procesadas;
- cantidad de documentos de cierto tipo;
- cantidad de registros descartados;
- cantidad de pares emitidos por los mappers;
- cantidad de outputs generados.

Los workers reportan estos contadores al coordinador. El coordinador los agrega y puede mostrarlos en páginas de estado. Esto sirve para monitorear progreso, detectar errores y validar invariantes del procesamiento.

## Performance

El paper muestra evaluaciones sobre clusters grandes. Dos ejemplos importantes son:

- **Grep distribuido**, buscando un patrón en aproximadamente 1 TB de datos.
- **Sort distribuido**, ordenando aproximadamente 1 TB de datos.

En el caso de sort, se observa claramente la diferencia entre las fases:

1. lectura de input;
2. shuffle por red;
3. escritura del output.

La lectura puede ser más rápida gracias a la localidad de datos. El shuffle queda más limitado por la red. La escritura final también depende del file system distribuido y de la replicación de la salida.

El paper muestra además que desactivar backup tasks aumenta notablemente el tiempo total de ejecución, porque unas pocas tareas lentas pueden producir una cola larga al final.

## Uso en Google

MapReduce se usó en Google para muchos tipos de procesamiento:

- indexación web;
- machine learning a gran escala;
- procesamiento de logs;
- análisis de queries populares;
- extracción de propiedades de páginas web;
- cómputos sobre grafos;
- clustering de noticias y productos.

Uno de los usos más importantes fue la reescritura del sistema de indexación de producción. Al expresar el pipeline como una secuencia de operaciones MapReduce, se redujo la complejidad del código y se delegaron en la biblioteca problemas como paralelización, distribución, tolerancia a fallas y manejo de máquinas lentas.

## Relación con cloud computing

MapReduce es parte de una etapa más amplia en la que empresas como Google, Amazon y Facebook desarrollaron infraestructura distribuida interna para resolver sus propios problemas de escala. Muchas de esas ideas evolucionaron hacia servicios de cloud computing.

La idea general del cloud es ofrecer capacidad de cómputo, almacenamiento o procesamiento como un servicio. En vez de que cada organización construya su propio cluster, puede usar infraestructura mantenida por un proveedor.

## Trabajo práctico asociado

El trabajo práctico busca implementar una versión simplificada de MapReduce.

Los objetivos principales son:

- usar Go;
- practicar comunicación con gRPC;
- implementar un coordinador y workers;
- ejecutar tareas map y reduce;
- soportar un ejemplo como word count;
- permitir ejecución secuencial y ejecución concurrente;
- manejar fallas de workers mediante reintentos;
- escribir tests automáticos;
- usar plugins de Go para cargar distintas implementaciones de map y reduce.

La versión del trabajo práctico simplifica algunas partes del sistema real. Por ejemplo, puede usar un file system compartido localmente en vez de implementar transferencia real de archivos entre máquinas. Aun así, mantiene las ideas centrales: coordinación, particionado, shuffle, reducción y reejecución ante fallas.

## Ideas principales

MapReduce funciona porque restringe el modelo de programación. No permite expresar cualquier algoritmo de cualquier manera, sino una familia de cómputos que pueden escribirse como transformaciones y agregaciones por clave.

Esa restricción permite que el framework haga automáticamente lo difícil:

- partir el input;
- ejecutar en paralelo;
- agrupar por clave;
- mover datos entre máquinas;
- ordenar datos intermedios;
- tolerar fallas;
- balancear carga;
- aprovechar localidad;
- reintentar tareas;
- reducir el impacto de máquinas lentas.

La abstracción es poderosa porque separa dos responsabilidades. El programador escribe la lógica del procesamiento. El sistema distribuido se ocupa de ejecutar esa lógica eficientemente en un cluster grande y poco confiable.
