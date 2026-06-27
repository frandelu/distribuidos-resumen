# Clase 13 — Bitcoin

## [Diapositivas](https://drive.google.com/file/d/1RE_q4eKTCJVm-SfnPaMSQIq7JC-jAe7y/view)

## Idea general

Bitcoin se puede estudiar como un sistema distribuido que mantiene un **ledger** o libro mayor de pagos en un entorno mucho más hostil que los sistemas vistos antes.

En sistemas como Raft, GFS, Dynamo o ZooKeeper suele existir una organización que controla los servidores. Los nodos pueden fallar por problemas de hardware, discos, red, energía o saturación, pero no se asume que actúen maliciosamente.

Bitcoin cambia el escenario:

```text
sistema distribuido clásico
  ├── nodos controlados por una misma organización
  ├── fallas aleatorias
  └── objetivo: seguir funcionando de forma consistente

Bitcoin
  ├── nodos controlados por personas distintas
  ├── no hay confianza mutua
  ├── cualquiera puede unirse
  └── objetivo: mantener un ledger común aun con actores deshonestos
```

Este tipo de escenario se relaciona con **fallas bizantinas**: los nodos no solo pueden caerse, sino también mentir, enviar mensajes contradictorios, incumplir el protocolo o actuar en beneficio propio.

Bitcoin es anterior a Raft. El paper original fue publicado en 2008 por Satoshi Nakamoto, mientras que Raft apareció varios años después. Aunque en la explicación se puede comparar Bitcoin con Raft, históricamente Bitcoin no nace como una extensión de Raft, sino como una solución propia para una red de pagos abierta y sin confianza centralizada.

## Bitcoin y Ethereum

Bitcoin es una red de pagos. El estado conceptual de la red puede pensarse como un gran libro mayor que indica quién tiene cuánto.

```text
Identidad        Balance
---------        -------
A                10 BTC
B                3 BTC
C                0.5 BTC
```

Ethereum es más general: no solo mantiene balances, sino que permite ejecutar programas distribuidos llamados **smart contracts**. En ese sentido, Ethereum funciona como una computadora distribuida programable, mientras que Bitcoin tiene un modelo más acotado y enfocado en transferencias de valor.

Bitcoin resulta más simple para introducir los conceptos de consenso, proof of work, forks, reorganizaciones y UTXOs.

## Motivación: pagos sin intermediarios de confianza

Los pagos electrónicos tradicionales dependen de instituciones financieras o plataformas centralizadas. Eso introduce varias propiedades prácticas:

```text
Usuario ──▶ intermediario financiero ──▶ receptor
```

El intermediario procesa el pago, valida identidades, puede revertir operaciones, cobra comisiones y puede bloquear cuentas o transacciones.

El paper original de Bitcoin parte de una crítica a este modelo: el comercio electrónico depende casi por completo de terceros confiables para procesar pagos. Aunque el sistema funciona en muchos casos, tiene costos y limitaciones.

### Problemas del modelo centralizado

#### Confianza obligatoria

Al pagar con una plataforma tradicional, se confía en que esa plataforma procese correctamente el pago, proteja los datos, no bloquee injustificadamente la cuenta y no revierta operaciones sin motivo válido.

```text
confianza en el intermediario
  ├── custodia o procesa información sensible
  ├── decide si una operación pasa o no
  └── puede bloquear cuentas o pagos
```

#### Costos y comisiones

Las plataformas de pago pueden cobrar comisiones fijas o proporcionales. En compras pequeñas una comisión fija puede ser cara. En transferencias grandes o internacionales, las comisiones proporcionales pueden volverse relevantes.

El costo visible para el comprador no siempre refleja el costo total. Muchas comisiones aparecen del lado del vendedor o del receptor del pago.

#### Pagos reversibles

En sistemas tradicionales, los pagos pueden revertirse. Eso puede ser útil para proteger al comprador, pero también implica que el vendedor nunca tiene una certeza absoluta inmediata de que el pago quedó finalizado.

