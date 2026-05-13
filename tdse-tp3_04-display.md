# Informe de Depuración y Análisis Temporal - Paso 04
## 1. Valores Obtenidos en Depuración
Los siguientes valores fueron capturados utilizando la herramienta *Live Expressions* de STM32CubeIDE luego de múltiples ejecuciones cíclicas.

**Unidad de medida:** Microsegundos (us)

| Index | Tarea (task_dta_list) | NOE (Ejecuciones) | LET (Último) | BCET (Mejor) | WCET (Peor) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | `task_dta_list[0]` | 42794 | 42 us | 42 us | 43 us |
| 1 | `task_dta_list[1]` | 42797 | 3 us | 3 us | 152 us |
| 2 | `task_dta_list[2]` | 42801 | 2 us | 2 us | 2 us |
| 3 | `task_dta_list[3]` | 42805 | 2 us | 2 us | 78 us |

## 2. Análisis de Restricciones Temporales
El sistema utiliza un ejecutor cíclico con un período de tick de **1 milisegundo (1000 us)**. 

Para garantizar el cumplimiento de las restricciones temporales, el tiempo de ejecución en el peor de los casos (WCET) sumado de todas las tareas debe ser menor al período del planificador.

* El **WCET máximo** registrado corresponde a `task_dta_list[1]` con **152 us**.
* Las demás tareas presentan WCETs de 43 us, 2 us y 78 us respectivamente.
* La suma total de los WCET en el peor de los casos teóricos es de **275 us** (43 + 152 + 2 + 78).

### Conclusión Técnica
El sistema **CUMPLE HOLGADAMENTE** con la restricción temporal del ejecutor cíclico. El tiempo de ejecución en el peor de los casos (275 us en total) representa menos del 30% del tiempo disponible por tick (1000 us). Esto demuestra que la implementación de las Máquinas de Estados no bloqueantes fue exitosa y el microcontrolador tiene tiempo de sobra para dormir o procesar otras interrupciones antes del próximo ciclo.
