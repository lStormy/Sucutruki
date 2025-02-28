>Son un tipo de [[User-level Thread]].

Los coroutines son utiles para implementar programas que son asincronos e inherentes. Es también una generalización del concepto de las subrutinas. Donde las subrutinas solo pueden "salir" devolviendo el control al "caller", en cambio las corrutinas, además pueden salir cediendole el control a otra corrutina. Cuando cede el control esta mantiene su contexto en memoria. Y la próxima vez que es llamado (siendo cedido por otra corrutina) esta continua desde donde dejó.
*Un pseudocodigo del libro*
```cpp
Queue g_queue;

Coroutine void produce() {
	while (true) {
		while (!g_queue.IsFull())
			CreateItemAddToQueue(g_queue);
		YieldToCoroutine(consume);
		//continue en el proximo Yield	
	}
}

Coroutine void consume() {
	while (true){
		while (!g_queue.IsEmpty) {
			ConsumeItemFromQueue(g_queue);
		}
		YieldToCoroutine(produce);
		//continua en el proximo Yield
	}
}
```

#### Como implementar Coroutines con Ucontext_H 
> Bibliografía: [Implementing Couroutines with ucontext](https://probablydance.com/2012/11/18/implementing-coroutines-with-ucontext/)

##### ¿Por qué implementarlo con ucontext? 
En vez de usar ucontext podrías usar boost el cual no es conveniente debido a que teine 600 includes y el archivo preprocesado tendría 75000 líneas. 

Boost intenta que su codigo compile en todas las maquinas y compiladores. Tiene que ser "exception safe" y "thread safe" sumado a esto tiene que ser lo más eficiente posible. Sin embargo para esto sacrifican tiempo de compilación. 

``` cpp
#include <ucontext.h>
#include <cstdint>
#include <memory>
#include <functional>
 
struct coroutine
{
    coroutine(const std::function<void (coroutine &)> func, size_t stack_size = SIGSTKSZ)
        : stack{new unsigned char[stack_size]}, func{func}
    {
        getcontext(&callee);
        callee.uc_link = &caller;
        callee.uc_stack.ss_size = stack_size;
        callee.uc_stack.ss_sp = stack.get();
        makecontext(&callee, reinterpret_cast<void (*)()>(&coroutine_call), 2, reinterpret_cast<size_t>(this) >> 32, this);
    }
    coroutine(const coroutine &) = delete;
    coroutine & operator=(const coroutine &) = delete;
    coroutine(coroutine &&) = default;
    coroutine & operator=(coroutine &&) = default;
 
    void operator()()
    {
        if (returned) return;
        swapcontext(&caller, &callee);
    }
 
    operator bool() const
    {
        return !returned;
    }
 
    void yield()
    {
        swapcontext(&callee, &caller);
    }
 
private:
    ucontext_t caller;
    ucontext_t callee;
    std::unique_ptr<unsigned char[]> stack;
    std::function<void (coroutine &)> func;
    bool returned = false;
 
    static void coroutine_call(uint32_t this_pointer_left_half, uint32_t this_pointer_right_half)
    {
        coroutine & this_ = *reinterpret_cast<coroutine *>((static_cast<size_t>(this_pointer_left_half) << 32) + this_pointer_right_half);
        this_.func(this_);
        this_.returned = true;
    }
};
```

###### Paso 1: 
```cpp
coroutine (const std::function<void (coroutine &)> func, size_t stack_size = SIGSTKSZ): stack {new unsigned char [stacksize]} func{func}
```
**Argumentos:**
- Una función que toma como argumento la dirección de memoria de una corrutina
- Una variable size_t que toma como valor la constante SIGSTKSZ 
- Después inicializa sus valores con los argumentos

###### Paso 2:
``` cpp
coroutine (const std::function<void (coroutine &)> func, size_t stack_size = SIGSTKSZ): stack {new unsigned char [stacksize]} func{func} {
        getcontext(&callee);
        callee.uc_link = &caller;
        callee.uc_stack.ss_size = stack_size;
        callee.uc_stack.ss_sp = stack.get();
        makecontext(&callee, reinterpret_cast<void (*)()>(&coroutine_call), 2, reinterpret_cast<size_t>(this) >> 32, this);
}
```
**Funcionamiento:**
- Captura el contexto actual de `callee` usando `getcontext`
- Crea un link de `callee` a `caller`, esto para cuando termine lo que está haciendo retorne al `caller` 
- Inicializa el stack y su tamaño
- Por último el `makecontext()` configura el `callee` que teníamos para ejecutar la función especifica cuando se cambia el contexto, en este caso es `couroutine_call`. Por último como argumentos pasa: 
		-  `reinterpret_cast<size_t>(this) >> 32` extrae la parte alta del puntero this
		- `This` se pasa como argumento
		- Esto permite reconstruir el puntero en arquitecturas de 64 bits donde `makecontext()` solo acepta argumentos de 32 bits.

###### Paso 3:

``` cpp
void operator()() {
    if (returned) return;
    swapcontext(&caller, &callee);
}
 
operator bool() const {
    return !returned;
}
 
void yield() {
    swapcontext(&callee, &caller);
}
```
**Funciones: 
- `Operator()` no devuelve nada si ya fue retornada, sino intercambia el contexto del `caller` con el `callee`
- `bool()` devuelve un true si no fue retornado
- `yield()` intercambia el contexto del `callee` con el `caller`, lo cual suspende la ejecución de la corrutina. Continua la ejecución desde el contexto donde estaba antes de llamarse a la corrutina.

###### Paso 4: 
``` cpp
ucontext_t caller;
ucontext_t callee;
std::unique_ptr<unsigned char[]> stack;
std::function<void (coroutine &)> func;
bool returned = false;
 
```
**Atributos:**
- `caller`: Contexto de lo que llama a esta corrutina
- `callee`: El contexto donde se ejecutara la corrutina
- `std::unique_ptr<unsigned char[]> stack`: El stack de la corrutina
- `func`: la función que ejecuta la rutina
- `returned`: Es True si ya ha retornado el contexto de `caller` por lo menos una vez.

###### Paso 5:
``` cpp
static void coroutine_call(uint32_t this_pointer_left_half, uint32_t this_pointer_right_half) {
    coroutine & this_ = *reinterpret_cast<coroutine * ((static_cast<size_t>(this_pointer_left_half) << 32) + this_pointer_right_half);
    this_.func(this_);
    this_.returned = true;
}

```
- Recrea una corrutina `this` apartir de dos partes: 
	- `static_cast<size_t> (this_pointer_left_half)`
	- this_pointer right_half
	- Corre hacia a la izquierda 32 bits a la parte izquierda y después suma la otra parte dejando las dos partes ordenadas y creando un puntero de 64 bits.
	- Lo castea a una corrutina `reinterpret_cast<coroutine *>`
- Ejecuta la función
- Y como ya fue llamada la función asigna un true al `returned`.
	
### **¿Por qué se usa esta técnica?**

- `makecontext()` en `ucontext.h` permite pasar argumentos a la función de contexto, pero solo admite **argumentos de 32 bits**.
- En sistemas de 64 bits, los punteros ocupan 64 bits, por lo que se dividen en **dos valores de 32 bits** para ser compatibles con `makecontext()`.
- La función `coroutine_call` se encarga de **reconstruir el puntero de 64 bits y ejecutar la corrutina**.


#### Implementar Coroutines sin necesidad de Ucontext
> Bibliografía [Handmade Coroutines ](https://probablydance.com/2013/02/20/handmade-coroutines-for-windows/)

##### Es importante explicar dos funciones primero: 

```cpp 
void stack_context::switch_into()
{
    asm("call switch_to_callable_context"
        : : "D"(&caller_stack_top), "S"(my_stack_top), "d" rbp_on_stack));
}
```
> Llama a una función en assembly que se llama "switch_to_callable_context" con los argumentos
- "D" (&caller_stack_top),
- "S" (my_stack_top),
- "d" (rbp_on_stack);

```assembly
switch_to_callable_context:
    pushq %rbp
    movq %rsp, %rbp
    // store rbx and r12 to r15 on the stack. these will be restored
    // after we jump back
    pushq %rbx
    pushq %r12
    pushq %r13
    pushq %r14
    pushq %r15
    movq %rsp, (%rdi) // store stack pointer
    // set up the other guy's stack pointer to make
    // debugging easier
    movq %rbp, (%rdx)
    jmp switch_point
```
>Con la llamada desde `switch_into` los registros:
- %rdi -> Dirección de `caller_stack_top` (para guardar el stack del llamador)
- %rsi -> my_stackt_top (puntero al nuevo stack)
- %rdx -> rbp_on_stack (puntero para almacenar el rbp)
>Lo que hace: 
1. Guarda el rbp actual en la pila, asegurando que cuando regresemos al contexto original, podamos restaurarlo.
2. Actualiza rbp para que refleje el nuevo frame de la función `switch_to_callable_context`
3. Guarda los registros rbx y r12 a r15 en la pila rsp
4. Guarda el *stack pointer* (rsp), en el `caller_stack_top` (que está guardado en el rdi).
5. Guarda rbp en `rbp_on_stack` (que está guardado en rdx)
6. Transfiere la ejecución a `switch_point`

```
switch_point:
    movq %rsi, %rsp     # Restaura el stack pointer del nuevo contexto
    popq %r15           # Restaura %r15
    popq %r14           # Restaura %r14
    popq %r13           # Restaura %r13
    popq %r12           # Restaura %r12
    popq %rbx           # Restaura %rbx
    popq %rbp           # Restaura %rbp
    retq                # Retorna al nuevo contexto

```

Esto lo que hace en resumidas cuentas es volver al contexto en el que inicialmente estabamos

``` cpp
static void * ensure_alignment(void * stack, size_t stack_size)
{
    static const size_t CONTEXT_STACK_ALIGNMENT = 16;
    unsigned char * stack_top = static_cast<unsigned char *>(stack) + stack_size;
    // if the user gave me a non-aligned stack, just cut a couple bytes off from the top
    return stack_top - reinterpret_cast<size_t>(stack_top) % CONTEXT_STACK_ALIGNMENT;
}
 
stack_context::stack_context(void * stack, size_t stack_size, void (* function)(void *), void * arg)
    : caller_stack_top(nullptr), my_stack_top(nullptr)
    , rbp_on_stack(nullptr)
{
    unsigned char * math_stack = static_cast<unsigned char *>(ensure_alignment(stack, stack_size));
    my_stack_top = math_stack - sizeof(void *) * 9;
    void ** initial_stack = static_cast<void **>(my_stack_top);
    // store the return address here to make the debuggers life easier
    asm("movq $switch_point, %0\n\t" : : "m"(initial_stack[8]));
    initial_stack[7] = nullptr; // will store rbp here to make the debuggers life easier
    asm("movq $callable_context_start, %0\n\t" : : "m"(initial_stack[6]));
    rbp_on_stack = initial_stack[5] = &initial_stack[7]; // initial rbp
    initial_stack[4] = &caller_stack_top; // initial rbx
    initial_stack[3] = reinterpret_cast<void *>(function); // initial r12
    initial_stack[2] = arg; // initial r13
    initial_stack[1] = nullptr; // initial r14
    initial_stack[0] = nullptr; // initial r15
}
```
- [ ] #ToDo:  Enteder esto.
- [x] #Hecho: Implementar corrutinas a partir de esa implementación. [Implementado en el repositorio del engine](https://github.com/lStormy/Nakash.git)
 

#ToDo Ideas para seguir implementando: 
1. Templates arguments
