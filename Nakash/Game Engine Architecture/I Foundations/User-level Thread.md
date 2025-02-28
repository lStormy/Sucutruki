Como los [[Threads]] "viven" en el espacio del kernel cada vez que se llame a una función para interactuar con un thread requiere un **Contex Switch hacia el kernel**. Los cuales son operaciones muy costosas. Acá entran los *Thread en el user-level*, los cuales son más "livianos" ya que no es necesario utilizar operaciones de Context Switching. Pero a la vez permiten esta forma de trabajar con multiples e independientes flujos de control, cada uno con su propio contexto de ejecución. 
El kernel no sabe nada de ellos, ya que estos están implementados por completo en el espacio de usuario. Cada uno de estos thread es representado por: Un tipo de dato ordinario que representa el id (normalmente un nombre), el contexto de ejecución (el contenido de los registros del CPU y el call stack). Cada uno de estos threads corre tdentre el contexto de un thread o [[Fibers]] "real".
Para poder implementar estos es necesario descubrir la forma de implementar el context switch. Desde un punto de vista más simplista el context switch lo único que hace es reemplazar los valores que tienen los registros del CPU. 

Referencias a futuro:
www.boost.org

Esta es la forma para poder implementar el Context Switching: 
```
switch_to_context:
	pushq %rbp
    movq %rsp, %rbp
    // store rbx and r12 to r15 on the stack. these will be restored
    // after we switch back
    pushq %rbx
    pushq %r12
    pushq %r13
    pushq %r14
    pushq %r15
    movq %rsp, (%rdi) // store stack pointer
switch_point:
    // set up the other guy's stack pointer
    movq %rsi, %rsp
    // and we are now in the other context
    // restore registers
    popq %r15
    popq %r14
    popq %r13
    popq %r12
    popq %rbx
    popq %rbp
    retq
```

## Análisis de chatgpt
#### **1. Guardar el contexto actual**

`pushq %rbp movq %rsp, %rbp`
- **`pushq %rbp`**: Guarda el valor actual del **registro de base (`rbp`)** en la pila.
- **`movq %rsp, %rbp`**: Establece `rbp` como el nuevo marco de pila.

`pushq %rbx pushq %r12 pushq %r13 pushq %r14 pushq %r15`
- Guarda los registros **rbx, r12, r13, r14 y r15** en la pila.
- Estos registros deben preservarse entre llamadas a funciones según la convención de llamadas de x86-64 System V ABI.

`movq %rsp, (%rdi) // store stack pointer`
- Guarda el valor del **puntero de pila (`rsp`)** en la dirección de memoria apuntada por `rdi`.
- `rdi` parece ser un puntero a una variable donde se almacena el **stack pointer del contexto actual**.

#### **2. Restaurar el contexto de otro proceso**

`movq %rsi, %rsp`
- Carga un nuevo valor en `rsp`, cambiando la pila al contexto de otro proceso.
- `rsi` contiene el puntero de pila del otro contexto previamente guardado.

#### **3. Restaurar los registros guardados**

`popq %r15 popq %r14 popq %r13 popq %r12 popq %rbx popq %rbp`
- Restaura los valores originales de los registros que se guardaron antes del cambio de contexto.

#### **4. Retornar al código del nuevo contexto**

`retq`
- Regresa a la ejecución del nuevo contexto, ya que la dirección de retorno estaba almacenada en la pila del otro contexto.[[]]