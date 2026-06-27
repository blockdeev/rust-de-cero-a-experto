# 🦀 Move Semantics & Lifetimes — Ejercicios Rustlings

> **Ejercicios oficiales de Rustlings resueltos y explicados** — sección `06_move_semantics`
> (5 ejercicios) y `16_lifetimes` (3 ejercicios), con las ideas clave que surgen de cada uno.

---

## 📑 Tabla de contenidos

1. [Conceptos clave antes de arrancar](#-conceptos-clave-antes-de-arrancar)
2. [Copy vs Move — la tabla definitiva](#-copy-vs-move--la-tabla-definitiva)
3. [Move Semantics — ejercicios](#-move-semantics--ejercicios)
4. [Lifetimes — ejercicios](#-lifetimes--ejercicios)
5. [Resumen mental](#-resumen-mental)

---

## 🧠 Conceptos clave antes de arrancar

### El sistema de módulos de tests

Todos los ejercicios de Rustlings incluyen un bloque de tests al final:

```rust
#[cfg(test)]        // solo compila cuando corrés tests, no en producción
mod tests {
    use super::*;   // importa todo lo que está definido arriba

    #[test]         // marca esta función como un test
    fn nombre_del_test() {
        assert_eq!(resultado, valor_esperado); // falla si no son iguales
    }
}
```

`assert_eq!` es el corazón — compara dos valores y si no coinciden, el test falla.
El ejercicio se considera resuelto cuando **todos los `assert_eq!` pasan**.

### Cómo marcar un ejercicio como completado

Rustlings usa `// I AM NOT DONE` como señal de que el ejercicio está pendiente.
Una vez que resolvés el ejercicio, **borrás esa línea** y Rustlings avanza al siguiente automáticamente.

> 💡 Los comentarios del enunciado (`// TODO`, `// Fix the...`) se dejan — son parte del ejercicio.
> Solo se borra el `// I AM NOT DONE`.

---

## 📊 Copy vs Move — la tabla definitiva

Una de las ideas más importantes que aparecen en estos ejercicios:

| Tipo | Dónde vive | Comportamiento | Ejemplos |
|---|---|---|---|
| **Tipos simples** | Stack | `Copy` — se duplican, el original sigue válido | `i32`, `f64`, `bool`, `char`, `usize` |
| **Tipos compuestos** | Heap | `Move` — se transfieren, el original queda inválido | `String`, `Vec<T>`, `Box<T>` |
| **Tuplas/arrays de Copy** | Stack | `Copy` | `(i32, bool)`, `[i32; 3]` |

```rust
// i32 implementa Copy → se copia, x sigue válido
let x: i32 = 5;
let y = x;
println!("{}", x); // ✅

// String NO implementa Copy → se mueve, s1 queda inválido
let s1 = String::from("hola");
let s2 = s1;
// println!("{}", s1); ❌ error: value borrowed after move
```

> 🔑 **La regla:** si un tipo maneja memoria en el heap, se mueve. Si vive entero en el stack
> con tamaño fijo conocido en compilación, se copia. `Vec` no implementa `Copy` porque
> copiar toda su memoria del heap sería costoso — Rust nunca lo hace silenciosamente.

---

## 📦 Move Semantics — ejercicios

### move_semantics1 — Mutabilidad básica

**Código original de Rustlings:**

```rust
// TODO: Fix the compiler error in this function.

fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    let vec = vec;
    vec.push(88);
    vec
}

fn main() {
    // You can optionally experiment here.
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn move_semantics1() {
        let vec0 = vec![22, 44, 66];
        let vec1 = fill_vec(vec0);
        assert_eq!(vec1, vec![22, 44, 66, 88]);
    }
}
```

**Error del compilador:**

```
error[E0596]: cannot borrow `vec` as mutable, as it is not declared as mutable
 --> src/main.rs:4:5
  |
4 |     vec.push(88);
  |     ^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
3 |     let mut vec = vec;
  |         +++
```

**Solución:**

```rust
fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    let mut vec = vec; // ← agregar mut
    vec.push(88);
    vec
}
```

**¿Qué se aprende?**

La mutabilidad vive donde se modifica el valor. La función recibe `vec` por move — la
propiedad pasa a la función — y para modificarlo adentro necesita ser `mut`.

---

### move_semantics2 — Dos vectores independientes

**Código original de Rustlings:**

```rust
// TODO: Fix the compiler error by finding a way to make this work
// without changing the `vec` parameter type in `fill_vec`.

fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    let mut vec = vec;
    vec.push(88);
    vec
}

fn main() {
    // You can optionally experiment here.
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn move_semantics2() {
        let vec0 = vec![22, 44, 66];
        let vec1 = fill_vec(vec0);
        // vec0 can't be accessed anymore because it is moved to fill_vec.
        assert_eq!(vec1, vec![22, 44, 66, 88]);
    }
}
```

**Error del compilador:**

```
error[E0382]: borrow of moved value: `vec0`
  |
  | let vec0 = vec![22, 44, 66];
  |     ---- move occurs because `vec0` has type `Vec<i32>`,
  |          which does not implement the `Copy` trait
  | let vec1 = fill_vec(vec0);
  |                      ---- value moved here
  | assert_eq!(vec0, [22, 44, 66]);
  |            ^^^^ value borrowed here after move
```

**Solución:**

```rust
let vec1 = fill_vec(vec0.clone()); // ← clonar vec0 antes de moverlo
```

**¿Por qué `.clone()` y no una referencia?**

El test quiere que `vec0` siga existiendo con `[22, 44, 66]` y `vec1` sea `[22, 44, 66, 88]`
— dos vectores independientes. Con una referencia la función no podría devolver un `Vec<i32>`
nuevo. Con `.clone()` se crea una copia completa en el heap: `vec0` queda intacto y la
función trabaja sobre la copia.

> ⚠️ `.clone()` es explícito a propósito — Rust nunca copia memoria del heap silenciosamente.
> Usalo cuando realmente necesitás dos copias independientes, no como solución por defecto.

---

### move_semantics3 — Mutabilidad en la firma

**Código original de Rustlings:**

```rust
// TODO: Fix the compiler error by only changing the `fill_vec` function
// signature (the function's input and output types and lifetimes).

fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    vec.push(88);
    vec
}

fn main() {
    // You can optionally experiment here.
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn move_semantics3() {
        let vec0 = vec![22, 44, 66];
        let vec1 = fill_vec(vec0);
        assert_eq!(vec1, vec![22, 44, 66, 88]);
    }
}
```

**Error del compilador:**

```
error[E0596]: cannot borrow `vec` as mutable, as it is not declared as mutable
  |
  | fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
  |             --- help: consider changing this to be mutable: `mut vec`
  |     vec.push(88);
  |     ^^^ cannot borrow as mutable
```

**Solución:**

```rust
fn fill_vec(mut vec: Vec<i32>) -> Vec<i32> { // ← mut directamente en la firma
    vec.push(88);
    vec
}
```

**Diferencia con move_semantics1:**

En el ejercicio 1 se hacía `let mut vec = vec` adentro del cuerpo. Acá el `mut` va
directamente en el parámetro de la firma — es equivalente pero más conciso. El enunciado
especifica que solo se puede cambiar la firma, por eso la solución es diferente.

**¿Y `vec0` en el test?**

No necesita `mut` porque el test no lo modifica directamente. `vec0` se mueve a la función
y deja de existir — la memoria se libera cuando `fill_vec` termina, antes del final de `main`.

---

### move_semantics4 — Non-Lexical Lifetimes (NLL)

**Código original de Rustlings:**

```rust
// TODO: Fix the compiler errors in the function without adding any new line.

fn move_semantics4() {
    let mut x = Vec::new();
    let y = &mut x;
    y.push(42);
    let z = &mut x;
    z.push(13);
    assert_eq!(x, [42, 13]);
}

fn main() {
    // You can optionally experiment here.
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn move_semantics4() {
        move_semantics4();
    }
}
```

**¿Por qué compila sin cambios?**

En Rust moderno (NLL — Non-Lexical Lifetimes), una referencia **muere en su último punto
de uso**, no al cerrar el scope. Por eso esto es válido:

```rust
let y = &mut x;
y.push(42);         // último uso de y → y muere acá

let z = &mut x;     // ✅ y ya no existe, podemos crear otra &mut
z.push(13);
```

Sin NLL (Rust antiguo), `y` hubiera vivido hasta el `}` del bloque y hubiera generado un
conflicto con `z`. NLL hace al compilador más inteligente — razona sobre flujo real de uso,
no sobre límites de scope léxico.

> 💡 Este ejercicio no requiere cambios — el objetivo es entender **por qué** compila.

---

### move_semantics5 — Cuándo tomar ownership y cuándo no

**Código original de Rustlings:**

```rust
#![allow(clippy::ptr_arg)]

// TODO: Fix the compiler errors without changing anything except adding or
// removing references (the character `&`).

// Shouldn't take ownership
fn get_char(data: String) -> char {
    data.chars().last().unwrap()
}

// Should take ownership
fn string_uppercase(mut data: &String) {
    data = data.to_uppercase();
    println!("{data}");
}

fn main() {
    let data = "Rust is great!".to_string();

    get_char(data);

    string_uppercase(&data);
}
```

**Error del compilador:**

```
error[E0308]: mismatched types
   |
   | fn string_uppercase(mut data: &String) {
   |                               ------- expected due to this parameter type
   |     data = data.to_uppercase();
   |            ^^^^^^^^^^^^^^^^^^^ expected `&String`, found `String`

error[E0382]: borrow of moved value: `data`
   |
   | let data = "Rust is great!".to_string();
   |     ---- move occurs because `data` has type `String`
   | get_char(data);
   |           ---- value moved here
   | string_uppercase(&data);
   |                  ^^^^^ value borrowed here after move
```

**Solución:**

```rust
// Shouldn't take ownership → recibe referencia
fn get_char(data: &String) -> char {
    data.chars().last().unwrap()
}

// Should take ownership → recibe el valor
fn string_uppercase(mut data: String) {
    data = data.to_uppercase();
    println!("{data}");
}

fn main() {
    let data = "Rust is great!".to_string();
    get_char(&data);         // ✅ solo lee, data sigue viva
    string_uppercase(data);  // ✅ toma ownership, lo modifica y lo destruye
}
```

**¿Qué hace `get_char` internamente?**

```rust
data.chars()        // convierte el String en un iterador de caracteres
    .last()         // agarra el último → devuelve Option<char>
    .unwrap()       // extrae el char del Option (explota si está vacío)
```

Para calcular el último carácter no necesita poseer el `String` — solo leerlo. Por eso
recibe `&String`. `string_uppercase` en cambio necesita modificar el valor, así que toma
ownership.

> 💡 **Regla práctica:** si una función solo necesita **leer**, usá `&T`. Si necesita
> **modificar sin devolver**, usá `&mut T`. Si necesita **consumir o transformar**, tomá ownership.

---

## ⏳ Lifetimes — ejercicios

### lifetimes1 — Función que devuelve una de dos referencias

**Código original de Rustlings:**

```rust
// The Rust compiler needs to know how to check whether supplied references are
// valid, so that it can let the programmer know if a reference is at risk of
// going out of scope before it is used. Remember, references are borrows and do
// not own their own data. What if their owner goes out of scope?

// TODO: Fix the compiler error by updating the function signature.

fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    // You can optionally experiment here.
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_longest() {
        assert_eq!(longest("abcd", "123"), "abcd");
        assert_eq!(longest("abc", "1234"), "1234");
    }
}
```

**Error del compilador:**

```
error[E0106]: missing lifetime specifier
  |
  | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value,
    but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
  | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
```

**Solución:**

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**¿Por qué es necesario `'a` acá?**

El compilador no puede saber en tiempo de compilación si va a devolver `x` o `y` — eso
depende del runtime. Sin `'a` no sabe cuánto tiempo es válido el resultado.

Con `'a` le firmás un contrato: **"el resultado vive exactamente lo que viva la más corta
de las dos referencias que recibo"**. Ahora puede rastrear el resultado y verificar que no
lo uses después de que alguna de las dos muera.

Los `<>` en `longest<'a>` son donde se **declaran** los parámetros genéricos — igual que
`<T>` para tipos, `<'a>` para lifetimes. Primero declarás, después usás.

---

### lifetimes2 — El problema del scope

**Código original de Rustlings:**

```rust
// TODO: Fix the compiler error by moving one line.

fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
    }
    println!("The longest string is '{result}'");
}
```

**Error del compilador:**

```
error[E0597]: `string2` does not live long enough
   |
   |         let string2 = String::from("xyz");
   |             ------- binding `string2` declared here
   |         result = longest(string1.as_str(), string2.as_str());
   |                                            ^^^^^^^^^^^^^^^^^ borrowed value
   |                                                              does not live long enough
   |     }
   |     - `string2` dropped here while still borrowed
   |     println!("The longest string is '{result}'");
   |              ----------------------------------- borrow later used here
```

**Solución — mover el `println!` adentro del scope:**

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(string1.as_str(), string2.as_str());
        println!("The longest string is '{result}'"); // ✅ antes de que string2 muera
    }
}
```

**¿Por qué falla el original?**

`longest` declara con `'a` que el resultado vive lo que viva la **más corta** de las dos
referencias. La más corta es `string2`, que muere al cerrar `}`. Entonces `result` solo es
válido dentro de ese scope — usarlo afuera es usar una referencia a memoria ya liberada.

---

### lifetimes3 — Lifetimes en structs

**Código original de Rustlings:**

```rust
// Lifetimes are also needed when structs hold references.
// TODO: Fix the compiler errors about the struct.

struct Book {
    author: &str,
    title: &str,
}

fn main() {
    let book = Book {
        author: "George Orwell",
        title: "1984",
    };
    println!("{} by {}", book.title, book.author);
}
```

**Error del compilador:**

```
error[E0106]: missing lifetime specifier
  |
  | struct Book {
  |     author: &str,
  |             ^ expected named lifetime parameter
  |     title: &str,
  |            ^ expected named lifetime parameter
```

**Solución:**

```rust
struct Book<'a> {
    author: &'a str, // ✅
    title: &'a str,
}

fn main() {
    let book = Book {
        author: "George Orwell",
        title: "1984",
    };
    println!("{} by {}", book.title, book.author);
}
```

**¿Qué significa `<'a>` en el struct?**

Cuando un struct guarda referencias adentro, **siempre necesita lifetime annotations**.
El compilador necesita saber cuánto tiempo van a vivir `author` y `title` para garantizar
que el struct no sobreviva a los datos que referencia.

Con `'a` le decís: **"este struct vive lo que vivan las referencias que contiene"**.

---

## 🧠 Resumen mental

```
¿La función solo lee el valor?          → &T  (referencia inmutable)
¿La función necesita modificarlo?       → &mut T  (referencia mutable)
¿La función consume o transforma?       → T  (toma ownership)
¿Necesitás dos copias independientes?  → .clone()  (costoso, usalo con criterio)

¿Una función devuelve una referencia
 de entre varias que recibe?            → anotar 'a explícitamente
¿Un struct guarda referencias?          → siempre necesita 'a

¿Una referencia muere cuándo?           → en su último punto de uso (NLL),
                                          no al cerrar el scope
```

### Copy vs Move — regla rápida

```
Tipo en el STACK con tamaño fijo  →  Copy  (i32, bool, char, f64…)
Tipo en el HEAP                   →  Move  (String, Vec, Box…)
```

---

> **Nota:** estos son los ejercicios oficiales de Rustlings resueltos con explicaciones propias.
> El material de estudio teórico complementario está en
> [`ownership_borrowing_lifetimes.md`](./ownership_borrowing_lifetimes.md).

---

*Notas de estudio — Rust desde cero · Serie: El sistema de tipos*
