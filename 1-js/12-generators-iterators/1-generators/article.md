
# Generators

Las funciones regulares sólo devuelven un único valor (o nada).

Los generadores pueden devolver ("yield") múltiples valores, posiblemente un número infinito de valores, uno tras otro, a petición. Trabajan muy bien con [iterables](info:iterable), permitiendo crear flujos de datos con facilidad.

## Generator functions

Para crear un generador, necesitamos una construcción sintáctica especial: `function*`, llamada "función del generador".

Se ve así:

```js
function* generateSequence() {
  yield 1;
  yield 2;
  return 3;
}
```

Cuando `generateSequence()` se llama, no ejecuta el código. En su lugar, devuelve un objeto especial, llamado "generador".

```js
// La "función generadora" crea el "objeto generador"
let generator = generateSequence();
```

El objeto `generator` puede ser percibido como una "llamada de función congelada":

![](generateSequence-1.svg)

Una vez creado, la ejecución del código se detiene al principio.

El método principal de un generador es `next()`. Cuando se le llama, se reanuda la ejecución hasta la declaración más cercana de  `yield <value>`. Entonces la ejecución se detiene, y el valor se devuelve al código exterior.

Por ejemplo, aquí creamos el generador y obtenemos su primer valor producido:

```js run
function* generateSequence() {
  yield 1;
  yield 2;
  return 3;
}

let generator = generateSequence();

*!*
let one = generator.next();
*/!*

alert(JSON.stringify(one)); // {value: 1, done: false}
```

EL resultado de `next()` siempre es un objeto:
- `value`: el valor producido.
- `done`: `false` si el código no ha terminado todavía, de lo contrario `true`.

A partir de ahora, sólo tenemos el primer valor:

![](generateSequence-2.svg)

Llamemos a `generator.next()` de nuevo. Entonces, se reanuda la ejecución y regresa la siguiente `yield`:

```js
let two = generator.next();

alert(JSON.stringify(two)); // {value: 2, done: false}
```

![](generateSequence-3.svg)

Y, si lo llamamos la tercera vez, entonces la ejecución alcanza la declaración de `return` que termina la función:
```js
let three = generator.next();

alert(JSON.stringify(three)); // {value: 3, *!*done: true*/!*}
```

![](generateSequence-4.svg)

Ahora el generador está listo. Deberíamos verlo desde `done:true` y procesa `value:3` como resultado final.

Las nuevas llamadas `generador.next()` ya no tienen sentido. Si las hacemos, devuelven el mismo objeto: `{done: true}`.

No hay forma de "hacer retroceder" un generador. Pero podemos crear otro llamando `generateSequence()`.

Hasta ahora, lo más importante que hay que entender es que las funciones del generador, a diferencia de las funciones normales, no ejecutan el código. Sirven como "fábricas de generadores". Ejecutar una `function*` devuelve un generador, y luego le pedimos valores.

```smart header="`function* f(…)` or `function *f(…)`?"
Es una cuestión secundaria, ambas sintaxis son correctas..

Pero normalmente se prefiere la primera sintaxis, ya que la estrella `*` denota que es una función generadora, describe el tipo, no el nombre, por lo que debe atenerse a la palabra clave `function`.
```

## Generators are iterable

Como ya habrán adivinado al ver el método `next()`, los generadores son [iterable](info:iterable).

Podemos obtener valores de bucle por `for..of`:

```js run
function* generateSequence() {
  yield 1;
  yield 2;
  return 3;
}

let generator = generateSequence();

for(let value of generator) {
  alert(value); // 1, then 2
}
```

Es una forma mucho más atractiva de trabajar con los generadores que llamar a `.next().value`, verdad?

...Pero tenga en cuenta: el ejemplo anterior muestra `1`, y luego `2`, y eso es todo. No se muestra `3`!

Es porque la iteración for-of ignora el último `value`, cuando `done: true`. Por lo tanto, si queremos que todos los resultados sean mostrados por `for..of`, debemos devolverlos con `yield`:

```js run
function* generateSequence() {
  yield 1;
  yield 2;
*!*
  yield 3;
*/!*
}

let generator = generateSequence();

for(let value of generator) {
  alert(value); // 1, then 2, then 3
}
```

Naturalmente, como los generadores son iterables, podemos llamar a toda la funcionalidad relacionada, por ejemplo, el operador de difusión `...`:

```js run
function* generateSequence() {
  yield 1;
  yield 2;
  yield 3;
}

let sequence = [0, ...generateSequence()];

alert(sequence); // 0, 1, 2, 3
```

