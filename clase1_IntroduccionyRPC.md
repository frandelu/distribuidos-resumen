# Introducción a sistemas distribuidos y RPC

## [Apuntes de Emma](https://drive.google.com/file/d/1nvDap74mlYvCX81Cc1maEGsZ_fKwhrgu/view)

## Sistema

Un sistema es un conjunto de componentes interconectados que interactúan entre sí para producir un comportamiento observable en su interfaz con el entorno.

El entorno no necesita conocer todos los detalles internos del sistema. Lo que observa es la interfaz: entradas, salidas y comportamiento visible. Por eso, para poder estudiar sistemas complejos se trabaja con abstracciones que esconden detalles innecesarios y dejan visible lo importante.

Tres abstracciones fundamentales para razonar sobre sistemas son:

- **Memoria**: dónde se guarda el estado.
- **Intérprete o procesador**: quién ejecuta instrucciones o transforma datos.
- **Enlaces de comunicación**: cómo se conectan componentes y cómo viajan los mensajes.

Un sistema puede verse como una combinación de componentes que almacenan información, ejecutan operaciones y se comunican mediante interfaces bien definidas.

## Memoria, procesamiento y comunicación

La memoria puede aparecer en muchos niveles:

- Registros y caché del procesador.
- RAM.
- Memoria flash.
- Discos magnéticos.
- Discos ópticos.
- Sistemas de archivos.
- Bases de datos.
- Sistemas distribuidos de almacenamiento.

La operación básica sobre memoria es leer y escribir:

- `write(nombre, valor)`: asociar un valor a un nombre o dirección.
- `read(nombre)`: obtener el valor asociado a ese nombre o dirección.

En general, una escritura cambia el estado del sistema y una lectura observa ese estado. Cuando esto se lleva a un sistema distribuido, aparece una dificultad: el estado puede estar repartido en varias máquinas, con comunicación por red y posibles fallas parciales.

El procesamiento también puede verse como una abstracción. Un intérprete toma instrucciones, consulta o modifica memoria y produce efectos observables. Un programa puede ser entendido como una secuencia de instrucciones ejecutadas por un intérprete sobre un determinado estado.

La comunicación aparece cuando los componentes no comparten memoria directamente. En ese caso se conectan mediante enlaces.

## Enlaces y mensajes

Un enlace permite que dos componentes intercambien información. Conceptualmente, puede modelarse con dos operaciones:

- `send(nombre, buffer)`: enviar bytes a un destino.
- `receive(nombre, buffer)`: recibir bytes desde un origen.

El contenido que viaja por un enlace suele pensarse como un mensaje. En niveles bajos son bytes; en niveles más altos se interpretan como estructuras con significado: requests, responses, comandos, eventos, objetos serializados, etc.

La comunicación por red se organiza en capas para separar responsabilidades. Cada capa ofrece una abstracción a la capa superior y se apoya en la capa inferior.

## Capas de red

El modelo OSI divide la comunicación en siete capas:

1. Física.
2. Enlace.
3. Red.
4. Transporte.
5. Sesión.
6. Presentación.
7. Aplicación.

En la práctica, para Internet se suele usar el modelo TCP/IP:

- **Acceso a la red**: comunicación con el medio físico y enlace local.
- **Internet / IP**: direccionamiento y ruteo entre redes.
- **Transporte**: comunicación extremo a extremo, por ejemplo TCP o UDP.
- **Aplicación**: protocolos de alto nivel como HTTP, DNS, RPC, etc.

El principio extremo a extremo indica que muchas garantías importantes deben resolverse en los extremos de la comunicación, no necesariamente dentro de la red. La red puede transportar paquetes, pero la aplicación o la capa de transporte deben ocuparse de aspectos como reintentos, orden, duplicados, confirmaciones o interpretación del mensaje.

## Sistema distribuido

Un sistema distribuido está formado por múltiples computadoras o nodos conectados por una red que cooperan para ofrecer un servicio o comportamiento común.

