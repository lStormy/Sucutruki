>La implementación la podes encontrar en [repositorio de Nakash](https://github.com/lStormy/Nakash)

## Como puede ser implementado
A partirde un *==`std::atmomic_flag`==*, en este caso dentro de una clase de C++. Un spinlock es obtenido utilizando un [[Race conditions|TAS]] para que atomicamente seteamos la flag (de `std::atomic_flag`) a `true`, reintentando en el while loop hasta que el TAS sea `true`.

Es importante que se utilice un *read-acquire memory ordering semantics* para leer el contenido actual del lock como parte de la operación TAS. Esto resguarda en contra del raro escenario en el cual el lock está siendo observado  mientras está siendo liberado cuando en realidad algún otro thread todavía no ha salido del estado critico. 

Cuando liberamos el spinlock, es importante usar una *write-release semantics* para asegurar que todo lo escrito hecho después de la llamada a `Unlock()` no está siendo observado por otros threads. 

```cpp
#include <thread>
#include <atomic>
#include <memory_resource>
class SpinLock {
	std::atomic_flag m_atomic;
public:
	SpinLock() : m_atomic(false) { }
	bool TryAcquire() {
		// use an acquire fence to ensure all subsequent
		// reads by this thread will be valid
		bool alreadyLocked = m_atomic.test_and_set( std::memory_order_acquire);
		return !alreadyLocked;
	}
	void Acquire(){
		// spin until successful acquire
		while (!TryAcquire()) {
			std::this_thread::yield();
		}
	}
	void Release() {
		// use release semantics to ensure that all prior
		// writes have been fully committed before we unlock
		m_atomic.clear(std::memory_order_release);
	}
};

int main () {
	return 0;
}
```
>Corre perfecto. 


# Push Lock 
Necesitamos un tipo de lock que ==permita cualquier número de lectores lo puedan adquirir concurrentemente==. Cuando un thread escritor quiera adquirir el lock, ==debería de esperar a que todos los demás threads lectores hayan terminado==, y después deeberá de adquirir el lock de en un ==modo especial "exclusivo"== que previene a cualquier lector o escritor en obtener acceso hasta que no haya completado la mutación del objeto. 
Se puede implementar este lock de una manera parecida al *Reentrant Lock* (disponible en el repositorio). En vez de tener el *thread id* guardamos la cantidad de *threads lectores*, cuando ==uno lo adquiere se suma la referencia y cuando uno lo libera se resta==. 

# Read-Copy-Update Lock
Consigue escalabilidad permitiendo que las lecturas puedan ocurrir concurrentemente con actualizaciones. 
#ToDo: Implementar esta garcha

# Lock-Not-Needed Assertions 
Hay veces que durante el juego puede suceder que dos operaciones se solapen a pesar de no ser posible para el programador. Pero en este caso si creemos de que no va a suceder de que se solapen entonces utilizamos uno de estos locks que no son tan caros como un spinlock. 
Utiliza un *Volatile* boolean en vez de un *atomic*, ya que no nos tenemos que garantizar de que siempre se arregle, sino que el 90% de los casos aproximadamente. Esto se debe a que como no va a ocurrir tanto entonces sería un gasto de recursos inecesarios utilizar un atomic. 

```
#if ASSERTIONS_ENABLED
#define BEGIN_ASSERT_LOCK_NOT_NECESSARY(L) (L).Acquire()
#define END_ASSERT_LOCK_NOT_NECESSARY(L) (L).Release()
#else
#define BEGIN_ASSERT_LOCK_NOT_NECESSARY(L)
#define END_ASSERT_LOCK_NOT_NECESSARY(L)
#endif
```
En el compilador: 
`g++ -DASSERTIONS_ENABLED -o my_program main.cpp`
