> Las interrupciones no son las únicas que pueden generar Data Race Bugs vistos en [[Race conditions]]. Los compiladores y el cpu pueden llegara introducir bugs sutiles en un programa concurrente por culpa del *Instructión reordering optimizactions* . 

## ¿Como nos aurrina esto?
Tomemos el caso de: 
1. `A = B + 1`; 
2. `B = 0`;
Lo cual va a ser traducido a: 
1. `mov eax, [B]`
2. `add eax, 1 
3. `mov [A], eax
4. `mov [B], 0
Lo cual por la lógica de ejecución *Out-of-order*, esto se termina re ordenando a: 
1. `mov eax, [B]`
2. `mov [B], 0
3. `add eax, 1 
4. `mov [A], eax
No habría problema si estamos trabajando con un solo thread. Pero supongamos que el thread1 está ejecutando esto, y el thread2 está esperando a que B sea 0 para poder tomar el valor de A. En ese caso va a leer un valor de A que no debería. 

>Jeff Preshing wrote a great blog post on this topic, available at http://preshing.com/20120625/memory-ordering-at-compile-time

## ¿Soluciones?
### Volatile:
==Volatile garantiza que una consecutiva cantidad de lecturas y escrituras de una variable no pueden ser optimizadas, por ende no puede reordenarlas.== Parece ser que arregla todo, pero no funciona tan confiablemente por varias razones. 
El principal problema es que ==puede no funcionar en algunas cpus en especifico==, por lo tanto es un problema para hacer codigo portable. 
==Además no previene al cpu de utilizar la out-of-order execution logic a re ordenar las instrucciones cuando está corriendo el programa.==
#### Memory cahing
Una memoria cache evita el trafico abultado en la main RAM manteniendo los datos frecuentados en el cache. Esto quiere decir que mientras que el dato está presnete en el cche, el cpu siempre va a intentar trabajar con la copia "cacheada", en vez de ir a por la copia accesible desde la main RAM. 
Esto último es más complicado en una maquina multicore. C==ada core tiene su propio cache y un cache compartido entre cores.== En caso de que un thread produzca un dato y después otro quiera leerlo, el primer thread no va a subir el dato al main RAM hasta que termine de ejecutarse. Por lo tanto ==el segundo thread va a intentar comunicarse con el primero pidiendole una copia del dato que necesita==, porque esto último sigue siendo menos costoso que leerlo de la RAM misma. Esto se llama *cache coherency protocol*.
##### Mesi protocol
Es un tipo de *Cache coherency protocol*. En donde cada linea de cache puede estar en los siguientes cuatro estados: 
1. **Modified**. Ha sido modificada (escrita) localmente.
2. **Exclusiva**. El bloque de memoria de la main RAM correspondiente a esta linea de cache existe solamente en el cache de un core en especifico, donde ningún otro core tiene una copia. 
3. **Compartida**. El bloque de memoria de la main RAM correspondiente a esta linea de cache existe en más de un cache del core, y todos los cores donde se encuentran tienen una copia identica.
4. **Invalidada**. Está línea de cache no tiene data valida. La próxima vez que se lea va a obetener una línea del cache de otro core, o desde la misma main RAM. 
==Todos los caches de los cores están conectados por un bus llamado *interconnect bus* (ICB).==
![[Pasted image 20250222173948.png]]
El problema está en que en las optimizaciones dentro del protocolo del cache coherency, puede suceder que dos instrucciones de lecturas y/o escrituras parezcan suceder, desde un punto de vista de otros cores en el sistema, en un orden que es el opuesto al orden en el que las instrucciones se suponían que tenían que ejecutarse.  
##### Memory Fences
Hay cuatro formas en la que una instrucción que fue ejecutada después que otra pase a la primera: 
1. Un read puede pasar a otro read
2. Un read puede pasar a un write
3. Un write puede pasar a write, ó
4. Un write puede pasar a otro read. 
Para impedir que esto suceda las cpus modernas proveen instrucciones en lenguaje de maquina llamadas *Memory fences* ó también *Memory Barriers*. 
==Estas fences pueden ser unidireccionales o bidireccionales.== Una instrucción unidireccional va a garantizar que ningún read o write que precede del orden de su programa  pueda tener un efecto después de este, o vice-versa. Una bidireccional va a prevenir efectos de "leakeage" de memoria en cualquier dirección. 
Todas las instrucciones de fence tienen dos efectos secundarios muy útiles: 
1. Sirven como compiler barriers, y 
2. previenen que la cpu ejecute out-of-order logic para reordenar instruccionesa al otro lado de las fences. 
#### Memory Ordering semantics
Hay 3 semanticas de ordenamientos, por lo menos las más importantes: 
1. **Release**: Garantiza que ningún write a una memoria compartida nunca va a ser pasado por otro read o write que lo precede en el orden del programa. Se lo llama "write release". 
2. **Acquire**: Garantiza que ningún read va a ser pasado por otro read o write que corre antes en el orden del programa. 
3. **Full fence**: Ningún read o write que ocurra antes del fence, en orden del programa, puede aparecer haber ocurrido antes del fence. 
##### Cuando usar un Acquire y cuando un Release
Un relese se suele usar en un esenario de productor, en el caso de que un thread haga dos writes consecutivos, y nos querramos asegurar de que todos los demás threads vean esos dos writes en el orden correct. ==Podemos asegurarnos de esto haciendo que el segundo de estos sea un **write-release**==. 
En el caso de Acquire es usado en un escenario de consumidor, por ejemplo que tnega que leer dos reads consecutivos y el segundo sea un condicional del primero. ==Nos aseguramos de del orden haciendo que el primero sea un **read-acquire**.== 

```cpp
int32_t
int32_t
g_data = 0;
g_ready = 0;
void ProducerThread() { // running on Core 1
	g_data = 42;
	// make the write to g_ready into a write-release
	// by placing a release fence *before* it
	RELEASE_FENCE();
	g_ready = 1;
}
void ConsumerThread() { // running on Core 2
	// make the read of g_ready into a read-acquire
	// by placing an acquire fence *after* it
	while (!g_ready)
		PAUSE();
	ACQUIRE_FENCE();
	// we can now read g_data safely...
	ASSERT(g_data == 42);
}
```

`RELEASE_FENCE();`: Espera a que todos las writes anteriores hayan sido hechas. 
`ACQUIRE_FENCE();`: Espera a que todos los reads anterioes hayan sido hecchos. 
### Memory order
`std::atomic<T>` permite declarar variables que producen sus operaciones de lectura y escritura utilizando fences para que sean atomicas. 
Pero para que en todos los casos estas funcionen correctamente usan full memory barriers. Esto se puede relajar pasasndo una *==memory order semantic==* (std::memory_order) a funciones que manipulan variables atomicas. 
1. **==Relaxed==**: 

