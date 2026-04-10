Este conjunto de archivos conforma el módulo encargado de **controlar las salidas físicas del sistema** (en este caso, un LED actuador). Utiliza una Máquina de Estados Finita (FSM) para aislar la lógica de control del hardware físico, permitiendo que el resto del sistema envíe comandos abstractos sin preocuparse por los detalles del microcontrolador.



A continuación, te presento el análisis detallado de los archivos y la evolución de sus variables.

---

### 1. Análisis del Funcionamiento de los Archivos

* **`task_actuator_attribute.h`**:
    Define la estructura de datos y el vocabulario del actuador.
    * **Eventos (`task_actuator_ev_t`)**: Órdenes que puede recibir (`EV_LED_IDLE`, `EV_LED_ACTIVE`).
    * **Estados (`task_actuator_st_t`)**: En qué estado lógico se encuentra el actuador (`ST_LED_IDLE`, `ST_LED_ACTIVE`).
    * **Estructuras**: Separa la configuración constante en memoria Flash (como el puerto GPIO, el pin y la polaridad lógica para encender/apagar) en `task_actuator_cfg_t`, de los datos variables en memoria RAM (como el estado actual, el contador de tiempo y las banderas) en `task_actuator_dta_t`.
* **`task_actuator.c`**:
    Contiene la lógica de inicialización y actualización periódica del actuador. Aquí reside la máquina de estados que evalúa si el LED debe encenderse o apagarse según los eventos recibidos y ejecuta los comandos de hardware a través de la librería HAL (`HAL_GPIO_WritePin`).
* **`task_actuator_interface.c`**:
    Proporciona el mecanismo de comunicación (API) para que otras tareas (como el Sistema) puedan enviarle órdenes al Actuador. Lo hace mediante la función `put_event_task_actuator`, la cual no utiliza una cola circular, sino que simplemente sobrescribe el evento actual y levanta una "bandera" (flag) para avisar que hay una orden nueva.

---

### 2. Evolución de las Variables del Actuador (`task_actuator_dta_list`)

Esta evolución describe qué ocurre con la estructura interna de datos cuando el sistema arranca y mientras se mantiene en el bucle principal.

| Variable | En `task_actuator_init()` | Durante `task_actuator_update()` |
| :--- | :--- | :--- |
| **`index`** | Itera de `0` a `ACTUATOR_DTA_QTY - 1` para inicializar todos los actuadores configurados. | Itera de `0` a `ACTUATOR_DTA_QTY - 1` en cada ciclo para actualizar la máquina de estados de cada actuador. |
| **`.tick`** *(Unidad: Milisegundos o ciclos de timer)* | Se inicializa en `0`. | Se puede incrementar periódicamente si la máquina de estados requiere medir un tiempo (por ejemplo, para apagar el LED automáticamente después de X milisegundos). |
| **`.state`** | Se fuerza al estado inicial: `ST_LED_IDLE`. | Cambia a `ST_LED_ACTIVE` cuando recibe un comando de encendido, y vuelve a `ST_LED_IDLE` cuando se le ordena apagarse. |
| **`.event`** | Se asume la inactividad: `EV_LED_IDLE`. | Almacena el último evento recibido desde el exterior (vía la interfaz). |
| **`.flag`** | Se inicializa en `false` (no hay comandos pendientes). | Cambia a `false` inmediatamente después de que la máquina de estados lee un evento válido. (Solo cambia a `true` cuando alguien llama a la interfaz). |

---

### 3. Comportamiento de la función `task_actuator_statechart(uint32_t index)`

Esta función es el núcleo de la tarea. Evalúa qué debe hacer el actuador físico basándose en su estado actual y en los mensajes recibidos. Funciona de la siguiente manera:

1.  **Lectura de punteros:** Obtiene la configuración constante (qué pin mover) y la estructura de datos dinámica (en qué estado estoy y qué evento recibí) correspondientes al actuador indicado por `index`.
2.  **Evaluación (`switch`)**: Revisa en qué estado se encuentra (`p_task_actuator_dta->state`):
    * **Si está en `ST_LED_IDLE`**: Comprueba dos condiciones simultáneas: si hay un mensaje nuevo (`flag == true`) **Y** si ese mensaje es de encendido (`event == EV_LED_ACTIVE`).
        * Si se cumplen, inmediatamente **baja la bandera** (`flag = false`) para no volver a procesar el mismo evento.
        * Enciende el LED físicamente usando `HAL_GPIO_WritePin()`, pasándole el estado configurado como "encendido" (`led_on`).
        * Cambia su estado interno a `ST_LED_ACTIVE`.
    * **Si está en `ST_LED_ACTIVE`**: De manera análoga (aunque el código anterior lo omitía por longitud), esperará a que `flag == true` y el evento sea `EV_LED_IDLE` para apagar el LED y retornar a `ST_LED_IDLE`.

---

### 4. Evolución de las Variables en la Interfaz (`put_event_task_actuator`)

A diferencia del `index`, la variable `identifier` se usa exclusivamente como argumento cuando otra parte del programa quiere comunicarse con el actuador. Esta función es un **Buzón de un solo mensaje**.

| Variable | Durante el arranque del sistema | Al ejecutar `put_event_task_actuator(evento, identifier)` |
| :--- | :--- | :--- |
| **`identifier`** | N/A | Toma el valor numérico del actuador que se desea controlar (por ejemplo, `ID_LED_A`). Sirve como índice para acceder a la lista correcta. |
| **`.event`** | N/A | **Se sobrescribe** inmediatamente con el nuevo valor pasado por parámetro (por ejemplo, `EV_LED_ACTIVE`). Si llega un evento nuevo antes de que el actuador lea el anterior, el anterior se pierde. |
| **`.flag`** | N/A | **Se establece en `true`**. Esta es la señal crítica. Le indica al `task_actuator_statechart` que el evento alojado en la variable `.event` es nuevo y debe ser procesado. |
