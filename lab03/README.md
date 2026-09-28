# Lab03 - Decodificador de 7 Segmentos, Multiplexores y Sumador Combinacional

## Integrantes
* [Andres Mateo Arias Aguilera](https://github.com/mateoaeora124)
* [Gabriel Cangrejo](https://github.com/gabriel11cangrejo)
* [Cesar Alberto Gomez](https://github.com/Cesar7772026)

---

## 1. Introducción y Fundamentos Teóricos

El presente laboratorio integra el diseño y la verificación de circuitos combinacionales complejos mediante la combinación de aritmética binaria, conmutación por multiplexores y decodificación para la visualización de datos en un display de 7 segmentos de ánodo común.

### 1.1 Decodificador de 7 Segmentos (Ánodo Común)
Un decodificador de 7 segmentos es un circuito combinacional que traduce una palabra binaria de $N$ bits a un patrón específico de activación en las 7 barras luminosas ($a..g$) de un indicador numérico/hexadecimal. En la configuración de **ánodo común**, los ánodos de los LEDs están interconectados a la línea de alimentación $V_{CC}$. Por ende, la lógica de control utiliza **activa en bajo**:
* **`0` Lógico:** Permite el flujo de corriente y **enciende** el segmento.
* **`1` Lógico:** Iguala el potencial y **apaga** el segmento.

Para un bus de entrada de 4 bits ($valor[3:0]$), la representación abarca del 0 al 15 (hexadecimal `0` al `F`). La expresión para el vector de salida $seg[6:0] = \{g, f, e, d, c, b, a\}$ sigue la regla:

$$seg_i = \begin{cases} 0 & \text{si el segmento } i \text{ se enciende (ON)} \\ 1 & \text{si el segmento } i \text{ se apaga (OFF)} \end{cases}$$

### 1.2 Multiplexores Combinacionales (MUX)
Los multiplexores son selectores de datos combinacionales que canalizan una de varias líneas de entrada hacia una única línea de salida compartida, basándose en el estado de una o más señales de control o selección ($sel$). En sistemas con displays numéricos, los multiplexores permiten:
1. Seleccionar qué dato de origen (por ejemplo, operando directo $A$, operando $B$, o el resultado de la suma $S_o$) se envía al decodificador de 7 segmentos.
2. Multiplexar en el tiempo múltiples dígitos para utilizar un número reducido de líneas de salida físicas hacia las pantallas.

### 1.3 Prevención de Latches en HDLs
En lenguajes de descripción de hardware (HDL) como Verilog, la descripción de bloques combinacionales mediante las sentencias `always @(*)` y `case` exige cubrir **todos los estados posibles de entrada**. La inclusión explícita de la cláusula `default` evita la inferencia accidental de *latches* (elementos de memoria imperceptibles que degradan el rendimiento temporal del sistema y provocan fallos en la síntesis).

---

## 2. Tablas de Verdad

### 2.1 Tabla de Verdad: Decodificador 7 Segmentos Individual (4 Bits / Hexadecimal)
Esta tabla describe la decodificación lógica directa del bus de entrada $valor[3:0]$ hacia el bus de salida de segmentos $seg[6:0] = \{g, f, e, d, c, b, a\}$ en lógica de ánodo común (`0` = ON, `1` = OFF):

| $valor[3:0]$ (Dec) | $valor[3:0]$ (Bin) | Carácter Visualizado | $seg[6:0]$ ($g f e d c b a$) | Hexadecimal |
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
El multiplexor de 4 bits permite elegir entre la salida directa de un operando de entrada o el resultado del sumador para enviarlo hacia el decodificador:

| Selección ($sel$) | Entrada $In_0[3:0]$ | Entrada $In_1[3:0]$ | Salida $Out[3:0]$ | Descripción del Canal |
| :---: | :---: | :---: | :---: | :---: |
| `0` | $Dato A$ | $Dato B$ | $In_0$ ($Dato A$) | Canalización directa del Operando A |
| `1` | $Dato A$ | $Suma$ | $In_1$ ($Suma$) | Canalización del Resultado del Sumador |

---

### 2.3 Tabla de Verdad: Sistema Integrado (Sumador de 3 Bits + Decodificador de 4 Bits)
Muestra el comportamiento completo del sumador combinacional con entradas de 3 bits $A[2:0]$ y $B[2:0]$ ($C_i = 0$), y su salida decodificada hacia el display de 7 segmentos:

| $A[2:0]$ (Dec) | $B[2:0]$ (Dec) | $A_2 A_1 A_0$ | $B_2 B_1 B_0$ | $C_o$ | $S_o[3:0]$ (Bin) | Suma Total | $S_{seg}[6:0]$ ($g f e d c b a$) |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 0 | 0 | `000` | `000` | 0 | `0000` | 0 | `1000000` |
| 0 | 1 | `000` | `001` | 0 | `0001` | 1 | `1111001` |
| 0 | 2 | `000` | `010` | 0 | `0010` | 2 | `0100100` |
| 0 | 3 | `000` | `011` | 0 | `0011` | 3 | `0110000` |
| 0 | 4 | `000` | `100` | 0 | `0100` | 4 | `0011001` |
| 0 | 5 | `000` | `101` | 0 | `0101` | 5 | `0010010` |
| 0 | 6 | `000` | `110` | 0 | `0110` | 6 | `0000010` |
| 0 | 7 | `000` | `111` | 0 | `0111` | 7 | `1111000` |
| 1 | 0 | `001` | `000` | 0 | `0001` | 1 | `1111001` |
| 1 | 1 | `001` | `001` | 0 | `0010` | 2 | `0100100` |
| 1 | 2 | `001` | `010` | 0 | `0011` | 3 | `0110000` |
| 1 | 3 | `001` | `011` | 0 | `0100` | 4 | `0011001` |
| 1 | 4 | `001` | `100` | 0 | `0101` | 5 | `0010010` |
| 1 | 5 | `001` | `101` | 0 | `0110` | 6 | `0000010` |
| 1 | 6 | `001` | `110` | 0 | `0111` | 7 | `1111000` |
| 1 | 7 | `001` | `111` | 0 | `1000` | 8 | `0000000` |
| 2 | 0 | `010` | `000` | 0 | `0010` | 2 | `0100100` |
| 2 | 1 | `010` | `001` | 0 | `0011` | 3 | `0110000` |
| 2 | 2 | `010` | `010` | 0 | `0100` | 4 | `0011001` |
| 2 | 3 | `010` | `011` | 0 | `0101` | 5 | `0010010` |
| 2 | 4 | `010` | `100` | 0 | `0110` | 6 | `0000010` |
| 2 | 5 | `010` | `101` | 0 | `0111` | 7 | `1111000` |
| 2 | 6 | `010` | `110` | 0 | `1000` | 8 | `0000000` |
| 2 | 7 | `010` | `111` | 0 | `1001` | 9 | `0011000` |
| 3 | 0 | `011` | `000` | 0 | `0011` | 3 | `0110000` |
| 3 | 1 | `011` | `001` | 0 | `0100` | 4 | `0011001` |
| 3 | 2 | `011` | `010` | 0 | `0101` | 5 | `0010010` |
| 3 | 3 | `011` | `011` | 0 | `0110` | 6 | `0000010` |
| 3 | 4 | `011` | `100` | 0 | `0111` | 7 | `1111000` |
| 3 | 5 | `011` | `101` | 0 | `1000` | 8 | `0000000` |
| 3 | 6 | `011` | `110` | 0 | `1001` | 9 | `0011000` |
| 3 | 7 | `011` | `111` | 0 | `1010` | 10 | `0001000` |
| 4 | 0 | `100` | `000` | 0 | `0100` | 4 | `0011001` |
| 4 | 1 | `100` | `001` | 0 | `0101` | 5 | `0010010` |
| 4 | 2 | `100` | `010` | 0 | `0110` | 6 | `0000010` |
| 4 | 3 | `100` | `011` | 0 | `0111` | 7 | `1111000` |
| 4 | 4 | `100` | `100` | 0 | `1000` | 8 | `0000000` |
| 4 | 5 | `100` | `101` | 0 | `1001` | 9 | `0011000` |
| 4 | 6 | `100` | `110` | 0 | `1010` | 10 | `0001000` |
| 4 | 7 | `100` | `111` | 0 | `1011` | 11 | `0000011` |
| 5 | 0 | `101` | `000` | 0 | `0101` | 5 | `0010010` |
| 5 | 1 | `101` | `001` | 0 | `0110` | 6 | `0000010` |
| 5 | 2 | `101` | `010` | 0 | `0111` | 7 | `1111000` |
| 5 | 3 | `101` | `011` | 0 | `1000` | 8 | `0000000` |
| 5 | 4 | `101` | `100` | 0 | `1001` | 9 | `0011000` |
| 5 | 5 | `101` | `101` | 0 | `1010` | 10 | `0001000` |
| 5 | 6 | `101` | `110` | 0 | `1011` | 11 | `0000011` |
| 5 | 7 | `101` | `111` | 0 | `1100` | 12 | `1000110` |
| 6 | 0 | `110` | `000` | 0 | `0110` | 6 | `0000010` |
| 6 | 1 | `110` | `001` | 0 | `0111` | 7 | `1111000` |
| 6 | 2 | `110` | `010` | 0 | `1000` | 8 | `0000000` |
| 6 | 3 | `110` | `011` | 0 | `1001` | 9 | `0011000` |
| 6 | 4 | `110` | `100` | 0 | `1010` | 10 | `0001000` |
| 6 | 5 | `110` | `101` | 0 | `1011` | 11 | `0000011` |
| 6 | 6 | `110` | `110` | 0 | `1100` | 12 | `1000110` |
| 6 | 7 | `110` | `111` | 0 | `1101` | 13 | `0100001` |
| 7 | 0 | `111` | `000` | 0 | `0111` | 7 | `1111000` |
| 7 | 1 | `111` | `001` | 0 | `1000` | 8 | `0000000` |
| 7 | 2 | `111` | `010` | 0 | `1001` | 9 | `0011000` |
| 7 | 3 | `111` | `011` | 0 | `1010` | 10 | `0001000` |
| 7 | 4 | `111` | `100` | 0 | `1011` | 11 | `0000011` |
| 7 | 5 | `111` | `101` | 0 | `1100` | 12 | `1000110` |
| 7 | 6 | `111` | `110` | 0 | `1101` | 13 | `0100001` |
| 7 | 7 | `111` | `111` | 0 | `1110` | 14 | `0000110` |

---

## 3. Descripción del Hardware en Verilog (HDL)

El diseño del sistema se encuentra estructurado de forma modular y jerárquica en Verilog.

### 3.1 Módulo Decodificador Base de 7 Segmentos (`7_seg.v`)

Módulo que decodifica una palabra binaria de 4 bits `valor[3:0]` a las salidas de 7 segmentos en lógica invertida (Ánodo Común):

```verilog
module decod_7seg (
    input      [3:0] valor, // número a mostrar (0 al 15 / Hexadecimal)
    output reg [6:0] seg    // segmentos a..g (orden: g,f,e,d,c,b,a)
);

    always @(*) begin
        case (valor)
            // Lógica Ánodo Común: 0 = Encendido, 1 = Apagado
            4'd0:  seg = 7'b1000000; // Enciende a,b,c,d,e,f (apaga g)
            4'd1:  seg = 7'b1111001; // Enciende b,c
            4'd2:  seg = 7 01010100; // Enciende a,b,d,e,g
            4'd3:  seg = 7'b0110000; // Enciende a,b,c,d,g
            4'd4:  seg = 7'b0011001; // Enciende b,c,f,g
            4'd5:  seg = 7'b0010010; // Enciende a,c,d,f,g
            4'd6:  seg = 7'b0000010; // Enciende a,c,d,e,f,g
            4'd7:  seg = 7'b1111000; // Enciende a,b,c
            4'd8:  seg = 7'b0000000; // Enciende a,b,c,d,e,f,g
            4'd9:  seg = 7'b0011000; // Enciende a,b,c,f,g
            4'd10: seg = 7'b0001000; // Enciende a,b,c,e,f,g (Carácter A)
            4'd11: seg = 7'b0000011; // Enciende c,d,e,f,g (Carácter b)
            4'd12: seg = 7'b1000110; // Enciende a,d,e,f (Carácter C)
            4'd13: seg = 7'b0100001; // Enciende b,c,d,e,g (Carácter d)
            4'd14: seg = 7'b0000110; // Enciende a,d,e,f,g (Carácter E)
            4'd15: seg = 7'b0001110; // Enciende a,e,f,g (Carácter F)
            
            // Default cubre cualquier estado no contemplado y evita latches inferidos
            default: seg = 7'b1111111; // Apaga todos los segmentos
        endcase
    end

endmodule
