# Lab01 - Introducción a la lógica combinacional

## Integrantes
  * [Andres Mateo Arias Aguilera](https://github.com/mateoaeora124)
  * Gabriel Cangrejo
  * Cesar Alberto Gomez
## informe

## Índice
1. [Introducción](#1-introducción)
2. [Simulación](#2-simulación)
3. [Evidencias](#3-evidencias)
4. [Guías interactivas](#4-guías-interactivas)
5. [Conclusiones](#5-conclusiones)
6. [Referencias](#6-referencias)

## 1. Introducción

### 1.1 Compuertas lógicas

#### Descripción

En este apartado se introducen los conceptos fundamentales de las **compuertas lógicas** y sus principios de funcionamiento. La comprensión de estos elementos teóricos es indispensable para validar y analizar las simulaciones de los circuitos digitales desarrolladas en las secciones posteriores de este laboratorio.
de esta manera podemos encontrar las siguientes compuertas lógicas básicas, las cuales son la base de la lógica de todo circuito digital:

![Compuertas Lógicas - Símbolos y Tablas de Verdad](https://github.com/user-attachments/assets/20d1d522-3ee2-472d-9aee-82ce33ca14d4)

---

### 1.2 Clasificación y Descripción de Compuertas Lógicas

* **AND (Y Lógica):** Entrega un nivel alto (`1`) en su salida únicamente cuando todas sus entradas se encuentran en nivel alto. Realiza la multiplicación booleana ($Y = A \cdot B$).
* **OR (O Lógica):** Entrega un nivel alto (`1`) si al menos una de sus entradas se encuentra en nivel alto. Realiza la suma booleana ($Y = A + B$).
* **NOT (Inversor):** Posee una sola entrada e invierte el estado lógico recibido ($Y = \bar{A}$).
* **NAND (NO-Y):** Salida equivalente a una compuerta AND seguida de un inversor. Su salida es nivel bajo (`0`) únicamente cuando todas sus entradas son altas ($Y = \overline{A \cdot B}$). Es una compuerta universal.
* **NOR (NO-O):** Salida equivalente a una compuerta OR seguida de un inversor. Su salida es nivel alto (`1`) únicamente cuando todas sus entradas son bajas ($Y = \overline{A + B}$). Es una compuerta universal.
* **XOR (O Exclusiva):** Entrega un nivel alto (`1`) en su salida únicamente cuando sus entradas presentan estados lógicos diferentes entre sí ($Y = A \oplus B$).
* **XNOR (NO-O Exclusiva):** Entrega un nivel alto (`1`) únicamente cuando sus entradas presentan estados lógicos iguales ($Y = \overline{A \oplus B}$).

---

### 1.3 Representación Operacional en Código y Álgebra Booleana

Para la descripción e implementación digital en lenguajes de programación y de descripción de hardware (HDL), las operaciones lógicas se representan mediante los siguientes operadores:

| Compuerta | Función Booleana | Operador Verilog | Operador (primitivas) VHDL |
| :--- | :---: | :---: | :---: |
| **AND** | $A \cdot B$ | `&` | `and` |
| **OR** | $A + B$ | `\|` | `or` |
| **NOT** | $\bar{A}$ | `~` | `not` |
| **NAND** | $\overline{A \cdot B}$ | `~(A & B)` | `nand` |
| **NOR** | $\overline{A + B}$ | `~(A \| B)` | `nor` |
| **XOR** | $A \oplus B$ | `^` | `xor` |
| **XNOR** | $\overline{A \oplus B}$ | `~(A ^ B)` | `xnor` |





