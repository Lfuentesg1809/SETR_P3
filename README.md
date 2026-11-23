# SETR_P3
Práctica 3 de SISTEMAS EMPOTRADOS Y DE TIEMPO REAL
# Práctica 3: Controlador de una máquina expendedora

## Desarrollo de la solución

Para realizar esta práctica decidí desarrollar el proyecto de forma progresiva, en lugar de conectar todos los componentes y programar su funcionamiento a la vez. Al principio fui trabajando sensor por sensor, comprobando que cada uno funcionaba correctamente de manera independiente antes de incorporar el siguiente. De esta forma, si aparecía algún error, era más sencillo identificar de dónde venía y solucionarlo.

Una vez que conseguía que un componente funcionase, lo integraba con los anteriores y volvía a comprobar que todo siguiese funcionando. Repetí este proceso hasta conseguir conectar todos los sensores y actuadores, teniendo en cuenta los pines disponibles del Arduino UNO y evitando conflictos entre ellos.

Una de las primeras dificultades fue precisamente la cantidad de cables y conexiones que necesitaba el montaje. Al utilizar tantos componentes, era fácil confundirse con los pines o realizar alguna conexión incorrecta. También tuve algunos problemas iniciales con las librerías necesarias para controlar los distintos dispositivos. Sin embargo, después de revisar las conexiones y ajustar el código, conseguí que todos los componentes funcionasen conjuntamente.

## Organización del programa

Para organizar el código he utilizado una **máquina de estados**, ya que me parecía la forma más clara de gestionar los diferentes modos de funcionamiento sin mezclar todas las tareas en un mismo bloque.

Para ello, he definido un `enum Estado` con los distintos estados del sistema, como el arranque, la espera, la lectura del ambiente, la selección de productos, la preparación y las diferentes opciones de administración.

La principal ventaja de esta estructura es que el programa sabe en todo momento en qué estado se encuentra y qué acciones debe realizar. Además, las transiciones entre estados se gestionan mediante una función común, `cambiar()`, que actualiza el estado, reinicia las variables de tiempo correspondientes y prepara la pantalla para mostrar la nueva información.

También he separado el código en funciones para gestionar los botones, el joystick, los sensores y la pantalla LCD. Así, el `loop()` principal queda más organizado y resulta más sencillo localizar errores o realizar modificaciones.

## Gestión del tiempo y ejecución no bloqueante

Una de las decisiones más importantes fue **evitar el uso de `delay()`**, ya que este detiene la ejecución del programa durante el tiempo indicado. Esto supondría un problema en una máquina expendedora, porque mientras se está preparando una bebida o mostrando información en pantalla, el sistema debe seguir siendo capaz de detectar otras acciones, como la pulsación del botón de administración.

Por este motivo, he utilizado `millis()` para controlar los intervalos de tiempo. En lugar de detener el programa, guardo el instante en el que comienza cada estado y voy comprobando cuánto tiempo ha transcurrido.

Para la medición de distancias también he utilizado `micros()`, ya que el sensor ultrasónico necesita trabajar con intervalos mucho más pequeños.

Esta solución permite que el programa siga ejecutando sus distintas tareas sin tener que esperar a que termine cada temporización.

**No he utilizado la librería ArduinoThread.** Aunque aparecía entre las recomendaciones del enunciado, decidí organizar las tareas mediante una máquina de estados y temporizadores basados en `millis()`. De esta manera, podía controlar las distintas operaciones sin añadir otra librería y manteniendo una estructura relativamente sencilla.

## Uso de interrupciones

He utilizado **interrupciones hardware por cambio de pin (PCINT)** para gestionar eventos relacionados con el botón externo y el sensor ultrasónico.

En el caso del botón externo, la interrupción registra que se ha producido un cambio de estado. Después, en el programa principal, se comprueba el nivel del botón y se calcula cuánto tiempo permanece pulsado utilizando `millis()`.

También he incorporado un sistema de antirrebote por software, con un intervalo de 35 milisegundos, para evitar que las pequeñas oscilaciones eléctricas de los pulsadores provoquen detecciones incorrectas.

En el sensor ultrasónico, las interrupciones permiten detectar los cambios de nivel de la señal ECHO y calcular la duración del pulso mediante `micros()`. De esta forma, evito utilizar `pulseIn()`, que podría mantener bloqueado el programa mientras espera la respuesta del sensor.

Además, he procurado que las rutinas de interrupción sean cortas y que el procesamiento principal se realice fuera de ellas. Esto ayuda a reducir el tiempo dedicado a atender las interrupciones.