```text
pago aprobado
  │
  ├── puede ser desconocido
  ├── puede entrar en disputa
  └── puede revertirse
```

Bitcoin busca pagos que, una vez suficientemente confirmados, sean prácticamente irreversibles.

#### Censura y bloqueo

Una plataforma centralizada puede cerrar cuentas, bloquear pagos o impedir que ciertos usuarios operen. No hace falta que sea censura estatal: puede ser una decisión privada de una empresa, un error de compliance o una política interna.

Bitcoin apunta a eliminar el punto único donde alguien puede decidir unilateralmente que una cuenta no puede transaccionar.

## Objetivos de una red de pagos descentralizada

Una red como Bitcoin busca tres propiedades principales.

| Propiedad | Idea |
|---|---|
| Permissionless | Cualquiera puede unirse a la red |
| Descentralizada | Cualquiera puede emitir transferencias válidas |
| Tamper-proof | No se puede falsificar la historia de pagos |

### Permissionless

No hay una autoridad que autorice a participar.

```text
levantar nodo
conectarse a la red
empezar a recibir y propagar transacciones
```

Esto contrasta con Raft, donde los nodos del cluster están definidos y administrados por una entidad que controla el sistema.

### Descentralizada

No existe una entidad central que procese todos los pagos.

```text
sin Stripe
sin Mercado Pago
sin banco central del sistema
sin servidor líder fijo
```

Cualquier usuario puede emitir una transacción y cualquier nodo puede recibirla y propagarla.

### Tamper-proof

La historia no debe poder falsificarse. En Bitcoin, esto se reduce principalmente a impedir el **double spend**.

Un double spend ocurre cuando alguien intenta gastar los mismos fondos dos veces.

```text
Jim tiene 10 BTC

Jim ── 5 BTC ──▶ Dwight
Jim ── 6 BTC ──▶ Darrel

Total intentado: 11 BTC
Saldo real: 10 BTC
```

El sistema debe aceptar una de esas transacciones y rechazar la otra.

## Ledger distribuido con Raft

Para entender el problema, primero se puede imaginar un ledger implementado con Raft.

```text
Jim = 10
Dwight = 20
Darrel = 1
```

Jim envía 5 BTC a Dwight.

```text
Tx(Jim, Dwight, 5)
```

En Raft, la transacción llega al líder. Si llega a un follower, este la redirige o la transmite al líder. El líder ordena la operación en su log.

```text
Cliente ── Tx ──▶ Nodo cualquiera ──▶ Líder
                                      │
                                      ▼
                                  Log ordenado
```

El líder aplica la transacción al estado materializado:

```text
Antes:
Jim = 10
Dwight = 20
Darrel = 1

Después:
Jim = 5
Dwight = 25
Darrel = 1
```

Si Jim intenta luego enviar 6 BTC a Darrel, el líder mira el estado actual y rechaza la transacción porque Jim solo tiene 5 BTC.

```text
Tx(Jim, Darrel, 6) ──▶ rechazada
```

Raft evita el double spend porque tiene un **líder centralizado** que impone un orden total sobre las operaciones.

```text
orden total del líder
        │
        ▼
una transacción se aplica antes que la otra
        │
        ▼
la segunda puede validarse contra el estado actualizado
```

## Entorno bizantino

En Bitcoin nadie confía en nadie.

No se puede asumir que los nodos sean honestos, que sigan el protocolo, que no envíen información contradictoria o que no intenten atacar la red.

```text
Raft
  ├── nodos conocidos
  ├── código controlado
  ├── fallas no maliciosas
  └── líder elegido por protocolo interno

Bitcoin
  ├── nodos desconocidos
  ├── cualquiera puede participar
  ├── actores potencialmente maliciosos
  └── no hay líder fijo
```

La red debe seguir funcionando si la mayoría del poder relevante del sistema actúa honestamente.

## Identidad sin autenticación centralizada

