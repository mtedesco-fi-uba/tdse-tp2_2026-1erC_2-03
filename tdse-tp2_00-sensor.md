
---

### 1. Funcionamiento de los Archivos

* **`task_sensor_attribute.h` y `task_system_attribute.h`:**
    Definen la "semántica" y las estructuras de datos de las tareas *Sensor* (el botón) y *System* (la lógica principal). 
    * Declaran los Eventos (`EV_BTN_UP`, `EV_BTN_DOWN`), Estados (`ST_BTN_IDLE`, `ST_BTN_ACTIVE`) y los Identificadores.
    * Separan la configuración que reside en memoria Flash (constante, ej. qué pin de hardware usar) de los datos que residen en RAM (variables que cambian, ej. el estado actual o el contador de ticks).
* **`task_sensor.c`:**
    Es la implementación de la tarea encargada de leer el botón. Aísla el hardware del resto de la aplicación. Periódicamente lee el estado físico del botón, actualiza su propia máquina de estados (generalmente para aplicar algoritmos antibrebote o *debouncing*) y, si detecta una pulsación válida, genera un evento de alto nivel para el sistema.
* **`task_system_interface.c`:**
    Implementa una **Cola Circular FIFO (First-In, First-Out)**. Funciona como un buzón de mensajes asíncrono. Permite que la tarea *Sensor* (Productor) envíe eventos a la tarea *System* (Consumidor) de forma segura, sin que ninguna de las dos se bloquee esperando a la otra.

---

### 2. Evolución de Variables en la Tarea Sensor

* **`index`:** Índice adimensional que itera sobre la lista de sensores configurados (si hubiera más de un botón).
* **`tick` (Unidad de medida: Milisegundos / ms o Ticks de sistema):** Es un temporizador local de la tarea, utilizado típicamente para medir cuánto tiempo lleva un botón presionado (para filtrar ruidos/rebotes).
* **`state`:** El estado actual de la máquina de estados del sensor.
* **`event`:** El último estímulo detectado (físico o lógico).

| Variable | Durante `task_sensor_init()` | Ejecución de `task_sensor_update()` |
| :--- | :--- | :--- |
| **`index`** | Recorre de `0` a `SENSOR_DTA_QTY - 1` para inicializar. | Itera continuamente de `0` a `SENSOR_DTA_QTY - 1` en cada vuelta del loop principal. |
| **`.tick`** | Se inicializa a `0`. | Se incrementa o se reinicia a `0` dentro del `statechart` dependiendo de si se está cronometrando un rebote o una pulsación larga. |
| **`.state`** | Se fuerza al estado inicial: `ST_BTN_IDLE`. | Cambia entre `ST_BTN_IDLE` y `ST_BTN_ACTIVE` según las transiciones lógicas del diagrama de estados. |
| **`.event`** | Se asume el estado de reposo: `EV_BTN_UP`. | Se actualiza en cada ciclo leyendo directamente el hardware (toma el valor `EV_BTN_DOWN` o `EV_BTN_UP`). |

---

### 3. Comportamiento de `task_sensor_statechart(uint32_t index)`

Esta función es el "cerebro" de la tarea sensor y se ejecuta en cada ciclo del procesador para el botón correspondiente al `index`:

1.  **Lectura de Configuración y Datos:** Obtiene los punteros a la estructura constante (qué pin leer, cuál es el nivel lógico de pulsación) y a la estructura dinámica (el estado y variables actuales del botón).
2.  **Mapeo de Hardware a Lógica:** Utiliza `HAL_GPIO_ReadPin()` para leer el estado eléctrico del pin. Compara ese estado con el valor predefinido de pulsación (`pressed`). Si coinciden, dictamina que el evento actual es `EV_BTN_DOWN`; de lo contrario, es `EV_BTN_UP`.
3.  **Evaluación de Transiciones (El `switch`):** Entra a un bloque `switch(p_task_sensor_dta->state)`. Dependiendo de si el estado actual es `IDLE` o `ACTIVE`, y cruzándolo con el evento recién detectado (y el temporizador `tick`), decide si debe cambiar a otro estado.
4.  **Generación de Salida:** Si la máquina de estados determina que ocurrió una pulsación limpia y válida, enviará un mensaje al sistema mediante la interfaz correspondiente (llamando a la función que encola el evento).

---

### 4. Evolución de las Variables de la Cola Circular (`event_task_system_queue`)

Esta cola permite que los eventos no se pierdan si la tarea *System* está ocupada cuando el *Sensor* detecta un cambio.

| Variable | Función / Propósito | En `init_event_task_system()` | En sucesivos `task_sensor_update()` (al usar `put_event...`) |
| :--- | :--- | :--- | :--- |
| **`head`** | Índice donde el Productor (*Sensor*) escribirá el **próximo** evento. | `0` | Se incrementa en `+1` cada vez que el Sensor detecta un evento válido y lo envía. Si llega a `QUEUE_LENGTH`, vuelve a `0` (comportamiento circular). |
| **`tail`** | Índice de donde el Consumidor (*System*) leerá el **próximo** evento. | `0` | Se mantiene igual en esta etapa. Solo se incrementará cuando la tarea *System* procese (lea) los eventos pendientes usando `get_event...`. |
| **`count`** | Cantidad de eventos sin leer almacenados en la cola actualmente. | `0` | Aumenta en `+1` con cada evento insertado por el Sensor. (Disminuirá cuando el System los lea). |
| **`queue[i]`** | El arreglo (buffer) que almacena físicamente los eventos. | Todas las posiciones de `0` a `QUEUE_LENGTH-1` se llenan con `EMPTY`. | Cuando se inserta un evento, `queue[head]` se sobrescribe con el nuevo evento (ej. `EV_SYS_ACTIVE`). Las demás posiciones quedan intactas hasta que el `head` pase por ellas. |
