# tdse-tp3_1erC_1-03
FIUBA - Electrónica - Taller de Sistemas Embebidos - Trabajo Práctico N°: 3 - LCD Display - System Setup Menu
# FIUBA - Electrónica - Taller de Sistemas Embebidos
## Trabajo Práctico N°: 3 - LCD Display - System Setup Menu
### Año-Cuatrimestre - Curso-Grupo
### Responsable de la entrega:
| Padrón | Apellidos, Nombres |   Fecha  | Deadline  |
| :----- | :----------------- | :------: | :-------: |
| 111106 | TOSCAN UMA         |  29/04/26| Semana 08 |

### Análisis de cumplimiento del ejecutor cíclico:

El planificador de nuestro sistema está configurado para ejecutar un tick cada **1 milisegundo (1000 us)**. Para que el sistema sea estable y determinista, el tiempo de ejecución en el peor de los casos (WCET) de las tareas debe ser estrictamente menor a este período.

Al observar los valores obtenidos en la depuración, notamos que la tarea `task_test` cumple holgadamente con un WCET de **36 us**. Sin embargo, la tarea `task_display` registra un WCET de **9886 us** (casi 10 milisegundos).

**Conclusión:** El código actual **NO cumple** con las restricciones temporales del ejecutor cíclico. El uso de la función original bloqueante (`displayStringWrite`), que envía todos los caracteres de la pantalla de una sola vez, excede ampliamente el límite de 1000 us. Esto demuestra la necesidad inminente de migrar la tarea de display hacia un diseño no bloqueante basado en máquinas de estados, enviando como máximo un dato o instrucción por milisegundo.