En una red centralizada, un usuario se autentica con usuario, contraseña, token, sesión o certificado emitido por una autoridad.

En Bitcoin no hay autenticación centralizada. La identidad se basa en criptografía de clave pública.

Se usan dos claves:

| Clave | Uso |
|---|---|
| Clave privada | Firmar mensajes |
| Clave pública | Verificar firmas |

La clave privada solo la conoce su dueño. La clave pública puede conocerla toda la red.

```text
firma = Sig(mensaje, clave_privada)

Verify(mensaje, firma, clave_publica) = true / false
```

Una transacción ya no es simplemente:

```text
Bob le manda 5 BTC a Alice
```

Sino:

```text
Bob emite un mensaje firmado que transfiere 5 BTC
hacia la clave pública de Alice
```

Ejemplo conceptual:

```text
TX:
  dst = pubkey_alice
  amount = 5 BTC
  signature = Sig(enc(dst) + enc(amount), secret_key_bob)
```

Cualquier nodo puede verificar que la transacción fue firmada por quien posee la clave privada correspondiente.

## Replay attack

Si la firma solo dependiera del destinatario y del monto, Alice podría tomar una transacción válida recibida de Bob y reenviarla muchas veces.

```text
Bob ── 5 BTC ──▶ Alice

Alice reenvía la misma TX
Alice reenvía la misma TX
Alice reenvía la misma TX
```

La firma seguiría siendo válida porque el contenido firmado sería exactamente el mismo.

Para evitarlo, la transacción debe incluir algo que la haga única. Una forma inicial de pensarlo es incluir el hash de una transacción previa.

```text
TX:
  prev_tx_hash
  dst = pubkey_alice
  amount = 5 BTC
  signature = Sig(enc(prev_tx_hash) + enc(dst) + enc(amount), secret_key_bob)
```

Si cambia el `prev_tx_hash`, cambia el mensaje firmado. Alice no puede generar una nueva firma válida porque no tiene la clave privada de Bob.

Más adelante aparece el modelo real de Bitcoin: las transacciones no referencian simplemente “la transacción anterior”, sino **outputs no gastados** de transacciones previas.

## Hashes criptográficos

Bitcoin usa hashes criptográficos como SHA-256.

Un hash criptográfico toma un input binario y produce un output de tamaño fijo.

```text
hash(a) = h_a
```

Sus propiedades importantes son:

| Propiedad | Idea |
|---|---|
| Determinismo | El mismo input siempre produce el mismo hash |
| Resistencia a preimagen | Dado `h`, es impracticable encontrar `a` tal que `hash(a)=h` |
| Efecto avalancha | Cambiar un bit del input cambia completamente el output |
| Identificador compacto | Un objeto grande puede representarse por su hash |

Si una transacción cambia aunque sea en un bit, su hash cambia completamente.

```text
TX original ── hash ──▶ h1
TX modificada ── hash ──▶ h2 completamente distinto
```

Esto permite usar hashes como identificadores de transacciones y bloques.

## Bloques

Las transacciones no se procesan de a una, sino agrupadas en bloques.

Agrupar transacciones reduce overhead y permite trabajar en lotes.

```text
Bloque 1
  ├── TX1
  ├── TX2
  └── TX3

Bloque 2
  ├── hash(Bloque 1)
  ├── TX4
  ├── TX5
  └── TX6
```

Cada bloque incluye el hash del bloque anterior. Eso crea una cadena de bloques:

```text
Bloque 1 ──hash──▶ Bloque 2 ──hash──▶ Bloque 3 ──hash──▶ ...
```

Si alguien modifica una transacción en un bloque viejo, cambia el hash de ese bloque. Entonces también cambia el contenido del bloque siguiente, porque el bloque siguiente contiene el hash anterior. La modificación se propaga hacia adelante.

```text
modificar Bloque 1
  │
  ├── cambia hash(Bloque 1)
  ├── invalida Bloque 2
  ├── invalida Bloque 3
  └── obliga a reconstruir toda la cadena posterior
```

