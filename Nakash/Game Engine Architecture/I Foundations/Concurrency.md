>Se utiliza cuando un programa puede funcionar en más de un control de flujo semi-independientes. Aunque también se puede utilizar para hacer mejor uso del "multicore computing plataform" 

### Programming models
Para que varios [[Threads]], dentro de un programa concurrente, puedan cooperar deben compartirse información y además sincronizar sus actividades. En otras palabras deberían de comunicarse. Para esto hay dos formas lograrlo: 

#### Message passing 
Estos mensajes son enviados a traves de la red, pasados por un proceso utilizando un "pipe" o transmitods a traves de una cola de mensajes en una memoria accesible por los dos, el que envía y el que recibe. Este modelo puede ser usado para una sola computadora y por varios threads dentro de un proceso corriendo fisicamente por distintas computadoras

#### Shared memory
En este tipo de comunicación dos o más threads reciben acceso a un mismo bloque de memoria fisica, por ende pueden operar directamente en cualquier "data objects" que resida en el area de memoria.  Este modelo solo funciona para threads que corren en una misma computadora con un banco fisico RAM que puede ser visto por todos los CPU cores. 

#### Distrubuted shared memory
A partir de la *ilusión* de shared memory entre computadoras separadas fisicamente se puede implementar por encima de un **message-passing-system**. 
Además se puede implementar un Message passing por encima de un **shared memory architecture**, implementando una cola de mensajes que reside dentro de la memoria compartida.

### Diferencias
**Physically-shared memory** por obvias razones es la forma más eficiente de compartir una mayor cantidad de datos,  esto es ya que no hay que copiarlos para poder transmitirlos.  Sin embargo el compartir recursos de cualquier tipo brinda problemas de sincronización.  Por otro lado el sistema de mensajería aliviana, pero no resuelve, los problemas de sincronización. 

