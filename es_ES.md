# Simetrías Rústicos

[English version](README.md)

En honor del [segundo cumpleaños de Rust estable][birthday], aquí está un
pequeño contribución a la documentación de Rust.

Muchos conceptos en Rust vienen en sistemas que emparejan o tienen una simetría
agradable. Este es un resumen o "cheat sheet" para algunos de estos.

Estas tablas pueden parecer intimidantes, pero reflejan la realidad de
programación de sistemas, con control de grano fino sobre la memoria. Si solo
está empezando con Rust, *no se preocupe*, y poner el documento a un lado.  Es
una referencia y una guía sobre cómo organizar estas cosas, si sabes vagamente
lo que son.

¡Agradezco las contribuciones! Sólo tienes que abrir una [pull request].

[birthday]: https://blog.rust-lang.org/2017/05/15/rust-at-two-years.html
[pull request]: https://github.com/kmcallister/rustic-symmetries/pulls


## Referencias

| | ¿Puedes [`Copy`][copy]? | ¿Puedes cambiar a través? |
| --- | --- | --- |
| [`&T`][ref] | sí | no |
| [`&mut T`][mut-ref] | no | sí |

Esto demuestra que ni `&T` ni `&mut T` es un subtipo del otro, en el sentido
[Liskov].


## Dueño y mutabilidad

El dueño controla cuando se destruye un valor. Un valor puede tener único
dueño, o una serie de referencias que comparten el posesión. En el segundo caso
utilizamos recuento de referencias.

La mutabilidad del interior se refiere a cualquier tipo de envoltura `Wrapper`
que podamos ir desde `& Wrapper<T>` a `&mut T`, o al menos tener algunas de
las capacidades de `& mut T`.

Los encabezamientos de las columnas se refieren (más o menos) al momento en que
los invariantes de seguridad de Rust son chequeados. Tenga en cuenta que no
puede ocurrir inseguridad de compartiendo entre hilos las estructuras inseguro
para hilos. El compilador simplemente rechazar su código, a través de la magia
del trait [`Sync`][sync].

| | Estático | Dinámica | Dinámica,<br>seguro para hilos |
| --- | --- | --- | --- |
| **Dueño directa** | `T` | | |
| **Dueño a través montículo** | [`Box<T>`][box] | | |
| **Dueño compartida** | | [`Rc<T>`][rc] | [`Arc<T>`][arc] |
| **Obtener, poner, comparar<br>y intercambiar, etc.** | [`&mut T`][mut-ref] | [`Cell<T>`][cell] | [`AtomicFoo`][atomic] |
| **Préstamo sin mutabilidad** | [`&T`][ref] | | |
| **Préstamo con mutabilidad,<br>o solo lector** | | | [`Mutex<T>`][mutex] |
| **Préstamo con mutabilidad,<br>o lectores múltiples** | [`&mut T`][mut-ref] | [`RefCell<T>`][refcell] | [`RwLock<T>`][rwlock] |
| **Préstamo con mutabilidad,<br>inseguro** | [`static mut`][static mut] | [`UnsafeCell<T>`][unsafecell] | [`UnsafeCell<T>`][unsafecell] |


## Tipos numéricos

| | Entero sin signo | Entero con signo | Punto flotante |
| --- | --- | --- | --- |
| **8 bits** | [`u8`][u8] | [`i8`][i8] | |
| **16 bits** | [`u16`][u16] | [`i16`][i16] | |
| **32 bits** | [`u32`][u32] | [`i32`][i32] | [`f32`][f32] |
| **64 bits** | [`u64`][u64] | [`i64`][i64] | [`f64`][f64] |
| **128 bits** | [`u128`][u128]<sup>α</sup> | [`i128`][i128]<sup>α</sup> | |
| **Tamaño de apuntadores** | [`usize`][usize] | [`isize`][isize] | |

<sup>α</sup> Sólo versión nocturna, en Rust 1.18.


## Cadenas

