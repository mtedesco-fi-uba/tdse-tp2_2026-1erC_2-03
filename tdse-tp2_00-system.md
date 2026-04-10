Este conjunto de archivos representa el **"cerebro"** de tu aplicación y la forma en que este se comunica con el mundo exterior. Aquí vemos cómo la tarea principal (el Sistema) procesa los eventos que le llegan (por ejemplo, del botón) y cómo toma decisiones para enviar órdenes a las salidas (el Actuador, que en este caso es un LED).

Aquí tienes el análisis detallado del funcionamiento y la evolución de las variables paso a paso.

---

### 1. Análisis del Funcionamiento de los Módulos

* **`task_system_attribute.h` / `task_actuator_attribute.h`:**
    Definen los "vocabularios" (Eventos y Estados) y las estructuras de datos en memoria (RAM) para el Sistema y el Actuador. 
    * El Sistema maneja estados como `ST_SYS_IDLE` y `ST_SYS_ACTIVE`.
    * El Actuador maneja estados análogos para el LED (`ST_LED_IDLE`, `ST_LED_ACTIVE`).
* **`task_system.c`:**
    Es la lógica central. Contiene una Máquina de Estados que evalúa qué hacer basándose en su estado actual y en los mensajes (eventos) que extrae de la cola. Funciona como el **Consumidor** de la cola de eventos del sensor y como el **Productor** de comandos para el actuador.
* **`task_system_interface.c`:**
    Contiene la misma cola circular que analizamos previamente, pero aquí vemos su uso desde la perspectiva del lector (el Sistema consume los eventos llamando a `get_event_task_system()`).
* **`task_actuator_interface.c`:**
    Proporciona la interfaz para enviar órdenes al Actuador. A diferencia del Sistema, el Actuador **no usa una cola circular**, sino un mecanismo más simple de comunicación directa mediante un *Flag* (bandera) y la sobreescritura del evento.

---

### 2. Evolución de las Variables del Sistema (`task_system_dta_list`)

En `task_system.c`, en lugar de un `index` numérico arbitrario, se utiliza el modo del sistema (`g_task_system_mode`, por ejemplo, `NORMAL` que equivale a `0`) como índice para acceder a los datos.

| Variable | En `task_system_init()` | Durante `task_system_update()` (con eventos entrantes) |
| :--- | :--- | :--- |
| **`index` (o `NORMAL`)** | Se inicializa la posición correspondiente al modo por defecto. | Permanece constante (ej. `0`) mientras el sistema esté en modo `NORMAL`. |
| **`.tick` (Milisegundos)** | `0` | Se usa típicamente como temporizador (ej. si el sistema debiera apagarse solo tras X segundos). |
| **`.state`** | `ST_SYS_IDLE` (Reposo) | Pasa a `ST_SYS_ACTIVE` cuando recibe un evento válido y cumple la condición del estado. |
| **`.event`** | `EV_SYS_IDLE` (Sin acción) | Toma el valor extraído de la cola circular mediante `get_event_task_system()` (ej. `EV_SYS_ACTIVE`). |
| **`.flag` (Booleano)** | `false` | Se pone en `true` cuando se detecta un nuevo evento en la cola. Vuelve a `false` apenas la máquina de estados procesa ese evento. |



---

### 3. Comportamiento de `task_system_statechart()` (Específicamente `task_system_normal_statechart()`)

Esta función se ejecuta constantemente en el bucle principal y sigue esta lógica secuencial:

1.  **Lectura de la Cola:** Primero, comprueba si hay algo en el "buzón" llamando a `any_event_task_system()`. Si la cola no está vacía, extrae el evento con `get_event_task_system()`, lo guarda en su variable `.event` y levanta su propia bandera (`.flag = true`) para recordar que tiene un evento pendiente de procesar.
2.  **Evaluación de Estados (`switch`)**:
    * Si está en **`ST_SYS_IDLE`**: Comprueba si tiene un evento pendiente (`flag == true`) Y si ese evento es `EV_SYS_ACTIVE` (alguien apretó el botón). Si es así, baja la bandera (`flag = false`), envía una orden de encender el LED llamando a `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)` y finalmente cambia su estado a **`ST_SYS_ACTIVE`**.
    * Si está en **`ST_SYS_ACTIVE`**: (Aunque el fragmento se corta, la lógica típica aquí sería esperar a que se libere el botón o pase un tiempo para enviar `EV_LED_IDLE` al actuador y volver a `ST_SYS_IDLE`).

---

### 4. Evolución de las Variables de la Cola Circular (`event_task_system_queue`)

Aquí vemos qué pasa cuando la tarea del Sistema **lee** los mensajes que había dejado el Sensor.

| Variable | En `init_event_task_system()` | En `task_system_update()` (al llamar a `get_event...`) |
| :--- | :--- | :--- |
| **`i`** | Itera de `0` a `QUEUE_LENGTH-1` para vaciar el arreglo inicializándolo en `EMPTY`. | No se utiliza durante la lectura normal. |
| **`head`** | `0` | No se modifica aquí (solo la modifica el Sensor al escribir). |
| **`tail`**| `0` | **Se incrementa en `+1`** cada vez que el Sistema lee un evento. Si llega al final del tamaño de la cola, vuelve a `0`. Es el "puntero de lectura". |
| **`count`** | `0` | **Disminuye en `-1`** porque el Sistema acaba de retirar y consumir un mensaje pendiente. |
| **`queue[i]`**| Se llena de valores nulos (`EMPTY`). | El valor leído sigue físicamente en la memoria del arreglo, pero lógicamente es ignorado porque `tail` ya avanzó a la siguiente posición. |

---

### 5. Evolución de Variables de la Interfaz del Actuador (`task_actuator_interface.c`)

Cuando el Sistema decide que hay que hacer algo (ej. prender el LED), llama a `put_event_task_actuator()`. Esta función no usa colas complejas, sino un mecanismo de "Buzón de 1 solo mensaje" con bandera.

| Variable | Al inicializar el sistema | Al ejecutarse `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)` |
| :--- | :--- | :--- |
| **`identifier`** | N/A | Recibe el valor `ID_LED_A` para saber a qué actuador específico va dirigida la orden. |
| **`.event`** | `EV_LED_IDLE` | Se sobrescribe directamente con el nuevo comando: `EV_LED_ACTIVE`. |
| **`.flag`** | `false` | Se fuerza a **`true`**. Esto es crítico: es la señal que usará la máquina de estados del Actuador (en su propio `update`) para darse cuenta de que recibió una orden nueva y no la misma orden vieja repetida. |

**Conclusión del flujo completo:**
Botón (Físico) -> `task_sensor` -> **Cola Circular** -> `task_system` -> **Variable directa (Flag)** -> `task_actuator` -> LED (Físico). 

