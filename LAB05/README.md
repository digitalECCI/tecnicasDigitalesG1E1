# Informe de Laboratorio: Sistema de Control de Semáforo Vehicular y Peatonal mediante Máquinas de Estados Finitos (FSM) en Verilog

## Integrantes
- [Andres Mateo Arias Aguilera](https://github.com/mateoaeora124)
- [Gabriel Cangrejo](https://github.com/gabriel11cangrejo)
- [Cesar Alberto Gomez](https://github.com/Cesar7772026)

---

## 1. Introducción y Fundamentos Teóricos

El presente laboratorio aborda el diseño, la simulación y la síntesis en hardware mediante Verilog HDL de un **sistema de control de tráfico inteligente** integrado por un semáforo vehicular de tres luces (Rojo, Amarillo, Verde), un semáforo peatonal de dos luces (Rojo, Verde) y una señal acústica mediante un zumbador (*buzzer*).

A diferencia de los circuitos combinacionales tradicionales, la gestión del tiempo y la secuencia de pasos de un semáforo exigen el uso de **lógica secuencial sincronizada por reloj** estructurada bajo el modelo de **Máquinas de Estados Finitos (FSM)**. El sistema procesa modos operativos avanzados como retención por flujo vehicular ($D$), aceleración nocturna ($sr$) y modo de emergencia por titileo ($jk\_switch$), garantizando estabilidad y ausencia de condiciones de carrera (*race conditions*) en entornos FPGA.

---

### 1.1 Máquinas de Estados Finitos (FSM) y Modelo de Moore
Una **Máquina de Estados Finitos (FSM)** es un modelo matemático de computación secuencial compuesto por un número finito de estados, transiciones entre dichos estados y acciones de salida.

$$S(t+1) = \delta(S(t), X(t))$$

$$Y(t) = \lambda(S(t))$$

Donde $S(t)$ representa el estado actual, $X(t)$ las entradas del sistema, $\delta$ la función delta de transición de estado, $Y(t)$ el vector de salidas y $\lambda$ la función de salida.

En una **Máquina de Moore**, las salidas $Y(t)$ dependen exclusivamente del estado presente $S(t)$ y no directamente de las entradas inmediatas. Esto proporciona estabilidad ante ruido combinacional en los buses de entrada y facilita la sincronización temporal.

```
                  +-----------------------------------+
                  |                                   |
                  v                                   |
           +--------------+   fin_tiempo = 1   +--------------+
           |    S_ROJO    | -----------------> |  S_AM_VERDE  |
           |   (2'b00)    |                    |   (2'b01)    |
           +--------------+                    +--------------+
                  ^                                   |
   fin_tiempo = 1 |                                   | fin_tiempo = 1
                  |                                   v
           +--------------+   fin_tiempo = 1   +--------------+
           |  S_AM_ROJO   | <----------------- |   S_VERDE    |
           |   (2'b11)    |                    |   (2 me; // (2'b10)   |
           +--------------+                    +--------------+
```

---

### 1.2 Codificación de Estados y Selección de Bits
Para representar los 4 estados principales del ciclo del semáforo se utiliza una codificación binaria explícita mediante directivas `localparam`, optimizando el uso de Flip-Flops en la FPGA:

$$n = \lceil \log_2(M) \rceil$$

Para $M = 4$ estados, se requieren $n = 2\text{ bits}$ ($\text{reg } [1:0] \text{ estado, prox\_estado}$):
- `S_ROJO` ($2'b00$): Vehicular en Rojo ($10\text{ s}$), Peatonal en Verde, Buzzer en emisión de pulsos.
- `S_AM_VERDE` ($2'b01$): Vehicular en Amarillo ($3\text{ s}$), Peatonal en Rojo, Buzzer apagado.
- `S_VERDE` ($2'b10$): Vehicular en Verde ($10\text{ s}$), Peatonal en Rojo, Buzzer apagado.
- `S_AM_ROJO` ($2'b11$): Vehicular en Amarillo ($3\text{ s}$), Peatonal en Rojo, Buzzer apagado.

---

### 1.3 Arquitectura de Tres Bloques `always` en Verilog
Para cumplir con los estándares de síntesis estricta en herramientas EDA como Intel Quartus Prime y evitar la inferencia indebida de *latches* (cerrojos combinacionales), la FSM se implementa dividiendo el código en **tres bloques `always` independientes**:

1. **Bloque 1 (Secuencial):** Actualización del registro de estado y contadores en el flanco activo del reloj (`always @(posedge clk or posedge rst)`).
2. **Bloque 2 (Combinacional - Transición):** Definición de la lógica del siguiente estado (`prox_estado`) mediante una estructura `case(estado)`.
3. **Bloque 3 (Combinacional - Salidas Moore):** Asignación de las salidas del sistema. Para garantizar cero *latches*, se inicializan **todas** las salidas con valores por defecto (`0`) al inicio del bloque antes del `case`.

---

### 1.4 Base de Tiempo y Divisor de Frecuencia a 50 MHz
Las tarjetas de desarrollo como la Terasic DE10-Lite cuentan con un oscilador de cristal de $f_{\text{FPGA}} = 50\text{ MHz}$ (Período $T = 20\text{ ns}$). Para generar una base de tiempo visible de $1\text{ s}$ ($1\text{ Hz}$), el divisor de frecuencia se calcula mediante el valor límite de conteo $N$:

$$N = \frac{f_{\text{FPGA}}}{f_{\text{Deseada}}} = \frac{50 \times 10^6\text{ Hz}}{1\text{ Hz}} = 50.000.000\text{ ciclos}$$

El contador acumula de $0$ a $49.999.999$. Al alcanzar este valor límite, conmuta la señal lógica `pulse_1s` durante un único ciclo de reloj para incrementar el contador de segundos interno (`contador_seg`).

---

### 1.5 Señalización Acústica del Peatón (Generación de Pulsos para Buzzer)
El zumbador peatonal (*buzzer*) no debe recibir una señal de corriente continua fija, sino un tren de pulsos (onda cuadrada) para emitir una alerta sonora intermitente. Esto se logra asignando `buzzer = blink` durante el estado `S_ROJO`, donde `blink` conmuta en cada pulso del divisor de tiempo de $1\text{ segundo}$.

---

## 2. Código Hardware HDL en Verilog

En esta sección se detallan los archivos del proyecto desarrollados bajo la arquitectura de tres bloques exigida para la tarjeta MAX 10 FPGA.

### 2.1 Módulo Principal FSM Semáforo (`traffic_light.v`)

```verilog
module traffic_light (
    input  wire clk,           // Reloj de entrada 50 MHz
    input  wire rst,           // Reset asíncrono
    input  wire sr,            // Switch día/noche (Acelera la base de tiempo)
    input  wire D,             // Sensor de tráfico (Detiene la FSM)
    input  wire jk_switch,     // Botón de emergencia

    // Semáforo Vehicular (3 luces)
    output reg  rojo,
    output reg  amarillo,
    output reg  verde,

    // Semáforo Peatonal (2 luces) y Zumbador
    output reg  peaton_rojo,
    output reg  peaton_verde,
    output reg  buzzer
);

    // 1. Codificación de estados (2 bits -> 4 estados)
    localparam S_ROJO     = 2'b00;
    localparam S_AM_VERDE = 2'b01;
    localparam S_VERDE    = 2'b10;
    localparam S_AM_ROJO  = 2'b11;

    reg [1:0]  estado, prox_estado;
    reg [25:0] timer;
    reg [3:0]  contador_seg;
    reg        fin_tiempo;
    reg        blink;

    // Divisor de base de tiempo: 50 MHz (50,000,000 ciclos = 1 s real)
    wire [25:0] max_timer = (sr) ? 26'd24_999_999 : 26'd49_999_999;
    wire pulse_1s = (timer >= max_timer);

    // Generador de pulso de 1 segundo y señal intermitente (blink)
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            timer <= 26'd0;
            blink <= 1'b0;
        end else begin
            if (pulse_1s) begin
                timer <= 26'd0;
                blink <= ~blink;
            end else begin
                timer <= timer + 1'b1;
            end
        end
    end

    // Duraciones por estado (10 s en rojo/verde, 3 s en amarillos)
    always @(*) begin
        case (estado)
            S_ROJO:     fin_tiempo = (contador_seg >= 4'd9);  // 10 s (0 a 9)
            S_AM_VERDE: fin_tiempo = (contador_seg >= 4'd2);  // 3 s  (0 a 2)
            S_VERDE:    fin_tiempo = (contador_seg >= 4'd9);  // 10 s (0 a 9)
            S_AM_ROJO:  fin_tiempo = (contador_seg >= 4'd2);  // 3 s  (0 a 2)
            default:    fin_tiempo = 1'b0;
        endcase
    end

    // =========================================================================
    // BLOQUE 1: Registro de estado (Bloque Secuencial)
    // =========================================================================
    always @(posedge clk or posedge rst) begin
        if (rst) begin
            estado       <= S_ROJO;
            contador_seg <= 4'd0;
        end else if (jk_switch) begin
            estado <= S_ROJO;
        end else if (pulse_1s && !D) begin
            if (fin_tiempo) begin
                estado       <= prox_estado;
                contador_seg <= 4'd0;
            end else begin
                contador_seg <= contador_seg + 1'b1;
            end
        end
    end

    // =========================================================================
    // BLOQUE 2: Lógica de transición (Bloque Combinacional)
    // =========================================================================
    always @(*) begin
        case (estado)
            S_ROJO:     prox_estado = fin_tiempo ? S_AM_VERDE : S_ROJO;
            S_AM_VERDE: prox_estado = fin_tiempo ? S_VERDE    : S_AM_VERDE;
            S_VERDE:    prox_estado = fin_tiempo ? S_AM_ROJO  : S_VERDE;
            S_AM_ROJO:  prox_estado = fin_tiempo ? S_ROJO     : S_AM_ROJO;
            default:    prox_estado = S_ROJO;
        endcase
    end

    // =========================================================================
    // BLOQUE 3: Lógica de salida Moore (Bloque Combinacional)
    // =========================================================================
    always @(*) begin
        // Valores por defecto al inicio para prevenir latches inferidos
        rojo         = 1'b0;
        amarillo     = 1'b0;
        verde        = 1'b0;
        peaton_rojo  = 1'b0;
        peaton_verde = 1'b0;
        buzzer       = 1'b0;

        if (jk_switch) begin
            // Modo Emergencia: Vehicular en Amarillo parpadeante, Peatón en Rojo
            amarillo    = blink;
            peaton_rojo = 1'b1;
        end else begin
            case (estado)
                S_ROJO: begin
                    rojo         = 1'b1;
                    peaton_verde = 1'b1;
                    buzzer       = blink; // Tren de pulsos activo durante cruce peatonal
                end

                S_AM_VERDE: begin
                    amarillo    = 1'b1; // Amarillo fijo en transición
                    peaton_rojo = 1'b1;
                end

                S_VERDE: begin
                    verde       = 1'b1;
                    peaton_rojo = 1'b1;
                end

                S_AM_ROJO: begin
                    amarillo    = 1'b1; // Amarillo fijo en transición
                    peaton_rojo = 1'b1;
                end

                default: begin
                    rojo        = 1'b1;
                    peaton_rojo = 1'b1;
                end
            endcase
        end
    end

endmodule
```

---

## 3. Bancos de Pruebas (Testbenches) y Simulación en GTKWave

Para verificar el comportamiento temporal del semáforo y validar la generación del tren de pulsos del zumbador sin esperar tiempos reales de simulación de física, se diseñó el banco de pruebas `tb_traffic_light.v`.

Este banco de pruebas incluye una rutina de **aceleración temporal en testbench** mediante las sentencias `force` y `release`, dosificando la simulación para que $1\text{ segundo real}$ transcurra cada $100\text{ ns}$ de simulación.

### 3.1 Testbench del Sistema (`tb_traffic_light.v`)

```verilog
`timescale 1ns / 1ps
`include "traffic_light.v"

module tb_traffic_light;

    reg clk;
    reg rst;
    reg sr;
    reg D;
    reg jk_switch;

    wire rojo;
    wire amarillo;
    wire verde;
    wire peaton_rojo;
    wire peaton_verde;
    wire buzzer;

    // Instanciación del módulo principal
    traffic_light uut (
        .clk          (clk),
        .rst          (rst),
        .sr           (sr),
        .D            (D),
        .jk_switch    (jk_switch),
        .rojo         (rojo),
        .amarillo     (amarillo),
        .verde        (verde),
        .peaton_rojo  (peaton_rojo),
        .peaton_verde (peaton_verde),
        .buzzer       (buzzer)
    );

    // Reloj de 50 MHz (Período T = 20 ns)
    always #10 clk = ~clk;

    // ACELERACIÓN TEMPORAL: Genera 1 segundo de conteo cada 100 ns de simulación
    always begin
        #80;
        force uut.timer = uut.max_timer;
        #20;
        release uut.timer;
    end

    initial begin
        // Archivos de volcado para visualización en GTKWave
        $dumpfile("tb_traffic_light.vcd");
        $dumpvars(0, tb_traffic_light);

        // 1. Reset Inicial
        clk       = 1'b0;
        rst       = 1'b1;
        sr        = 1'b0;
        D         = 1'b0;
        jk_switch = 1'b0;

        #100;
        rst = 1'b0; // Inicia ciclo en S_ROJO (10 s simulados = 1000 ns)

        // 2. Transición del ciclo completo normal
        #3000;

        // 3. Activación del sensor de tráfico (Detiene la FSM en el estado actual)
        D = 1'b1;
        #400;
        D = 1'b0;

        // 4. Modo Emergencia (Activa parpadeo amarillo y peatón en rojo)
        #400;
        jk_switch = 1'b1;
        #800;
        jk_switch = 1'b0;

        #1000;
        $finish;
    end

endmodule
```

---

### 3.2 Asignación de Pines para Síntesis en Intel Quartus Prime (`.qsf`)

Las restricciones físicas de pines se configuraron en el archivo `.qsf` del proyecto para mapear los controles a los switches integrados y las salidas al conector GPIO de expansión (JP1) operando al estándar `3.3-V LVTTL`:

```tcl
# =========================================================================
# ASIGNACION DE PINES DE10-LITE (MAX 10 FPGA)
# =========================================================================

set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to clk
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to rst
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to sr
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to D
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to jk_switch
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to rojo
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to amarillo
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to verde
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to peaton_rojo
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to peaton_verde
set_instance_assignment -name IO_STANDARD "3.3-V LVTTL" -to buzzer

# Reloj Interno 50 MHz
set_location_assignment PIN_P11 -to clk

# Entradas en Switches de la tarjeta (SW[0..3])
set_location_assignment PIN_C10 -to rst
set_location_assignment PIN_C11 -to sr
set_location_assignment PIN_D12 -to D
set_location_assignment PIN_C12 -to jk_switch

# Salidas en Conector GPIO JP1 (Pines 1 al 6)
set_location_assignment PIN_V10 -to rojo         # Pin 1 GPIO (GPIO_[0])
set_location_assignment PIN_W10 -to amarillo     # Pin 2 GPIO (GPIO_[1])
set_location_assignment PIN_V9  -to verde        # Pin 3 GPIO (GPIO_[2])
set_location_assignment PIN_W9  -to peaton_rojo  # Pin 4 GPIO (GPIO_[3])
set_location_assignment PIN_V8  -to peaton_verde # Pin 5 GPIO (GPIO_[4])
set_location_assignment PIN_W8  -to buzzer       # Pin 6 GPIO (GPIO_[5])
```

---

## 4. Guías Interactivas

Para complementar la parte teórica, la simulación de máquinas de estado y los diagramas de tiempo en la FPGA, se puede acceder a la guía interactiva a través del siguiente enlace:

🌐 **[Acceder a las Guías Interactivas en Google Sites desde correo ECCI](https://sites.google.com/ecci.edu.co/simuladorfpgade10-lite?usp=sharing)**

---

## 5. Conclusiones

1. **Beneficios de la Estructura de Tres Bloques `always`:** La separación estricta entre el registro de estado secuencial, la lógica de transición combinacional y el bloque de salidas Moore previene la inferencia de *latches* no deseados en Intel Quartus Prime. Esto asegura un circuito digital determinista y de alto rendimiento.
2. **Prevención de Latching por Salidas por Defecto:** La inicialización explícita de las variables de salida (`rojo`, `amarillo`, `verde`, `peaton_rojo`, `peaton_verde`, `buzzer`) a `0` al inicio del bloque de salida Moore garantiza que cada combinación de entradas genere una función lógica combinacional pura sin retención de memoria no deseada.
3. **Generación de Trenes de Pulsos en Periféricos Acústicos:** La asignación de la señal oscilante `blink` a la salida del `buzzer` únicamente durante el estado `S_ROJO` permite generar una señal de audio intermitente efectiva para cruces peatonales sin requerir módulos de hardware adicionales.
4. **Técnicas de Aceleración en Bancos de Prueba (Testbenches):** El uso de manipulaciones temporales como `force` y `release` sobre los registros de conteo del temporizador en el testbench permite simular ciclos de varios segundos en una ventana de nanosegundos en GTKWave, manteniendo el archivo de código Verilog 100% compatible con la FPGA real.
5. **Acondicionamiento Físico y Estándares I/O:** La configuración del estándar de voltaje a `3.3-V LVTTL` en las restricciones de pines del archivo `.qsf` es indispensable para garantizar la integridad eléctrica de las señales enviadas a través de la barra de pines GPIO (JP1) hacia la protoboard externa.
