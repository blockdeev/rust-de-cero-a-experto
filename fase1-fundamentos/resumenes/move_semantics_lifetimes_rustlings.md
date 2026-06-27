# 🦀 Move Semantics & Lifetimes — Ejercicios Rustlings

> **Ejercicios oficiales de Rustlings resueltos y explicados** — sección `06_move_semantics`
> (6 ejercicios) y `16_lifetimes` (3 ejercicios), con las ideas clave que surgen de cada uno.

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

**El error:**

```
cannot borrow `vec` as mutable, as it is not declared as mutable
help: consider changing this to be mutable
  |
  |     let mut vec = vec;
```

**Código original:**

```rust
fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    vec.push(88); // ❌ vec no es mutable
    vec
}
```

**Solución:**

```rust
fn fill_vec(vec: Vec<i32>) -> Vec<i32> {
    let mut vec = vec; // declaramos vec como mutable dentro de la función
    vec.push(88);      // ✅ ahora sí podemos modificarlo
    vec
}
```

**¿Qué se aprende?**

La mutabilidad vive donde se modifica el valor. La función recibe `vec` por move — la
propiedad pasa a la función — y para modificarlo adentro necesita ser `mut`.

---

### move_semantics2 — Dos vectores independientes

**El error:**

```
error[E0382]: borrow of moved value: `vec0`
  --> ...
23 |         let vec1 = fill_vec(vec0);
   |                             ---- value moved here
25 |         assert_eq!(vec0, [22, 44, 66]); ← value borrowed here after move
```

**El test quiere:**

```rust
assert_eq!(vec0, [22, 44, 66]);      // vec0 sin modificar
assert_eq!(vec1, [22, 44, 66, 88]); // vec1 con el 88 agregado
```

**Solución:**

```rust
let vec1 = fill_vec(vec0.clone()); // clonamos vec0 → dos vectores independientes
```

**¿Por qué `.clone()` y no una referencia?**

Con una referencia, `fill_vec` no podría devolver un `Vec<i32>` nuevo — estaría modificando
el mismo vector. Terminarías con un solo vector modificado, no dos independientes con
contenidos distintos. `.clone()` crea una copia completa en el heap — `vec0` queda intacto
con `[22, 44, 66]` y `vec1` recibe `[22, 44, 66, 88]`.

> ⚠️ `.clone()` es explícito a propósito — Rust nunca copia memoria del heap silenciosamente.
> Usalo cuando realmente necesitás dos copias independientes, no como solución por defecto.

---

### move_semantics3 — Mutabilidad en la firma

**El error:**

```
cannot borrow `vec` as mutable, as it is not declared as mutable
help: consider changing this to be mutable
  |
  | fn fill_vec(mut vec: Vec<i32>) -> Vec<i32>
```

**Solución:**

```rust
fn fill_vec(mut vec: Vec<i32>) -> Vec<i32> { // mut directamente en la firma
    vec.push(88);
    vec
}
```

**Diferencia con move_semantics1:**

En el ejercicio 1 se hacía `let mut vec = vec` adentro del cuerpo. Acá el `mut` va
directamente en el parámetro de la firma — es equivalente pero más conciso.

**¿Y vec0 en el test?**

No necesita `mut` porque el test no lo modifica directamente. `vec0` se mueve a la función
y deja de existir — la memoria se libera cuando `fill_vec` termina, antes del final de `main`.

```rust
fn main() {
    let vec0 = vec![22, 44, 66];
    let vec1 = fill_vec(vec0);  // vec0 se mueve acá
                                // fill_vec termina → vec0 muere
    assert_eq!(vec1, [22, 44, 66, 88]); // solo verificamos vec1
}
```

---

### move_semantics4 — Non-Lexical Lifetimes (NLL)

**El ejercicio:**

```rust
fn move_semantics4() {
    let mut x = Vec::new();
    let y = &mut x;
    y.push(42);
    let z = &mut x;
    z.push(13);
    assert_eq!(x, [42, 13]);
}
```

**¿Por qué compila?**

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

---

### move_semantics5 — Cuándo tomar ownership y cuándo no