## Gossip de transacciones

En Bitcoin no se envía una transacción a todos los nodos directamente. Eso sería demasiado costoso en ancho de banda.

Se usa un mecanismo de **gossip**:

```text
Nodo recibe TX
  │
  ├── la guarda en su buffer local
  ├── la reenvía a algunos vecinos
  └── esos vecinos hacen lo mismo
```

Eventualmente, la transacción se propaga por la red.

Cada nodo mantiene un conjunto local de transacciones pendientes, es decir, transacciones que conoce pero que todavía no fueron incluidas en un bloque.

```text
transacciones pendientes
  ├── conocidas por el nodo
  ├── todavía no están en la blockchain
  └── pueden ser incluidas por un minero en un bloque futuro
```

## Double spend sin líder

Sin líder centralizado aparece el problema de orden.

Jim puede enviar dos transacciones incompatibles a distintas partes de la red:

```text
TX1: Jim ── 5 BTC ──▶ Dwight
TX2: Jim ── 6 BTC ──▶ Darrel
```

Si Jim tiene 10 BTC, no puede ejecutar ambas.

Pero si TX1 llega primero a una parte de la red y TX2 llega primero a otra parte, cada grupo puede creer que la transacción que vio primero es la válida.

```text
Parte A de la red:
  aplica TX1
  Jim = 5
  Dwight = 25

Parte B de la red:
  aplica TX2
  Jim = 4
  Darrel = 7
```

Cuando luego se propagan ambas, cada parte rechaza la otra porque ya quedó incompatible con su estado local.

Resultado:

```text
red inconsistente
```

Lo que se perdió es el **orden total** que en Raft daba el líder.

```text
problema central de Bitcoin:
  elegir un orden común de transacciones
  sin líder fijo
  sin identidades confiables
  en una red abierta
```

## Por qué no alcanza con votar

Una primera idea sería votar.

```text
1 nodo = 1 voto
```

Pero en una red permissionless eso no funciona, porque cualquiera puede crear muchas identidades.

```text
Sybil attack:
  crear miles o millones de claves públicas
  aparentar ser muchos participantes
  controlar la votación
```

Otra idea sería:

```text
1 IP = 1 voto
```

Tampoco sirve, porque es posible conseguir muchas direcciones IP o controlar muchas máquinas.

Bitcoin necesita una medida de participación más difícil de falsificar.

## Proof of Work

La solución de Bitcoin es **Proof of Work**.

La idea general es exigir una prueba de que se gastó poder de cómputo.

```text
para proponer un bloque
hay que resolver un problema costoso
```

La idea tiene antecedentes en trabajos de Cynthia Dwork y Moni Naor sobre mecanismos anti-spam para correo electrónico. Para enviar un mensaje, el emisor debía resolver un problema computacional: barato de verificar, pero caro de producir.

Bitcoin reutiliza esa idea para elegir quién puede proponer el próximo bloque.

### Problema difícil de generar y fácil de verificar

Buscar una preimagen exacta sería impracticable:

```text
encontrar x tal que hash(x) = 0
```

Se usa una versión relajada:

```text
encontrar x tal que hash(x) < N
```

En Bitcoin, `x` puede pensarse como el bloque completo más un campo variable llamado `nonce`.

```text
Bloque:
  ├── hash del bloque anterior
  ├── transacciones
  └── nonce
```

El minero prueba distintos valores de `nonce` hasta encontrar uno que haga que el hash del bloque cumpla la condición de dificultad.

```text
while hash(bloque + nonce) >= target:
    nonce += 1
```

Generar la prueba es lento porque implica probar muchísimos hashes. Verificarla es fácil: alcanza con hashear el bloque una vez y chequear que cumple la condición.

```text
minero:
  prueba millones de nonces

verificador:
  hash(bloque)
  chequea leading zeros / target
```

## Elección de líder

