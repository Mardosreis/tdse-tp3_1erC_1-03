Trabajo Práctico: Control de LCD Display mediante Máquina de Estados y Porting de Código C
1. Introducción y Objetivos
El objetivo de este trabajo es implementar un controlador para un display LCD (típicamente el controlador HD44780 o compatible) utilizando un enfoque de diseño modular. Se abarcan tres áreas principales:

Porting de Código (Hardware Abstraction Layer): Separar la lógica del LCD de las dependencias específicas del microcontrolador.

Modelado (Statechart): Diseñar la lógica de inicialización y escritura utilizando diagramas de estado para evitar el uso de retardos bloqueantes (delays) prolongados.

Implementación en C: Traducir el modelo a código C eficiente y portable.

2. System Setup y Estrategia de Porting
Para que el código sea portable a cualquier microcontrolador (STM32, PIC, AVR, ESP32, etc.), no debemos usar funciones específicas de hardware (como HAL_GPIO_WritePin) directamente en la lógica del LCD. En su lugar, creamos una capa de abstracción.

Interfaz de Porting (Hardware Abstraction Layer - HAL)
Definimos las funciones que el usuario debe proveer para que el LCD funcione en su sistema específico.

C
// lcd_port.h
#ifndef LCD_PORT_H
#define LCD_PORT_H

#include <stdint.h>

// Definición de pines lógicos
typedef enum {
    LCD_PIN_RS,
    LCD_PIN_EN,
    LCD_PIN_D4,
    LCD_PIN_D5,
    LCD_PIN_D6,
    LCD_PIN_D7
} lcd_pin_t;

// Estado del pin
typedef enum {
    LCD_PIN_LOW = 0,
    LCD_PIN_HIGH = 1
} lcd_pin_state_t;

// Funciones a implementar por el usuario según su microcontrolador
void LCD_Port_Init(void);
void LCD_Port_WritePin(lcd_pin_t pin, lcd_pin_state_t state);
uint32_t LCD_Port_GetTicks(void); // Para manejar retardos no bloqueantes

#endif // LCD_PORT_H
3. Modelado del Sistema (Statechart)
El controlador HD44780 requiere una secuencia de inicialización estricta con tiempos de espera específicos (por ejemplo, esperar 15ms, luego 4ms, luego 100us). Para evitar usar retardos bloqueantes que detengan el microcontrolador, modelamos el sistema como una máquina de estados.

Definición de Estados
STATE_UNINIT: Estado inicial. Espera a que se llame a la función de inicio.

STATE_INIT_WAIT: Espera los retardos de la secuencia de inicialización de 4 bits.

STATE_READY: El LCD está inicializado y listo para recibir comandos o datos.

STATE_SENDING_NIBBLE: Estado transitorio manejando el pulso del pin EN (Enable).

4. Implementación en C (C Coding)
Con el hardware abstraído y el modelo de estados definido, procedemos a escribir el código del controlador principal.

Archivo de Cabecera (lcd_app.h)
C
// lcd_app.h
#ifndef LCD_APP_H
#define LCD_APP_H

#include <stdint.h>
#include <stdbool.h>

typedef enum {
    LCD_STATE_UNINIT,
    LCD_STATE_INIT_WAIT,
    LCD_STATE_READY,
    LCD_STATE_ERROR
} lcd_state_t;

void LCD_Init(void);
void LCD_Process(void); // Función que se llama continuamente en el bucle principal (o timer)
bool LCD_Print(const char* str); // Retorna true si fue aceptado

#endif // LCD_APP_H
Archivo Fuente (lcd_app.c - Lógica de Estados)
C
// lcd_app.c
#include "lcd_app.h"
#include "lcd_port.h"

static lcd_state_t current_state = LCD_STATE_UNINIT;
static uint32_t wait_timestamp = 0;
static uint8_t init_step = 0;

void LCD_Init(void) {
    LCD_Port_Init();
    current_state = LCD_STATE_INIT_WAIT;
    wait_timestamp = LCD_Port_GetTicks();
    init_step = 0;
}

// Bucle de la máquina de estados
void LCD_Process(void) {
    uint32_t current_ticks = LCD_Port_GetTicks();

    switch(current_state) {
        case LCD_STATE_UNINIT:
            // No hacer nada hasta que se llame a LCD_Init()
            break;

        case LCD_STATE_INIT_WAIT:
            // Secuencia típica de inicialización para 4-bits
            if (init_step == 0 && (current_ticks - wait_timestamp >= 15)) {
                // Enviar comando 0x30
                // (Pseudocódigo) LCD_SendNibble(0x03);
                wait_timestamp = current_ticks;
                init_step++;
            } 
            else if (init_step == 1 && (current_ticks - wait_timestamp >= 5)) {
                // Enviar comando 0x30 nuevamente
                // (Pseudocódigo) LCD_SendNibble(0x03);
                wait_timestamp = current_ticks;
                init_step++;
            }
            // ... (completar secuencia de inicialización 0x32, 0x28, 0x0C, 0x01) ...
            else if (init_step >= 5) {
                current_state = LCD_STATE_READY;
            }
            break;

        case LCD_STATE_READY:
            // Revisar si hay datos en el buffer de transmisión y enviarlos
            break;

        case LCD_STATE_ERROR:
            // Manejo de errores
            break;
    }
}
5. Conclusión del Diseño
Al utilizar esta arquitectura, logramos un sistema robusto:

Reutilizable: El archivo lcd_app.c nunca cambia. Si pasamos de un Arduino a un STM32, solo reescribimos las 3 funciones de lcd_port.c.