La diferencia central con una máquina aislada es que ya no hay una única CPU y una única memoria local compartida. Cada nodo tiene su propio procesador y su propia memoria, y la coordinación entre nodos ocurre mediante mensajes.

Esto introduce una idea clave: en un sistema distribuido no existe un estado global inmediato y compartido. Cada nodo observa su propio estado local y recibe información de otros nodos con cierto retraso, a través de la red.

## Motivos para distribuir un sistema

Distribuir un sistema puede tener varios objetivos.

### Escalabilidad

Cuando una sola máquina no alcanza, se puede escalar de dos maneras:

- **Escalabilidad vertical**: mejorar una máquina agregando más CPU, memoria, disco o capacidad de red.
- **Escalabilidad horizontal**: agregar más máquinas y repartir el trabajo entre ellas.

La escalabilidad horizontal suele ser más flexible, pero obliga a resolver problemas de coordinación, partición de datos, balanceo de carga, consistencia y fallas parciales.

### Tolerancia a fallas

Un sistema distribuido puede seguir funcionando aunque algunos componentes fallen. Para lograrlo se usan técnicas como replicación, redundancia, reintentos, detección de fallas y failover.

La tolerancia a fallas no aparece automáticamente por tener más máquinas. De hecho, al agregar nodos también se agregan más puntos posibles de falla. El diseño debe contemplar qué pasa cuando un nodo se cae, cuando un mensaje se pierde, cuando la red se particiona o cuando una respuesta llega tarde.

### Compartir recursos

Varios clientes o servicios pueden compartir almacenamiento, cómputo, bases de datos, archivos, colas, caches o servicios externos.

El desafío es administrar el acceso concurrente a esos recursos y definir reglas claras de consistencia.

### Economía de escala

Puede ser más barato usar muchas máquinas comunes que una única máquina muy grande. Los centros de datos aprovechan esta idea: miles de servidores relativamente estándar coordinados para ofrecer capacidad agregada.

### Modularidad forzada

Distribuir obliga a definir límites claros entre componentes. Cada servicio expone una interfaz y oculta su implementación. Esto ayuda a separar responsabilidades, pero también introduce costos de comunicación y fallas parciales. -> Compartimentalización de fallas.

### Responsabilidades administrativas separadas

Distintos componentes pueden estar bajo control de equipos, organizaciones o dominios administrativos diferentes. Esto es común en Internet, microservicios, sistemas empresariales y servicios cloud.

## Fallas parciales

Una propiedad central de los sistemas distribuidos es que pueden ocurrir fallas parciales. En una computadora local, si el proceso está funcionando, normalmente se asume que memoria, CPU y llamadas locales están disponibles. En un sistema distribuido, un nodo puede estar vivo mientras otro no responde, la red puede perder mensajes, o una operación remota puede ejecutarse aunque la respuesta nunca vuelva.

Ejemplos de problemas típicos:

- El cliente envía un request y no llega al servidor.
- El servidor recibe el request, ejecuta la operación, pero la respuesta no llega al cliente.
- La respuesta llega duplicada.
- El request llega duplicado.
- Dos nodos tienen visiones distintas sobre si un tercero está vivo o caído.
- Una parte del sistema queda aislada por una partición de red.

Por eso, en sistemas distribuidos no alcanza con pensar en éxito o error como en una llamada local. Muchas veces aparece un tercer estado: no se sabe si la operación ocurrió.

## Compartimentalización y aislamiento de fallas

Una forma de diseñar sistemas robustos es dividirlos en compartimentos. La idea es que una falla no arrastre necesariamente a todo el sistema.

La analogía de un barco con compartimentos estancos sirve para entenderlo: si entra agua en una sección, el barco puede seguir flotando si el daño queda contenido. En sistemas distribuidos ocurre algo parecido con regiones, zonas de disponibilidad, réplicas, shards, particiones, circuit breakers y límites de carga.

La compartimentalización busca que una falla sea parcial y administrable, no total.

## Centros de datos y servicios cloud