En Bitcoin el líder no se elige por votación ni por un algoritmo de elección clásico.

```text
el líder aparece
```

El primer nodo que encuentra una prueba de trabajo válida propaga su bloque. Los demás verifican la prueba y, si el bloque es válido, empiezan a trabajar sobre él.

```text
Miner encuentra bloque válido
  │
  ├── lo propaga por gossip
  ├── otros nodos verifican PoW
  └── si es válido, construyen encima
```

Esto reemplaza la elección de líder de Raft por una competencia probabilística basada en poder de cómputo.

```text
más poder de cómputo
  │
  ▼
mayor probabilidad de proponer el próximo bloque
```

## Mineros

Los nodos que intentan construir bloques válidos se llaman **mineros**.

Un minero:

```text
1. Recibe transacciones pendientes.
2. Arma un bloque candidato.
3. Prueba nonces hasta cumplir la dificultad.
4. Propaga el bloque si encuentra una solución.
```

Los mineros tienen incentivos económicos para incluir transacciones y proponer bloques válidos.

Si proponen un bloque inválido, el resto de la red lo ignora.

## Forks

Puede pasar que dos mineros encuentren bloques válidos casi al mismo tiempo.

```text
Bloque 1
  ├── Bloque 2A
  └── Bloque 2B
```

Ambos bloques pueden ser válidos, pero representan historias distintas. Algunos nodos recibirán primero `Bloque 2A`, otros recibirán primero `Bloque 2B`.

La regla local inicial es simple:

```text
si recibo dos bloques de la misma altura,
trabajo sobre el primero que me llegó
```

Pero eso puede dividir temporalmente a la red.

## Longest chain y reorgs

La regla de fork choice de Bitcoin es seguir la cadena más larga, o más precisamente la cadena con mayor trabajo acumulado.

```text
Bloque 1
  ├── Bloque 2A
  └── Bloque 2B ── Bloque 3
```

Si un nodo estaba trabajando sobre `Bloque 2A` y luego recibe una rama más larga que continúa desde `Bloque 2B`, reorganiza su estado y pasa a trabajar sobre la rama más larga.

Esto se llama **reorg**.

```text
estado anterior:
Bloque 1 → Bloque 2A

estado nuevo:
Bloque 1 → Bloque 2B → Bloque 3
```

Un reorg cambia historia reciente. Por eso Bitcoin prioriza disponibilidad sobre consistencia fuerte inmediata.

En términos CAP:

```text
availability > consistencia fuerte inmediata
```

La consistencia es eventual y probabilística.

## Confirmaciones y finalización probabilística

En Raft, una entrada queda finalizada de forma determinística cuando alcanza las condiciones de commit.

En Bitcoin no hay finalización determinística absoluta. Una transacción se vuelve cada vez más difícil de revertir cuanto más bloques se construyen encima.

```text
Bloque con TX ── Bloque +1 ── Bloque +2 ── Bloque +3 ── ...
```

Cada bloque posterior es una confirmación adicional.

Cuanto más vieja es una transacción, más difícil es reorganizarla, porque para eliminarla habría que construir una cadena alternativa más larga desde antes de esa transacción.

```text
para revertir una TX vieja:
  ├── construir una rama alternativa desde antes de la TX
  ├── resolver PoW para cada bloque de esa rama
  └── superar el trabajo acumulado de la cadena honesta
```

Para un pago chico, puede alcanzar con esperar pocas confirmaciones. Para un pago grande, conviene esperar más confirmaciones.

```text
café       → pocas confirmaciones
inmueble   → muchas confirmaciones
```

## Ataques con reorgs

Un atacante que quiere desconocer una transacción pasada debe construir una cadena alternativa donde esa transacción no exista.

```text
Cadena honesta:
B1 → B2A(TX: Jim paga a Dwight) → B3 → B4

Cadena atacante:
B1 → B2B(sin esa TX) → ...
```

