
---

### 1. Análisis de los Archivos Fuente

#### `startup_stm32f103rbtx.s` (Código de Inicio en Ensamblador)
Este archivo es lo primero que se ejecuta cuando el microcontrolador se enciende o se reinicia. Es el puente entre el hardware puro y tu código en C. Sus funciones principales son:
* **Definición de la Tabla de Vectores (`g_pfnVectors`):** Crea una tabla en la memoria que le dice al procesador en qué dirección de memoria está el código para manejar cada interrupción (por ejemplo, dónde ir cuando ocurre un Reset, un error de hardware o salta un temporizador).
* **Inicialización de Memoria (`Reset_Handler`):** Cuando ocurre un Reset, el procesador salta aquí. Este código copia los valores iniciales de tus variables globales/estáticas desde la memoria Flash (no volátil) a la memoria RAM (sección `.data`) y rellena con ceros las variables no inicializadas (sección `.bss`).
* **Salto a C:** Llama a `SystemInit` (para configuración básica del sistema) y finalmente ejecuta un salto (`bl main`) a la función `main()` en tu archivo `main.c`.

#### `main.c` (Cuerpo Principal del Programa)
Aquí reside el flujo principal de configuración y ejecución de tu aplicación.
* **Inicialización de la HAL (`HAL_Init()`):** Inicializa la capa de abstracción de hardware (Hardware Abstraction Layer), configura el Flash prefetch y establece el SysTick para generar interrupciones cada 1 milisegundo.
* **Configuración del Reloj (`SystemClock_Config()`):** Ajusta los osciladores y multiplicadores (PLL) para definir la velocidad a la que "late" el microcontrolador.
* **Inicialización de Periféricos:** Llama a `MX_GPIO_Init()` para configurar pines (como el botón `B1_Pin` y el LED `LD2_Pin`) y `MX_USART2_UART_Init()` para habilitar la comunicación serie.
* **Bucle Principal:** Llama a `app_init()` (tu código de usuario) y entra en el `while(1)`, donde ejecuta continuamente `app_update()`.

#### `stm32f1xx_it.c` (Manejadores de Interrupciones)
Este archivo contiene las Rutinas de Servicio de Interrupción (ISRs). Son funciones que el procesador ejecuta automáticamente cuando ocurre un evento de hardware, interrumpiendo lo que sea que esté haciendo en el `main()`.
* **`SysTick_Handler()`:** Se ejecuta cada 1 milisegundo. Llama a `HAL_IncTick()`, que simplemente incrementa una variable global usada por la librería HAL para llevar la cuenta del tiempo (usada en funciones como `HAL_Delay()`).
* **`EXTI15_10_IRQHandler()`:** Se dispara cuando ocurre un cambio de estado físico en los pines 10 a 15 (en este caso, cuando se presiona el botón configurado en `B1_Pin`). Delega la acción a `HAL_GPIO_EXTI_IRQHandler()`.

---

### 2. Evolución de `SysTick` y `SystemCoreClock`

Vamos a seguir el rastro desde el Reset hasta el bucle `while(1)`.

**Paso 1: `Reset_Handler` (en `startup_stm32f103rbtx.s`)**
* **`SystemCoreClock`:** Al arrancar, el microcontrolador utiliza su oscilador interno (HSI) por defecto, que normalmente es de 8 MHz. Si la función `SystemInit` (llamada desde aquí) no hace cambios profundos en el PLL, `SystemCoreClock` vale 8,000,000 (8 MHz).
* **`SysTick`:** El temporizador de sistema está apagado. El hardware no está contando y no hay interrupciones.

**Paso 2: Inicio de `main()` -> `HAL_Init()`**
* **`SystemCoreClock`:** Sigue en su valor por defecto (ej. 8 MHz).
* **`SysTick`:** `HAL_Init()` enciende el temporizador SysTick para que genere una interrupción cada 1 ms utilizando el reloj actual de 8 MHz. A partir de este momento, la variable interna de la HAL (`uwTick`) comienza a incrementar de 0 en adelante, aumentando 1 unidad cada milisegundo a través del `SysTick_Handler`.

**Paso 3: `SystemClock_Config()` (en `main.c`)**
Aquí ocurre el cambio drástico de velocidad. Observando el código:
* Usa el oscilador interno (HSI).
* Configura el PLL multiplicando la fuente por 16 (`RCC_PLL_MUL16`) y dividiendo la fuente (HSI) por 2 (`RCC_PLLSOURCE_HSI_DIV2`).
* Cálculo: (8 MHz / 2) * 16 = **64 MHz**.
* **`SystemCoreClock`:** Pasa de 8 MHz a **64 MHz**. La librería HAL actualiza automáticamente esta variable interna.
* **`SysTick`:** Al final de la configuración del reloj, la HAL reconfigura automáticamente el temporizador SysTick. Como el procesador ahora va más rápido (64 MHz), ajusta el valor de recarga (reload value) del SysTick para garantizar que siga interrumpiendo **exactamente cada 1 ms**, sin verse afectado por el aumento de velocidad.

**Paso 4: Llegada al `while(1)`**
* **`SystemCoreClock`:** Se mantiene estable y constante en **64 MHz** (64,000,000 Hz).
* **`SysTick`:** El contador sigue funcionando de fondo. La variable del "tick" (incrementada por `HAL_IncTick()`) ya tendrá un valor mayor a cero, dependiendo de cuántos milisegundos hayan pasado inicializando periféricos (`MX_GPIO_Init`, `app_init`, etc.). Seguirá incrementando de a 1 indefinidamente mientras el programa esté en el bucle principal.

---
