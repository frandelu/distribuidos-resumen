# Clase 1 - Introducción a sistemas distribuidos y RPC

## [Apuntes de Emma](https://drive.google.com/file/d/1nvDap74mlYvCX81Cc1maEGsZ_fKwhrgu/view)

## Idea general

Un sistema distribuido no se estudia solamente como “computadoras conectadas por red”. La red es importante, pero el foco está en entender sistemas grandes formados por componentes que cooperan, fallan, se comunican y ofrecen una interfaz común hacia afuera.

Un sistema puede pensarse como un conjunto de componentes interconectados que interactúan para producir un comportamiento observable en su interfaz con el entorno. Desde afuera no interesa cada detalle interno: interesa qué operaciones expone, qué respuestas devuelve y qué garantías ofrece.

Esa forma de pensar obliga a trabajar con abstracciones. Un mismo componente puede verse como una caja negra desde afuera y, al mismo tiempo, estar compuesto internamente por otros subsistemas. La granularidad depende de qué se quiera analizar.

En sistemas distribuidos, esta idea aparece constantemente: una base de datos, un file system, un balanceador, un clúster de cómputo o un servicio web pueden verse como sistemas con interfaces, pero internamente están compuestos por muchos nodos y protocolos.

## Cómo se estudia la materia

La forma de estudiar sistemas distribuidos es a partir de sistemas reales y papers de ingeniería. La idea no es ver únicamente definiciones abstractas, sino estudiar cómo empresas como Google, Amazon o Facebook resolvieron problemas concretos de escala, almacenamiento, cómputo y comunicación.

Los papers funcionan como fuentes primarias: describen sistemas que existieron o existen, con sus decisiones de diseño, problemas, trade-offs y limitaciones. Al comparar varios sistemas aparecen patrones comunes: coordinadores, masters, réplicas, logs, shards, balanceadores, protocolos de consenso, mecanismos de recuperación, consistencia eventual y tolerancia a fallas.

Los grandes ejes de aplicación son:

- **Compute**: cómo repartir cómputo grande entre muchas máquinas. MapReduce es el ejemplo clásico.
- **Storage**: cómo almacenar datos en sistemas distribuidos, desde file systems hasta bases de datos distribuidas.
- **Stream processing y mensajería**: cómo procesar eventos y datos en tiempo real o casi real.
- **Cloud computing**: cómo se organizan grandes plataformas de infraestructura y servicios en la nube.

El objetivo no es que cada estudiante termine implementando DynamoDB o Google File System desde cero en la vida profesional, sino entender qué garantías ofrecen esos sistemas, por qué aparecen sus limitaciones y cómo razonar sobre ellos cuando se usan como piezas de sistemas más grandes.

## Tres abstracciones fundamentales

Para pensar sistemas, hay tres abstracciones básicas que aparecen una y otra vez:

1. **Memoria**.
2. **Intérpretes o procesadores**.
3. **Enlaces de comunicación**.

Estas abstracciones no siempre corresponden directamente a hardware. Pueden aparecer en muchos niveles.

## Memoria

La memoria es cualquier componente que permite guardar y recuperar estado. Puede ser hardware o software.

Ejemplos:

- Registros y caché del procesador.
- RAM.
- Flash memory.
- Discos magnéticos.
- Sistemas RAID.
- File systems.
- Bases de datos.
- Sistemas distribuidos de almacenamiento.

La interfaz conceptual de una memoria puede resumirse así:

```text
write(nombre, valor)
valor = read(nombre)
```

La memoria asocia un nombre, dirección o clave con un valor. Si se escribe un valor y luego se lee el mismo nombre, se espera recuperar ese valor, aunque en sistemas distribuidos esta expectativa se vuelve más compleja por replicación, caches, concurrencia y fallas.

Los sistemas de storage distribuidos que aparecen durante la materia son, en el fondo, abstracciones de memoria distribuidas: permiten escribir y leer datos, pero lo hacen usando varias máquinas.

## Intérpretes o procesadores

Un intérprete es cualquier componente que ejecuta instrucciones sobre un estado. Puede ser:

- Un procesador físico.
- Una máquina virtual.
- Un runtime como Node.js o la JVM.
- Un programa que interpreta comandos.
- Un motor que ejecuta tareas distribuidas.

Un intérprete toma instrucciones, consulta o modifica memoria y produce efectos observables. Aprender a programar también puede verse como aprender a controlar un intérprete mediante un lenguaje.