Los sistemas distribuidos modernos suelen ejecutarse en centros de datos con muchas máquinas conectadas por redes internas de alta velocidad. Sobre esa infraestructura se construyen servicios de almacenamiento, cómputo, bases de datos, colas, caches y balanceadores.

Un incidente en un servicio base puede afectar a muchos servicios construidos encima. Por ejemplo, una interrupción importante en un servicio de almacenamiento puede impactar aplicaciones, sitios web, procesos batch, sistemas de monitoreo y otros servicios dependientes.

Esto muestra que las dependencias distribuidas no son detalles secundarios: forman parte del comportamiento real del sistema.

## NFS como ejemplo de sistema distribuido

NFS permite que un cliente acceda a archivos remotos como si fueran parte de su sistema de archivos local.

La arquitectura general incluye:

- Procesos de usuario en el cliente.
- Un sistema de archivos virtual.
- Un cliente NFS.
- Comunicación por red.
- Un servidor NFS.
- Un sistema de archivos local del servidor.
- Disco local del servidor.

La abstracción buscada es que el acceso remoto sea parecido al acceso local. Sin embargo, hay diferencias importantes.

### Problemas de NFS

El acceso a un archivo remoto tiene latencia variable y mucho mayor que un acceso a memoria o disco local. Además, la red puede fallar, el servidor puede no responder y puede haber múltiples clientes accediendo al mismo archivo al mismo tiempo.

Aparecen varias decisiones de diseño:

- Cómo manejar locks.
- Cómo mantener consistencia entre clientes.
- Cómo responder ante fallas parciales.
- Cuándo cachear datos.
- Cómo invalidar caches.
- Cómo balancear robustez y performance.

Un diseño más robusto puede ser más lento. Un diseño más performante puede relajar garantías de consistencia. No hay una solución universal: depende del caso de uso.

## Modelos de organización

### Cliente-servidor

En el modelo cliente-servidor, un cliente envía pedidos y un servidor responde. El servidor ofrece un servicio y el cliente lo consume.

La interacción típica es request/response:

1. El cliente arma un mensaje con argumentos.
2. El cliente envía el request.
3. El servidor espera mensajes.
4. El servidor recibe el request.
5. El servidor extrae los argumentos.
6. El servidor ejecuta la operación.
7. El servidor arma una respuesta.
8. El cliente recibe la respuesta y revisa el resultado.

Este modelo aparece en HTTP, bases de datos, servicios internos, APIs, sistemas de archivos remotos y RPC.

### Peer-to-peer

En el modelo peer-to-peer, los nodos tienen roles más simétricos. Un nodo puede actuar como cliente y servidor al mismo tiempo. La comunicación puede darse entre pares sin un servidor central único.

Este modelo aparece en sistemas de intercambio de archivos, blockchains, protocolos de gossip y algunas arquitecturas descentralizadas.

## HTTP y arquitectura web

HTTP es un ejemplo conocido de comunicación cliente-servidor. Un cliente envía un request y un servidor devuelve una response.

En una arquitectura web real, rara vez hay un único servidor aislado. Puede haber:

- Clientes o navegadores.
- Un balanceador de carga.
- Múltiples servidores de aplicación.
- Bases de datos.
- Almacenamiento de archivos.
- Réplicas.
- Backups.
- Nodos primarios y secundarios.

El balanceador distribuye tráfico entre servidores. Las bases de datos pueden tener un nodo principal y réplicas. Los backups permiten recuperación. Cada componente agrega capacidad, pero también agrega nuevas formas de falla.

## RPC

RPC significa Remote Procedure Call: llamada a procedimiento remoto.

La idea es permitir que un programa invoque una operación que se ejecuta en otra máquina usando una interfaz parecida a una llamada local.

En lugar de que el programador arme manualmente mensajes de red para cada operación, RPC ofrece una abstracción de procedimiento:

```text
resultado = servicio.operacion(argumentos)
```

Aunque parezca una llamada común, internamente ocurre comunicación por red.

