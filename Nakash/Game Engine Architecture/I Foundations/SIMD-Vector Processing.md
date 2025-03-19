>**Single instruction multple data**. Refiere a la habilidad de la mayoría de los microprocesadores podernos en poder llevar a cabo operaciones matematicas en multples data items en paralelo, utilizando una *single machine instruction*. 
 Como veremos más adelante SIMD puede ser combinado hacia una forma de paralelismo conocida como **single instruction multiple thread (SIMT)**.

### Una breve introducción (o eso espero)
Intel introdujo un sen te instrucciones que permitían calculos SIMD ser hechos en 8 integers de 8-bit, en 4 de 16-bit, en 2 de 32-bit enpaquetados en un registro especial de 64-bits MMX ("multimedia extensios" o "matrix math extensions"). Seguido a esto llegó un set de instrucciones llamado *Streaming SIMD extensions*, que utiliza registros de 128-bit que pueden contener ==integer  o IEE floating-point data==. El más usado comunmente en el engine es el "**Packed 32-bit floating-point mode**". en este modo, cuatro valores de tipo float de 32-bit son empaquetados en un solo registro de 128.bit.
Después de esto llega el *advance vector extensions (avx)*. Los registro AVX son 256 bits de ancho, permitiendo a una sola instruccion operarse en pares de ==hasta ocho floating-point de 32-bit operandos en paralelo==. 

### Instrucciones SEE: Set y sus registros 
Vamos a utilizar un subset de este tipos de instruccipnes que se usa con los *Packed 32-bit floating-point mode*. Estás instrucciones están denotadas por un sufijo _PS_, lo cuál significa que estamos trabajando con datos empaquetados y que cada elemento es un _Single precision floating-point_.
Cada registro se denotan como XMMi (donde cada i es un número de 0 a 15).

#### Algunas funciones útiles de SEE: 
Primero que nada hay que incluir *<x86intrin.h>* 
- __m128 _mm_set_ps(float w, float z, float y, float x);
This intrinsic initializes an __m128 variable with the four floating-point
values provided.
- __m128 _mm_load_ps(const float* pData);
This intrinsic loads four floats from a C-style array into an __m128
variable. The input array must be 16-byte aligned.
 -  **void _mm_store_ps(float* pData, __m128 v);
This intrinsic stores the contents of an __m128 variable into a C-style
array of four floats, which must be 16-byte aligned.
 -  __m128 _mm_add_ps(__m128 a, __m128 b);
This intrinsic adds the four pairs of floats contained in the variables
a and b in parallel and returns the result.
- __m128 _mm_mul_ps(__m128 a, __m128 b);
This intrinsic multiplies the four pairs of floats

Utilizar estos vectores termina siendo más eficiente ya que permite procesar todos los elementos dentro de estos vectores de forma paralela. Un ejemplo aplicando una suma: 

```cpp 
void sumVect (int cont, float * result, const float * A, const float * B) {
	assert (cont % 4 == 0); 
	for (int i = 0; i < cont ; i+=4) {
		__m128 a = _mm_load_ps(&A[0]); 
		__m128 b = _mm_load_ps(&B[0]);
		__m128 r = _mm_add_ps (a, b); 
		_mm_store_ps(result, r); 
	}
}
```
Al procesarse los 4 elementos de los vectores XMM en paralelo entoncces esto va a ser casi 4 veces más rápido que hacerlo linealmente. El problema es que tiene que estar cargando valores y después sumarlos, por eso restringe la capacidad que tiene.  


```cpp 
void vect_sqrt (int count, float * __intrinsict__ r, float * __intrinsic__ a) {
	assert (count % 4 == 0); 
	__m128 vz = _mm_set1_ps (0.0f); 
	for (int i = 0; i < count; i+= 4) { 
		__m128 va = _m_load_ps (a + i); //Aritmetica de punteros
		__m128 r; 
		if (_m_cmpge_ps (va, vz)) {
			vr = _m_sqrt_ps (va);
		} else {
			vr = vz; 
		}
		_m_store_ps (r, vr); 
	}
}

```