| Formato | Prestar<sup>β</sup> | ¿Prestar subcadena? | Cambiar | Copiar cuando cambiado | Dueño, en montículo |
| --- | --- | --- | --- | --- | --- |
| Cualquier bytes | [`&[u8]`][slice] | sí | [`&mut`&nbsp;`[u8]`][slice] | [`Cow<[u8]>`][cow]<br> | [`Vec<u8>`][vec] |
| [UTF-8] | [`&str`][str] | sí | [`&mut`&nbsp;`str`][str]<sup>α</sup> | [`Cow<str>`][cow]<br> | [`String`][string] |
| Dependiente de<br>la plataforma | [`&OsStr`][osstr] | no | | [`Cow<OsStr>`][cow] | [`OsString`][osstring] |
| Ruta de sistema de archivos | [`&Path`][path] | no | | [`Cow<Path>`][cow] | [`PathBuf`][pathbuf] |
| `NUL`-terminado, seguro | [`&CStr`][cstr] | no | | [`Cow<CStr>`][cow] | [`CString`][cstring] |
| `NUL`-terminado, plano | [`*const`<br>`c_char`][c_char]<sup>γ</sup> | sí<sup>δ</sup> | [`*mut`<br>`c_char`][c_char]<sup>γ</sup> | | [`*mut`<br>`c_char`][c_char]<sup>γε</sup> |

<sup>α</sup> Casi inútil, porque la mayoría de los cambios podrían cambiar la
longitud de un punto de código UTF-8. Una excepción es [conversion de mayúscula
/ minuscula sólo para ASCII][make_ascii_lowercase].

<sup>β</sup> En la mayoría de los casos, puede prestar memoria estática (por
ejemplo, una cadena literal) con un tipo como `&' static str`.

<sup>γ</sup> Con apuntadoras planos, estás en tu propio respecto a las
semanticas de dueño / préstamo. Cualquier buena biblioteca C documentará sus
expectativas.

<sup>δ</sup> Puede cortar el frente de una cadena `NUL`-terminado, pero no el final.

<sup>ε</sup> Sobre el principio general de que si estás el dueño, puedes
cambiar el valor. Pero puedes usar [`*const c_char`][c_char] en su lugar.


## Funciones y closures

Las *capturadas* o *variables libres* de un closure son las variables usados en
un expresion de lambda, que no son definidos en el lambda o su lista de argumentos.
Las capturadas vienen del entorno del expression de lambda.

Rust infiere cuál(es) trait(s) un closure puede implementar de cómo las
capturas son usados. Puede forzar que los valores se muevan en un closure
prefijando [la palabra clave `move`][move-closure].

Cada lambda y `fn` tiene su propio tipo único, sin nombre (¿tipo Voldemort?).
Esto permite el envío estático y inserción en linea. Cada uno de estos tipos
sin nombre puede ser forzados a la apropiada `fn` / `Fn` / `FnMut` / `FnOnce`.

| | Es | ¿Puedes cambiar las capturadas? | ¿Puedes mover de las capturadas? |
| --- | --- | --- | --- |
| [`fn(A) -> B`][fnptr] | tipo | sin capturadas | sin capturadas |
| [`Fn(A) -> B`][fn] | trait | no | no |
| [`FnMut(A) -> B`][fnmut] | trait | sí | no |
| [`FnOnce(A) -> B`][fnonce] | trait | sí | sí |
| [`FnBox(A) -> B`][fnbox]<sup>α</sup> | trait | sí | sí |

<sup>α</sup> Sólo versión nocturna, en Rust 1.18.


## Tamaños

Esta tabla describe los tamaños de algunos tipos comunes.

Una *palabra de la maquina* es un apuntador o un entero de tamaño del
apuntador.

El primer tamaño es el tamaño del propio valor: el material que vive en la
pila si lo pones en una variable `let`. El segundo tamaño es el tamaño de
cualquier *dueño* datos en el montón.

Suponemos que `T`, `A`, `B`, `C` son [`Sized`][sized].