## Middleware RPC

RPC se implementa como una capa de middleware entre la aplicación y la red.

La aplicación invoca una función. El middleware se encarga de convertir esa llamada en mensajes, enviarlos por red, recibir la respuesta y reconstruir el resultado.

Esto permite separar la lógica de negocio de los detalles de comunicación. La aplicación trabaja con operaciones de alto nivel, mientras el sistema RPC maneja transporte, codificación, decodificación, errores y respuestas.

## Stubs

Los stubs son piezas de código generadas o escritas para ocultar los detalles de red.

### Stub del cliente

El stub del cliente tiene la misma apariencia que una función local. Cuando se lo llama:

1. Toma los argumentos.
2. Los serializa.
3. Arma un mensaje.
4. Envía el request al servidor.
5. Espera una respuesta.
6. Deserializa el resultado.
7. Devuelve el valor al programa cliente.

### Stub del servidor

El stub del servidor recibe mensajes desde la red. Cuando llega un request:

1. Deserializa los argumentos.
2. Llama a la función real del servicio.
3. Toma el resultado.
4. Lo serializa.
5. Envía la respuesta al cliente.

Gracias a los stubs, la aplicación cliente y la aplicación servidor pueden escribirse como si compartieran una interfaz de funciones, aunque estén en máquinas distintas.

## Serialización y marshalling

Para enviar una llamada por red, los argumentos y resultados deben convertirse a bytes. A este proceso se lo llama serialización o marshalling.

El receptor realiza la operación inversa: toma los bytes y reconstruye los valores originales. Esto se llama deserialización o unmarshalling.

Este paso es necesario porque la red solo transporta bytes. Las estructuras de datos del lenguaje, los objetos, los enteros, strings o listas deben representarse en un formato común.

Problemas a resolver:

- Representación de tipos.
- Orden de bytes.
- Compatibilidad entre lenguajes.
- Versionado de mensajes.
- Campos opcionales.
- Errores de interpretación.

## Ejemplo: servicio de tiempo

Un ejemplo simple de RPC es un servicio que devuelve la hora.

El cliente invoca algo como:

```text
GET_TIME()
```

El stub del cliente convierte esa llamada en un mensaje. El servidor recibe el mensaje, ejecuta la operación real para obtener la hora y devuelve una respuesta. El cliente recibe el resultado y continúa.

Aunque la interfaz parece una función simple, hay una secuencia de mensajes de ida y vuelta por red.

## RPC no es una llamada local

RPC intenta parecer una llamada local, pero no lo es.

Diferencias importantes:

- Una llamada local tiene latencia muy baja; una llamada remota depende de la red.
- Una llamada local normalmente falla junto con el proceso; una llamada remota puede fallar parcialmente.
- En una llamada local no suelen aparecer mensajes duplicados; en una llamada remota sí pueden aparecer por reintentos.
- En una llamada local se sabe si se ejecutó; en una llamada remota puede quedar la duda.
- En una llamada local se comparten tipos, memoria y proceso; en una llamada remota hay serialización y procesos separados.

Por eso, ocultar demasiado la diferencia entre local y remoto puede ser peligroso. La abstracción de RPC es cómoda, pero el programador debe recordar que detrás hay red.

## Problemas de RPC

Los dos problemas principales son:

1. **Mayor latencia**.
2. **Nuevas formas de falla**.

La latencia cambia la forma de diseñar. No conviene hacer miles de llamadas remotas pequeñas si se puede enviar una operación más gruesa. Las interfaces distribuidas suelen diseñarse para reducir viajes de ida y vuelta.

Las fallas obligan a definir semánticas claras. Ante un timeout, el cliente no sabe necesariamente si el servidor no recibió el request, si lo recibió y falló, si lo ejecutó pero la respuesta se perdió, o si la respuesta está demorada.

## Reintentos y duplicados

Cuando un cliente no recibe respuesta, puede reintentar. Pero un reintento puede causar duplicados.

Ejemplo:

1. El cliente envía `borrar_archivo(x)`.
2. El servidor borra el archivo.
3. La respuesta se pierde.
4. El cliente reintenta.
5. El servidor recibe de nuevo `borrar_archivo(x)`.

Si la operación no está diseñada para ejecutarse más de una vez, el duplicado puede generar problemas.

Esto lleva a estudiar idempotencia y semánticas de ejecución.

## Idempotencia

Una operación es idempotente si ejecutarla una vez o varias veces produce el mismo efecto final.

Ejemplos usualmente idempotentes:

- Leer un valor.
- Setear una variable a un valor fijo.
- Borrar un recurso si el resultado deseado es que no exista.

Ejemplos no idempotentes:

- Incrementar un contador.
- Transferir dinero.
- Agregar un elemento repetido a una lista.
- Crear una orden de compra sin identificador único.

La idempotencia facilita los reintentos. Si una operación es idempotente, repetirla por timeout es menos riesgoso.

## Semánticas de llamada remota

### At-least-once

La semántica at-least-once intenta que la operación se ejecute al menos una vez. Si no llega respuesta, el cliente reintenta.

Ventaja:

- Aumenta la probabilidad de que la operación ocurra.

Problema:

- La operación puede ejecutarse más de una vez.

Funciona bien para operaciones idempotentes, pero puede ser peligrosa para operaciones no idempotentes.

### At-most-once

La semántica at-most-once intenta que la operación se ejecute como máximo una vez.

Para esto se suelen usar identificadores de request, detección de duplicados y cacheo de respuestas. Si llega un request repetido, el servidor puede devolver la respuesta anterior en lugar de ejecutar de nuevo.

Ventaja:

- Evita ejecuciones duplicadas.

Problema:

- Ante ciertas fallas, el cliente puede no saber si la operación se ejecutó o no.

### Exactly-once

Exactly-once significa que la operación se ejecuta una y solo una vez.

En un sistema distribuido con fallas de red, exactly-once general es imposible de garantizar solo con mensajes y reintentos. Se pueden construir aproximaciones para casos específicos usando identificadores únicos, transacciones, logs persistentes, deduplicación e idempotencia, pero no se elimina la dificultad fundamental: si se pierde la comunicación, distinguir entre “no se ejecutó” y “se ejecutó pero se perdió la respuesta” puede ser imposible.

## Diseño de APIs distribuidas

Al diseñar una interfaz distribuida conviene considerar:

- Reducir la cantidad de llamadas remotas.
- Preferir operaciones idempotentes cuando sea posible.
- Usar identificadores únicos para requests o comandos.
- Definir timeouts.
- Definir reintentos.
- Definir qué errores son recuperables.
- Pensar qué pasa si la respuesta llega tarde.
- Pensar qué pasa si el cliente reintenta.
- Pensar qué estado se guarda en cliente y servidor.

Una API remota no debería diseñarse como si cada método fuera una llamada local gratuita. Cada invocación cruza la red y puede fallar de formas nuevas.

## Resumen conceptual

Los sistemas distribuidos combinan procesamiento, memoria y comunicación en múltiples nodos independientes conectados por red.

Distribuir permite escalar, compartir recursos, tolerar fallas y separar responsabilidades, pero introduce latencia, concurrencia, coordinación, ausencia de estado global inmediato y fallas parciales.

RPC ofrece una abstracción cómoda para invocar operaciones remotas como si fueran procedimientos locales. Esa abstracción se implementa con stubs, serialización, mensajes request/response y middleware.

La dificultad central de RPC es que la llamada parece local, pero semánticamente no lo es. La red puede demorar, duplicar, perder o reordenar mensajes. Por eso, las operaciones remotas necesitan semánticas explícitas: at-least-once, at-most-once, idempotencia, deduplicación, timeouts y manejo de fallas.

El diseño de sistemas distribuidos consiste en elegir cuidadosamente qué garantías se ofrecen, qué costos se aceptan y cómo se comporta el sistema cuando algo falla.
