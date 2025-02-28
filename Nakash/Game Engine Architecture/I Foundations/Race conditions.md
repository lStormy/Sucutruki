>En [[Concurrency]] una race condition sucede cuando el programa puede variar dependiendo del  estado del tiempo. Esto quiere decir que el programa puede llegar a comportarse de distinta manera cuando la relativa cantidad de secuanecia de eventos al rededor del sistema cambia. 

## Critical Races
El hecho de que el programa cambie dependiendo del tiempo no causa effectos inadecuados. Sin embargo, una *critical race* es una *Race condition* que puede llegar a cuasar un comportamiento inadecuado del programa. 
Estos tipos de "bugs" a veces son "extraños" o incluso "imposibles"
>Programmers often call these kinds of issues Heisenbugs

### Data Races
Estos son un tipo de Critical Races, en donde dos controles de flujos interfieren el uno con el otro mientras leen  y/o escriben en un bloque de datos compartidos, resultando en datos corruptos.
La idea en concurrencia es eliminar la mayor cantidad de "Data Races" posibles y evitar estos bugs.
En resumidas cuentas, un data race bug ocurre: 
- Un thread previene a otro en un solo core,
- O dos o más operaciones criticas ocurren a la vez. 
##### Critical operations
Son aquellas operaciones que pueden leer o cambiar un objeto compartido en particular. Para garantizar que el objeto compartido está libre de cualquier "data race bug", se debe asegurar que ninguna de sus "critical operaciones" se pueden interrumpir el una con la otra. Cuando una se vuelve interrumpible se llama *atomic operation*. 

****
> A critical operation can be said to have executed atomically if its invocation and response are not interrupted by another critical operaration on that same object. 
****

Si a una operación critica la interrumpe una *operación no critica* entonces no tiene por qué salir nada mal. Sin embargo el problema ocurre si a una operación critica se la interrumpe con otra operación critica en el mismo objeto. Si a una operación critica se la interrumpe con otra operación critica, la cuales suceden en dos objetos distintos, entonces está todo bien. 
Podemos asegurar que una operación critica ocurra de forma atomica si y solo si ocurre instantaneamente, de esta forma sería interrumpible, ya que ninguna otra operación podría caber en un espacio tiempo negativo. 

#### Como convertir una operación a una operación atomica 
>The easiest way and most reliable way to accomplish this is to use a special object called mutex. An object provided by the operating system that acts like a padlock, and it can be locked and unlocked by a Thread. 

Como un [[Mutex]] solo puede ser activado por un Thread a la vez, si activamos el mutex todo lo que esté resguardado por este va a ser una operación atomica. Esto nos permite operar criticamente sobre un objeto sin la consecuencia de la corrupción de datos. 
Ahora esto nos deja con un sistema en el que está separado por distintos controles de flujo que suceden de forma ordenada, orden el cuál no conocemos, pero no se solapan entre sí. 

Devuelta volvemos a la explicación del por qué usar [[User-level Thread]], ya que el utilizar herramientas de "Thread synchronization" se utilizan llamadas al kernel, los cuales requieren un context switch hacía un modo protegido. 
>Such context Switches can cost upwards of 1000 clock cycles.

Debido a esto, quizá sea mejor implementar algún tipo de "atomicity and synchronization tool" o cambiar a "lock-free programming". 

### Utilizando operaciones atomicas para transformar operaciones criticas en atomicas
>La falopa es terrible

Hay algunas instrucciones en lenguaje de maquina que pueden ser atomicas, aunque hay otras que jamás se pueden asumir como atomicas. ==Algunos CPU permiten cualquier instrucción sea *forzada* a ser ejecutada de forma atomica especificando un "prefix" en la instrucción en el lenguaje assembly==.
Diferentes CPUs e ISAS (instruction set Achitecture) proveen distintos sets de instrucciones atomicas. Pero todas caen en 2 categorías: 
- Atomic reads and writes, and
- Atomic read-modify-write (RMW) instructions. 

La instrucción RMW más simple es el TAS (test-and-set). De forma atomica setea una variable booleana a true y retorna si valor previo (para que pueda ser testeada).
``` cpp
bool TAS (bool * plock) {
	const bool old = *plock; 
	*plock = true;
	return old; 
}
```
El Tas puede ser usado para implementar un tipo de  estructura simple de lock llamada [[Spin Lock]]. 
Un ejemplo de implementación podría ser este: 

``` cpp
void SpinLockTas(bool * plock) {
	while (_tas(plock) == true) {
		//Alguien tiene el lock
		PAUSE(); 
	}
}
```


Algunas CPU proveen una instrucción llamada *compare-and-Swap* (CAS). Cheuqa el valor existente en la memoria y lo cambia atomicamente con un nuevo valor que *Si y solo si*  el valor existente es igual al esperado (ingresado por el usuario). 

```cpp
bool CAS (int * pValue, int expectedValue, int newValue) {
	if (*pValue == expectedValue) {
		*pValue = newValue; 
		return true; 
	}
	return false;
}
```

==Ayuda a detectar Data Races==, comparando el valor que está en la memoria que tenemos en el tiempo que vamos a escribir el valor antes de invocar el RMW.  
Sin embargo el problema está en el caso de "ABA", si leemos A, después alguien cambia A a ser B y al instante pasa B a ser A entonces no estaríamos dandonos cuenta de que sucedió un data race. 
Este problema lo resuelve Load linker/store conditional (LL/SC). El load linker lee, guarda en en un registro "link register" y después SC cambia el valor si y solo si el valro  la dirección que tiene el valor es igual al del registro. 
LL/SC solo neceista un instante de la memoria para poder llevarse a cabo. Por otro lado CAS necesita dos.

Siguiendo a partir de [[Instruction Reordering]]