Para que la red acepte la cadena atacante, esa cadena debe superar a la cadena honesta en trabajo acumulado.

Esto exige controlar una fracción enorme del poder de cómputo. Mientras la mayoría del poder de cómputo siga construyendo sobre la cadena honesta, la cadena atacante queda atrás.

```text
seguridad de Bitcoin ≈ costo de superar el trabajo acumulado de la red
```

## Dificultad

La dificultad de Proof of Work determina qué tan difícil es encontrar un bloque válido.

Una forma intuitiva de pensarlo es la cantidad de ceros iniciales requeridos en el hash.

```text
más ceros requeridos
  │
  ▼
menor probabilidad de éxito por intento
  │
  ▼
más trabajo esperado
```

Bitcoin ajusta la dificultad de manera determinística para mantener un ritmo aproximado de bloques.

El objetivo es que se genere un bloque aproximadamente cada 10 minutos.

```text
si los bloques aparecen demasiado rápido:
  aumenta la dificultad

si los bloques aparecen demasiado lento:
  baja la dificultad
```

Esto permite que la red se adapte al aumento del poder de cómputo y al hardware especializado.

Al principio se podía minar con CPU. Luego se usaron GPU. Después aparecieron dispositivos especializados para calcular hashes a gran velocidad.

```text
CPU → GPU → hardware especializado para minería
```

## Emisión de moneda

En monedas tradicionales, un banco central emite dinero.

En Bitcoin, la emisión ocurre cuando se mina un bloque.

El minero incluye una transacción especial llamada **coinbase transaction**, donde se crea nueva moneda y se envía a sí mismo la recompensa permitida por el protocolo.

```text
Miner encuentra bloque válido
  │
  ├── incluye coinbase transaction
  ├── recibe recompensa
  └── puede incluir fees de transacciones
```

La recompensa no es arbitraria. Está definida por el protocolo. Si un minero intenta asignarse más de lo permitido, los demás nodos ignoran el bloque.

```text
bloque con reward inválida ──▶ descartado
```

## Halving

La recompensa por bloque se reduce periódicamente en un evento llamado **halving**.

Cada cierto intervalo de bloques, aproximadamente cada 4 años, la recompensa baja a la mitad.

```text
reward inicial
  ↓
reward / 2
  ↓
reward / 4
  ↓
...
```

Eventualmente, alrededor del año 2140, ya no se emitirán nuevos BTC. A partir de ahí, los incentivos para los mineros dependerán de las comisiones de transacción.

La recompensa cumple dos funciones:

```text
1. Emisión y distribución de moneda.
2. Incentivo económico para asegurar la red.
```

## UTXO

Bitcoin no guarda el estado global como un balance por cuenta al estilo:

```text
Alice = 10 BTC
Bob = 5 BTC
```

El modelo real es **UTXO**: *Unspent Transaction Outputs*.

Un UTXO es una salida de una transacción previa que todavía no fue gastada.

```text
Transacción previa:
  output: Alice recibe 3 BTC

Mientras Alice no lo use como input en otra transacción,
es un UTXO.
```

Para gastar, Alice arma una nueva transacción que referencia outputs no gastados previos.

```text
Inputs:
  UTXO1: Alice recibió 2 BTC
  UTXO2: Alice recibió 3 BTC

Output:
  Bob recibe 5 BTC
```

La propia transacción contiene la prueba de que Alice tiene fondos suficientes, porque referencia los outputs previos que le pertenecen.

## Cambio en UTXO

Los UTXOs se gastan enteros. No se puede gastar “la mitad” de un UTXO.

Si Alice tiene un UTXO de 5 BTC y quiere enviar 4 BTC a Bob, la transacción consume el UTXO completo y genera dos outputs:

```text
Input:
  UTXO de Alice: 5 BTC

Outputs:
  Bob: 4 BTC
  Alice: 1 BTC  (cambio)
```

Es parecido a pagar con un billete y recibir vuelto.