En sistemas distribuidos, los nodos suelen combinar procesamiento local con comunicación remota: ejecutan parte de la lógica, modifican su propio estado y se coordinan con otros nodos mediante mensajes.

## Enlaces de comunicación

Un enlace permite que dos componentes intercambien información. Conceptualmente, puede modelarse con operaciones como:

```text
send(nombre, buffer)
receive(nombre, buffer)
```

El enlace conecta componentes que no comparten memoria directamente. La unidad abstracta que viaja por el enlace es el **mensaje**.

Un mensaje es un bloque de información que una máquina envía a otra. En niveles bajos son bytes; en niveles altos puede representar un request HTTP, una response, una operación RPC, un comando, un evento o un objeto serializado.

A diferencia de una memoria local, un enlace de comunicación puede perder mensajes, duplicarlos, demorarlos o entregarlos con latencia variable. Por eso, muchos problemas de sistemas distribuidos aparecen justamente por la diferencia entre “leer/escribir memoria local” y “mandar/recibir mensajes por red”.

## Capas de red

La comunicación por red suele organizarse en capas. El modelo OSI separa siete niveles:

1. Física.
2. Enlace.
3. Red.
4. Transporte.
5. Sesión.
6. Presentación.
7. Aplicación.

En la práctica, el modelo de Internet o TCP/IP simplifica esta separación:

- **Acceso a la red**: medio físico y enlace local.
- **Red / IP**: direccionamiento y ruteo entre redes.
- **Transporte**: comunicación extremo a extremo, principalmente TCP o UDP.
- **Aplicación**: protocolos como HTTP, RPC, DNS, etc.

Para diseñar sistemas distribuidos, normalmente no se profundiza en cómo se implementa Ethernet o cómo se rutean paquetes IP, pero sí importa saber qué garantías ofrece la capa de transporte.

TCP ofrece un stream confiable y ordenado de bytes. UDP ofrece datagramas más simples, sin las mismas garantías. La elección importa: algunas semánticas de RPC dependen de que el transporte no duplique mensajes por sí mismo, por ejemplo usando TCP.

## Mensajes, streams y abstracciones

TCP puede verse como un stream de bytes. Sobre ese stream, las aplicaciones construyen mensajes. Aunque internamente todo sean bytes, los protocolos suelen separar la comunicación en unidades con significado.

En otros contextos, como procesamiento de streams, también se habla de streams, pero el significado es distinto: no se trata necesariamente de un stream TCP, sino de una secuencia de eventos o mensajes de negocio.

El concepto se repite en distintos niveles:

- TCP: stream de bytes.
- HTTP: requests y responses sobre conexiones.
- RPC: llamadas remotas convertidas en mensajes.
- Kafka o sistemas de mensajería: streams de eventos.

Las palabras se reutilizan, pero el nivel de abstracción cambia.

## Qué es un sistema distribuido

Un sistema distribuido está formado por varios nodos que tienen memoria y procesamiento propios, conectados mediante una red.

No es lo mismo que una máquina multicore. En una máquina multicore, varios procesadores pueden acceder a una memoria compartida. En un sistema distribuido, cada nodo tiene su propia memoria local y no puede escribir directamente en la memoria de otro nodo.

La coordinación se hace enviando mensajes.

Esto cambia por completo el modelo mental:

- No hay memoria global inmediata.
- No hay locks compartidos tradicionales basados en memoria común.
- No hay una visión única e instantánea del estado del sistema.
- La red puede fallar o demorar mensajes.
- Dos nodos pueden tener información distinta al mismo tiempo.

Muchas técnicas conocidas de concurrencia local no se pueden trasladar directamente a sistemas distribuidos porque dependen de memoria compartida y operaciones atómicas del procesador.

## Por qué distribuir un sistema

Distribuir un sistema tiene costos, pero también beneficios importantes.

### Escalabilidad

La escalabilidad permite aumentar la capacidad del sistema.

Hay dos formas principales:

- **Escalabilidad vertical**: usar una máquina más grande, con más CPU, memoria, disco o red.
- **Escalabilidad horizontal**: agregar más máquinas y repartir la carga.

La escalabilidad horizontal es central en sistemas distribuidos. Permite crecer agregando nodos relativamente baratos, pero exige resolver particionamiento, balanceo, coordinación y consistencia.

Un ejemplo clásico es el uso de muchas máquinas comunes para resolver problemas enormes. Google popularizó el enfoque de usar clusters de commodity hardware para tareas como indexar la web completa. En vez de depender de una supercomputadora única, se usan muchas máquinas más simples coordinadas por software.

