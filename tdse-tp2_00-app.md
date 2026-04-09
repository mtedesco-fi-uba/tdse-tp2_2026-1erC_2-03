### 1. Análisis del Funcionamiento de los Módulos

* **`app.c`:** Es el núcleo de tu aplicación. Contiene el bucle principal (`app_update()`) y la inicialización (`app_init()`). Su función principal aquí es actuar como un "Cyclic Executive" (ejecutivo cíclico), despachando las tareas de forma secuencial y midiendo el tiempo que tarda cada una en ejecutarse usando la estructura `task_dta_list`.
* **`logger.c` y `logger.h`:** Implementan un sistema de registro de mensajes (logs). Proveen macros como `LOGGER_INFO()` para enviar texto a través de un puerto serie (UART) o semihosting, lo cual es vital para depuración.
* **`systick.c`:** Maneja el temporizador de sistema de ARM (SysTick), que genera una interrupción periódica (generalmente cada 1 milisegundo). Es la base de tiempo macroscópica del sistema para saber cuándo lanzar ciertas tareas.
* **`dwt.h`:** Hace referencia al *Data Watchpoint and Trace*. Este es un bloque de hardware dentro del procesador Cortex-M que incluye un contador de ciclos de reloj de 32 bits (CYCCNT). Es capaz de contar a la velocidad exacta del núcleo (ej. a 64 MHz), lo que permite medir tiempos a nivel de **microsegundos (µs)** de forma extremadamente precisa sin sobrecargar el procesador.

---

### 2. Evolución de las Variables

A continuación, se detalla la evolución de las variables durante el ciclo de vida del programa.

* **`index` (Adimensional):** Es el índice del arreglo utilizado en el bucle `for` o `while` dentro de `app_update()` para recorrer la lista de tareas.
* **`g_app_tick_cnt` (Milisegundos - ms):** Es el contador global del sistema. Se incrementa asíncronamente en el manejador de interrupciones del SysTick.
* **`g_app_runtime_us` (Microsegundos - µs):** Tiempo de ejecución medido a través del DWT, representando cuánto tiempo ha estado corriendo la aplicación o una porción específica de la misma.
* **Métricas de `task_dta_list[index]`:**
    * **`NOE` (Number Of Executions - Adimensional):** Cuenta cuántas veces se ha ejecutado la tarea.
    * **`LET` (Last Execution Time - µs):** El tiempo que tardó la tarea en su última ejecución.
    * **`BCET` (Best Case Execution Time - µs):** El menor tiempo registrado para ejecutar la tarea.
    * **`WCET` (Worst Case Execution Time - µs):** El mayor tiempo registrado para ejecutar la tarea.

#### Tabla de Evolución Temporal

| Variable | Durante `app_init()` | 1ra ejecución en `app_update()` | Ejecuciones sucesivas en `app_update()` |
| :--- | :--- | :--- | :--- |
| **`g_app_tick_cnt`** | `0` (Se resetea). | Incrementado automáticamente de fondo por hardware (ej. `1`, `2`...). | Sigue creciendo asíncronamente de a 1 ms. |
| **`index`** | - | Va de `0` hasta la cantidad total de tareas configuradas. | Se repite el ciclo (`0` al máximo) en cada vuelta del loop principal. |
| **`.NOE`** | `0` | Pasa a valer `1` después de que la tarea se ejecuta. | Se incrementa en +1 en cada ejecución (`2`, `3`, `4`...). |
| **`.LET`** | `0` | Adquiere el valor del tiempo recién medido (ej. `15 µs`). | Se sobreescribe con el nuevo tiempo medido en cada vuelta. |
| **`.BCET`** | Valor máximo (ej. `0xFFFFFFFF`) para forzar un reemplazo. | Adquiere el valor de `LET` porque `LET < BCET_INICIAL`. | Se actualiza **solo si** el nuevo `LET` es menor que el `BCET` histórico. |
| **`.WCET`** | `0` | Adquiere el valor de `LET` porque `LET > 0`. | Se actualiza **solo si** el nuevo `LET` es mayor que el `WCET` histórico. |
| **`g_app_runtime_us`**| `0` | Acumula o mide el tiempo total de la iteración. | Crece a medida que el DWT sigue contando los ciclos de reloj acumulados. |

---

### 3. El Impacto de usar `LOGGER_INFO()`

Utilizar funciones de log como `LOGGER_INFO()` dentro de una de las tareas o en el flujo principal tiene un **impacto drástico y negativo** sobre los tiempos de ejecución.

* **Impacto en `task_dta_list[index].WCET` (Tiempo del peor caso):** Imprimir por un puerto serie (como el USART2 que configuraste a 115200 baudios en el `main.c` previo) es un proceso extremadamente lento para el procesador. Enviar un simple mensaje de 20 caracteres por UART bloquea o retrasa la ejecución durante varios milisegundos. Como resultado, el tiempo de la última ejecución (`LET`) se disparará. Dado que este nuevo valor de `LET` será excepcionalmente alto, **sobrescribirá inmediatamente el `WCET`**, arruinando cualquier métrica previa de tiempo real que tuvieras para esa tarea.
* **Impacto en `g_app_runtime_us`:**
    Al tardar más tiempo ejecutando la impresión del log, el contador global de microsegundos registrará un aumento sustancial. El tiempo total de tu ciclo (el "runtime" de la iteración) ya no reflejará el costo computacional real de tus algoritmos, sino el tiempo que el microcontrolador pasó esperando para enviar bytes por el cable.

> **Consejo práctico:** En sistemas de tiempo real, los *logs* deben desactivarse durante las pruebas de rendimiento (profiling), o bien, deben implementarse utilizando técnicas de memoria en anillo (Ring Buffers) no bloqueantes, para que el `WCET` represente puramente la lógica de control.

### Evolución de variables
NOE: adimensional  
LET, BCET, WCET: µs  
  
<img width="558" height="406" alt="image" src="https://github.com/user-attachments/assets/9b1a089d-b3c8-4da0-9458-970c94217ecb" />