```text
pagás con 5
comprás por 4
recibís 1 de cambio
```

En Bitcoin, el “cambio” es otro output que vuelve a una dirección controlada por Alice.

## Ventajas del modelo UTXO

### Unicidad de transacciones

Como una transacción referencia outputs previos específicos, el contenido queda naturalmente ligado a inputs únicos. Esto ayuda a evitar replay attacks.

```text
TX nueva = inputs previos + outputs nuevos + firma
```

Si cambian los inputs, cambia la transacción.

### Estado inmutable

En lugar de modificar balances globales, Bitcoin marca outputs como gastados o no gastados.

```text
UTXO disponible ── usado como input ──▶ gastado
```

Esto convierte el estado en un conjunto de registros inmutables más una marca de disponibilidad.

```text
operaciones principales:
  ├── agregar nuevos outputs
  └── invalidar outputs gastados
```

### Reorgs más simples

Con UTXOs, un reorg implica cambiar qué outputs se consideran gastados según la rama activa.

```text
rama A:
  UTXO X gastado
  UTXO Y disponible

rama B:
  UTXO X disponible
  UTXO Y gastado
```

Esto es más simple que deshacer una secuencia compleja de modificaciones sobre balances globales.

### Paralelismo

Los UTXOs son independientes. Si dos transacciones consumen UTXOs distintos, pueden validarse en paralelo.

```text
TX1 consume UTXO A
TX2 consume UTXO B

A y B independientes → validación paralelizable
```

En un modelo de cuenta global, varias transacciones sobre la misma cuenta suelen requerir orden secuencial.

### Light clients

El modelo UTXO facilita clientes livianos. Un cliente puede seguir solamente los outputs que le pertenecen, sin mantener todo el estado global.

```text
Light client
  ├── escucha bloques
  ├── detecta outputs propios
  ├── guarda sus UTXOs
  └── arma transacciones con esos UTXOs
```

No aporta la misma seguridad que un nodo completo, pero permite operar sin almacenar toda la historia y todo el estado de la red.

## Comparación con Ethereum

Bitcoin usa UTXOs y está orientado a pagos.

Ethereum usa un modelo de cuentas y ejecuta smart contracts. Su estado se parece más a una computadora distribuida con memoria modificable.

```text
Bitcoin
  ├── red de pagos
  ├── UTXOs
  ├── scripting limitado
  └── finalización probabilística por PoW + longest chain

Ethereum
  ├── computadora distribuida
  ├── cuentas y contratos
  ├── máquina virtual general
  └── consenso más complejo con proof of stake y finalización
```

En Ethereum, las reorganizaciones y la ejecución paralela son más complejas porque hay estado global mutable. En Bitcoin, el modelo UTXO simplifica esas operaciones.

## Finalidad en Bitcoin y Ethereum

Bitcoin tiene finalización probabilística.

```text
más confirmaciones → menor probabilidad de reorg
```

Teóricamente, una cadena podría reorganizarse desde muy atrás si un atacante consigue superar el trabajo acumulado de la cadena principal. En la práctica, la probabilidad baja rápidamente a medida que se acumulan confirmaciones.

Ethereum, en cambio, usa mecanismos de votación y finalización propios de proof of stake. Una vez que un bloque queda finalizado bajo esas reglas, la garantía es más fuerte que la de Bitcoin.

## Repositorios, protocolo e implementaciones

Bitcoin no es un único programa controlado por una única persona. Es un protocolo.

Como ocurre con TCP/IP, pueden existir múltiples implementaciones mientras respeten las reglas del protocolo.

```text
protocolo Bitcoin
  ├── implementación A
  ├── implementación B
  ├── implementación C
  └── nodo propio escrito desde cero
```

Un nodo válido es cualquier implementación que pueda conectarse a la red, recibir bloques, validar reglas, propagar transacciones y seguir el consenso.