### Tolerancia a fallas

Distribuir también permite que el sistema siga funcionando aunque algunos componentes fallen. Si hay varias réplicas o varios nodos capaces de hacer el mismo trabajo, la falla de uno no necesariamente implica la caída total del sistema.

Esto no ocurre automáticamente. Agregar máquinas también aumenta la cantidad de puntos posibles de falla. La tolerancia a fallas aparece cuando el sistema está diseñado para detectar fallas, aislarlas, redirigir tráfico, reconstruir estado y continuar operando.

La propiedad clave es que los sistemas distribuidos pueden tener **fallas parciales**: una parte falla y otra sigue funcionando.

### Compartir recursos

Un sistema distribuido permite compartir recursos ubicados en diferentes máquinas:

- Archivos.
- Bases de datos.
- Cómputo.
- Colas.
- Caches.
- Servicios externos.

El desafío es coordinar el acceso a esos recursos, especialmente cuando hay concurrencia y fallas.

### Economía de escala

Puede ser más barato usar muchas máquinas comunes que una única máquina enorme. Los data centers modernos se construyen alrededor de esta idea: miles de servidores relativamente estándar coordinados para ofrecer almacenamiento, cómputo y servicios de alto nivel.

Esta economía de escala también explica parte del origen práctico del cloud computing. Grandes proveedores construyen infraestructura masiva y luego venden capacidad de cómputo, storage y red como servicios.

### Modularidad forzada

Distribuir un sistema obliga a separar componentes mediante interfaces. Un servicio no puede acceder directamente a la memoria interna de otro: debe comunicarse usando una API, mensajes o protocolos definidos.

Esto fuerza límites más claros entre módulos. Esa separación puede mejorar la organización del sistema y reducir acoplamiento, pero también introduce latencia y fallas de red.

### Separación administrativa

En sistemas grandes, distintos equipos pueden ser responsables de distintos servicios. Cada equipo define, mantiene y evoluciona su componente, siempre que respete la interfaz que expone hacia los demás.

Esta separación es una de las motivaciones detrás de arquitecturas orientadas a servicios y microservicios, aunque lo importante no es que los servicios sean “micro”, sino que tengan límites claros, responsabilidades definidas e interfaces estables.

## Fallas parciales y compartimentalización

Una falla parcial ocurre cuando una parte del sistema falla, pero el resto continúa funcionando. Esta propiedad es una ventaja y un problema al mismo tiempo.

Es una ventaja porque permite diseñar sistemas que resistan fallas. Es un problema porque obliga a razonar sobre estados intermedios: un nodo puede estar vivo, otro caído, la red entre ambos puede estar rota, y un tercero puede tener una visión distinta.

La analogía de los compartimentos estancos de un barco ayuda a entender la idea. Si un compartimento se inunda, el barco puede seguir flotando si el daño queda aislado. Pero si se rompen demasiados compartimentos, el sistema completo falla.

En sistemas distribuidos ocurre algo similar. Un sistema puede tolerar la caída de algunos nodos, pero si se caen demasiados o si falla un subsistema crítico, la falla puede propagarse.

Un incidente grande en un servicio base puede afectar muchos servicios dependientes. Un ejemplo típico es un servicio de almacenamiento cloud que falla y arrastra aplicaciones, sistemas de monitoreo, sitios web y otros servicios que dependen de él. El problema no es solo que falle un nodo, sino cómo se propaga esa falla por las dependencias.

## Transparencia de distribución

Muchos sistemas distribuidos intentan ocultar que están distribuidos. Esta idea se llama transparencia de distribución: desde afuera, el sistema parece una sola cosa aunque internamente tenga muchas máquinas.

La transparencia es útil, pero exagerarla puede ser peligrosa. Si una interfaz remota intenta parecer idéntica a una interfaz local, el programador puede olvidar que existen latencia, timeouts, fallas parciales y problemas de red.

Una abstracción distribuida debe ocultar detalles innecesarios, pero no debe ocultar las diferencias fundamentales entre una llamada local y una llamada remota.

## NFS como ejemplo

NFS, Network File System, permite montar archivos remotos como si fueran parte del sistema de archivos local.

La idea es cómoda: una aplicación usa operaciones de archivos normales y el sistema se encarga de hablar con un servidor remoto. La arquitectura general incluye:

- Procesos de usuario en el cliente.
- Virtual File System.
- Cliente NFS.
- Red.
- Servidor NFS.
- File system local del servidor.
- Disco del servidor.

