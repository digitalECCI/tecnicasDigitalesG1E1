# Informe de Laboratorio: Decodificador de 7 Segmentos, Multiplexores y Sumador Combinacional

## Integrantes
* [Andres Mateo Arias Aguilera](https://github.com/mateoaeora124)
* [Gabriel Cangrejo](https://github.com/gabriel11cangrejo)
* [Cesar Alberto Gomez](https://github.com/Cesar7772026)

---

## 1. Introducción y Fundamentos Teóricos

El presente laboratorio integra el diseño y la verificación de circuitos combinacionales avanzados en Verilog HDL, combinando la aritmética binaria, la conmutación por multiplexores y la decodificación para visualización en displays de 7 segmentos de ánodo común.

### 1.1 Decodificador de 7 Segmentos (Ánodo Común)
Un decodificador de 7 segmentos traduce un vector binario de entrada a un conjunto de señales que controlan los segmentos de un display numérico/hexadecimal ($a, b, c, d, e, f, g$). En la configuración de **ánodo común**, los ánodos de los diodos LED están conectados internamente a la fuente de alimentación $V_{CC}$, por lo que la activación ocurre con **lógica activa en bajo**:
- **`0` Lógico:** Permite la conducción de corriente y **enciende** el segmento.
- **`1` Lógico:** Bloquea la corriente y **apaga** el segmento.

Para un bus de entrada de 4 bits ($valor[3:0]$), la representación cubre el rango del 0 al 15 (hexadecimal `0` al `F`). El vector de salida se mapea en el orden $seg[6:0] = \{g, f, e, d, c, b, a\}$.

### 1.2 Multiplexores Combinacionales (MUX)
Los multiplexores son selectores de datos combinacionales que dirigen una de varias líneas de entrada hacia una única salida compartida mediante una señal de control o selección ($sel$). En sistemas con decodificadores y pantallas, permiten alternar dinámicamente entre distintas fuentes de datos (como la entrada directa o el resultado de un sumador) para su visualización.

### 1.3 Prevención de Latches en Verilog
Para garantizar la síntesis de un circuito combinacional puro en Verilog, las sentencias `always @(*)` y las estructuras `case` deben definir un valor de salida para **todas las combinaciones posibles de entrada**. La inclusión explícita de la cláusula `default` evita la inferencia inadvertida de *latches* (elementos de memoria no deseados que introducen retardos y fallos de temporización).

---

## 2. Tablas de Verdad

### 2.1 Tabla de Verdad: Decodificador 7 Segmentos Individual (4 Bits / Hexadecimal)
Describe la decodificación lógica directa de la entrada `valor[3:0]` hacia la salida de segmentos `seg[6:0] = {g, f, e, d, c, b, a}` en lógica de ánodo común (`0` = Encendido, `1` = Apagado):

| `valor[3:0]` (Dec) | `valor[3:0]` (Bin) | Carácter Visualizado | `seg[6:0]` ($g f e d c b a$) | Hexadecimal |
| :---: | :---: | :---: | :---: | :---: |
| 0 | `0000` | 0 | `1000000` | `0x40` |
| 1 | `0001` | 1 | `1111001` | `0x79` |
| 2 | `0010` | 2 | `0100100` | `0x24` |
| 3 | `0011` | 3 | `0110000` | `0x30` |
| 4 | `0100` | 4 | `0011001` | `0x19` |
| 5 | `0101` | 5 | `0010010` | `0x12` |
| 6 | `0110` | 6 | `0000010` | `0x02` |
| 7 | `0111` | 7 | `1111000` | `0x78` |
| 8 | `1000` | 8 | `0000000` | `0x00` |
| 9 | `1001` | 9 | `0011000` | `0x18` |
| 10 | `1010` | A | `0001000` | `0x08` |
| 11 | `1011` | b | `0000011` | `0x03` |
| 12 | `1100` | C | `1000110` | `0x46` |
| 13 | `1101` | d | `0100001` | `0x21` |
| 14 | `1110` | E | `0000110` | `0x06` |
| 15 | `1111` | F | `0001110` | `0x0E` |
| Default | - | Apagado | `1111111` | `0x7F` |

---

### 2.2 Tabla de Verdad: Multiplexor Selector de Datos 2 a 1 (4 Bits)
Selecciona entre dos fuentes de datos de 4 bits para enviarlas al decodificador:

| Selección (`sel`) | Entrada `in0[3:0]` | Entrada `in1[3:0]` | Salida `out[3:0]` | Canal Seleccionado |
| :---: | :---: | :---: | :---: | :---: |
| `0` | Dato $A$ | Dato $B$ / Suma | `in0` ($Dato A$) | Canal 0 |
| `1` | Dato $A$ | Dato $B$ / Suma | `in1` ($Dato B$ / Suma) | Canal 1 |

---

### 2.3 Tabla de Verdad: Sistema Integrado (Sumador de 3 Bits + Decodificador de 4 Bits)
Esta tabla detalla el comportamiento del sistema integrado `top_sum_7seg`, donde las entradas $A[2:0]$ y $B[2:0]$ ($C_i = 0$) se suman, concatenando el acarreo $C_o$ para formar un bus de 4 bits $SalidaT = \{C_o, S_o\}$, el cual se decodifica hacia el display:

| $A[2:0]$ (Dec) | $B[2:0]$ (Dec) | $A_2 A_1 A_0$ | $B_2 B_1 B_0$ | $C_o$ | $S_o[2:0]$ | $SalidaT[3:0]$ | Suma Total | `Sseg[6:0]` ($g f e d c b a$) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 0 | 0 | `000` | `000` | 0 | `000` | `0000` | 0 | `1000000` |
| 0 | 1 | `000` | `001` | 0 | `001` | `0001` | 1 | `1111001` |
| 0 | 2 | `000` | `010` | 0 | `010` | `0010` | 2 | `0100100` |
| 0 | 3 | `000` | `011` | 0 | `011` | `0011` | 3 | `0110000` |
| 0 | 4 | `000` | `100` | 0 | `100` | `0100` | 4 | `0011001` |
| 0 | 5 | `000` | `101` | 0 | `101` | `0101` | 5 | `0010010` |
| 0 | 6 | `000` | `110` | 0 | `110` | `0110` | 6 | `0000010` |
| 0 | 7 | `000` | `111` | 0 | `111` | `0111` | 7 | `1111000` |
| 1 | 0 | `001` | `000` | 0 | `001` | `0001` | 1 | `1111001` |
| 1 | 1 | `001` | `001` | 0 | `010` | `0010` | 2 | `0100100` |
| 1 | 7 | `001` | `111` | 1 | `000` | `1000` | 8 | `0000000` |
| 2 | 6 | `010` | `110` | 1 | `000` | `1000` | 8 | `0000000` |
| 2 | 7 | `010` | `111` | 1 | `001` | `1001` | 9 | `0011000` |
| 3 | 7 | `011` | `111` | 1 | `010` | `1010` | 10 (A) | `0001000` |
| 4 | 7 | `100` | `111` | 1 | `011` | `1011` | 11 (b) | `0000011` |
| 5 | 7 | `101` | `111` | 1 | `100` | `1100` | 12 (C) | `1000110` |
| 6 | 7 | `110` | `111` | 1 | `101` | `1101` | 13 (d) | `0100001` |
| 7 | 7 | `111` | `111` | 1 | `110` | `1110` | 14 (E) | `0000110` |

---

## 3. Código Hardware HDL en Verilog

A continuación se presentan los módulos Verilog implementados en el proyecto.

### 3.1 Módulo Decodificador Base (`7_seg.v`)
Implementación del decodificador de 7 segmentos para valores de 4 bits (0 a 15) en lógica de Ánodo Común:

```verilog
module decod_7seg (
    input      [3:0] valor, // número a mostrar (0 al 15)
    output reg [6:0] seg    // segmentos a..g (orden: g,f,e,d,c,b,a)
);

    always @(*) begin
        case (valor)
            // Lógica Ánodo Común: 0 = Encendido, 1 = Apagado
            4'd0: seg = 7'b1000000; // Enciende a,b,c,d,e,f (apaga g)
            4'd1: seg = 7'b1111001; // Enciende b,c
            4'd2: seg = 7'b0100100; // Enciende a,b,d,e,g
            4'd3: seg = 7'b0110000; // Enciende a,b,c,d,g
            4'd4: seg = 7'b0011001; // Enciende b,c,f,g
            4'd5: seg = 7'b0010010; // Enciende a,c,d,f,g
            4'd6: seg = 7'b0000010; // Enciende a,c,d,e,f,g
            4'd7: seg = 7'b1111000; // Enciende a,b,c
            4'd8: seg = 7'b0000000; // Enciende todos
            4'd9: seg = 7'b0011000; // Enciende a,b,c,f,g
            4'd10: seg = 7'b0001000; // Enciende a,b,c,e,f,g (A)
            4'd11: seg = 7'b0000011; // Enciende c,d,e,f,g (b)
            4'd12: seg = 7'b1000110; // Enciende a,d,e,f (C)
            4'd13: seg = 7'b0100001; // Enciende b,c,d,e,g (d)
            4'd14: seg = 7'b0000110; // Enciende a,d,e,f,g (E)
            4'd15: seg = 7'b0001110; // Enciende a,e,f,g (F)
            // Default cubre cualquier estado no contemplado y evita latches inferidos
            default: seg = 7'b1111111; // Apaga todos los segmentos
        endcase
    end

endmodule
```

### 3.2 Módulo Sumador Aritmético Soportado (`sumador_4_bit.v`)
Módulo jerárquico auxiliar utilizado por la estructura principal para efectuar la suma binaria:

```verilog
// Sumador elemental de 1 bit
module Sumador_1bit (
    input A,
    input B,
    input Ci,
    output So,
    output Co
);
    assign So = A ^ B ^ Ci;
    assign Co = (A & B) | (A & Ci) | (B & Ci);
endmodule

// Sumador de 4 bits por propagación de acarreo (Ripple Carry)
module Sumador_4bit (
    input  [3:0] A,
    input  [3:0] B,
    input        Ci,
    output [2:0] So,
    output       Co
);

    wire C0, C1;

    Sumador_1bit bit0 (.A(A[0]), .B(B[0]), .Ci(Ci), .So(So[0]), .Co(C0));
    Sumador_1bit bit1 (.A(A[1]), .B(B[1]), .Ci(C0), .So(So[1]), .Co(C1));
    Sumador_1bit bit2 (.A(A[2]), .B(B[2]), .Ci(C1), .So(So[2]), .Co(Co));
endmodule
```

### 3.3 Módulo Multiplexor Auxiliar 2 a 1 de 4 Bits (`mux_2a1.v`)
Módulo de conmutación de bus de datos de 4 bits:

```verilog
module mux_2a1 (
    input  [3:0] in0,
    input  [3:0] in1,
    input        sel,
    output [3:0] out
);

    assign out = (sel) ? in1 : in0;
endmodule
```

### 3.4 Módulo Superior Integrador (`top_sum_7seg.v`)
Estructura jerárquica que conecta el sumador de 4 bits con el decodificador de 7 segmentos mediante la concatenación del acarreo de salida $C_o$ y la suma $S_o$ ($SalidaT = \{C_o, S_o\}$):

```verilog
//`include "sumador_4_bit.v"
//`include "7_seg.v"

module top_sum_7seg (
    input  [2:0] A,
    input  [2:0] B,
    input        Ci,
    output [6:0] Sseg
);

    wire [2:0] So;
    wire [3:0] SalidaT;
    wire       Co;

    // Instancia del sumador de 4 bits
    Sumador_4bit sumador (
        .A({1'b0, A}),
        .B({1'b0, B}),
        .Ci(Ci),
        .So(So),
        .Co(Co)
    );

    assign SalidaT = {Co, So}; 

    // Instancia del decodificador de 7 segmentos
    decod_7seg decodificador (
        .valor(SalidaT),
        .seg(Sseg)
    );
endmodule
```

---

## 4. Bancos de Pruebas (Testbenches) y Simulación

### 4.1 Testbench del Decodificador (`7_seg_TB.v`)
Banco de pruebas para la evaluación secuencial de los estímulos en el módulo decodificador base:

```verilog
`include "7_seg.v"
`timescale 1s / 1s

module tb_decod_7seg();

    reg [3:0] valor;
    wire [6:0] seg;

    decod_7seg uut (
        .valor(valor),
        .seg(seg)
    );

    integer i;

    initial begin
        // Bucle para probar los valores posibles
        for (i = 0; i < 10; i = i + 1) begin
            valor = i;
            #10;
        end
    end

    initial begin: TEST_CASE
        $dumpfile("simu.vcd");
        $dumpvars(-1, uut);
        #100 $finish; 
    end
endmodule
```

### 4.2 Testbench del Sistema Integrado (`top_sum_7seg_tb.v`)
Banco de pruebas que recorre las 64 combinaciones posibles de entrada para el módulo `top_sum_7seg` mediante dos bucles anidados:

```verilog
`include "7_seg_4b.v"
`timescale 1s / 1s

module top_sum_7seg_tb();

    reg [2:0] A, B;
    reg Ci;
    wire [6:0] Sseg;

    // Instanciación del módulo Top
    top_sum_7seg uut (
        .A(A),
        .B(B),
        .Ci(Ci),
        .Sseg(Sseg)
    );

    integer i, j;

    initial begin
        // Prueba de todas las combinaciones posibles de 0 a 7
        for (i = 0; i < 8; i = i + 1) begin
            for (j = 0; j < 8; j = j + 1) begin
                A = i;
                B = j;
                Ci = 0;
                #10;
            end
        end
    end

    initial begin: TEST_CASE
        $dumpfile("simu.vcd");
        $dumpvars(-1, uut);
        #700 $finish;
    end
endmodule
```

### 4.3 Análisis y Verificación de Resultados
La compilación y simulación se llevaron a cabo utilizando las herramientas de software libre **Icarus Verilog** (`iverilog`) y **GTKWave**:

1. **Prueba del Decodificador (`tb_decod_7seg`)**:
   - Durante el intervalo $t=0\text{s}$ a $t=10\text{s}$, con `valor = 4'd0`, la salida toma el valor `seg = 7'b1000000` (`0x40`), encendiendo los segmentos $a, b, c, d, e, f$ y apagando el segmento $g$.
   - Para $t=70\text{s}$, con `valor = 4'd7`, la salida cambia a `seg = 7'b1111000` (`0x78`), encendiendo únicamente los segmentos $a, b, c$.

2. **Prueba Integrada (`top_sum_7seg_tb`)**:
   - Al aplicar $A=7$ (`3'b111`) y $B=7$ (`3'b111`), la suma produce $S_o = 6$ (`3'b110`) y un acarreo de salida $C_o = 1$.
   - El vector concatenado $SalidaT = \{C_o, S_o\}$ equivale a `4'b1110` (14 en decimal).
   - El decodificador traduce `4'b1110` como `Sseg = 7'b0000110` (`0x06`), mostrando correctamente la letra hexadecimal `E` en la pantalla.

---

## 5. Sustentación en Video y Recursos Adicionales

- **Video Demostración de Sustentación:** [Ver Video en YouTube](#)
- **Página y Guía Interactiva del Laboratorio:** [Sitio Web Interactivo del Proyecto](#)

---

## 6. Conclusiones

1. **Diseño Modular Jerárquico:** La estructuración del proyecto en bloques independientes (`decod_7seg`, `Sumador_4bit` y `top_sum_7seg`) permitió verificar individualmente la lógica de decodificación antes de integrarla con los componentes aritméticos.
2. **Interpretación y Formato de Lógica Invertida:** La correcta definición del patrón de bits en el decodificador garantiza la compatibilidad con dispositivos físicos de ánodo común, donde cada bit en `0` activa el respectivo diodo emisor de luz.
3. **Evitación de Latches e Integridad Sintetizable:** La especificación de todos los casos posibles (0 a 15) junto con la cláusula `default` en la sentencia `case` asegura la generación de lógica combinacional pura multiplexada, eliminando retroalimentaciones imprevistas o memorias no deseadas.
4. **Verificación Automatizada:** Los bancos de pruebas con bucles deterministas facilitaron la validación completa del espacio de estados de las entradas en tiempo de simulación reducido.

---

## 7. Referencias

- [1] M. M. Mano y M. D. Ciletti, *Digital Design: With an Introduction to the Verilog HDL, VHDL, and SystemVerilog*, 6th ed. Upper Saddle River, NJ, USA: Pearson, 2017.
- [2] IEEE Standard Verilog Hardware Description Language, IEEE Std 1364-2005, 2006.