Las actualizaciones del protocolo no se deciden de forma puramente automática. Hay discusiones, implementadores, comunidades, propuestas y adopción gradual. En sistemas descentralizados, la dimensión social y política no desaparece: aparece en cómo se coordinan cambios de protocolo.

## Trabajo en blockchain

Hay distintos niveles de trabajo técnico en blockchain.

### Infraestructura

Implementación de nodos, clientes de ejecución, clientes de consenso, redes peer-to-peer, bases de datos internas, sincronización, propagación de bloques y optimización de rendimiento.

```text
infraestructura
  ├── nodos
  ├── consenso
  ├── ejecución
  ├── networking
  └── almacenamiento
```

### Smart contracts

Desarrollo de programas que corren sobre redes como Ethereum.

Ejemplos:

```text
exchanges descentralizados
stablecoins
NFTs
escrow
sistemas de propiedad digital
juegos
```

Un contrato puede actuar como intermediario programable. Por ejemplo, un contrato de escrow recibe recursos de dos partes y los libera bajo reglas definidas por código.

### Criptografía

Desarrollo de primitivas criptográficas para privacidad, pruebas de conocimiento cero, verificación eficiente, identidad y cómputo verificable.

```text
zero knowledge proofs
fully homomorphic encryption
privacidad de datos
votación verificable
identidad descentralizada
```

### Aplicaciones

Construcción de productos sobre redes existentes: wallets, plataformas de pagos, sistemas de tickets, herramientas de auditoría, exploradores de bloques, infraestructura para desarrolladores y más.

## Testnets y desarrollo

Para experimentar no hace falta usar dinero real.

Existen **testnets**, redes de prueba con monedas sin valor económico real.

```text
testnet
  ├── permite probar contratos
  ├── permite levantar nodos
  ├── permite enviar transacciones
  └── evita gastar fondos reales
```

También es posible levantar una red local o usar wallets que se conectan a nodos de terceros.

Una wallet no necesariamente es un nodo completo. Muchas wallets envían transacciones a un gateway o proveedor de nodos, lo que mejora la usabilidad pero introduce cierto grado de centralización práctica.

## Resumen conceptual

Bitcoin resuelve una red de pagos distribuida sin confianza centralizada combinando varias piezas.

```text
Identidad
  └── claves públicas y firmas digitales

Tamper-proofing
  ├── hashes criptográficos
  ├── transacciones identificables
  └── bloques encadenados por hash

Orden
  ├── Proof of Work
  ├── líder probabilístico
  └── longest chain

Consistencia
  ├── eventual
  ├── reorgs posibles
  └── finalidad probabilística

Estado
  └── UTXOs en lugar de balances globales
```

La diferencia central con Raft es que Raft parte de un cluster controlado, con nodos conocidos y un líder elegido explícitamente. Bitcoin parte de una red abierta, sin confianza, con nodos potencialmente maliciosos. Por eso reemplaza la autoridad del líder por una competencia de proof of work y reemplaza la finalización determinística por una seguridad probabilística basada en trabajo acumulado.

## Bibliografía mencionada

- Bitcoin whitepaper: `https://bitcoin.org/bitcoin.pdf`
- How Bitcoin Works Under the Hood: `https://www.youtube.com/watch?v=Lx9zgZCMqXE`
- Bitcoin Developer Guide — Transactions: `https://developer.bitcoin.org/devguide/transactions.html`
- 3Blue1Brown — How does Bitcoin actually work?: `https://www.youtube.com/watch?v=bBC-nXj3Ng4`
- Blockchain 101 — Visual Demo: `https://www.youtube.com/watch?v=_160oMzblY8`
- MIT lecture on Bitcoin: `https://www.youtube.com/watch?v=K_euhRou98Y`
- Programming Bitcoin, Jimmy Song: `https://www.oreilly.com/library/view/programming-bitcoin/9781492031482/`
- Mastering Bitcoin, Andreas M. Antonopoulos and David A. Harding: `https://github.com/bitcoinbook/bitcoinbook/blob/develop/BOOK.md`