No bloqueante: Al usar LCD_Process() basado en una máquina de estados y "Ticks", el CPU puede realizar otras tareas (como leer sensores o comunicarse por UART) mientras el LCD cumple sus tiempos de espera mecánicos.

Análisis de los Archivos Fuente
A. Capa de Sistema y Planificador (Scheduler)
app_it.c (Interrupciones del Sistema): Contiene la rutina de atención de interrupciones del temporizador del sistema (SysTick). La función HAL_SYSTICK_Callback(void) se ejecuta periódicamente (generalmente cada 1 milisegundo) y actualiza el contador global de "ticks" de la aplicación (g_app_tick_cnt). Es el "latido" del sistema.

app.c (Main Application Loop): Es el corazón del planificador. Revisa continuamente si hubo un "tick" del sistema (evaluando el contador decremental modificado por app_it.c). Si el tiempo avanzó, itera a través de un arreglo de tareas configuradas (task_cfg_list) y llama a sus respectivas funciones de actualización (task_x_update). También perfila el rendimiento, calculando los tiempos de ejecución de las tareas como LET (Last Execution Time), BCET (Best Case) y WCET (Worst Case).

systick.c: Provee retardos bloqueantes en el orden de los microsegundos (systick_delay_us) leyendo directamente el registro de conteo regresivo del hardware SysTick->VAL. A diferencia de los retrasos por tareas, este delay detiene el procesador, pero es estrictamente necesario para cumplir con los requerimientos temporales ultracortos durante la inicialización a bajo nivel del display (tiempos de setup y hold del hardware).

B. Capa de Driver y Hardware Abstraction (Display)
display.h y display.c: Implementan el driver de bajo nivel para un display alfanumérico (tipo HD44780). Maneja la comunicación tanto para conexiones de 8-bits como de 4-bits (DISPLAY_CONNECTION_GPIO_4BITS). Aquí se encuentra la lógica electrónica de manipular los pines RS, EN (Enable) y los buses de datos D0-D7. Funciones como displayCharPositionWrite() envían los comandos precisos al hardware físico.

C. Capa de Tareas de Aplicación (Tasks)
task_display_interface.h y task_display_interface.c: Actúan como un mecanismo de Comunicación Inter-Tareas (IPC). Implementan el patrón Message Passing. Contienen la función put_event_task_display(), la cual es llamada por otras tareas (como test) para solicitar una escritura en el LCD. Esta función levanta una bandera (flag = true), setea el evento EV_DSP_UPDATE, y copia los datos (coordenadas y texto) en una estructura compartida (task_display_dta), desacoplando a las tareas creadoras de información del driver que las muestra.

task_test_attribute.h y task_test.c: Implementa la tarea generadora de información (Tarea Productora). Tiene variables internas como tick y counter. En su inicialización, inyecta los primeros mensajes (ej. "LCD Display Test" y " Porting C code ") llamando a la interfaz de display.

task_display.c: Es la Tarea Consumidora. Se encarga exclusivamente de revisar si hay mensajes pendientes en la interfaz y gestionar el volcado de esos mensajes hacia el driver (display.c) mediante una máquina de estados.

2. Comportamiento de las Funciones de Statechart
Ambas funciones emplean el modelo de Máquina de Estados Finitos (FSM). Al ser llamadas desde el planificador cooperativo (app.c), su ejecución debe ser no bloqueante (Run-to-completion). Recrean la ilusión de multitarea guardando el estado en el que se quedaron y retornando inmediatamente el control al sistema.

void task_test_statechart(void)
La función principal de la tarea de prueba es generar estímulos para el sistema o realizar una temporización lógica (sin bloquear usando retardos).

Comportamiento esperado: Maneja internamente un temporizador descendente basado en llamadas. Cada vez que el planificador llama a esta función (y avanza el tiempo del sistema), la FSM descuenta su atributo p_task_test_dta->tick.

Cuando el tick llega a 0 (evento de timeout), la máquina transiciona. Típicamente incrementa un contador (p_task_test_dta->counter), reinicia el timer tick con un valor máximo (ej. DEL_TEST_XX_MAX), y genera un texto dinámico (como "Contador: X").

A continuación, llama a put_event_task_display(...) para enviar la actualización al LCD.

void task_display_statechart(void)
Esta función gobierna cómo y cuándo se actualiza físicamente el LCD. El uso de esta máquina garantiza que si actualizar toda la pantalla requiere tiempo mecánico, el microcontrolador (en este caso la Nucleo-64) no quede "congelado".

El código task_display.c (según los fragmentos proporcionados) opera de la siguiente manera:

Evaluación de Estado Inicial (ST_DSP_IDLE o Espera):

La máquina descansa en estado de reposo comprobando las banderas compartidas de la interfaz.

Si detecta que la bandera está arriba (p_task_display_dta->flag == true) y que el evento indica una actualización (EV_DSP_UPDATE), la máquina decide transicionar al estado de actualización (ST_DSP_UPDATE).

Transición y Procesamiento (ST_DSP_UPDATE):

Al ingresar a este estado, baja la bandera (flag = false) para reconocer o consumir el evento y evitar sobre-escrituras.

Mueve los índices lógicos de fila y columna (row = 0, column = 0).

Llama al driver de bajo nivel (displayCharPositionWrite) para preparar el hardware, indicando en qué coordenada específica se van a volcar los datos de la memoria RAM del display (la variable ddram local) hacia la interfaz física.

Nota de escalabilidad: En diseños no bloqueantes puros, este estado puede dividirse en múltiples ciclos (enviar un solo carácter por cada llamado al statechart) para mantener el WCET (Worst Case Execution Time) de app.c bajo control estricto. Al concluir la actualización de la cadena de texto, la máquina retorna a ST_DSP_IDLE.