El problema es que la interfaz de file system local asume ciertas propiedades que no siempre valen sobre la red. Un disco local tiene cierta latencia, ciertas fallas y ciertas garantías. Un servidor remoto puede no responder, la red puede caerse, la operación puede quedar bloqueada o puede devolver errores que una aplicación local no esperaba.

NFS muestra el riesgo de hacer una abstracción demasiado transparente: parece local, pero no se comporta exactamente como local.

## Características problemáticas de los enlaces

Los enlaces de comunicación tienen propiedades que no pueden ignorarse.

### Latencia variable

Una operación remota tarda mucho más que una operación local y, además, la latencia varía. Puede depender de congestión, distancia física, routers, carga del servidor o problemas intermedios.

Esto afecta decisiones como timeouts, cantidad de llamadas remotas y granularidad de las APIs.

### Ausencia de memoria compartida

No se puede asumir que dos nodos comparten memoria. Cualquier intento de simular memoria compartida encima de red trae costos y problemas de consistencia.

### Fallas parciales

Un nodo puede fallar mientras otros siguen vivos. La red puede fallar entre dos nodos pero no entre otros. Una respuesta puede perderse aunque el request haya llegado.

### Concurrencia

Cada nodo ejecuta de forma independiente. Los eventos ocurren concurrentemente, sin un reloj global perfecto y sin un orden único obvio.

Los locks locales basados en memoria compartida no funcionan directamente. Para coordinar nodos distribuidos hacen falta otros mecanismos: consenso, leases, logs replicados, transacciones distribuidas, coordinadores, etc.

## Organización cliente-servidor

En la organización cliente-servidor hay una asimetría clara:

- El **cliente** necesita algo.
- El **servidor** ofrece un servicio.

La comunicación típica es request/response:

```text
cliente  -- request  --> servidor
cliente  <-- response -- servidor
```

El cliente inicia el pedido y el servidor responde. Esta idea aparece en HTTP, bases de datos, APIs internas, servicios cloud y RPC.

La diferencia con un modelo peer-to-peer es que en peer-to-peer los nodos tienen roles más simétricos. Un nodo puede actuar como cliente y servidor según la operación. Protocolos como Raft tienen una estructura más peer-to-peer en el sentido de que todos los nodos pueden ocupar roles similares, aunque temporalmente haya un líder.

## HTTP como ejemplo de cliente-servidor

HTTP es un ejemplo clásico de request/response. Un navegador envía un request y el servidor responde con una página, un JSON, un archivo o un código de error.

En una arquitectura real, no suele haber un único servidor:

```text
cliente -> load balancer -> servidores web -> base de datos
```

Un balanceador de carga recibe tráfico y lo distribuye entre varias instancias del mismo servicio. Si los servidores son stateless, escalar es relativamente simple: se agregan más instancias y el balanceador reparte requests.

La base de datos es más difícil de escalar porque tiene estado. No alcanza con crear varias bases independientes, porque una escritura en una no aparecería automáticamente en las otras. Para eso se usan estrategias como primary/backup, réplicas de lectura, sharding o bases distribuidas diseñadas específicamente para escalar.

## Load balancers y servicios

Un load balancer es un componente que distribuye tráfico entre varias instancias. Es útil cuando varias máquinas ejecutan el mismo servicio y cualquiera puede atender un request.

Si el servicio no tiene estado local relevante, el balanceo es simple. Si el servicio tiene estado, aparecen problemas de consistencia, replicación y ruteo.

En arquitecturas con muchos servicios, cada servicio puede exponer su propia interfaz y estar detrás de su propio mecanismo de balanceo o descubrimiento. Lo importante es que los clientes no dependan de una instancia particular, sino de una abstracción de servicio.

## RPC

RPC significa **Remote Procedure Call**, llamada a procedimiento remoto.

La idea es invocar una operación que se ejecuta en otra máquina con una interfaz parecida a una llamada local:

```text
resultado = servicio.operacion(argumentos)
```

Sin RPC, cada operación remota requeriría escribir manualmente código para:

1. Armar un mensaje.
2. Serializar argumentos.
3. Enviar bytes por un socket.
4. Esperar una respuesta.
5. Recibir bytes.
6. Deserializar el resultado.
7. Manejar errores.

RPC encapsula ese patrón repetitivo en una capa intermedia.

## Middleware

Un middleware es una capa entre la aplicación y la red. En RPC, el middleware oculta parte del trabajo de comunicación:

```text
aplicación
RPC / middleware
sockets / transporte
red
```

La aplicación invoca una función. La capa RPC transforma esa llamada en mensajes, los envía, recibe la respuesta y reconstruye el resultado.

El término middleware se usó mucho en sistemas distribuidos, aunque es amplio y a veces ambiguo. En este contexto, interesa como capa que abstrae comunicación entre componentes distribuidos.

## Stubs

RPC suele implementarse con **stubs**.

### Stub del cliente

El stub del cliente parece una función local. Cuando la aplicación lo llama:

1. Recibe los argumentos.
2. Los serializa.
3. Arma un request.
4. Envía el request por red.
5. Espera la response.
6. Deserializa el resultado.
7. Devuelve el resultado a la aplicación.

### Stub del servidor

El stub del servidor hace el proceso inverso:

1. Recibe el request.
2. Deserializa los argumentos.
3. Llama a la función real implementada en el servidor.
4. Toma el resultado.
5. Lo serializa.
6. Devuelve la response.

Gracias a los stubs, la aplicación trabaja con procedimientos de alto nivel, no con bytes y sockets.

## Serialización y marshalling

La red transporta bytes. Por eso, los datos de alto nivel deben convertirse a una representación binaria o textual.

Ese proceso se llama serialización o marshalling. La operación inversa se llama deserialización o unmarshalling.

Problemas asociados:

- Cómo representar tipos.
- Cómo mantener compatibilidad entre lenguajes.
- Cómo versionar mensajes.
- Cómo manejar campos opcionales.
- Cómo evitar ambigüedades.
- Cómo validar datos recibidos.

En RPC moderno, estas tareas suelen estar automatizadas por herramientas que generan código a partir de una especificación de interfaz.

## gRPC

gRPC es una implementación moderna de RPC desarrollada por Google.

La interfaz se define en archivos `.proto`. Allí se describen servicios, métodos, requests y responses. A partir de esa definición se genera código para distintos lenguajes.

Ejemplo conceptual:

```proto
service TimeService {
  rpc GetTime(TimeRequest) returns (TimeResponse);
}

message TimeRequest {}

message TimeResponse {
  int64 unix_time = 1;
}
```

A partir de esa especificación, la herramienta genera:

- Código cliente para invocar el servicio.
- Código servidor para registrar la implementación real.
- Serialización y deserialización.
- Estructuras de datos para requests y responses.

El programador implementa la lógica del servidor y usa el cliente generado. No escribe manualmente la comunicación por sockets.

## REST y RPC

REST también permite comunicación entre servicios, típicamente sobre HTTP. La diferencia es de estilo.

REST organiza operaciones alrededor de recursos y métodos HTTP como `GET`, `POST`, `PUT` y `DELETE`. RPC organiza operaciones como procedimientos explícitos.

RPC suele ser más directo cuando se quiere modelar una API como operaciones con nombres específicos. REST suele ser cómodo cuando se trabaja con recursos expuestos por HTTP, especialmente para clientes web o APIs públicas.

No hay una única respuesta universal. La elección depende del contexto, los clientes, la infraestructura, el tipo de operación y las herramientas disponibles.

## RPC no es una llamada local

Aunque RPC intenta parecer una llamada local, semánticamente no lo es.

Diferencias principales:

- Una llamada remota tiene mucha más latencia.
- La latencia es variable.
- La red puede fallar.
- El servidor puede ejecutar la operación pero la respuesta puede perderse.
- El request puede no llegar.
- Puede haber reintentos y duplicados.
- No hay memoria compartida.
- Hay serialización y deserialización.

Esta diferencia es central. Una abstracción RPC demasiado transparente puede llevar a diseñar mal el sistema, por ejemplo haciendo demasiadas llamadas pequeñas o ignorando timeouts.

## Fallas en RPC

En una llamada remota pueden ocurrir situaciones que no aparecen en una llamada local.

Caso normal:

```text
cliente -> request -> servidor
cliente <- response <- servidor
```

Fallas posibles:

1. El request no llega al servidor.
2. El request llega, el servidor ejecuta la operación, pero la response no llega al cliente.

Desde el punto de vista del cliente, ambos casos pueden verse igual: envió un request y no recibió respuesta.

Esto genera incertidumbre. Ante un timeout, el cliente no sabe si la operación no ocurrió o si ocurrió pero se perdió la respuesta.

## Semánticas de entrega

Las semánticas de entrega describen qué garantías ofrece el sistema ante fallas y reintentos.