In the code above, `...generateSequence()` convierte el iterable objeto generador en un conjunto de elementos (lea más sobre la sintaxis de difusión en el capítulo [](info:rest-parameters-spread-operator#spread-operator))

## Using generators instead of iterables

Hace algún tiempo, en el capítulo [](info:iterable) creamos un objeto iterable `range`  que devuelve valores `from..to`.

Aquí, recordemos el código:

```js run
let range = {
  from: 1,
  to: 5,

  // for..of llama a este método una vez desde el principio
  [Symbol.iterator]() {
    // ...devuelve el objeto iterado:
    // en adelante, for..of trabajar sólo con ese objeto, pidiéndole los siguientes valores
    return {
      current: this.from,
      last: this.to,

      // next() es llamado en cada iteración por el bucle for..of 
      next() {
        // debería devolver el valor como un objeto {done:.., value :...}
        if (this.current <= this.last) {
          return { done: false, value: this.current++ };
        } else {
          return { done: true };
        }
      }
    };
  }
};

alert([...range]); // 1,2,3,4,5
```

Usar un generador para hacer secuencias iterativas es mucho más elegante:

```js run
function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) {
    yield i;
  }
}

let sequence = [...generateSequence(1,5)];

alert(sequence); // 1, 2, 3, 4, 5
```

...¿Pero qué pasa si queremos mantener un objeto personalizado `range` ?

## Converting Symbol.iterator to generator

Podemos obtener lo mejor de ambos mundos proporcionando un generador como `Symbol.iterator`:

```js run
let range = {
  from: 1,
  to: 5,

  *[Symbol.iterator]() { // una abreviatura de [Symbol.iterator]: function*()
    for(let value = this.from; value <= this.to; value++) {
      yield value;
    }
  }
};

alert( [...range] ); // 1,2,3,4,5
```

El objecto `range` es ahora iterable.

Eso funciona bastante bien, porque cuando `range[Symbol.iterator]` es llamado:
- devuelve un objeto (ahora un generador)
- que tiene `.next()` método (yep, un generador lo tiene)
- que devuelve valores en la forma `{value: ..., done: true/false}` (comprueba, exactamente lo que hace el generador).

No es una coincidencia, por supuesto. El objetivo de los generadores es hacer más fácil las iteraciones, así que podemos ver eso.

La última variante con un generador es mucho más concisa que el código iterable original, y mantiene la misma funcionalidad.

```smart header="Los generadores pueden continuar para siempre"
En los ejemplos anteriores generamos secuencias finitas, pero también podemos hacer un generador que produzca valores para siempre. Por ejemplo, una secuencia interminable de números pseudo-aleatorios.

Eso seguramente requeriría un `break` in `for..of`, de lo contrario el bucle se repetiría para siempre y se colgaría.
```

## Generator composition

La composición de los generadores es una característica especial que les permite "incrustar" de forma transparente los generadores entre sí.

Por ejemplo, nos gustaría generar una secuencia de:
- dígitos `0..9` (códigos de caracteress 48..57),
- seguido de letras del alfabeto `a..z` (códigos de caracteres 65..90)
- y letras mayúsculas `A..Z` (códigos de caracteres 97..122)

Luego planeamos crear contraseñas seleccionando caracteres de la misma secuencia (podría añadir también caracteres de sintaxis), pero necesitamos generar primero la secuencia.

Ya tenemos `function* generateSequence(start, end)`. Reutilicémoslo para entregar 3 secuencias una tras otra, juntas son exactamente lo que necesitamos.

En una función regular, para combinar los resultados de otras múltiples funciones, las llamamos, almacenamos los resultados, y luego nos unimos al final.

Para los generadores, podemos hacerlo mejor, así:

```js run
function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

function* generatePasswordCodes() {

*!*
  // 0..9
  yield* generateSequence(48, 57);

  // A..Z
  yield* generateSequence(65, 90);

  // a..z
  yield* generateSequence(97, 122);
*/!*

}

let str = '';

for(let code of generatePasswordCodes()) {
  str += String.fromCharCode(code);
}

alert(str); // 0..9A..Za..z
```

La directiva especial de `yield*` en el ejemplo es responsable de la composición. Delega la ejecución a otro generador. O, para decirlo de forma sencilla, hace funcionar los generadores y transmite de forma transparente sus rendimientos al exterior, como si fueran hechos por el propio generador de llamada.

El resultado es el mismo que si alineamos el código de los generadores anidados:

```js run
function* generateSequence(start, end) {
  for (let i = start; i <= end; i++) yield i;
}

function* generateAlphaNum() {

*!*
  // yield* generateSequence(48, 57);
  for (let i = 48; i <= 57; i++) yield i;

  // yield* generateSequence(65, 90);
  for (let i = 65; i <= 90; i++) yield i;

  // yield* generateSequence(97, 122);
  for (let i = 97; i <= 122; i++) yield i;
*/!*

}

let str = '';

for(let code of generateAlphaNum()) {
  str += String.fromCharCode(code);
}

alert(str); // 0..9A..Za..z
```

La composición de un generador es una forma natural de insertar un flujo de un generador en otro.

Funciona incluso si el flujo de valores del generador anidado es infinito. Es simple y no utiliza memoria extra para almacenar resultados intermedios.

## "yield" is a two-way road

Hasta este momento, los generadores eran como "iteradores repotenciados". Y así es como se usan a menudo.

Pero de hecho son mucho más poderosos y flexibles.

Eso se debe a que el `yield` es un camino de doble sentido: no sólo devuelve el resultado en el exterior, sino que también puede pasar el valor en el interior del generador.

Para ello, deberíamos llamar a `generator.next(arg)`, con un argumento. Ese argumento se convierte en el resultado de `yield`.

Veamos un ejemplo:

```js run
function* gen() {
*!*
  // Pasar una pregunta al código exterior y esperar una respuesta
  let result = yield "2 + 2?"; // (*)
*/!*

  alert(result);
}

let generator = gen();

let question = generator.next().value; // <-- yield devuelve el valor

generator.next(4); // --> pasar el resultado al generador  
```

![](genYield2.svg)

1. La primera llamada `generador.next()` siempre es sin argumento. Inicia la ejecución y devuelve el resultado del primer `yield` ("2+2"). En este punto el generador detiene la ejecución (todavía en esa línea).
2. Entonces, como se muestra en la imagen de arriba, el resultado de `yield` entra en la variable `question` del código de llamada.
3. En `generator.next(4)`, el generador se reanuda, y `4` obtiene como resultado: `let result = 4`.

Tenga en cuenta que el código externo no tiene que llamar inmediatamente a `next(4)`. Puede llevar tiempo calcular el valor. Este también es un código válido:

```js
// reanudar el generador después de algún tiempo
setTimeout(() => generator.next(4), 1000);
```

La sintaxis puede parecer un poco extraña. Es muy poco común que una función y el código de llamada se pasen valores entre sí. Pero eso es exactamente lo que está pasando.

Para hacer las cosas más obvias, aquí hay otro ejemplo, con más llamadas:

```js run
function* gen() {
  let ask1 = yield "2 + 2?";

  alert(ask1); // 4

  let ask2 = yield "3 * 3?"

  alert(ask2); // 9
}

let generator = gen();

alert( generator.next().value ); // "2 + 2?"

alert( generator.next(4).value ); // "3 * 3?"

alert( generator.next(9).done ); // true
```

La imagen de la ejecución:

![](genYield2-2.svg)

1. La primera `.next()` comienza la ejecución... Llega a la primera `yield`.
2. El resultado se devuelve al código exterior.
3. El segundo `.next(4)` pasa `4` al generador como resultado de la primera `yield`, y reanuda la ejecución.
4. ...Llega al segundo `yield`, que se convierte en el resultado de la llamada del generador.
5. La tercera `next(9)` pasas `9` al generador como resultado de la segunda `yield` y reanuda la ejecución que llega al final de la función, así que `done: true`.

Es como un juego de "ping-pong". Cada `next(value)` (excluyendo el primero) pasa un valor al generador, que se convierte en el resultado del actual `yield`, y luego obtiene de vuelta el resultado del siguiente `yield`.

## generator.throw

Como observamos en los ejemplos anteriores, el código externo puede pasar un valor al generador, como resultado de `yield`.

...Pero también puede iniciar (arrojar) un error allí. Eso es natural, ya que un error es una especie de resultado.

Para pasar un error a un `yield`, deberíamos llamar a `generator.throw(err)`. En ese caso, el `err` es arrojado en la línea con `yield`.

Por ejemplo, aquí el resultado de `"2 + 2?"` conduce a un error:

```js run
function* gen() {
  try {
    let result = yield "2 + 2?"; // (1)

    alert("La ejecución no llega hasta aquí, porque la excepción es arrojada por encima");
  } catch(e) {
    alert(e); // muestra el error
  }
}

let generator = gen();

let question = generator.next().value;

*!*
generator.throw(new Error("La respuesta no se encuentra en mi base de datos")); // (2)
*/!*
```

El error, arrojado en el generador en la línea `(2)` lleva a una excepción en la línea `(1)` with `yield`. En el ejemplo anterior, `try..catch` lo capta y muestra.

Si no lo encontramos, entonces, como cualquier excepción, se "cae" el generador en el código de llamada..

La línea actual del código de llamada es la línea con `generator.throw`, etiquetada como `(2)`. Así que podemos capturarla aquí, así:

```js run
function* generate() {
  let result = yield "2 + 2?"; // El error está aquí
}

let generator = generate();

let question = generator.next().value;

*!*
try {
  generator.throw(new Error("La respuesta no se encuentra en mi base de datos"));
} catch(e) {
  alert(e); // muestra el error
}
*/!*
```

Si no encontramos el error allí, entonces, como de costumbre, cae a través del código de llamada externo (si existe) y, si no lo encontramos, mata el script.

## Summary

- Los generadores son creados por las funciones de los generadores `function*(…) {…}`.
- Dentro de los generadores (solamente) existe un operador `yield`.
- El código externo y el generador pueden intercambiar resultados a través de llamadas `next/yield`.

En el Javascript moderno, los generadores se usan muy poco. Pero a veces son útiles, porque la capacidad de una función de intercambiar datos con el código de llamada durante la ejecución es única.

Además, en el próximo capítulo aprenderemos los generadores de asíncronos, que se utilizan para leer flujos de datos generados asincrónicamente en un bucle `for`.

En la programación de la web a menudo trabajamos con datos en flujo, por ejemplo la necesidad de obtener resultados paginados, por lo que es un caso de uso muy importante.
