# Máquina-Expendedora

En esta práctica el objetivo es implementar el funcionamiento de una máquina expendedora utilizando componentes de Arduino como joystick, LCD, etc

**Montaje**

<img width="1075" height="731" alt="Captura desde 2026-11-16 17-12-10" src="https://github.com/user-attachments/assets/75c675f6-3434-4156-9e77-b999a6bbc986" />

Para los pines elegidos, no todas las decisiones son arbitrarias.

- Para el botón he elegido el pin 2 porque Arduino uno solo permite interrupciones externas en ese y en 3.
- El led rojo está en el pin 11 porque necesitámos uno PWM para ir incrementando la luz.
- Para el Joystick necesitamos pines analógicos.

**Resumen del funcionamineto**

Al arrancar la placa, se enciende 3 veces un led azul esperando 1 segundo entre una y otra. A la vez, en la LDC se puede leer CARGANDO..., durante el mismo tiempo.

Una vez finalizado esto, entra en modo servicio, el sensor de ultrasonidos empezará a medir distancias. Si la distancia es mayor a 1m escribe en la LCD ESPERANDO CLIENTE hasta que detecte a menos de 1m. 

En ese momento empiza una cuenta de 5s donde aparecerá la temperatura y la humedad de ese instante en la LCD. Tras esto nos aparecerá un menú de bebidas y precio que se puede manejar con el joystick, si se le pulsa, estas eligiendo esa bebida y pondrá Preparando Café... durante un tiempo aleatorio entre 4 y 8, en este tiempo un led rojo se irá encendiendo de manera proporcional a la cantidad de tiempo consumida,  y al acabar RETIRAR BEBIDA por 3s. 

https://github.com/user-attachments/assets/c28767f6-5ae4-4fba-a9a8-fec7fcb84a72

Este es el modo servicio pero si pulsas entre 2s y 3s el botón, puedes reiniciar el comportamiento de servicio.

También se puede pulsar más de 5s y entonces entras en modo administrador, en este caso siempre estarán los dos leds encendidos durante todo el proceso de este modo. Se abrirá otro menú en la pantalla donde podrás elegir entre diferentes opciones de visualización de temperatura, distancia o contando, las cuales se van actualizando al instante. También una interactiva donde podrás cambiar precios aumentando o disminuyendolos en 5 céntímos. Si presionas el joystick se guardará ese precio hasta que se apague la placa.

Para salir de este modo es necesario volver a presionar el botón al menos 5s y regresarás a servicio, donde en el caso de que hayas hecho cambios en los precios, podrás verlos actualizados.

Durante el código, se irá pasando periódicamente por un watchdog por si la placa se quedase bloqueada poder reiniciarla.

https://github.com/user-attachments/assets/e2656519-bff9-4c93-8847-f8a493378bcd

https://github.com/user-attachments/assets/8c14e3bc-1be9-4e27-a781-f8393eb2be03

**Implementación**

El programa tiene 3 modos: ARRANQUE, SERVICIO o ADMIN, para servicio también hay fases: ESPERANDO_CLIENTE, MOSTRANDO_DATOS, SELECCIONAR_PRODUCTO, PREPARANDO o RETIRAR_BEBIDA, y dos listas de precios por si se actualizan.

El loop tiene solo dos líneas ya que contamos con hilos y un controlador. También, muy importante el reseteo del watchdog.

Cada hilo se encarga de una tarea:
- led_thread -> blink_led(), cuando el modo sea arranque se ejecutará cada segundo.
- button_thread -> check_button()
- servicio_thread -> servicio_task(), cuando el modo sea servicio, se ejecutarán sus tareas.
- admin_thread -> admin_task(), cuando el modo sea administración, se ejecutarán sus tareas.

Para las interrupciones utilizo attachInterupt para las del botón y TimerOne para el contador. Es necesario declarar sus variables como volatil para poder comprobar sus valores en ese instante de tiempo

El watchdog es un temporizador del microcontrolador que reinicia la placa si el programa no lo resetea a tiempo, se define en el setup y s ereinicia en el loop.

> Código al detalle

*Arranque*

En el setup directamente se escribe en la LCD y el hilo llama a su función cada segundo donde se cambiará su estado de true a false, que es lo mismo que high a low. Cada vez que el estado sea true/high se aumentará un contador el cual cuando se llegue a 3 cambiará el modo a servicio y se transicionará. 

*Servicio*


  
