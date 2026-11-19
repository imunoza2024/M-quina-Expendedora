# Máquina-Expendedora

En esta práctica el objetivo es implementar el funcionamiento de una máquina expendedora utilizando componentes de Arduino como joystick, LCD, etc.

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

El loop tiene solo dos líneas ya que contamos con hilos, interrupciones, el setup y un controlador. También, muy importante el reseteo del watchdog.

Cada hilo se encarga de una tarea:
- led_thread -> blink_led(), cuando el modo sea arranque se ejecutará cada segundo.
- button_thread -> check_button()
- servicio_thread -> servicio_task(), cuando el modo sea servicio, se ejecutarán sus tareas.
- admin_thread -> admin_task(), cuando el modo sea administración, se ejecutarán sus tareas.

Para las interrupciones utilizo attachInterupt para las del botón y TimerOne para el contador. Es necesario declarar sus variables como volatil para poder comprobar sus valores en ese instante de tiempo

El watchdog es un temporizador del microcontrolador que reinicia la placa si el programa no lo resetea a tiempo, se define en el setup y se reinicia en el loop.



> Código al detalle

*Arranque*

En el setup directamente se escribe en la LCD y el hilo llama a su función cada segundo donde se cambiará su estado de true a false, que es lo mismo que high a low. Cada vez que el estado sea true/high se aumentará un contador el cual cuando se llegue a 3 cambiará el modo a servicio y se transicionará. 

*Servicio*

Primero entra la fase esperando cliente con su función check_presence (), donde hasta que no llega una medida menos a 1m, no se pasa a la siguiente fase mostando_datos.

En esta, la función show_tem_humd () es llamada hasta que se cumple el plazo de tiempo gracias a un condicional y millis en dos instantante de tiempos. Cuando se cumple, se pasa a la fase seleccionar_producto.

Ahora con show_products() podemos interactuar con el menú y sabremos si el joystick ha sido presionado cuando su función joystick_pressed () devuelva true, con esto se pasará a la fase preparando.

Preparando tiene su función prepare_drink() donde en cada vuelta, se calcula el tiempo transcurrido. Mientras sea menor que prep_time, escribe en el LCD y actualiza el brillo del LED2 se pasa  la fase retirar_bebida.

Con take_drink(), funcion de la fase retirar_bebida, escribimos en la LCD y se pasa de nuevo a la fase esperando_cliente.

Check_button gestiona la pulsación del botón, y dependiendo del modo en el que esté y el la duración de esta, entrará en cancelación o en modo admin (entrada o salida). La interrupción detect_button() se activa con CHANGE.

*Administración*

Siempre estarán encendidos ambos leds. 

Con show_options() mostaremos otro menú de opciones: temperatura y humedad, distancia, contador y editar precios, que llamarán asus respectivas funciones: show_admin_temperature(), show_distance(), show_counter() y edit_prices().

En edit_prices(), si pulsamos un café, nos permite cambiar en 0,05 el precio de este subiendo o bajando el joystick. Volviendo a pulsar confirmas el precio y vuelves al menú de cafés. 

La función back_to_menu nos permite ir hacia atras, es decir, si estamos en contador y queremos volver al menú de admin, moviendo el joystick hacia la izquierda, se ejcutará la función.

*¿Qué he usado y por qué?*

Threads y controller: no queremos usar espera activa, por ejemplo, para los leds, el botón o el contador. Esta librería nos permite ejecutar tareas con distintos periodos sin necesidad de hacer delay().

En mi caso, aademás de esta funcionalidad, lo uso para organizar el código entre tareas del servicio y del administrador, por ejemplo. Estas no están en funcionamiento continuamente, sino que, al iniciar debe pasar por una condición if (mode != servicio o admin) return;. Solo entra el hilo del modo en el que esté.

Interrupciones: 

- Para el botón, cuando se presiones o suelte. Aplico antirrebotes para evitar problemas con el botón. Además, noInterrupts()/interrupts() para evitar lecturas a medias.

- Para el contador, con TimerOne que ejecuta cada segundo e incrementa el contador. Debe ser independiente a lo que esté haciendo el programa ya que queremos calculos exactos del tiempo.

Watchdog: si por un fallo el programa se queda bloqueado más de 2s , wdt_reset() deja de ejecutarse y se reiniciará automáticamente.

<img width="2000" height="900" alt="Foto1" src="https://github.com/user-attachments/assets/d9040f49-25d3-4264-a7c4-a9947d749b5e" />

  
<img width="2000" height="900" alt="Foto2" src="https://github.com/user-attachments/assets/9e1b0043-c33f-478c-b6bb-5d7e4df5e8aa" />