| Tipo | Tamaño del valor | Contenido | Tamaño del montón |
| --- | --- | --- | --- |
| [`bool`][bool] | 1 byte | 0 o 1 | |
| [`()`][tuple] | ¡vacío! | | |
| [`(A, B, C)`][tuple]<br>[`struct`][struct] | suma de `A`, `B`, `C` + padding / alineación | valores de tipo `A`, `B`, `C` | si dueño es `A`, `B`, o `C` |
| [`enum`][enum] | tamaño de la etiqueta<br>+ máx de variantes<br>+ padding / alineación | etiqueta + solo variante | si dueño es variante |
| [`[T; n]`][array] | `n` × tamaño de `T` | `n` elementos de tipo `T` | si dueño es `T` |
| [`&T`][ref], [`&mut T`][mut-ref]<br>[`*const T`][rawptr], [`*mut T`][rawptr] | 1 palabra | apuntador | |
| [`Box<T>`][box] | 1 palabra | apuntador | tamaño de `T` |
| [`Option<T>`][option] | 1 palabra + tamaño de `T` + padding / alineación (pero vea abajo) | etiqueta + opcionalmente `T` | si dueño es `T`, si `Option` es `Some` |
| [`Option<&T>`][option]<br>[`Option<&mut T>`][option] | 1 palabra<sup>β</sup> | apuntador o `NULL` | |
| [`Option<Box<T>>`][option] | 1 palabra<sup>β</sup> | apuntador o `NULL` | tamaño de `T`, si `Option` es `Some` |
| [`[T]`][slice], [`str`][str] | tamaño dinámica | elementos o puntos de código | |
| [`&[T]`][slice] | 2 palabras | apuntador, longitud (en elementos) | |
| [`&str`][str] | 2 palabras | apuntador, longitud (en bytes) | |
| [`Box<[T]>`][box] | 2 palabras | apuntador, longitud (en elementos) | longitud × tamaño de `T` |
| [`Box<str>`][box] | 2 palabras | apuntador, longitud (en bytes) | longitud (bytes) |
| [`Vec<T>`][vec] | 3 palabras | apuntador, longitud, capacidad | capacidad × tamaño de `T` |
| [`String`][string] | 3 palabras | apuntador, longitud, capacidad | capacidad (bytes) |
| [`Trait`][trait-object] | tamaño dinámica | campos del tipo concreto | si dueño es tipo concreto |
| [`&Trait`][trait-object] | 2 palabras | apuntador al valor concreto, apuntador a vtable | |
| [`Box<Trait>`][trait-object] | 2 palabras | apuntador al valor concreto, apuntador a vtable | tamaño del tipo concreto |
| [`fn` específico utilizado como valor][fnptr]<sup>α</sup> | ¡vacío! | | |
| [lambda específico][closure] | depende de las capturadas,<br>pero se conoce estáticamente | capturadas | si dueño es capturadas |
| [`fn(A) -> B`][fnptr]<br>[`unsafe fn(A) -> B`][fnptr]<br>[`extern fn(A) -> B`][fnptr] | 1 palabra<sup>α</sup> | apuntador a código | |
| [`PhantomData<T>`][phantomdata] | ¡vacío! | | |
| [`Rc<T>`][rc]<br>[`Arc<T>`][arc] | 1 palabra | apuntador | 2 palabras + tamaño de `T` + padding / alineación |
| [`Cell<T>`][cell] | tamaño de `T` | `T` | si dueño es `T` |
| [`AtomicT`][atomic] | tamaño de `T` | `T` | |
| [`RefCell<T>`][refcell] | 1 palabra + tamaño de `T` + padding / alineación | bandera de préstamo, `T` | si dueño es `T` |
| [`Mutex<T>`][mutex]<br>[`RwLock<T>`][rwlock] | 2 palabras + tamaño de `T` + padding / alineación | bandera de veneno, apuntador a mutex de sistema, `T` | si dueño es `T`, + mutex de sistema |

<sup>α</sup> Estos son apuntadores a funciones. Técnicamente, pueden tener un
tamaño diferente desde un apuntador de datos, pero esto no ocurre en
arquitecturas comunes.

<sup>β</sup> Esta optimización se aplica realmente a cualquier enum en forma de
`Option` que contiene, en algún lugar, un campo que no puede ser 0.