### At-least-once

La semántica **at-least-once** intenta que la operación llegue y se ejecute al menos una vez.

Si no llega una respuesta, el cliente o el stub reintenta:

```text
request
request
request
...
```

El problema es que el servidor puede recibir el mismo request varias veces. Por eso, esta semántica requiere que la operación sea idempotente o que el servidor pueda detectar duplicados.

Sirve bien para operaciones como lecturas o escrituras idempotentes, pero puede ser peligrosa para operaciones como transferir dinero, incrementar un contador o crear una orden.

### At-most-once

La semántica **at-most-once** intenta que la operación se ejecute como máximo una vez.

Una forma simple de lograrlo es no reintentar automáticamente. Si no hay respuesta, se reporta error. Si llega `OK`, se sabe que el servidor ejecutó la operación una vez. Si llega error o timeout, queda la incertidumbre: pudo no haberse ejecutado o pudo haberse ejecutado sin que la respuesta volviera.

Esta semántica evita duplicados automáticos, pero no garantiza que la operación haya ocurrido.

### Exactly-once

La semántica **exactly-once** sería la ideal: ejecutar la operación exactamente una vez.

En sistemas distribuidos con fallas de red, exactly-once general no puede garantizarse de forma absoluta. El problema fundamental es que, si se pierde la comunicación, el cliente no puede distinguir perfectamente entre “el servidor no ejecutó” y “el servidor ejecutó pero la respuesta se perdió”.

Se pueden construir aproximaciones útiles, pero no eliminar la dificultad fundamental.

## Idempotencia

Una operación es idempotente si ejecutarla una vez o muchas veces produce el mismo efecto final.

Ejemplos idempotentes:

- Leer un valor.
- Setear `x = 5`.
- Borrar un recurso si el resultado esperado es que no exista.

Ejemplos no idempotentes:

- `x = x + 1`.
- Transferir dinero.
- Crear una orden sin identificador único.
- Agregar un elemento duplicable a una lista.

La idempotencia es una herramienta clave para hacer reintentos seguros.

## Aproximar exactly-once con idempotency IDs

Una forma práctica de acercarse a exactly-once es agregar un identificador único a cada operación.

Ejemplo:

```text
add_money(account_id, amount, idempotency_id)
```

El cliente genera un `idempotency_id` único y lo manda con el request. El servidor guarda los IDs ya procesados en almacenamiento persistente.

Si llega un request con un ID nuevo, lo procesa y guarda el ID. Si llega de nuevo el mismo ID por un retry, no vuelve a ejecutar la operación; devuelve la respuesta previa o ignora el duplicado.

Esto transforma una operación no idempotente en una operación efectivamente idempotente desde el punto de vista del protocolo.

No resuelve mágicamente todos los problemas, pero es una técnica común para manejar reintentos sin duplicar efectos.

## Diseño de APIs distribuidas

Al diseñar una API distribuida conviene pensar explícitamente:

- Qué operaciones son idempotentes.
- Qué operaciones pueden reintentarse.
- Qué pasa si el request se duplica.
- Qué pasa si la response se pierde.
- Qué timeouts se usan.
- Qué errores son definitivos y cuáles son ambiguos.
- Qué identificadores únicos hacen falta.
- Qué estado persistente permite deduplicar.
- Qué garantías ofrece la capa RPC.

Una API remota no debe diseñarse como si fuera una llamada local gratuita. Cada llamada cruza la red y puede fallar de formas nuevas.

## Resumen

Un sistema distribuido combina nodos independientes, memoria local, procesamiento y comunicación por mensajes. La ausencia de memoria compartida y la presencia de red hacen que aparezcan problemas de latencia, concurrencia, fallas parciales y coordinación.

Distribuir permite escalar, tolerar fallas, compartir recursos, aprovechar economía de escala y separar responsabilidades, pero introduce complejidad real.

RPC ofrece una abstracción cómoda para invocar operaciones remotas mediante una interfaz parecida a una función local. Esa comodidad se implementa con stubs, serialización, mensajes request/response y middleware.

La idea central es que RPC parece una llamada local, pero no lo es. La red puede perder, demorar o duplicar mensajes; el servidor puede ejecutar una operación aunque la respuesta no vuelva; y el cliente puede quedar sin saber qué ocurrió.

Por eso, las semánticas de entrega, la idempotencia, los reintentos, los timeouts y los identificadores únicos son parte esencial del diseño de sistemas distribuidos.
