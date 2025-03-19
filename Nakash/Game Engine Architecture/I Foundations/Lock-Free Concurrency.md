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


>Concurrency is a vast and profound topic, and in this chapter we’ve only just
scratched the surface. As always, the goal of this book is merely to build aware-
ness and serve as a jumping-off point for further learning.
• For a complete discussion of implementing a lock-free singly-linked list, check out Herb Sutter’s talk at CppCon 2014, which is where the example above came from. The talk is available on YouTube in two parts:
>- https://www.youtube.com/watch?v=c1gO9aB9nbs, and 
 >- https://www.youtube.com/watch?v=CmxkPChOcvw.

>This lecture by Geoff Langdale of CMU provides a great overview: https: //www.cs.cmu.edu/~410-s05/lectures/L31_LockFree.pdf.

>Also be sure to check out this presentation by Samy Al Bahra for a clear and easily digestible overview of pretty much every topic under the sun related to concurrent programming: http://concurrencykit.org/presentations/lockfree_introduction/#/.

>Mike Acton’s excellent talk on concurrent thinking is a must-read; it is available at http://cellperformance.beyond3d.com/articles/public/concurrency_rabit_hole.pdf.

>These two online books are great resources for concurrent programming:http://greenteapress.com/semaphores/LittleBookOfSemaphores.pdf and https://www.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.2011.01.02a.pdf.

>Some excellent articles on lock-free programming and how atomics, barriers and fences work can be found on Jeff Preshing’s blog: http://preshing.com/20120612/an-introduction-to-lock-free-programming.

>This page has great information about memory barriers on Linux: https: //www.mjmwired.net/kernel/Documentation/memory-barriers.txt#305