[UTF-8]: https://en.wikipedia.org/wiki/UTF-8
[static mut]: https://doc.rust-lang.org/book/const-and-static.html#mutability
[copy]: https://doc.rust-lang.org/std/marker/trait.Copy.html
[ref]: http://rust-lang.github.io/book/second-edition/ch04-02-references-and-borrowing.html
[mut-ref]: http://rust-lang.github.io/book/second-edition/ch04-02-references-and-borrowing.html#mutable-references
[Liskov]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
[rc]: https://doc.rust-lang.org/std/rc/
[arc]: https://doc.rust-lang.org/std/sync/struct.Arc.html
[box]: https://doc.rust-lang.org/std/boxed/
[mutex]: https://doc.rust-lang.org/std/sync/struct.Mutex.html
[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[rwlock]: https://doc.rust-lang.org/std/sync/struct.RwLock.html
[unsafecell]: https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html
[slice]: https://doc.rust-lang.org/std/primitive.slice.html
[vec]: https://doc.rust-lang.org/std/vec/struct.Vec.html
[str]: https://doc.rust-lang.org/std/primitive.str.html
[string]: https://doc.rust-lang.org/std/string/struct.String.html
[osstr]: https://doc.rust-lang.org/std/ffi/struct.OsStr.html
[osstring]: https://doc.rust-lang.org/std/ffi/struct.OsString.html
[make_ascii_lowercase]: https://doc.rust-lang.org/std/primitive.str.html#method.make_ascii_lowercase
[path]: https://doc.rust-lang.org/std/path/struct.Path.html
[pathbuf]: https://doc.rust-lang.org/std/path/struct.PathBuf.html
[cow]: https://doc.rust-lang.org/std/borrow/enum.Cow.html
[cstr]: https://doc.rust-lang.org/std/ffi/struct.CStr.html
[cstring]: https://doc.rust-lang.org/std/ffi/struct.CString.html
[c_char]: https://docs.rs/libc/0.2.22/libc/type.c_char.html
[cell]: https://doc.rust-lang.org/std/cell/struct.Cell.html
[atomic]: https://doc.rust-lang.org/std/sync/atomic/index.html
[sync]: https://doc.rust-lang.org/std/marker/trait.Sync.html
[i8]: https://doc.rust-lang.org/std/primitive.i8.html
[i16]: https://doc.rust-lang.org/std/primitive.i16.html
[i32]: https://doc.rust-lang.org/std/primitive.i32.html
[i64]: https://doc.rust-lang.org/std/primitive.i64.html
[i128]: https://doc.rust-lang.org/std/primitive.i128.html
[isize]: https://doc.rust-lang.org/std/primitive.isize.html
[u8]: https://doc.rust-lang.org/std/primitive.u8.html
[u16]: https://doc.rust-lang.org/std/primitive.u16.html
[u32]: https://doc.rust-lang.org/std/primitive.u32.html
[u64]: https://doc.rust-lang.org/std/primitive.u64.html
[u128]: https://doc.rust-lang.org/std/primitive.u128.html
[usize]: https://doc.rust-lang.org/std/primitive.usize.html
[f32]: https://doc.rust-lang.org/std/primitive.f32.html
[f64]: https://doc.rust-lang.org/std/primitive.f64.html
[tuple]: https://doc.rust-lang.org/std/primitive.tuple.html
[struct]: https://rust-lang.github.io/book/second-edition/ch05-00-structs.html
[enum]: https://rust-lang.github.io/book/second-edition/ch06-00-enums.html
[trait-object]: https://rust-lang.github.io/book/second-edition/ch17-02-trait-objects.html
[fnptr]: https://rust-lang.github.io/book/second-edition/ch19-05-advanced-functions-and-closures.html#function-pointers
[fn]: https://doc.rust-lang.org/std/ops/trait.Fn.html
[fnmut]: https://doc.rust-lang.org/std/ops/trait.FnMut.html
[fnonce]: https://doc.rust-lang.org/std/ops/trait.FnOnce.html
[fnbox]: https://doc.rust-lang.org/std/boxed/trait.FnBox.html
[closure]: https://rust-lang.github.io/book/second-edition/ch13-01-closures.html
[phantomdata]: https://doc.rust-lang.org/std/marker/struct.PhantomData.html
[option]: https://doc.rust-lang.org/std/option/enum.Option.html
[move-closure]: https://doc.rust-lang.org/book/closures.html#move-closures
[sized]: https://doc.rust-lang.org/std/marker/trait.Sized.html
[rawptr]: https://doc.rust-lang.org/std/primitive.pointer.html
[array]: https://doc.rust-lang.org/std/primitive.array.html
[bool]: https://doc.rust-lang.org/std/primitive.bool.html


## See also

[rust-learning cheat sheets](https://github.com/ctjhoa/rust-learning#cheat-sheets)

[Rust Cheatsheet](http://phaiax.github.io/rust-cheatsheet/)

[Rust Iterator Cheat Sheet](https://danielkeep.github.io/itercheat_baked.html)

[The Periodic Table of Rust Types](http://cosmic.mearie.org/2014/01/periodic-table-of-rust-types/)

[Cheatsheet for Futures](https://rufflewind.com/img/rust-futures-cheatsheet.html)

[Time complexity for collections](https://doc.rust-lang.org/std/collections/#sequences)
