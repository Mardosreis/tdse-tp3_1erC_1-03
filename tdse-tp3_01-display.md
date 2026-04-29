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
