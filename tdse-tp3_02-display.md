## Valores obtenidos de task_dta_list
Los siguientes valores fueron capturados mediante la depuración con STM32CubeIDE luego de más de 67.000 ejecuciones cíclicas.

**Unidad de medida:** Microsegundos (us)

| Index | Tarea | NOE (Ejecuciones) | LET (Último) | BCET (Mejor) | WCET (Peor) |
| :---: | :--- | :---: | :---: | :---: | :---: |
| **0** | `task_test` | 67364 | 2 us | 2 us | 36 us |
| **1** | `task_display` | 67367 | 2 us | 2 us | 1996 us |

## Análisis de restricciones temporales

El planificador (ejecutor cíclico) del sistema está configurado para ejecutarse mediante interrupciones del SysTick cada **1 milisegundo (1000 us)**. Para garantizar un comportamiento determinista, el tiempo de ejecución en el peor de los casos (WCET) de cada tarea debe ser estrictamente menor a este período.

Al analizar los valores obtenidos:
1. La **Tarea 0** cumple con la restricción, ya que su WCET es de apenas **36 us**.
2. La **Tarea 1** **NO CUMPLE** con la restricción temporal. Su WCET registrado es de **1996 us** (prácticamente 2 milisegundos).

**Conclusión:** El código actual viola las restricciones temporales del ejecutor cíclico. El exceso de tiempo se debe a que la tarea del display utiliza funciones bloqueantes para escribir en la pantalla LCD de forma secuencial, deteniendo el procesador y superando el límite de 1000 us impuesto por el sistema. Esta es la justificación principal para migrar el diseño hacia una arquitectura de máquinas de estado no bloqueante.
