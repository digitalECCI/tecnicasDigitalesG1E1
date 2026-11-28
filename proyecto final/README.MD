# Informe de Proyecto Final: Enlace de Comunicación Serial UART entre FPGA (DE10-Lite) y PC en Verilog

## Integrantes
- [Andres Mateo Arias Aguilera](https://github.com/mateoaeora124)
- [Gabriel Cangrejo](https://github.com/gabriel11cangrejo)
- [Cesar Alberto Gomez](https://github.com/Cesar7772026)

---

## 1. Introducción y Fundamentos Teóricos

Este proyecto final cubre el diseño, la simulación y la síntesis en hardware, con Verilog HDL, de un **enlace de comunicación serial UART bidireccional** entre la tarjeta Terasic **DE10-Lite (FPGA Intel MAX 10, 10M50DAF484C7G)** y un PC con la terminal **PuTTY**. El sistema tiene estas partes:

- **Sentido FPGA → PC (transmisión):** el usuario fija un número de **0 a 1023** en los switches `SW9..SW0` y presiona **KEY0**. La FPGA convierte el valor a texto decimal y lo envía por UART; el número aparece impreso en PuTTY (por ejemplo `1023`).
- **Sentido PC → FPGA (recepción):** el usuario escribe un número de 0 a 1023 en PuTTY y presiona **Enter**. La FPGA recibe los dígitos, los convierte a binario y muestra el valor en los **10 LEDs** `LEDR9..LEDR0`.
- Un **generador de baudios** que reemplaza al reloj compartido (9600 baudios, formato **8-N-1**).
- **Conversión binario ↔ decimal** en los dos sentidos: `bin2dec` (Double Dabble) para transmitir y `dec2bin` (acumulación `valor·10 + dígito`) para recibir.
- **Retroalimentación:** la FPGA hace **eco** de lo que se escribe en PuTTY, responde `ERR` si el número es inválido, muestra en `HEX3..HEX0` el valor de los switches en decimal y `Er` en `HEX5..HEX4` cuando la última línea recibida fue inválida.

La correspondencia de recursos es directa: la DE10-Lite tiene 10 switches y 10 LEDs, es decir **10 bits**, que cubren exactamente el rango $0$ a $2^{10}-1 = 1023$.

```
        PC (PuTTY)                               FPGA (DE10-Lite)
      +------------+                     +----------------------------------+
      |            | -- TX ---> RX ----->| uart_rx -> dec2bin -> LEDR9..0   |
      |  Terminal  |   (USB-TTL 3.3 V)   |               |  (eco)           |
      |   serial   | <-- RX <--- TX -----| uart_tx <- ctrl_tx <- SW9..0+KEY0|
      +------------+                     +----------------------------------+
```

A diferencia de la caja fuerte (laboratorio anterior), donde todo ocurría dentro de la tarjeta, aquí el sistema tiene que **ponerse de acuerdo con otro equipo** que no comparte su reloj. Todo el diseño gira alrededor de esa idea.

---

### 1.1 Comunicación Serial vs. Paralela

| | Paralelo | Serial |
|---|---|---|
| Cómo viajan los bits | Varios a la vez, **una línea por bit** | **Uno tras otro** por una sola línea |
| Velocidad | Rápido en distancias cortas | Menor por línea, pero escala a distancia |
| Cableado | Muchos cables; problemas de sincronía (*skew*) a distancia | Pocos cables; ideal a distancia |
| Hardware | Directo | Requiere **serializar y deserializar** (registros de desplazamiento) |

En el tablero se dibujó la diferencia: el byte `1001 1100` en paralelo necesita **×8 líneas** (un registro las presenta todas a la vez), mientras que en serie viaja por **×1 línea**, desplazándose bit a bit dentro de un registro. **La UART es serial**, por eso el **registro de desplazamiento** es su pieza central: en `uart_tx.v` convierte paralelo → serie y en `uart_rx.v` serie → paralelo.

```
 Paralelo (x8 líneas)                  Serial (x1 línea)
  +-----------------+                   +---+---+---+---+---+---+---+---+
  |1 0 0 1 1 1 0 0  |                   | 1 | 0 | 0 | 1 | 1 | 1 | 0 | 0 |---> línea TX
  +-+-+-+-+-+-+-+-+-+                   +---+---+---+---+---+---+---+---+
    | | | | | | | |                       registro de desplazamiento: sale un bit por tick
    v v v v v v v v
```

---

### 1.2 Modos de Transmisión y Sincronía

**Dirección del flujo:**
- **Simplex:** un solo sentido (ej.: un sensor que solo transmite).
- **Half-duplex:** ambos sentidos, pero no a la vez (se turnan la línea).
- **Full-duplex:** ambos sentidos **simultáneamente**.

**La UART es full-duplex**: usa dos líneas independientes, `TX` y `RX`, así que puede enviar y recibir al mismo tiempo. En este diseño `uart_tx` y `uart_rx` son bloques totalmente separados, y el testbench lo comprueba presionando KEY0 mientras se escribe en PuTTY (sección 3.1).

**Síncrono vs. asíncrono:**
- **Síncrono** (SPI, I²C): emisor y receptor comparten una **línea de reloj**; el receptor sabe exactamente cuándo muestrear.
- **Asíncrono** (UART): **no hay reloj compartido**. Ambos acuerdan de antemano una **velocidad (baud rate)** y un formato de trama.

| | UART | SPI | I²C |
|---|---|---|---|
| Sincronía | Asíncrono | Síncrono | Síncrono |
| Líneas | 2 (TX, RX) | 4 (SCLK, MOSI, MISO, SS) | 2 (SDA, SCL) |
| Reloj | No comparte | Compartido | Compartido |
| Dispositivos | Punto a punto | 1 maestro / N esclavos | Multi-dispositivo (direcciones) |
| Velocidad | Baja–media | Alta | Media |

> **La pregunta clave de la UART:** sin un cable de reloj, ¿cómo sabe el receptor dónde empieza y termina cada bit? La respuesta es toda la gracia del protocolo: **la trama** (start/stop) marca el inicio y el fin, y **el baud rate acordado** dice cuánto dura cada bit.

---

### 1.3 La Trama UART (8-N-1)

La línea en reposo (*idle*) se mantiene en **nivel alto (`1`)**. Cada byte se "enmarca" con bits especiales:

| Campo | Descripción |
|---|---|
| Idle | Línea en reposo, nivel alto (`1`) |
| Start | 1 bit en bajo (`0`): avisa que empieza un dato |
| Datos | 5 a 9 bits (típico 8), **LSB primero** |
| Paridad | Opcional: 1 bit para detección de error |
| Stop | 1 o 2 bits en alto (`1`): cierre de trama |

El formato elegido es el típico **8-N-1**: 8 bits de datos, sin paridad (*None*), 1 bit de stop. Cada carácter ocupa **10 bits** en la línea.

Ejemplo de clase: enviar el byte `0x41` (`'A'` = `0100 0001`), **LSB primero**:

```
          IDLE START  D0  D1  D2  D3  D4  D5  D6  D7  STOP IDLE
   1 ─────────┐   ┌───┐                       ┌───┐   ┌────────────
              │   │   │                       │   │   │
   0          └───┘   └───────────────────────┘   └───┘
                0   1   0   0   0   0   0   1   0   1
```

En el tablero también se trabajó un ejemplo **con paridad par**: para el dato `1010 1110` hay cinco unos, así que con paridad par se agrega `P = 1` para que el total sea par (y `P = 0` cuando el número de unos ya es par). En este proyecto se usa **sin paridad**, que es la configuración por defecto de PuTTY y de casi todos los adaptadores; la detección de errores se hace con el bit de stop (*framing error*) y validando que los caracteres sean dígitos.

---

### 1.4 Baud Rate y Generador de Baudios

El **baud rate** es el número de símbolos por segundo. En UART cada símbolo es un bit, así que la duración de un bit es:

$$T_{bit} = \frac{1}{\text{baud}} = \frac{1}{9600} \approx 104{,}17\ \mu s$$

Valores típicos: 9600, 19200, 115200 baudios. Emisor y receptor **deben** usar el mismo baud rate: es el "acuerdo" que reemplaza al reloj compartido.

Igual que en el cronómetro y el semáforo, un **divisor** cuenta ciclos del reloj del sistema para marcar cada bit:

$$N = \frac{f_{clk}}{\text{baud}} = \frac{50 \times 10^6}{9600} = 5208{,}33 \approx 5208 \text{ ciclos por bit}$$

Cada 5208 ciclos se genera un **tick de bit** y se envía un bit. Es exactamente la misma técnica de base de tiempo de los laboratorios anteriores. Con 5208 cuentas el baud rate real es $50\times10^6/5208 = 9600{,}6$ baudios, un error de solo **0,006 %**. El contador necesita **13 bits** ($2^{12} = 4096 < 5208 \le 2^{13} = 8192$).

---

### 1.5 Sobremuestreo 16× en el Receptor

El transmisor solo necesita un tick por bit. El receptor, en cambio, no sabe **cuándo** empezará a llegar la trama, así que muestrea la línea **16 veces por bit** (`tick16`):

$$N_{16} = \frac{50 \times 10^6}{9600 \times 16} = 325{,}5 \approx 326 \text{ ciclos}$$

```
 RX  ‾‾‾‾‾‾‾‾‾‾\_______________________________/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
               |<---------- START ----------->|<------------ D0 ------------->|
 tick16        | 0 1 2 3 4 5 6 7 | 0 1 2 3 ...                         ... 15 |
                                ^                                             ^
                    centro del start (8 ticks):              centro de D0 (16 ticks
                    ¿sigue en 0? -> trama válida             después): se muestrea
```

1. En reposo, el receptor espera el **flanco de bajada** del start.
2. Cuenta **8 ticks16** (medio bit) y revisa que la línea **siga en 0**. Si volvió a 1, era ruido (*glitch*) y vuelve a reposo.
3. Desde el centro del start cuenta **16 ticks16** para caer en el **centro** de cada bit de datos, lejos de los flancos, donde la señal es más estable.
4. Tras D7, muestrea el **stop**: si es `1` la trama es válida; si es `0` hay **error de trama** (*framing error*).

Muestrear en el centro deja un margen de ±½ bit para tolerar diferencias de reloj entre los dos equipos. A lo largo de los 9,5 bits que van del start al centro del stop, el error acumulado debe ser menor a ½ bit, es decir, menos de ~5 % de diferencia de baud rate. Con 326 cuentas el error es de 0,16 %, muy por debajo de ese límite.

---

### 1.6 Representación del Número: 8 bits vs. 10 bits

Un byte UART transporta solo **8 bits** (0–255), pero el rango 0–1023 requiere **10 bits**. En el tablero se hizo la cuenta: con 8 bits el máximo es `FF` = `1111 1111` = **255**. Por eso el número viaja como **texto** (dígitos ASCII), uno por trama:

```
 SW = 1023 = 11_1111_1111 (binario)
        |
     bin2dec (Double Dabble)
        |
   '1'    '0'    '2'    '3'    '\r'   '\n'        <- 6 tramas UART
   0x31   0x30   0x32   0x33   0x0D   0x0A
```

En ASCII los dígitos `'0'..'9'` ocupan los códigos `0x30..0x39`, así que convertir un dígito BCD a ASCII es solo **anteponer `0011`** (sumar `0x30`), y al revés basta con tomar el **nibble bajo** del carácter. La conversión binario ↔ decimal es la parte central del proyecto.

---

### 1.7 Conversión Binario → Decimal: Algoritmo Double Dabble (`bin2dec.v`)

Para enviar el número hay que separar sus dígitos decimales (miles, centenas, decenas, unidades). Dividir entre 10 en hardware es costoso, así que se usa el algoritmo **Double Dabble** ("desplazar y sumar 3"):

1. Se arma un registro `[ mil | cen | dec | uni | binario ]` con las columnas BCD en cero.
2. Se repite **10 veces** (una por bit): si alguna columna BCD vale **5 o más**, se le suma **3**; luego se desplaza todo un bit a la izquierda.
3. Al final, las columnas BCD contienen los dígitos decimales.

¿Por qué sumar 3? Al desplazar se multiplica por 2. Si una columna vale 5 o más, el doble sería ≥ 10 y debería "llevar" a la siguiente columna. Sumar 3 antes de desplazar equivale a sumar 6 después, que es justo la corrección que convierte un valor ≥ 10 en binario a BCD con acarreo.

Ejemplo con `37 = 00_0010_0101`:

| Paso | Operación | Decenas | Unidades | Binario restante |
|---|---|---|---|---|
| 0 | Inicio | 0000 | 0000 | 00_0010_0101 |
| 1–4 | Desplazar ×4 (salen solo ceros) | 0000 | 0000 | 10_0101_0000 |
| 5 | Desplazar | 0000 | 0001 | 00_1010_0000 |
| 6 | Desplazar | 0000 | 0010 | 01_0100_0000 |
| 7 | Desplazar | 0000 | 0100 | 10_1000_0000 |
| 8 | Desplazar | 0000 | 1001 | 01_0000_0000 |
| 9a | Uni = 9 ≥ 5 → +3 | 0000 | **1100** | |
| 9 | Desplazar | 0001 | 1000 | 10_0000_0000 |
| 10a | Uni = 8 ≥ 5 → +3 | 0001 | **1011** | |
| 10 | Desplazar | **0011** | **0111** | → **3** y **7** |

El módulo es **puramente combinacional** (el `for` se desenrolla en 10 etapas de comparadores y sumadores de 4 bits). Además entrega `n_dig`, la cantidad de dígitos significativos, para no enviar ceros a la izquierda (`"37"` y no `"0037"`).

---

### 1.8 Conversión Decimal → Binario: Acumulación (`dec2bin.v`)

En recepción los dígitos llegan **del más significativo al menos significativo**, uno por trama. Se acumulan con:

$$\text{valor} \leftarrow \text{valor} \times 10 + \text{dígito}$$

La multiplicación por 10 no necesita multiplicador: $10x = 8x + 2x$, es decir **dos desplazamientos y una suma** (`{acum,3'b000} + {acum,1'b0}`).

Ejemplo, escribiendo `725` + Enter en PuTTY:

| Carácter | ASCII | Dígito | Acumulador |
|---|---|---|---|
| `'7'` | 0x37 | 7 | 0·10 + 7 = **7** |
| `'2'` | 0x32 | 2 | 7·10 + 2 = **72** |
| `'5'` | 0x35 | 5 | 72·10 + 5 = **725** |
| Enter | 0x0D | — | LEDs ← 725 = `10_1101_0101` |

**Validaciones:**
- Si el acumulado supera **1023** se marca **desborde** y la línea se rechaza.
- Si llega un carácter que no es dígito (letra, espacio, retroceso) o una trama con **framing error**, la línea se marca **inválida**.
- Al presionar Enter: si la línea es válida, los LEDs toman el nuevo valor; si no, **conservan el anterior**, PuTTY muestra `ERR` y los displays `Er`.
- Un Enter sin dígitos no cambia nada (solo salta de línea).

---

### 1.9 Sincronización de Entradas y Antirrebote

Todas las entradas externas son **asíncronas** respecto al reloj de 50 MHz:

1. **Línea RX:** el PC transmite con su propio reloj, así que los flancos de RX pueden caer en cualquier momento. Se pasa por un **sincronizador de 2 flip-flops** dentro de `uart_rx.v` (inicializado en `1`, el nivel de reposo, para no detectar un start falso al salir del reset).
2. **Switches SW9..SW0:** sincronizador de 2 flip-flops en `proyecto_uart.v`, igual que en la caja fuerte.
3. **KEY0:** se reutiliza `antirrebote.v` de la caja fuerte: sincronizador de 2 FF + filtro de **20 ms** + **detector de flanco**. Genera **un solo pulso** de 1 ciclo por pulsación, de modo que el número se transmite **una sola vez** aunque el botón se mantenga presionado o rebote.

Los pulsadores KEY de la DE10-Lite son **activos en bajo**, por eso en el módulo superior se usan `~key_tx` y `rst = ~key_reset`.

---

### 1.10 Máquinas de Estados

Se usan tres FSM, todas escritas con la **arquitectura de tres bloques `always`** de los laboratorios anteriores (registro de estado, transición combinacional y salida Moore).

**FSM del transmisor (`uart_tx.v`):**

```
              tx_start                tick                tick
 +--------+ (carga byte) +----------+ ------> +---------+ ------> +---------+
 | S_IDLE |------------->| S_ESPERA |         | S_START |         | S_DATOS |--+ tick y n_bit < 7:
 | tx = 1 |              |  tx = 1  |         |  tx = 0 |         |tx=shift0|<-+ desplaza, n_bit++
 +--------+              +----------+         +---------+         +---------+
     ^                                                                 |
     |                  tick (tx_done)        +---------+  tick y      |
     +----------------------------------------| S_STOP  |<-------------+
                                              |  tx = 1 |  n_bit == 7
                                              +---------+
```

`S_ESPERA` alinea el inicio del start con un tick del generador de baudios. Sin ese estado, si el tick llegara justo después de `tx_start`, el bit de start quedaría más corto que los demás. La salida `tx` es **registrada** para que el cable nunca vea *glitches*.

**FSM del receptor (`uart_rx.v`):**

```
               rx = 0 (flanco de bajada)
 +--------+ ---------------------------> +---------+
 | S_IDLE |                              | S_START |  cuenta 8 ticks16
 +--------+ <--------------------------- +---------+
     ^       centro del start y rx = 1        |
     |       (era ruido)                      | centro del start y rx = 0
     |                                        v
     |                                   +---------+  cada 16 ticks16: muestrea
     |                                   | S_DATOS |  shift <= {rx, shift[7:1]}
     |                                   +---------+
     |                                        | 8 bits recibidos
     |                                        v
     |     centro del stop:              +---------+
     +---- rx = 1 -> rx_done  -----------| S_STOP  |
           rx = 0 -> rx_err (framing)    +---------+
```

**FSM del secuenciador de mensajes (`ctrl_tx.v`):**

```
         hay petición pendiente
 +--------+ (elige mensaje e idx = 0) +---------+   1 ciclo   +----------+
 | S_IDLE |-------------------------->| S_ENVIA |------------>| S_ESPERA |
 +--------+                           |tx_start |             +----------+
      ^                               +---------+                  |
      |                                    ^   tx_done y no último | 
      |                                    +-----------------------+ idx++
      |          tx_done y último byte                              |
      +-------------------------------------------------------------+
```

`ctrl_tx` arma los mensajes byte a byte a partir de una tabla tipo ROM (como `decod_mensaje.v` en la caja fuerte):

| Mensaje | Bytes | Cuándo |
|---|---|---|
| `M_NUM` | dígitos ASCII (1 a 4) + `\r\n` | Al presionar KEY0 |
| `M_ECO` | el carácter recibido | Al recibir cualquier carácter que no sea Enter |
| `M_CRLF` | `\r\n` | Enter con número válido (o vacío) |
| `M_ERR` | `\r\nERR\r\n` | Enter con número inválido |

Como un mensaje puede llegar mientras otro se está enviando, cada tipo tiene un **registro de petición pendiente**. Cuando el transmisor queda libre se atiende primero el eco (para que lo escrito salga antes del salto de línea) y luego el número de los switches.

---

## 2. Código Hardware HDL en Verilog

El proyecto tiene **9 módulos**. Cada uno está en su propio archivo con la misma convención de nombres y comentarios de los laboratorios anteriores. `antirrebote.v` se reutiliza tal cual de la caja fuerte.

```
                                   proyecto_uart.v
 +-----------------------------------------------------------------------------------------+
 |                         +------------+   tick16   +---------+  rx_data/rx_done/rx_err    |
 |  clk ------------------>|  baud_gen  |----------->| uart_rx |<--- uart_rx (GPIO_1)       |
 |                         | 5208 / 326 |            | FSM+2FF |           |                |
 |                         +------------+            +---------+           v                |
 |                              | tick                                +---------+           |
 |                              v                                     | dec2bin |--> LEDR9..0
 |  +---------+  tx_start  +---------+                                | val*10+d|--> error -> HEX5..4 "Er"
 |  | uart_tx |<-----------| ctrl_tx |<--- eco_car / fin_linea / ----+---------+           |
 |  |FSM+shift|  tx_data   |(bin2dec)|     fin_error                                       |
 |  +---------+  tx_done-->+---------+                                                     |
 |      |                     ^  ^                                                          |
 |      v                     |  +-- enviar <-- antirrebote.v <-- KEY0                      |
 |  uart_tx (GPIO_0)          +----- SW9..0 --> 2FF ---------> bin2dec --> decod_7seg x4 --> HEX3..0
 |                                                                                         |
 |  KEY1 --> rst (a todos los módulos)                                                     |
 +-----------------------------------------------------------------------------------------+
```

| Archivo | Descripción |
|---|---|
| `proyecto_uart.v` | Módulo superior: integra UART, switches, botón, LEDs y displays |
| `baud_gen.v` | Generador de baudios: tick de bit (5208) y tick 16× (326) |
| `uart_tx.v` | Transmisor UART: FSM + registro de desplazamiento (serializa un byte) |
| `uart_rx.v` | Receptor UART: sincronizador + FSM con sobremuestreo 16× (deserializa un byte) |
| `bin2dec.v` | Binario de 10 bits → 4 dígitos BCD/ASCII (Double Dabble) |
| `dec2bin.v` | Dígitos ASCII recibidos → valor de 10 bits (`valor·10 + dígito`) con validación |
| `ctrl_tx.v` | Secuenciador: arma los mensajes (número, eco, `ERR`) y los entrega byte a byte |
| `antirrebote.v` | Sincronizador + antirrebote + detector de flanco (de la caja fuerte) |
| `decod_7seg.v` | Decodificador de 7 segmentos (ánodo común) |
| `tb_proyecto_uart.v` | Testbench del sistema completo, ambos sentidos |
| `tb_uart_tiempo_real.v` | Testbench a 9600 baudios reales (loopback de `'A'`) |

### 2.1 Módulo Principal (`proyecto_uart.v`)

El módulo principal interconecta todos los bloques. Invierte los pulsadores (activos en bajo), sincroniza los switches y reparte los dos sentidos de la comunicación. El parámetro `SIMULATION` se propaga a `antirrebote` y `baud_gen` para acelerar el testbench sin tocar el código sintetizable. Hay **dos instancias de `bin2dec`**: una dentro de `ctrl_tx` (con el valor congelado al presionar KEY0) y otra aquí para mostrar en vivo el valor de los switches en los displays.

```verilog
module proyecto_uart (
    input  wire       clk,          // PIN_P11 (50 MHz)
    input  wire [9:0] sw,           // SW9..SW0: número a transmitir (0..1023)
    input  wire       key_tx,       // PIN_B8 (KEY0 - ENVIAR, Active-Low)
    input  wire       key_reset,    // PIN_A7 (KEY1 - RESET, Active-Low)

    input  wire       uart_rx,      // GPIO_[1] (PIN_W10) <- TX del adaptador USB-TTL
    output wire       uart_tx,      // GPIO_[0] (PIN_V10) -> RX del adaptador USB-TTL

    output wire [9:0] ledr,         // LEDR9..LEDR0: número recibido desde PuTTY (binario)
    output wire [6:0] disp0,        // HEX0 \
    output wire [6:0] disp1,        // HEX1  | Valor de los switches en decimal
    output wire [6:0] disp2,        // HEX2  |
    output wire [6:0] disp3,        // HEX3 /
    output wire [6:0] disp4,        // HEX4 \  "Er" si la última línea recibida
    output wire [6:0] disp5         // HEX5 /  fue inválida
);

    // SIMULATION = 1 acelera el antirrebote y el baud rate para el testbench
    parameter SIMULATION = 0;
    parameter BAUD       = 9600;

    // RESET = KEY1 (activo en bajo)
    wire rst = ~key_reset;

    // ---------------------------------------------------------------------
    // Entradas de usuario
    // ---------------------------------------------------------------------
    // Sincronizador de 2 flip-flops para los 10 switches
    reg [9:0] sw_s0 = 10'd0;
    reg [9:0] sw_s1 = 10'd0;
    always @(posedge clk) begin
        sw_s0 <= sw;
        sw_s1 <= sw_s0;
    end

    // Antirrebote + detector de flanco para KEY0: un solo envío por pulsación
    wire enviar;
    antirrebote #(
        .SIMULATION(SIMULATION)
    ) db_tx (
        .clk     (clk),
        .rst     (rst),
        .entrada (~key_tx),      // KEY0 es activo en bajo
        .nivel   (),
        .subida  (enviar),       // Pulso al presionar
        .bajada  ()
    );

    // ---------------------------------------------------------------------
    // Generador de baudios: tick (1x) para TX y tick16 (16x) para RX
    // ---------------------------------------------------------------------
    wire tick, tick16;
    baud_gen #(
        .BAUD       (BAUD),
        .SIMULATION (SIMULATION)
    ) baud_inst (
        .clk    (clk),
        .rst    (rst),
        .tick   (tick),
        .tick16 (tick16)
    );

    // ---------------------------------------------------------------------
    // Sentido PC -> FPGA:  uart_rx -> dec2bin -> LEDs
    // ---------------------------------------------------------------------
    wire [7:0] rx_data;
    wire       rx_done, rx_err;

    uart_rx rx_inst (
        .clk     (clk),
        .rst     (rst),
        .tick16  (tick16),
        .rx      (uart_rx),       // Se sincroniza con 2 FF dentro del receptor
        .rx_data (rx_data),
        .rx_done (rx_done),
        .rx_err  (rx_err)
    );

    wire [9:0] valor_rx;
    wire       error_rx;
    wire       eco_car, fin_linea, fin_error;

    dec2bin d2b_inst (
        .clk       (clk),
        .rst       (rst),
        .rx_data   (rx_data),
        .rx_done   (rx_done),
        .rx_err    (rx_err),
        .valor     (valor_rx),
        .error     (error_rx),
        .eco_car   (eco_car),
        .fin_linea (fin_linea),
        .fin_error (fin_error)
    );

    assign ledr = valor_rx;

    // ---------------------------------------------------------------------
    // Sentido FPGA -> PC:  switches + KEY0 -> ctrl_tx (bin2dec) -> uart_tx
    // ---------------------------------------------------------------------
    wire       tx_start, tx_done;
    wire [7:0] tx_data;

    ctrl_tx ctrl_inst (
        .clk       (clk),
        .rst       (rst),
        .enviar    (enviar),
        .valor_sw  (sw_s1),
        .eco_car   (eco_car),
        .car       (rx_data),
        .fin_linea (fin_linea),
        .fin_error (fin_error),
        .tx_done   (tx_done),
        .tx_start  (tx_start),
        .tx_data   (tx_data)
    );

    uart_tx tx_inst (
        .clk      (clk),
        .rst      (rst),
        .tick     (tick),
        .tx_start (tx_start),
        .tx_data  (tx_data),
        .tx       (uart_tx),
        .tx_busy  (),
        .tx_done  (tx_done)
    );

    // ---------------------------------------------------------------------
    // Displays: HEX3..HEX0 = switches en decimal, HEX5..HEX4 = "Er"
    // ---------------------------------------------------------------------
    localparam [4:0] C_OFF = 5'h10;
    localparam [4:0] C_E   = 5'h0E;
    localparam [4:0] C_R   = 5'h11;

    wire [3:0] sw_mil, sw_cen, sw_dec, sw_uni;
    wire [2:0] sw_ndig;

    bin2dec conv_disp (
        .bin     (sw_s1),
        .bcd_mil (sw_mil),
        .bcd_cen (sw_cen),
        .bcd_dec (sw_dec),
        .bcd_uni (sw_uni),
        .ascii   (),
        .n_dig   (sw_ndig)
    );

    // Ceros a la izquierda apagados (muestra "  37" y no "0037")
    wire [4:0] c3 = (sw_ndig >= 3'd4) ? {1'b0, sw_mil} : C_OFF;
    wire [4:0] c2 = (sw_ndig >= 3'd3) ? {1'b0, sw_cen} : C_OFF;
    wire [4:0] c1 = (sw_ndig >= 3'd2) ? {1'b0, sw_dec} : C_OFF;
    wire [4:0] c0 = {1'b0, sw_uni};
    wire [4:0] c5 = error_rx ? C_E : C_OFF;
    wire [4:0] c4 = error_rx ? C_R : C_OFF;

    decod_7seg hex0 (.codigo(c0), .seg(disp0));
    decod_7seg hex1 (.codigo(c1), .seg(disp1));
    decod_7seg hex2 (.codigo(c2), .seg(disp2));
    decod_7seg hex3 (.codigo(c3), .seg(disp3));
    decod_7seg hex4 (.codigo(c4), .seg(disp4));
    decod_7seg hex5 (.codigo(c5), .seg(disp5));

endmodule
```

### 2.2 Generador de Baudios (`baud_gen.v`)

Dos contadores independientes sobre el reloj de 50 MHz, como muestra el diagrama de bloques de clase (*Divisor → tick → TX, tick 16× → RX*). Los valores se calculan con redondeo a partir de los parámetros `CLK_FREQ` y `BAUD`, así que cambiar a 115200 baudios es solo cambiar un parámetro.

```verilog
module baud_gen (
    input  wire clk,       // Reloj de 50 MHz
    input  wire rst,       // Reset asíncrono activo en alto
    output reg  tick,      // Pulso de 1 ciclo por cada bit (9600 Hz)    -> Transmisor
    output reg  tick16     // Pulso de 1 ciclo 16 veces por bit (153.6 kHz) -> Receptor
);

    // Igual que en el cronómetro y el semáforo: un divisor cuenta ciclos del
    // reloj del sistema para marcar cada bit.
    parameter CLK_FREQ   = 50_000_000;
    parameter BAUD       = 9600;
    parameter SIMULATION = 0;

    // N = f_clk / baud (redondeado al entero más cercano)
    //   Tick de bit : 50e6 / 9600        = 5208.3 -> 5208 cuentas (104.16 us)
    //   Tick 16x    : 50e6 / (9600 * 16) =  325.5 ->  326 cuentas (6.52 us)
    // En simulación: 1 bit = 32 ciclos y tick16 cada 2 ciclos (misma relación 16:1)
    localparam integer DIV_BIT_REAL = (CLK_FREQ + BAUD / 2) / BAUD;          // 5208
    localparam integer DIV_16_REAL  = (CLK_FREQ + BAUD * 8) / (BAUD * 16);   //  326

    localparam [15:0] DIV_BIT = SIMULATION ? 16'd32 : DIV_BIT_REAL[15:0];
    localparam [15:0] DIV_16  = SIMULATION ? 16'd2  : DIV_16_REAL[15:0];

    reg [15:0] cnt_bit;
    reg [15:0] cnt_16;

    // Divisor 1: tick de bit para el transmisor
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cnt_bit <= 16'd0;
            tick    <= 1'b0;
        end else if (cnt_bit >= DIV_BIT - 1'b1) begin
            cnt_bit <= 16'd0;
            tick    <= 1'b1;
        end else begin
            cnt_bit <= cnt_bit + 1'b1;
            tick    <= 1'b0;
        end
    end

    // Divisor 2: tick de sobremuestreo 16x para el receptor
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            cnt_16 <= 16'd0;
            tick16 <= 1'b0;
        end else if (cnt_16 >= DIV_16 - 1'b1) begin
            cnt_16 <= 16'd0;
            tick16 <= 1'b1;
        end else begin
            cnt_16 <= cnt_16 + 1'b1;
            tick16 <= 1'b0;
        end
    end

endmodule
```

### 2.3 Transmisor UART (`uart_tx.v`)

Es el bloque más sencillo: una FSM de 5 estados carga el byte en paralelo en el registro `shift` y lo desplaza a la derecha en cada tick, de modo que por `shift[0]` sale primero el LSB. `tx_done` avisa a `ctrl_tx` que puede enviar el siguiente byte.

```verilog
module uart_tx (
    input  wire       clk,        // Reloj de 50 MHz
    input  wire       rst,        // Reset asíncrono activo en alto
    input  wire       tick,       // Tick de bit del generador de baudios (1 por bit)
    input  wire       tx_start,   // Pulso: cargar tx_data y empezar a transmitir
    input  wire [7:0] tx_data,    // Byte a transmitir

    output reg        tx,         // Línea serial de salida (reposo = 1)
    output reg        tx_busy,    // 1 mientras hay una trama en curso
    output reg        tx_done     // Pulso de 1 ciclo al terminar el bit de stop
);

    // Trama 8-N-1:  IDLE(1) | START(0) | D0 .. D7 (LSB primero) | STOP(1) | IDLE(1)

    // Codificación explícita de estados (3 bits -> 5 estados)
    localparam S_IDLE   = 3'b000;  // Línea en reposo, esperando tx_start
    localparam S_ESPERA = 3'b001;  // Byte cargado, espera el próximo tick para alinear el start
    localparam S_START  = 3'b010;  // Bit de start (0) durante 1 tick
    localparam S_DATOS  = 3'b011;  // 8 bits de datos, LSB primero
    localparam S_STOP   = 3'b100;  // Bit de stop (1) durante 1 tick

    reg [2:0] estado, prox_estado;
    reg [7:0] shift;               // Registro de desplazamiento (serializador)
    reg [2:0] n_bit;               // Bit de datos actual (0..7)
    reg       tx_comb;             // Valor de la línea según el estado

    // BLOQUE 1: Registro de estado y camino de datos (Secuencial)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            estado  <= S_IDLE;
            shift   <= 8'd0;
            n_bit   <= 3'd0;
            tx      <= 1'b1;       // La línea arranca en reposo (alto)
            tx_done <= 1'b0;
        end else begin
            estado  <= prox_estado;
            tx      <= tx_comb;    // Salida registrada: sin glitches en el cable
            tx_done <= 1'b0;

            case (estado)
                S_IDLE:  if (tx_start) shift <= tx_data;   // Carga paralela del byte
                S_START: n_bit <= 3'd0;
                S_DATOS: if (tick) begin
                             shift <= {1'b0, shift[7:1]};   // Desplaza: sale el LSB
                             n_bit <= n_bit + 1'b1;
                         end
                S_STOP:  if (tick) tx_done <= 1'b1;
                default: ;
            endcase
        end
    end

    // BLOQUE 2: Lógica de transición (Combinacional)
    always @(*) begin
        case (estado)
            S_IDLE:   prox_estado = tx_start ? S_ESPERA : S_IDLE;
            S_ESPERA: prox_estado = tick ? S_START : S_ESPERA;
            S_START:  prox_estado = tick ? S_DATOS : S_START;
            S_DATOS:  prox_estado = (tick && n_bit == 3'd7) ? S_STOP : S_DATOS;
            S_STOP:   prox_estado = tick ? S_IDLE : S_STOP;
            default:  prox_estado = S_IDLE;
        endcase
    end

    // BLOQUE 3: Lógica de salida Moore (Combinacional con valores por defecto)
    always @(*) begin
        tx_comb = 1'b1;
        tx_busy = (estado != S_IDLE);

        case (estado)
            S_START: tx_comb = 1'b0;       // Start
            S_DATOS: tx_comb = shift[0];   // Dato, LSB primero
            default: ;                     // IDLE, ESPERA y STOP: línea en alto
        endcase
    end

endmodule
```

### 2.4 Receptor UART (`uart_rx.v`)

Es el bloque más complejo. Sincroniza la línea, detecta el flanco del start, lo **valida a mitad de bit**, muestrea cada dato en su centro y valida el stop. El byte entra al registro por la izquierda (`{rx_s1, shift[7:1]}`), así que después de 8 desplazamientos el primer bit recibido (LSB) queda en `shift[0]`.

```verilog
module uart_rx (
    input  wire       clk,        // Reloj de 50 MHz
    input  wire       rst,        // Reset asíncrono activo en alto
    input  wire       tick16,     // Tick de sobremuestreo (16 por bit)
    input  wire       rx,         // Línea serial de entrada (asíncrona)

    output reg  [7:0] rx_data,    // Último byte recibido
    output reg        rx_done,    // Pulso de 1 ciclo: byte válido en rx_data
    output reg        rx_err      // Pulso de 1 ciclo: error de trama (stop = 0)
);

    // Codificación explícita de estados (2 bits -> 4 estados)
    localparam S_IDLE  = 2'b00;  // Esperando el flanco de bajada del start
    localparam S_START = 2'b01;  // Cuenta 8 ticks16 hasta el centro del start
    localparam S_DATOS = 2'b10;  // Muestrea cada bit en su centro (cada 16 ticks16)
    localparam S_STOP  = 2'b11;  // Muestrea y valida el bit de stop

    reg [1:0] estado, prox_estado;
    reg [3:0] s_cnt;             // Contador de ticks16 dentro de un bit (0..15)
    reg [2:0] n_bit;             // Bit de datos actual (0..7)
    reg [7:0] shift;             // Registro de desplazamiento (deserializador)

    // ETAPA 1: Sincronizador de 2 flip-flops (la línea RX es asíncrona al reloj)
    // Arranca en 1 (reposo) para no ver un start falso al salir del reset
    reg rx_s0, rx_s1;
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            rx_s0 <= 1'b1;
            rx_s1 <= 1'b1;
        end else begin
            rx_s0 <= rx;
            rx_s1 <= rx_s0;
        end
    end

    // Momentos de muestreo
    wire centro_start = tick16 && (s_cnt == 4'd7);    // 8 ticks después del flanco
    wire centro_bit   = tick16 && (s_cnt == 4'd15);   // 16 ticks después del centro anterior

    // BLOQUE 1: Registro de estado y camino de datos (Secuencial)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            estado  <= S_IDLE;
            s_cnt   <= 4'd0;
            n_bit   <= 3'd0;
            shift   <= 8'd0;
            rx_data <= 8'd0;
            rx_done <= 1'b0;
            rx_err  <= 1'b0;
        end else begin
            estado  <= prox_estado;
            rx_done <= 1'b0;
            rx_err  <= 1'b0;

            case (estado)
                S_IDLE: s_cnt <= 4'd0;

                S_START: if (tick16) begin
                    if (s_cnt == 4'd7) begin
                        s_cnt <= 4'd0;           // Centro del start: se reinicia la cuenta
                        n_bit <= 3'd0;
                    end else
                        s_cnt <= s_cnt + 1'b1;
                end

                S_DATOS: if (tick16) begin
                    s_cnt <= s_cnt + 1'b1;       // 15 -> 0 por desborde natural de 4 bits
                    if (s_cnt == 4'd15) begin
                        shift <= {rx_s1, shift[7:1]};   // Entra por la izquierda: LSB primero
                        n_bit <= n_bit + 1'b1;
                    end
                end

                S_STOP: if (tick16) begin
                    s_cnt <= s_cnt + 1'b1;
                    if (s_cnt == 4'd15) begin
                        if (rx_s1) begin         // Stop válido (1)
                            rx_data <= shift;
                            rx_done <= 1'b1;
                        end else
                            rx_err  <= 1'b1;     // Framing error: stop en 0
                    end
                end
            endcase
        end
    end

    // BLOQUE 2: Lógica de transición (Combinacional)
    always @(*) begin
        case (estado)
            S_IDLE:  prox_estado = (rx_s1 == 1'b0) ? S_START : S_IDLE;  // Flanco de bajada
            S_START: begin
                if (centro_start)
                    prox_estado = rx_s1 ? S_IDLE : S_DATOS;  // Si volvió a 1 era ruido
                else
                    prox_estado = S_START;
            end
            S_DATOS: prox_estado = (centro_bit && n_bit == 3'd7) ? S_STOP : S_DATOS;
            S_STOP:  prox_estado = centro_bit ? S_IDLE : S_STOP;
            default: prox_estado = S_IDLE;
        endcase
    end

endmodule
```

### 2.5 Conversor Binario → Decimal ASCII (`bin2dec.v`)

```verilog
module bin2dec (
    input  wire [9:0]  bin,       // Valor binario de 10 bits (0..1023)

    output wire [3:0]  bcd_mil,   // Dígitos BCD
    output wire [3:0]  bcd_cen,
    output wire [3:0]  bcd_dec,
    output wire [3:0]  bcd_uni,
    output wire [31:0] ascii,     // {mil, cen, dec, uni} en ASCII ('0' = 8'h30)
    output wire [2:0]  n_dig      // Dígitos significativos (1..4), sin ceros a la izquierda
);

    // Algoritmo "Double Dabble" (desplazar y sumar 3), puramente combinacional:
    // en cada una de las 10 iteraciones, toda columna BCD >= 5 se le suma 3
    // y luego se desplaza todo un bit a la izquierda.
    //
    //   reg_dd = [ mil | cen | dec | uni | binario ]
    //            25:22 21:18 17:14 13:10    9:0
    integer   i;
    reg [25:0] reg_dd;

    always @(*) begin
        reg_dd = {16'd0, bin};
        for (i = 0; i < 10; i = i + 1) begin
            if (reg_dd[13:10] >= 4'd5) reg_dd[13:10] = reg_dd[13:10] + 4'd3;
            if (reg_dd[17:14] >= 4'd5) reg_dd[17:14] = reg_dd[17:14] + 4'd3;
            if (reg_dd[21:18] >= 4'd5) reg_dd[21:18] = reg_dd[21:18] + 4'd3;
            if (reg_dd[25:22] >= 4'd5) reg_dd[25:22] = reg_dd[25:22] + 4'd3;
            reg_dd = reg_dd << 1;
        end
    end

    assign bcd_mil = reg_dd[25:22];
    assign bcd_cen = reg_dd[21:18];
    assign bcd_dec = reg_dd[17:14];
    assign bcd_uni = reg_dd[13:10];

    // Conversión BCD -> ASCII: '0' = 0x30, así que basta con anteponer 4'h3
    assign ascii = {4'h3, bcd_mil, 4'h3, bcd_cen, 4'h3, bcd_dec, 4'h3, bcd_uni};

    // Cantidad de dígitos a enviar ("7" y no "0007")
    assign n_dig = (bcd_mil != 4'd0) ? 3'd4 :
                   (bcd_cen != 4'd0) ? 3'd3 :
                   (bcd_dec != 4'd0) ? 3'd2 : 3'd1;

endmodule
```

### 2.6 Conversor Decimal ASCII → Binario (`dec2bin.v`)

Además de acumular, genera los pulsos que le dicen a `ctrl_tx` qué responder (eco, salto de línea o `ERR`). El dígito se obtiene del nibble bajo del carácter (`rx_data[3:0]`) porque `'0'..'9'` = `0x30..0x39`.

```verilog
module dec2bin (
    input  wire       clk,        // Reloj de 50 MHz
    input  wire       rst,        // Reset asíncrono activo en alto
    input  wire [7:0] rx_data,    // Byte recibido (carácter ASCII)
    input  wire       rx_done,    // Pulso: llegó un byte válido
    input  wire       rx_err,     // Pulso: error de trama en el receptor

    output reg  [9:0] valor,      // Último número válido (0..1023) -> LEDs
    output reg        error,      // 1 = la última línea fue inválida (se borra con una válida)

    // Pulsos de 1 ciclo para el eco hacia PuTTY (los usa ctrl_tx)
    output reg        eco_car,    // Reflejar el carácter recibido
    output reg        fin_linea,  // Enter aceptado (o vacío) -> "\r\n"
    output reg        fin_error   // Enter con error          -> "\r\nERR\r\n"
);

    localparam [7:0] ASCII_0  = 8'h30;
    localparam [7:0] ASCII_9  = 8'h39;
    localparam [7:0] ASCII_CR = 8'h0D;   // Enter en PuTTY
    localparam [7:0] ASCII_LF = 8'h0A;   // Algunas terminales envían LF

    reg  [9:0]  acum;          // Valor parcial acumulado
    reg         hay_digitos;   // Se recibió al menos un dígito
    reg         desborde;      // El número superó 1023
    reg         invalido;      // Llegó un carácter que no es dígito o hubo error de trama

    wire        es_digito = (rx_data >= ASCII_0) && (rx_data <= ASCII_9);
    wire        es_enter  = (rx_data == ASCII_CR) || (rx_data == ASCII_LF);
    wire [3:0]  digito    = rx_data[3:0];       // '0'..'9' = 0x30..0x39 -> nibble bajo

    // valor = valor * 10 + dígito   (x10 = x8 + x2: dos desplazamientos y una suma)
    wire [13:0] siguiente = {acum, 3'b000} + {acum, 1'b0} + digito;

    always @(posedge clk or posedge rst) begin
        if (rst) begin
            valor       <= 10'd0;
            error       <= 1'b0;
            acum        <= 10'd0;
            hay_digitos <= 1'b0;
            desborde    <= 1'b0;
            invalido    <= 1'b0;
            eco_car     <= 1'b0;
            fin_linea   <= 1'b0;
            fin_error   <= 1'b0;
        end else begin
            eco_car   <= 1'b0;
            fin_linea <= 1'b0;
            fin_error <= 1'b0;

            if (rx_err) begin
                invalido <= 1'b1;                    // Trama corrupta: la línea queda inválida

            end else if (rx_done) begin
                if (es_digito) begin
                    eco_car     <= 1'b1;
                    hay_digitos <= 1'b1;
                    if (siguiente > 14'd1023)
                        desborde <= 1'b1;            // Fuera de rango: se deja de acumular
                    else if (!desborde)
                        acum <= siguiente[9:0];

                end else if (es_enter) begin
                    if (invalido || desborde) begin
                        fin_error <= 1'b1;           // LEDs conservan el valor anterior
                        error     <= 1'b1;
                    end else begin
                        fin_linea <= 1'b1;
                        if (hay_digitos) begin       // Enter solo (vacío) no cambia nada
                            valor <= acum;
                            error <= 1'b0;
                        end
                    end
                    // Se prepara para el siguiente número
                    acum        <= 10'd0;
                    hay_digitos <= 1'b0;
                    desborde    <= 1'b0;
                    invalido    <= 1'b0;

                end else begin
                    eco_car  <= 1'b1;                // Se refleja, pero invalida la línea
                    invalido <= 1'b1;
                end
            end
        end
    end

endmodule
```

### 2.7 Secuenciador de Mensajes (`ctrl_tx.v`)

```verilog
module ctrl_tx (
    input  wire       clk,         // Reloj de 50 MHz
    input  wire       rst,         // Reset asíncrono activo en alto

    // Peticiones de envío (pulsos de 1 ciclo)
    input  wire       enviar,      // KEY0: enviar el número de los switches
    input  wire [9:0] valor_sw,    // Valor de los switches (sincronizado)
    input  wire       eco_car,     // dec2bin: reflejar un carácter recibido
    input  wire [7:0] car,         // Carácter a reflejar
    input  wire       fin_linea,   // dec2bin: Enter aceptado -> "\r\n"
    input  wire       fin_error,   // dec2bin: Enter con error -> "\r\nERR\r\n"

    // Interfaz con uart_tx
    input  wire       tx_done,     // Pulso: terminó el byte actual
    output reg        tx_start,    // Pulso: transmitir tx_data
    output reg  [7:0] tx_data      // Byte a transmitir
);

    // ---------------------------------------------------------------------
    // Mensajes que se pueden enviar (cada uno es una secuencia de bytes)
    //   M_NUM : dígitos ASCII del número (1 a 4) + "\r\n"    ej. "1023\r\n"
    //   M_ECO : el carácter recibido (eco de lo que se escribe en PuTTY)
    //   M_CRLF: "\r\n"
    //   M_ERR : "\r\nERR\r\n"
    // ---------------------------------------------------------------------
    localparam [1:0] M_NUM  = 2'd0;
    localparam [1:0] M_ECO  = 2'd1;
    localparam [1:0] M_CRLF = 2'd2;
    localparam [1:0] M_ERR  = 2'd3;

    localparam [7:0] CR = 8'h0D;
    localparam [7:0] LF = 8'h0A;

    // Codificación explícita de estados (2 bits -> 3 estados)
    localparam S_IDLE   = 2'b00;  // Sin mensaje en curso, atiende peticiones pendientes
    localparam S_ENVIA  = 2'b01;  // Pulso tx_start con el byte idx del mensaje
    localparam S_ESPERA = 2'b10;  // Espera tx_done del transmisor

    reg [1:0] estado, prox_estado;
    reg [1:0] tipo;               // Mensaje en curso
    reg [2:0] idx;                // Byte actual dentro del mensaje
    reg [9:0] valor_msg;          // Número que se está enviando (congelado)
    reg [7:0] car_msg;            // Carácter de eco que se está enviando

    // Peticiones pendientes (se atienden cuando el transmisor queda libre)
    reg       pend_num, pend_eco, pend_crlf, pend_err;
    reg [9:0] valor_pend;
    reg [7:0] car_pend;

    // Conversión binario -> decimal ASCII del número a enviar
    wire [31:0] ascii;
    wire [2:0]  n_dig;
    bin2dec conv_tx (
        .bin     (valor_msg),
        .bcd_mil (), .bcd_cen (), .bcd_dec (), .bcd_uni (),
        .ascii   (ascii),
        .n_dig   (n_dig)
    );

    // Longitud de cada mensaje
    reg [2:0] largo;
    always @(*) begin
        case (tipo)
            M_NUM:   largo = n_dig + 3'd2;
            M_ECO:   largo = 3'd1;
            M_CRLF:  largo = 3'd2;
            default: largo = 3'd7;   // M_ERR
        endcase
    end

    wire ultimo = (idx == largo - 1'b1);

    // Byte idx del mensaje en curso (tabla tipo ROM, como decod_mensaje en la caja fuerte)
    wire [2:0] pos = idx + (3'd4 - n_dig);   // Salta los ceros a la izquierda
    reg  [7:0] byte_sel;
    always @(*) begin
        byte_sel = CR;
        case (tipo)
            M_NUM: begin
                if (idx < n_dig) begin
                    case (pos[1:0])
                        2'd0: byte_sel = ascii[31:24];   // Mil
                        2'd1: byte_sel = ascii[23:16];   // Centenas
                        2'd2: byte_sel = ascii[15:8];    // Decenas
                        2'd3: byte_sel = ascii[7:0];     // Unidades
                    endcase
                end else
                    byte_sel = (idx == n_dig) ? CR : LF;
            end
            M_ECO:  byte_sel = car_msg;
            M_CRLF: byte_sel = (idx == 3'd0) ? CR : LF;
            M_ERR: begin
                case (idx)
                    3'd0: byte_sel = CR;
                    3'd1: byte_sel = LF;
                    3'd2: byte_sel = "E";
                    3'd3: byte_sel = "R";
                    3'd4: byte_sel = "R";
                    3'd5: byte_sel = CR;
                    default: byte_sel = LF;
                endcase
            end
        endcase
    end

    wire hay_pendiente = pend_eco | pend_err | pend_crlf | pend_num;

    // BLOQUE 1: Registro de estado, mensaje y peticiones (Secuencial)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            estado     <= S_IDLE;
            tipo       <= M_CRLF;
            idx        <= 3'd0;
            valor_msg  <= 10'd0;
            car_msg    <= 8'd0;
            pend_num   <= 1'b0;
            pend_eco   <= 1'b0;
            pend_crlf  <= 1'b0;
            pend_err   <= 1'b0;
            valor_pend <= 10'd0;
            car_pend   <= 8'd0;
        end else begin
            estado <= prox_estado;

            case (estado)
                // Toma la petición pendiente de mayor prioridad.
                // El eco va primero para que lo escrito salga antes del salto de línea.
                S_IDLE: begin
                    idx <= 3'd0;
                    if (pend_eco) begin
                        tipo <= M_ECO;  car_msg <= car_pend;  pend_eco <= 1'b0;
                    end else if (pend_err) begin
                        tipo <= M_ERR;  pend_err <= 1'b0;
                    end else if (pend_crlf) begin
                        tipo <= M_CRLF; pend_crlf <= 1'b0;
                    end else if (pend_num) begin
                        tipo <= M_NUM;  valor_msg <= valor_pend;  pend_num <= 1'b0;
                    end
                end

                S_ESPERA: if (tx_done && !ultimo) idx <= idx + 1'b1;

                default: ;
            endcase

            // Registro de peticiones nuevas (van después: si coinciden con
            // el borrado de arriba, gana la petición nueva)
            if (enviar)    begin pend_num <= 1'b1; valor_pend <= valor_sw; end
            if (eco_car)   begin pend_eco <= 1'b1; car_pend   <= car;      end
            if (fin_linea) pend_crlf <= 1'b1;
            if (fin_error) pend_err  <= 1'b1;
        end
    end

    // BLOQUE 2: Lógica de transición (Combinacional)
    always @(*) begin
        case (estado)
            S_IDLE:   prox_estado = hay_pendiente ? S_ENVIA : S_IDLE;
            S_ENVIA:  prox_estado = S_ESPERA;
            S_ESPERA: begin
                if (tx_done)
                    prox_estado = ultimo ? S_IDLE : S_ENVIA;
                else
                    prox_estado = S_ESPERA;
            end
            default:  prox_estado = S_IDLE;
        endcase
    end

    // BLOQUE 3: Lógica de salida Moore (Combinacional)
    always @(*) begin
        tx_start = (estado == S_ENVIA);
        tx_data  = byte_sel;
    end

endmodule
```

### 2.8 Sincronizador y Antirrebote (`antirrebote.v`)

Reutilizado sin cambios de la caja fuerte. Se usa la salida `subida` para que **cada pulsación** de KEY0 genere **un solo envío**.

```verilog
module antirrebote (
    input  wire clk,       // Reloj de 50 MHz
    input  wire rst,       // Reset asíncrono activo en alto
    input  wire entrada,   // Switch (activo en alto: 1 = ON)
    output reg  nivel,     // Nivel filtrado del switch
    output reg  subida,    // Pulso de 1 ciclo cuando el switch pasa a ON
    output reg  bajada     // Pulso de 1 ciclo cuando el switch pasa a OFF
);

    // Parámetro declarado dentro del módulo (igual que en el cronómetro)
    parameter SIMULATION = 0;

    // Tiempo que la señal debe permanecer estable para aceptarla
    // 50 MHz -> 1,000,000 ciclos = 20 ms
    localparam [19:0] N_ESTABLE = SIMULATION ? 20'd4 : 20'd1_000_000;

    reg        sync_0, sync_1;   // Sincronizador de 2 flip-flops
    reg        nivel_prev;       // Para detectar flancos
    reg [19:0] contador;

    // ETAPA 1: Sincronización (evita metaestabilidad)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            sync_0 <= 1'b0;
            sync_1 <= 1'b0;
        end else begin
            sync_0 <= entrada;
            sync_1 <= sync_0;
        end
    end

    // ETAPA 2: Antirrebote (solo acepta el cambio si se mantiene N_ESTABLE ciclos)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            nivel    <= 1'b0;
            contador <= 20'd0;
        end else if (sync_1 != nivel) begin
            if (contador >= N_ESTABLE - 1'b1) begin
                nivel    <= sync_1;
                contador <= 20'd0;
            end else begin
                contador <= contador + 1'b1;
            end
        end else begin
            contador <= 20'd0;  // Rebote: la señal volvió, se reinicia la cuenta
        end
    end

    // ETAPA 3: Detector de flancos (un solo pulso por cada cambio)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            nivel_prev <= 1'b0;
            subida     <= 1'b0;
            bajada     <= 1'b0;
        end else begin
            nivel_prev <= nivel;
            subida     <=  nivel & ~nivel_prev;
            bajada     <= ~nivel &  nivel_prev;
        end
    end

endmodule
```

### 2.9 Decodificador de 7 Segmentos (`decod_7seg.v`)

```verilog
module decod_7seg (
    input      [4:0] codigo, // 0..15 = dígito hexadecimal, 16.. = caracteres especiales
    output reg [6:0] seg     // Segmentos: {g, f, e, d, c, b, a} (Ánodo común: 0 = Encendido)
);

    always @(*) begin
        case (codigo)
            // Dígitos
            5'h00: seg = 7'b100_0000; // 0
            5'h01: seg = 7'b111_1001; // 1
            5'h02: seg = 7'b010_0100; // 2
            5'h03: seg = 7'b011_0000; // 3
            5'h04: seg = 7'b001_1001; // 4
            5'h05: seg = 7'b001_0010; // 5
            5'h06: seg = 7'b000_0010; // 6
            5'h07: seg = 7'b111_1000; // 7
            5'h08: seg = 7'b000_0000; // 8
            5'h09: seg = 7'b001_0000; // 9
            5'h0A: seg = 7'b000_1000; // A
            5'h0B: seg = 7'b000_0011; // b
            5'h0C: seg = 7'b100_0110; // C
            5'h0D: seg = 7'b010_0001; // d
            5'h0E: seg = 7'b000_0110; // E
            5'h0F: seg = 7'b000_1110; // F

            // Caracteres especiales
            5'h10: seg = 7'b111_1111; // Apagado
            5'h11: seg = 7'b010_1111; // Letra 'r'  (e,g)
            5'h12: seg = 7'b000_0111; // Letra 't'  (d,e,f,g)

            default: seg = 7'b111_1111;
        endcase
    end

endmodule
```

---

## 3. Bancos de Pruebas (Testbenches) y Simulación en GTKWave

La verificación se hizo con **Icarus Verilog** y **GTKWave** usando dos bancos de pruebas:

| Testbench | Escala | Propósito |
|---|---|---|
| `tb_proyecto_uart.v` | `SIMULATION = 1` (rápida) | Sistema completo en **ambos sentidos**, errores y full-duplex, con **verificación automática** (`[OK]` / `[FALLA]`) |
| `tb_uart_tiempo_real.v` | `SIMULATION = 0` (**9600 baudios reales**) | Medir la duración real de cada bit y comprobar TX → RX en lazo cerrado con el byte `'A'` |

Igual que en la caja fuerte, la aceleración se logra con el `parameter SIMULATION`, que reduce los temporizadores **conservando la relación 16:1** entre `tick` y `tick16`:

| Temporizador | Real | Simulación |
|---|---|---|
| Antirrebote KEY0 | 20 ms (1.000.000 ciclos) | 4 ciclos (80 ns) |
| Tick de bit (`tick`) | 5208 ciclos (104,16 µs) | 32 ciclos (640 ns) |
| Tick 16× (`tick16`) | 326 ciclos (6,52 µs) | 2 ciclos (40 ns) |

**Cómo ejecutar las simulaciones** (desde la carpeta del proyecto):

```bash
iverilog -o build/tb_proyecto_uart.out tb_proyecto_uart.v
cd build && vvp tb_proyecto_uart.out && gtkwave tb_proyecto_uart.gtkw

iverilog -o build/tb_uart_tiempo_real.out tb_uart_tiempo_real.v
cd build && vvp tb_uart_tiempo_real.out && gtkwave tb_uart_tiempo_real.gtkw
```

Los archivos `.gtkw` de la carpeta `build/` cargan las señales ya organizadas por sentido (FPGA → PC y PC → FPGA).

### 3.1 Testbench del Sistema Completo (`tb_proyecto_uart.v`)

El banco de pruebas modela un **"PuTTY virtual"** conectado a la FPGA:

- **Receptor del PC:** un bloque `always @(negedge fpga_tx)` decodifica cada trama que transmite la FPGA (muestrea en el centro de cada bit), guarda los últimos 16 bytes en `terminal` e **imprime los caracteres en consola como lo haría PuTTY**.
- `pc_byte(b, stop_bueno)`: el PC transmite un byte por la línea RX de la FPGA. Con `stop_bueno = 0` genera a propósito una trama con **framing error**.
- `pc_escribir("725", enter)`: escribe una cadena carácter por carácter, opcionalmente seguida de Enter (`0x0D`, lo que envía PuTTY).
- `presionar_key0`: simula KEY0 **con rebotes mecánicos** al presionar y al soltar.
- `enviar_switches(v)`: pone `v` en los switches, presiona KEY0 y espera la transmisión.
- `verificar(cond, msg)`: imprime `[OK]` o `[FALLA]` y cuenta los errores.

Escenarios verificados:
1. **FPGA → PC:** 1023, 0, 37 y 500 llegan como texto, sin ceros a la izquierda, **una sola vez por pulsación** a pesar de los rebotes, con el bit de start de exactamente 1 T<sub>bit</sub>.
2. **PC → FPGA:** `725`, `1023`, `0` y `0042` aparecen en binario en los LEDs solo después del Enter, y PuTTY recibe el eco.
3. **Entradas inválidas:** `1024` (fuera de rango), `99999` (desborde), `12a` (letra) y una trama con stop en 0 se rechazan; los LEDs conservan el valor anterior, PuTTY muestra `ERR` y los displays `Er`. Un número válido posterior borra el indicador.
4. **Full-duplex:** se presiona KEY0 mientras se escribe en PuTTY; ambos sentidos funcionan a la vez y no se pierde ningún byte.

Salida de consola (lo que aparece sin `[OK]` es lo que "imprime PuTTY"):

```
[OK]    Reset: linea TX en reposo (1) y LEDs en 0

--- Sentido FPGA -> PC (switches + KEY0) ---
Terminal PuTTY:
1023
[OK]    SW=1023 -> PuTTY recibe "1023\r\n"
[OK]    Una sola transmision por pulsacion (con rebotes)
[OK]    Bit de start dura 1 T_bit (32 ciclos)
0
[OK]    SW=0    -> "0\r\n"
37
[OK]    SW=37   -> "37\r\n" (sin ceros a la izq.)
[OK]    HEX muestra '  37' en decimal
500
[OK]    SW=500  -> "500\r\n"
[OK]    Todas las tramas con stop valido

--- Sentido PC -> FPGA (PuTTY -> LEDs) ---
Terminal PuTTY (eco):
725[OK]    Sin Enter los LEDs no cambian

[OK]    "725" + Enter -> LEDs = 1011010101
[OK]    Eco "725\r\n" en PuTTY
1023
[OK]    "1023" -> LEDs = 1111111111
0
[OK]    "0" -> LEDs = 0000000000
0042
[OK]    "0042" -> LEDs = 42 (ceros a la izquierda)

--- Entradas invalidas ---
1024
ERR
[OK]    "1024" fuera de rango: LEDs conservan 42
[OK]    PuTTY muestra "ERR"
[OK]    HEX5..HEX4 muestran 'Er'
99999
ERR
[OK]    "99999" (desborde) rechazado
12a
ERR
[OK]    "12a" (caracter no numerico) rechazado
5
ERR
[OK]    Framing error (stop = 0) invalida la linea

[OK]    Enter vacio no cambia los LEDs
512
[OK]    "512" valido tras errores -> LEDs = 1000000000
[OK]    Indicador 'Er' se borra

--- Full-duplex: KEY0 mientras se escribe en PuTTY ---
321
9
[OK]    Recepcion correcta mientras se transmite
[OK]    Llegan el eco ("9\r\n") y el numero ("321\r\n")
[OK]    KEY1 (RESET) -> LEDs en 0

=== Simulacion terminada: 0 falla(s) ===
```

Resultado de simulación en GTKWave, sentido **FPGA → PC** (SW = 1023, KEY0): se ve la pulsación con rebotes, el único pulso `enviar`, los 6 bytes `31 30 32 33 0D 0A` en `tx_data` y sus tramas en `fpga_tx`:

![Simulación de la transmisión en GTKWave](figs/simu_tx.png)

Resultado de simulación en GTKWave, sentido **PC → FPGA** (`725` + Enter): se ven las tramas en `pc_tx`, el contador `s_cnt` del sobremuestreo, los bytes en `rx_data`, el acumulador `7 → 72 → 725` y el cambio de `ledr` al llegar el Enter:

![Simulación de la recepción en GTKWave](figs/simu_rx.png)

### 3.2 Testbench a Escala Real (`tb_uart_tiempo_real.v`)

Este banco instancia `baud_gen`, `uart_tx` y `uart_rx` con `SIMULATION = 0` (5208 y 326 cuentas reales) y conecta **TX directamente a RX** (*loopback*). Transmite el mismo ejemplo de clase, `0x41` (`'A'`), e imprime la duración de cada nivel de la línea. Para que el `.vcd` quede pequeño solo se vuelcan las señales lentas.

```
Divisor de bit : 5208 cuentas (50 MHz / 9600)
Divisor 16x    : 326 cuentas (50 MHz / 153600)

Enviando 0x41 ('A' = 0100_0001), LSB primero: 0 | 1 0 0 0 0 0 1 0 | 1
  t =  104.29 us  linea -> 0   (nivel anterior duro  104.29 us = 1.00 bits)
  t =  208.45 us  linea -> 1   (nivel anterior duro  104.16 us = 1.00 bits)
  t =  312.61 us  linea -> 0   (nivel anterior duro  104.16 us = 1.00 bits)
  t =  833.41 us  linea -> 1   (nivel anterior duro  520.80 us = 5.00 bits)
  t =  937.57 us  linea -> 0   (nivel anterior duro  104.16 us = 1.00 bits)
  t = 1041.73 us  linea -> 1   (nivel anterior duro  104.16 us = 1.00 bits)

Trama completa (10 bits): 1041.58 us
Byte recibido en el lazo : 0x41 ('A')
[OK]    Loopback correcto
[OK]    10 bits x 104.16 us = 1.0416 ms

=== Simulacion terminada: 0 falla(s) ===
```

La medición confirma la trama `0 | 1 0 0 0 0 0 1 0 | 1`: cada bit dura **104,16 µs**, los cinco ceros seguidos (D1..D5) duran 520,8 µs = 5 bits, y la trama completa de 10 bits dura **1,0416 ms**. El receptor reconstruye `0x41` sin error de trama.

![Trama de 'A' a 9600 baudios en GTKWave](figs/simu_tiempo_real.png)

### 3.3 Síntesis en Intel Quartus Prime

El proyecto compila sin errores en Quartus Prime 18.0 Lite. Resumen del *fitter* y del análisis de tiempos:

| Recurso | Uso |
|---|---|
| Elementos lógicos | 433 / 49,760 ( < 1 % ) |
| Registros | 205 |
| Pines | 67 / 360 ( 19 % ) |
| Memoria / multiplicadores / PLL | 0 |
| **Fmax** (Slow 1200 mV 85 °C) | **74.58 MHz** (requerido: 50 MHz) |

El diseño cabe holgadamente (< 1 % del MAX 10) y cumple tiempos con margen. El archivo `proyecto_uart.sdc` declara el reloj de 50 MHz y marca como *false path* las entradas asíncronas (que ya pasan por sincronizadores de 2 FF) y las salidas lentas.

Los avisos *"Output pins are stuck at VCC or GND"* son esperados: `HEX3` solo puede mostrar `1` o estar apagado (el máximo es 1023), y `HEX5`/`HEX4` solo muestran `E`/`r` o nada, así que varios segmentos nunca cambian.

### 3.4 Asignación de Pines (`.qsf`)

Los pines se tomaron del **DE10-Lite User Manual**. Todos usan el estándar `3.3-V LVTTL`. La entrada `uart_rx` tiene **pull-up interno** para que, con el adaptador desconectado, la línea quede en reposo (`1`) y no se reciban tramas falsas.

| Señal | Pin FPGA | Recurso de la tarjeta |
|---|---|---|
| `clk` | PIN_P11 | Reloj de 50 MHz (MAX10_CLK1_50) |
| `key_tx` | PIN_B8 | KEY0 (ENVIAR) |
| `key_reset` | PIN_A7 | KEY1 (RESET) |
| `sw[0..9]` | C10, C11, D12, C12, A12, B12, A13, A14, B14, F15 | SW0..SW9 |
| `uart_tx` | PIN_V10 | GPIO_[0], JP1 pin 1 → RX del adaptador |
| `uart_rx` | PIN_W10 | GPIO_[1], JP1 pin 2 ← TX del adaptador (pull-up) |
| `ledr[0..9]` | A8, A9, A10, B10, D13, C13, E14, D14, A11, B11 | LEDR0..LEDR9 |
| `disp0..disp5` | ver `.qsf` | HEX0..HEX5 |

```tcl
# =========================================================================
# Reloj (50 MHz interno)
# =========================================================================
set_location_assignment PIN_P11 -to clk
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to clk

# =========================================================================
# KEY0 - ENVIAR (pulsador de la tarjeta, activo en bajo)
# =========================================================================
set_location_assignment PIN_B8 -to key_tx
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to key_tx

# =========================================================================
# KEY1 - RESET (pulsador de la tarjeta, activo en bajo)
# =========================================================================
set_location_assignment PIN_A7 -to key_reset
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to key_reset

# =========================================================================
# Switches SW9..SW0 - número a transmitir (0..1023)
# =========================================================================
set_location_assignment PIN_C10 -to sw[0]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[0]
set_location_assignment PIN_C11 -to sw[1]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[1]
set_location_assignment PIN_D12 -to sw[2]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[2]
set_location_assignment PIN_C12 -to sw[3]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[3]
set_location_assignment PIN_A12 -to sw[4]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[4]
set_location_assignment PIN_B12 -to sw[5]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[5]
set_location_assignment PIN_A13 -to sw[6]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[6]
set_location_assignment PIN_A14 -to sw[7]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[7]
set_location_assignment PIN_B14 -to sw[8]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[8]
set_location_assignment PIN_F15 -to sw[9]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sw[9]

# =========================================================================
# UART en header GPIO JP1 (3.3 V)
#   Pin 1 GPIO (GPIO_[0]) -> uart_tx -> RX del adaptador USB-TTL
#   Pin 2 GPIO (GPIO_[1]) <- uart_rx <- TX del adaptador USB-TTL (pull-up: reposo = 1)
#   Pin 12 o 30          -> GND común con el adaptador
# =========================================================================
set_location_assignment PIN_V10 -to uart_tx
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to uart_tx
set_location_assignment PIN_W10 -to uart_rx
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to uart_rx
set_instance_assignment -name WEAK_PULL_UP_RESISTOR ON -to uart_rx

# =========================================================================
# LEDR9..LEDR0 - número recibido desde PuTTY (binario)
# =========================================================================
set_location_assignment PIN_A8 -to ledr[0]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[0]
set_location_assignment PIN_A9 -to ledr[1]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[1]
set_location_assignment PIN_A10 -to ledr[2]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[2]
set_location_assignment PIN_B10 -to ledr[3]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[3]
set_location_assignment PIN_D13 -to ledr[4]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[4]
set_location_assignment PIN_C13 -to ledr[5]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[5]
set_location_assignment PIN_E14 -to ledr[6]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[6]
set_location_assignment PIN_D14 -to ledr[7]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[7]
set_location_assignment PIN_A11 -to ledr[8]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[8]
set_location_assignment PIN_B11 -to ledr[9]
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to ledr[9]
```

### 3.5 Montaje Físico y Configuración de PuTTY

Se usa un adaptador **USB-TTL** (CP2102, CH340 o FT232) con el selector/jumper de voltaje en **3.3 V**. Las líneas se **cruzan**: lo que una transmite, el otro lo recibe.

```
   Adaptador USB-TTL (3.3 V)              DE10-Lite, header JP1 (GPIO)
  +--------------------------+           +------------------------------+
  |  TXD  -------------------|---------->| pin 2  GPIO_[1]  uart_rx  W10 |
  |  RXD  <------------------|-----------| pin 1  GPIO_[0]  uart_tx  V10 |
  |  GND  -------------------|-----------| pin 12 GND                    |
  |  VCC  (NO SE CONECTA)    |           |                               |
  +--------------------------+           +------------------------------+
           | USB
           v
          PC (PuTTY, COMx)
```

| Elemento | Conexión |
|---|---|
| TXD del adaptador | JP1 pin 2 (GPIO_[1], W10) = `uart_rx` |
| RXD del adaptador | JP1 pin 1 (GPIO_[0], V10) = `uart_tx` |
| GND del adaptador | JP1 pin 12 (o pin 30): **GND común** |
| VCC del adaptador | **No se conecta** (la tarjeta tiene su propia alimentación por USB) |

> **Importante:** el GPIO de la DE10-Lite es de **3,3 V**. No se debe usar un adaptador en 5 V ni niveles RS-232 (±12 V), porque pueden dañar la FPGA.

**Configuración de PuTTY:**

| Opción | Valor |
|---|---|
| Connection type | **Serial** |
| Serial line | `COMx` (ver *Administrador de dispositivos → Puertos COM*) |
| Speed (baud) | **9600** |
| Data bits / Stop bits / Parity | **8 / 1 / None** (8-N-1) |
| Flow control | **None** |
| Terminal → Local echo | **Force off** (la FPGA ya hace eco) |
| Terminal → Local line editing | **Force off** (cada tecla se envía de inmediato) |

**Prueba de los dos sentidos:**
1. **Switches + KEY0 → PuTTY:** se fija un número en `SW9..SW0` (los displays `HEX3..HEX0` lo muestran en decimal para comprobarlo) y se presiona KEY0. El número aparece en PuTTY en una línea nueva.
2. **PuTTY → LEDs:** se escribe un número de 0 a 1023 (se ve el eco) y se presiona Enter. Los LEDs `LEDR9..LEDR0` muestran el valor en binario. Si el número es inválido aparece `ERR` y `Er` en `HEX5..HEX4`.

---

## 4. Respuestas a las Preguntas de Análisis

**1. ¿Por qué un solo byte UART no alcanza para representar los números 0–1023, y cómo lo resuelve su diseño?**
Un byte tiene 8 bits, así que solo representa $2^8 = 256$ valores (0 a 255, `FF`). El rango 0–1023 tiene $2^{10} = 1024$ valores y necesita 10 bits. El diseño envía el número **como texto decimal**: `bin2dec` separa el valor de los switches en hasta 4 dígitos con Double Dabble, cada dígito se convierte a ASCII (`0x30 + dígito`) y viaja en su propia trama, seguido de `\r\n`. En recepción, `dec2bin` acumula los dígitos con `valor·10 + dígito` hasta el Enter. Así cualquier número de 0 a 1023 ocupa entre 1 y 4 tramas, y además es legible directamente en PuTTY.

**2. Calcule el número de cuentas del generador de baudios para 9600 baudios con el reloj de 50 MHz.**

$$N = \frac{f_{clk}}{\text{baud}} = \frac{50\,000\,000}{9600} = 5208{,}33 \rightarrow 5208 \text{ cuentas por bit}$$

El contador va de 0 a 5207 y necesita **13 bits** ($2^{13} = 8192$). El bit dura $5208 \times 20\text{ ns} = 104{,}16\ \mu s$ y el baud rate real es $9600{,}6$, con un error de 0,006 %. Para el sobremuestreo del receptor: $50 \times 10^6 / (9600 \times 16) = 325{,}5 \rightarrow 326$ cuentas (error de 0,16 %). El testbench a escala real lo confirma: `Divisor de bit : 5208 cuentas` y bits de 104,16 µs.

**3. ¿Por qué el receptor es más complejo que el transmisor?**
El transmisor **decide** cuándo empieza la trama y solo tiene que poner cada bit en la línea durante un tick. El receptor, en cambio, **no sabe cuándo llegará** la trama ni tiene el reloj del emisor, así que debe:
- sincronizar una señal asíncrona (2 FF),
- detectar el flanco de bajada del start,
- **validar** el start a mitad de bit para descartar ruido,
- sobremuestrear a 16× para ubicar el **centro** de cada bit y tolerar diferencias de reloj,
- validar el bit de stop y reportar el error de trama.

Por eso usa un reloj 16 veces más rápido, un contador extra (`s_cnt`) y más condiciones de transición.

**4. ¿Por qué se detecta el flanco del botón y no su nivel para transmitir?**
Una pulsación humana dura cientos de milisegundos, es decir, decenas de millones de ciclos de 50 MHz. Si se transmitiera mientras el **nivel** está en alto, el número se enviaría una y otra vez durante toda la pulsación y PuTTY se llenaría de copias. Detectando el **flanco** (`subida` de `antirrebote.v`) se genera un único pulso de 1 ciclo por pulsación, que se registra como una sola petición en `ctrl_tx`. El antirrebote de 20 ms evita además que los rebotes mecánicos generen flancos extra. El testbench lo verifica: con rebotes al presionar y al soltar llegan exactamente 6 bytes (`"1023\r\n"`).

**5. ¿Qué es un error de trama (framing error) y cómo lo detectaría el receptor?**
Es una trama cuyo **bit de stop no vale 1**. Ocurre cuando emisor y receptor tienen baud rates distintos (el receptor pierde la alineación), cuando hay ruido en la línea, o cuando se desconecta el cable a mitad de un byte (la línea queda en 0). El receptor lo detecta en `S_STOP`: en el centro del bit de stop muestrea la línea y, si lee `0` en lugar de `1`, **descarta el byte** y genera el pulso `rx_err` en vez de `rx_done`. En este diseño `dec2bin` marca entonces la línea como inválida, y al presionar Enter responde `ERR` sin modificar los LEDs. El testbench lo prueba enviando una trama con stop en 0.

**6. ¿Por qué es necesario sincronizar la línea RX antes de usarla en la FSM?**
La línea RX cambia según el reloj del PC, que no tiene ninguna relación con el de 50 MHz de la FPGA. Si un flanco cae justo en la ventana de *setup/hold* de un flip-flop, éste puede entrar en **metaestabilidad** (un nivel indefinido durante un tiempo). Además, la señal llega a varios flip-flops de la FSM a la vez (registro de estado, contador, registro de desplazamiento): sin sincronizar, unos podrían ver `0` y otros `1` en el mismo ciclo, y la FSM saltaría a un estado incoherente. El sincronizador de 2 FF da un ciclo completo para que la metaestabilidad se resuelva y entrega a toda la FSM **una única versión estable** de RX (`rx_s1`).

**7. ¿Qué diferencia habría entre enviar el número en binario crudo y enviarlo como texto ASCII?**

| | Binario crudo | Texto ASCII (este diseño) |
|---|---|---|
| Bytes por número | 2 fijos (10 bits → 2 bytes) | 1 a 4 dígitos + `\r\n` (3 a 6 bytes) |
| Eficiencia | Mayor | Menor |
| Legible en PuTTY | **No**: se verían símbolos raros o caracteres de control (`0x03`, `0xFF`) | **Sí**: se lee `1023` directamente |
| Escribirlo desde el teclado | Prácticamente imposible | Natural |
| Delimitación | Hay que acordar el orden de los 2 bytes (byte alto/bajo) y es fácil desalinearse | El Enter marca el final del número |
| Conversión en la FPGA | Ninguna (solo partir los 10 bits) | `bin2dec` y `dec2bin` |

El binario crudo es más eficiente y sirve para comunicación máquina–máquina (por ejemplo con un script en Python), pero el enunciado pide usar una terminal donde una persona lee y escribe, y para eso el texto ASCII es la opción correcta, a cambio de la lógica de conversión.

---

## 5. Conclusiones

1. **Comunicación sin reloj compartido:** la UART demuestra que dos sistemas pueden entenderse sin un cable de reloj si acuerdan el baud rate y el formato de trama. Los bits de start y stop resuelven la pregunta clave del protocolo (dónde empieza y termina cada byte), y el sobremuestreo 16× permite tolerar la pequeña diferencia entre los relojes de la FPGA y del PC.
2. **Reutilización de técnicas conocidas:** el generador de baudios es la misma base de tiempo del cronómetro y del semáforo (5208 cuentas por bit), el antirrebote se reutilizó sin cambios de la caja fuerte y las tres FSM siguen la arquitectura de tres bloques `always`. Esto redujo el riesgo y permitió concentrarse en lo nuevo: el protocolo y la conversión de datos.
3. **El registro de desplazamiento como pieza central:** serializar (TX) y deserializar (RX) se reduce a cargar o ir llenando un registro de 8 bits que se desplaza en cada tick. La diferencia de complejidad entre ambos lados viene de que el receptor debe descubrir la temporización por su cuenta.
4. **Conversión binario ↔ decimal:** el límite de 8 bits por trama obligó a transmitir el número como texto. El algoritmo Double Dabble convirtió binario a BCD sin divisores, y la acumulación `valor·10 + dígito` (con `×10 = ×8 + ×2`) hizo la conversión inversa con desplazamientos y sumas. Validar el rango y los caracteres evitó que entradas inválidas alteraran los LEDs.
5. **Diseño pensado para el usuario:** el eco, el mensaje `ERR`, el indicador `Er` y la visualización en decimal de los switches no eran obligatorios, pero hicieron la demostración clara y fácil de depurar. El secuenciador con peticiones pendientes permitió usar los dos sentidos al mismo tiempo (full-duplex) sin perder bytes.
6. **Verificación antes del hardware:** el `parameter SIMULATION` permitió simular en microsegundos lo que en hardware dura milisegundos, y el "PuTTY virtual" del testbench comprobó automáticamente los dos sentidos, los errores y el caso full-duplex. El testbench a escala real confirmó los 104,16 µs por bit con el mismo ejemplo de `'A'` de clase, y una carrera en el estímulo del testbench (cambiar `tx_start` en el mismo flanco en que se muestrea) recordó la importancia de aplicar los estímulos en el flanco de bajada.

