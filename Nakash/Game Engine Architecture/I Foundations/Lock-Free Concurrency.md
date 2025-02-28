> Refiere a la práctica, de [[Concurrency]], intentando  prevenir [[Threads]] en entrar en un estado de bloqueo, en otras palabras en que siempre estén disponibles. Contrario a la creencia de que se basa en no usar [[Mutex]]. 
> Es una forma de evadir las [[Race conditions]] y es una tecnica proveniente de la colección de "non-blocking concurrent programming techniques". 

### Podemos organizar las tecnicas de "non-blocking" en las siguientes categorías: 
#### 1. Blocking
Un algoritmo de bloqueo es uno el que pone a dormir los threads esperando a un recurso a volverse disponible. Esto puede llevar a ["Deadlock, livelock, starvation y priority inversion"](Mutex) 
#### 2. Obstruction freedom
==Un algoritmo de este estilo garantiza que un solo thread pueda completar su trabajo en una estricto numero de pasos==, mientras todos los otros threads del sistema esten suspendidos. Este thread solitario está llevando a cabo una "solo exection" y a veces este tipo de algoritmos se lellama "solo-terminating". Ningún algoritmo que use un lock puede llamarse obstruction freedom, ==ya que si cualquiera de los trheads termina estando suspendido mientras mantiene un lock el thread solitario puede llegar a estancarse sin terminar su ejecución==.
#### 3. Lock freedom 
*Por cada programa infinitamente largo, una cantidad infinita de operaciones van a ser completadas*. De forma intuitiva, un algoritmo de este estilo garantiza que siempre va a haber algún thread en el sistema que puede hacer progresos. También prohíbe el uso de locks. 
Este es del tipo *Transaction-based*, esto evita el Deadlock pero puede permitir algunos threads a sufrir "Starvation"
#### 4. Wait freedom
==Provee todo lo que garantiza el lock freedom, pero además garantiza que no haya starvation.== En otras palabras, todos los threads puede hacer progresos y ningún thread tiene permitido "Starve" de forma indeterminada. 

##### Progesimos desde acá a [[Race conditions]]


