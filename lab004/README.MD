# Informe de Laboratorio: Contador Digital y Cronómetro en Display de 7 Segmentos

## Integrantes
- [Andres Mateo Arias Aguilera](https://github.com/mateoaeora124)
- [Gabriel Cangrejo](https://github.com/gabriel11cangrejo)
- [Cesar Alberto Gomez](https://github.com/Cesar7772026)

---

## 1. Introducción y Fundamentos Teóricos

El presente laboratorio aborda el diseño, la simulación y la implementación en hardware mediante Verilog HDL de un **cronómetro digital de alta precisión** multiplexado en tiempo. El sistema es capaz de realizar la medición del tiempo discriminando unidades de milésimas de segundo, centésimas de segundo, décimas de segundo y segundos enteros. 

A diferencia de los circuitos combinacionales puros desarrollados en prácticas anteriores, este sistema integra **lógica secuencial sincronizada por reloj**, permitiendo el almacenamiento de estados dinámicos, la división de frecuencias de reloj de alto rendimiento (FPGA) a frecuencias visibles y el control multiplexado de displays de 7 segmentos de ánodo común.

---

### 1.1 Lógica Secuencial y Máquinas de Estado
La **lógica secuencial** comprende el estudio y diseño de circuitos digitales cuyos estados de salida $Y(t)$ dependen no únicamente del vector de entradas actuales $X(t)$, sino también del historial o estado interno almacenado $Q(t)$:

$$Y(t) = f(X(t), Q(t))$$

$$Q(t+1) = g(X(t), Q(t))$$

En un sistema sincrónico, las transiciones de estado ocurren exclusivamente en instantes de tiempo determinados por los flancos (de subida o bajada) de una señal de reloj global ($\text{CLK}$). Esto garantiza la eliminación de condiciones de carrera (*race conditions*) y permite estructurar máquinas de estado finitas (FSM) robustas para el control numérico y temporal.

---

### 1.2 Elementos de Memoria y Registros de Almacenamiento

#### 1.2.1 Flip-Flops Base
Los Flip-Flops son las unidades biestables fundamentales capaces de almacenar $1\text{ bit}$ de información:
- **Flip-Flop Tipo D (Data):** Transfiere el valor presente en la entrada $D$ a la salida $Q$ exactamente en el flanco activo del reloj ($\text{posedge CLK}$). Es la estructura predilecta para la síntesis de contadores y registros en FPGAs.
- **Flip-Flop Tipo T (Toggle):** Conmuta el estado de su salida ($Q(t+1) = \overline{Q(t)}$) si la entrada $T=1$ al llegar el flanco de reloj.
- **Flip-Flop SR y JK:** Estructuras conmutativas que permiten operaciones de activación ($\text{Set}$), desactivación ($\text{Reset}$) y conmutación ($\text{Toggle}$), evitando el estado indeterminado del SR clásico.
---
![Flip-Flop Tipo D y Tipo T](figs/flipflop.jpg)
---
#### 1.2.2 Registros de Datos y Desplazamiento
Un registro es un arreglo n-dimensional de Flip-Flops sincronizados por la misma línea de reloj. En este laboratorio, los registros de almacenamiento permiten retener las cuentas BCD (*Binary-Coded Decimal*) correspondientes a las milésimas, centésimas, décimas y segundos.

---

### 1.3 Contadores Digitales y Codificación BCD (Binary-Coded Decimal)
Un **contador digital** es un circuito secuencial estructurado por una cadena de Flip-Flops que recorre una secuencia binaria predeterminada de estados ante cada pulso de reloj.

#### Contadores Módulo N y Cascada BCD (Módulo 10)
Para representar de forma natural las fracciones de segundo y los segundos enteros en pantallas decimales, los contadores se configuran en código **BCD (Módulo 10)**, restringiendo la cuenta al rango $0 \rightarrow 9$ ($0000_2 \rightarrow 1001_2$). 

Cuando un contador alcanza su valor máximo ($N-1 = 9$), genera una señal de acarreo de salida (*Carry Out / Enable Out*) que actúa como la señal de habilitación de conteo para la siguiente etapa de mayor jerarquía:

$$\text{Frecuencia Flujo Contador Extensión} = \frac{f_{\text{Entrada}}}{10}$$
---
![Contadores BCD en Cascada](figs/cascada.png)
---
### 1.4 Divisor de Frecuencia por Contador Módulo N
Las FPGAs disponen de osciladores de cristal integrados que generan señales de reloj de alta frecuencia (por ejemplo, $50\text{ MHz}$ o $100\text{ MHz}$). Para registrar intervalos temporales precisos como $1\text{ ms}$ ($1000\text{ Hz}$), es indispensable diseñar un **divisor de frecuencia**.

Un divisor de frecuencia se implementa mediante un registro contador de paso de tamaño $k$ bits ($2^k \ge N$) con el valor límite $N$:

$$N = \frac{f_{\text{FPGA}}}{f_{\text{Deseada}}}$$

#### Ejemplo Matemático de Cómputo para $1\text{ ms}$:
Para un reloj de entrada en la FPGA de $f_{\text{FPGA}} = 50\text{ MHz}$ y una frecuencia objetivo de $f_{\text{salida}} = 1000\text{ Hz}$ ($1\text{ ms}$):

$$N = \frac{50 \times 10^6\text{ Hz}}{1000\text{ Hz}} = 50.000\text{ ciclos}$$

El contador acumula del $0$ al $49.999$. Al alcanzar $N-1$, conmuta una señal de pulso habilitador de $1$ ciclo de reloj y reinicia la cuenta a $0$, logrando una base de tiempo exacta y sin fluctuación (*jitter*) para la medición del cronómetro.

---

### 1.5 Multiplexación Temporal de Displays de 7 Segmentos
Para visualizar múltiples dígitos (milésimas, centésimas, décimas y segundos) sin utilizar una cantidad excesiva de pines de la FPGA, se utiliza la **multiplexación en el dominio del tiempo**:
1. Los buses de segmentos $seg[6:0]$ de todos los displays se conectan en paralelo a un único bus común.
2. Las líneas de selección de ánodos de cada display ($an[3:0]$) se activan de manera secuencial e individual a una frecuencia superior a $60\text{ Hz}$ (típicamente entre $100\text{ Hz}$ y $1\text{ kHz}$).
3. Gracias al efecto de la **persistencia retiniana** del ojo humano, la alternancia rápida crea la ilusión de que todos los displays están encendidos de forma continua sin parpadeo (*flicker*).