**El error:**

```
error[E0308]: mismatched types
  | data = data.to_uppercase();
  |        ^^^^^^^^^^^^^^^^^^^ expected &String, found String

error[E0382]: borrow of moved value: `data`
  | get_char(data);       ← value moved here
  | string_uppercase(&data); ← value borrowed here after move
```

**Código original (con los comentarios del ejercicio):**

```rust
// Shouldn't take ownership
fn get_char(data: String) -> char {   // ❌ toma ownership — MAL
    data.chars().last().unwrap()
}

// Should take ownership
fn string_uppercase(mut data: &String) { // ❌ recibe referencia — MAL
    data = data.to_uppercase();
    println!("{data}");
}

fn main() {
    let data = "Rust is great!".to_string();
    get_char(data);          // data se mueve acá
    string_uppercase(&data); // ❌ data ya no existe
}
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

### move_semantics6 — Structs que guardan referencias

**El error:**

```
error[E0106]: missing lifetime specifier
  | author: &str,   ← expected named lifetime parameter
  | title: &str,    ← expected named lifetime parameter
```

**Código original:**

```rust
struct Book {
    author: &str, // ❌ el compilador no sabe cuánto vive esta referencia
    title: &str,
}
```

**Solución:**

```rust
struct Book<'a> {
    author: &'a str, // ✅ declaramos que Book no puede vivir más que sus referencias
    title: &'a str,
}
```

**¿Qué significa `<'a>` en el struct?**

Los `<>` son para declarar parámetros genéricos. Igual que podés declarar un tipo genérico
`<T>`, acá declarás un lifetime genérico `<'a>` — primero lo declarás, después lo usás.

El compilador necesita saber cuánto tiempo van a vivir `author` y `title`. Sin `'a` no tiene
forma de verificar que `Book` no sobreviva a los `String` a los que apunta. Con `'a` le decís:
**"este struct vive lo que vivan las referencias que contiene"** — igual que la referencia más corta.

---

## ⏳ Lifetimes — ejercicios

### lifetimes1 — Función que devuelve una de dos referencias

**El error:**

```
error[E0106]: missing lifetime specifier
  | fn longest(x: &str, y: &str) -> &str
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value,
    but the signature does not say whether it is borrowed from x or y
```

**Código original:**

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() { x } else { y }
}
```

**Solución:**

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**¿Por qué es necesario `'a` acá?**

El compilador no puede saber en tiempo de compilación si va a devolver `x` o `y` — eso
depende del runtime. Y `x` e `y` pueden venir de variables con vidas completamente distintas.
Sin `'a` el compilador está ciego — no sabe cuánto tiempo es válido el resultado.

Con `'a` le firmás un contrato: **"el resultado vive exactamente lo que viva la más corta
de las dos referencias que recibo"**. Ahora puede rastrear el resultado y verificar que no
lo uses después de que alguna de las dos muera.

---

### lifetimes2 — El problema del scope

**Código original (no compila):**

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(&string1, &string2);
    }                           // string2 muere acá
    println!("'{result}'");     // ❌ result podría apuntar a string2 muerta
}
```

**Solución — mover el `println!` adentro del scope:**

```rust
fn main() {
    let string1 = String::from("long string is long");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(&string1, &string2);
        println!("'{result}'"); // ✅ result se usa antes de que string2 muera
    }
}
```

**¿Por qué falla el original?**

`longest` declara con `'a` que el resultado vive lo que viva la **más corta** de las dos
referencias. La más corta es `string2`, que muere al cerrar `}`. Entonces `result` solo es
válido dentro de ese scope — usarlo afuera es usar una referencia a memoria ya liberada.

---

### lifetimes3 — Lifetimes en structs

**El error:**

```
error[E0106]: missing lifetime specifier
  | author: &str,
  | title: &str,
```

**Código original:**

```rust
struct Book {
    author: &str, // ❌
    title: &str,
}
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

**Regla general:**

Cuando un struct guarda referencias adentro, **siempre necesita lifetime annotations**.
El compilador necesita garantizar que el struct no sobreviva a los datos que referencia.

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