## Uso del Watchdog

Otro recurso que he incorporado es el **temporizador Watchdog**, utilizando la librería `avr/wdt.h`.

He configurado el Watchdog con un tiempo de 2 segundos mediante `wdt_enable(WDTO_2S)` y lo refresco periódicamente en el `loop()` utilizando `wdt_reset()`.

La razón de utilizarlo es aumentar la seguridad y fiabilidad del sistema. Si por algún error el programa se quedase bloqueado y dejase de ejecutar el bucle principal, el Watchdog provocaría automáticamente un reinicio del microcontrolador.

También desactivo inicialmente el Watchdog en el `setup()` antes de volver a configurarlo, para evitar posibles reinicios continuos después de un bloqueo.

## Gestión de sensores, joystick y actuadores

Para evitar lecturas innecesarias, he establecido distintos intervalos de actualización según el componente.

Por ejemplo, el sensor DHT11 se consulta aproximadamente cada 2,2 segundos, ya que no es necesario medir continuamente la temperatura y la humedad. Por otro lado, el sensor ultrasónico utiliza un sistema de medición basado en fases y temporizaciones con `micros()`.

También he controlado la actualización de la pantalla LCD para que no se esté borrando y escribiendo constantemente. La pantalla se actualiza cuando cambia el estado, cuando se modifica alguna selección o cuando corresponde refrescar una lectura. Esto permite reducir operaciones innecesarias y evitar parpadeos en la visualización.

Para el joystick he implementado un sistema que detecta los movimientos y exige volver a la posición central antes de registrar otro desplazamiento. Así se evita que una sola inclinación provoque múltiples cambios de opción demasiado rápidos.

En cuanto al LED que indica el progreso de preparación, he utilizado **PWM mediante `analogWrite()`**. Esto permite modificar progresivamente su brillo en función del tiempo transcurrido, sin necesidad de bloquear el programa.

Para gestionar los precios de los productos he decidido almacenarlos como números enteros en céntimos, en lugar de utilizar variables decimales. Esto facilita realizar incrementos y decrementos exactos de 5 céntimos y evita posibles errores de precisión al trabajar con números en coma flotante.

## Dificultades encontradas

Las principales dificultades al comienzo de la práctica estuvieron relacionadas con el montaje físico y la integración de todos los componentes.

Al haber tantos cables conectados al Arduino, tuve que prestar especial atención a la distribución de los pines. Además, algunos componentes necesitaban librerías específicas, por lo que también tuve que comprobar que estuvieran correctamente instaladas y que su utilización fuera compatible con el resto del programa.

Otra parte importante fue conseguir que las distintas tareas funcionasen conjuntamente sin interferir unas con otras. Por eso opté por utilizar la máquina de estados y evitar los retardos bloqueantes, ya que así podía mantener el control del sistema desde un único bucle principal.

El desarrollo progresivo, probando primero cada sensor por separado y después integrándolo con los demás, fue especialmente útil para resolver estos problemas.

## Esquemático del circuito

Para representar las conexiones del circuito he utilizado **Tinkercad**, ya que inicialmente intenté realizar el esquema con Fritzing, pero no conseguí hacerlo correctamente con esa herramienta.

Una limitación que encontré en Tinkercad fue que no disponía del módulo joystick utilizado en la práctica. Por ese motivo, aunque el joystick no aparece representado físicamente en el esquema, he indicado los pines a los que está conectado en el montaje real.

En concreto, el joystick utiliza los pines analógicos **A0 y A1** para sus dos ejes y el pin digital **7** para el pulsador integrado.

De esta manera, aunque no aparezca el componente en el dibujo, quedan especificadas sus conexiones y se puede entender cómo está integrado en el circuito.

## Conclusión

Esta práctica me ha servido para aprender a integrar distintos sensores y actuadores en un mismo proyecto y, sobre todo, a organizar el código para que todas las tareas puedan gestionarse de forma coordinada.

Al principio, la mayor dificultad fue conseguir que funcionasen correctamente todas las conexiones y librerías, pero desarrollar el proyecto poco a poco me permitió detectar y solucionar los problemas con mayor facilidad.

Considero que las decisiones más importantes han sido utilizar una máquina de estados, gestionar los tiempos sin `delay()`, incorporar interrupciones para determinados eventos y utilizar el Watchdog como mecanismo de recuperación ante bloqueos.

Gracias a esta organización, el programa resulta más fácil de entender, mantener y ampliar, además de permitir una respuesta más rápida a los eventos del sistema.
