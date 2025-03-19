>A Mutex is an operating system object that allows critical operations to be made atomic

#### Posibles estados
1. Unlocked
2. Locked

Lo más importante sobre un mutex es que garantiza que solo un [Thread](Threads) lo va a utilizar en un cierto tiempo. Por lo tanto si juntamos todas las operaciones de un objeto compartiod en particular en un mutex, entonces esas operaciones se vuelven atomicas, de forma relativa en una con la otra. En otras palabras se vuelven mutuamente exclusivas (mutex).

##### Aclaración:
Hay muchas más estructuras que implementan estos tipos de soluciones a [[Race conditions|Critical operations]] y pasarlas a atomic operations.
Como *Semaforos*, *Condition variables* y quizá *Futures and promise* (este último no lo conozco tanto). 

***Pero además hay un problema con esta forma de concurrencia...***

### Problemas de la "Lock-Based" [[Concurrency]]
Además de las [[Race conditions|Atomicidades]] hay otros problemas en [Concurrencia](Concurrency), incluso habiendo usado resguardado todos los objetos compartidos en memoria usando locks. 

#### 1. Deadlocks
Una situación en la que ningún Thread en el sistema puede lograr un progreso, resultando en un "hang". Esto ocurre cuando todos los Threads están esperando a que algo suceda, en el estado Bloqueado. Pero como ningún Thread está en un estado de Correr, entonces nunca se van a volver disponibles, por lo tanto todo el programa se congela o "hangs".
 
```cpp
cpp
void Thread1() {
	g_mutexa.lock(); //holds lock for resource A
	g_mutexb.lock(); //Sleeps waiting for resource B
	//...
}

void Thread2 () {
	g_mutexb.lock(); //holds lock for b
	g_mutexa.lock(); //Sleeps waiting a
	//...
}
```

**E**sto se puede solucionar logrando que las **"Condiciones de Coffman"** no se cumplan: 
1. *Mutual exclusión*: Un solo thread puede recibir acceso acceso exclusivo a un solo recurso via un mutex lock
2. *Hold and wait*: Un thread debe mantener un lock cuando va a dormir esperando por otro lock
3. *No lock preemption*. Nadie (ni siquiera el Kernel) tiene permitido romper un lock mantenido por un thread durmiendo
4. *Circular wait*: Debe haber una salida ciclica en un grafo de dependencia de threads. 

![[Pasted image 20250221173516.png]]
>Ejemplo de grafo de dependencia de threads a partir del codigo que está arriba... 

Como romper las condiciones 1 y 4 es hacer "trampa" entonces la solución a este problema termina siendo concentrandose en evitar las condiciones 2 y 3.

**C**ondición 2: ==Se puede evitar utilizando menos locks.== Por ejemplo en el codigo que tenemos podemos ver como hay una dependencia circular entre T1 y T2. Si damos acceso exclusivo utilizando solo un Lock entonces o T1 depende de T2 o al revés. 
**C**ondición 4: ==Se puede solucionar imponiendo un orden global a todos los "Lock-Tacking" en el sistema==. En el ejemplo de dos Threads de arriba, podemos solucionarlo asegurandonos que siempre se habilite el lock del recurso A antes del lock del recurso B. Esto va a funcionar por que un thread va a obtener el lock en el recurso A antes de intentar tomar cualquiera de los otros locks. Esto efectivamente pone todos los threads a dormir por consecuencia asegurandose que tomar el lock del recurso B siempre funcionando. 

#### 2. Livelock: 
Otra solución a ==Deadlock== es utilizar una función `pthread_mutex_trylock()` va a intentar tomar el lock, si no lo logra pone a dormir el thread y lo vuelve a intentar. 
Cuando un Thread usa una estrategía explicita, como re-intentos programados por tiempo. para evadir o solucionar un Deadlock, un nuevo problema puede aparecer: ==Los threads pueden llegar a estar todo su tiempo intentando resolver un deadlock, incluso más que intentando hacer un trabajo real==. Esto último es un *Livelock*.

#### 3. Starvation 
==Sucede cuando hay uno o más threads que no reciban ningún tiempo de execución por parte del CPU==. Puede suceder si uno o más "higher-priority" Threads falla en recobrar el control del cpu, derivando en una prevención de los "lower-priority" threads de ejecutarse. *Livelock* es un tipo de *Starvation*, en donde un algoritmo de resolución de *Deadlock*, de forma efectiva, anúla a todos los threads ("starves" all threads) de su habilidad de hacer un "trabajo real". 

#### 4. Priority Inversion
Mutex locks pueden llevar a situaciones donde "lower-priority" threads actuan como si fueran unos "highest-priority" threads en el sistema. 
#ToDo: Desarrollarlo

#### 5. The dining Philosofers
Es una forma famosa de ilustrar los problemas de Deadlock, Livelock y starvation. 
==Hay una mesa circular, cada filosofo está sentado al rededor de esta mesa, todos con un plato de fideos adelante==. Entre cada filosofo hay un ==cubierto para comer==. Los filosofos desean alternar entre *pensar* (lo cual pueden hacer sin la necesidad de usar los cubiertos) y *comer* (Necesitan dos cubiertos). El problema es encontrar un patrón en el que puedan alternar entre comer y pensar sin sufrir ninguno de los problemas antes mencionados. 

#ToDo: Probablemente en un futuro tengamos que volver a la parte "Some Rules for Thumb Concurrency". 