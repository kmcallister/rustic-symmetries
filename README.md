# Rustic Symmetries

[Version en español](es_ES.md)

In honor of the [second birthday of stable Rust][birthday], here is a small
contribution to Rust documentation.

Many concepts in Rust come in matching sets or have some pleasing symmetry.
This is a summary or "cheat sheet" for some of these.

Some of these tables may look intimidating, but they reflect the reality of
systems programming with fine-grained control over memory. If you're just
getting started with Rust, *don't panic* and do set this article aside! It's
intended as a reference and a guide to how to organize these things in your
head once you vaguely know what they are.

Contributions are welcome! Just open a [pull request].

[birthday]: https://blog.rust-lang.org/2017/05/15/rust-at-two-years.html
[pull request]: https://github.com/kmcallister/rustic-symmetries/pulls


## References

| | Can [`Copy`][copy]? | Can mutate through? |
| --- | --- | --- |
| [`&T`][ref] | yes | no |
| [`&mut T`][mut-ref] | no | yes |

This demonstrates that neither `&T` nor `&mut T` is a subtype of the other, in
the [Liskov] sense.


## Ownership and mutability

Ownership controls when a value is destroyed. A value can have either a unique
owner, or a number of references which collectively share ownership. The latter
case usually involves reference counting.

Interior mutability refers to any wrapper type `Wrapper` such that we can go
from `&Wrapper<T>` to `&mut T`, or at least have some of the capabilities of
`&mut T`.

The column headings here refer (more or less) to the point in time at which
Rust's safety invariants are checked. Note that no unsafety can occur due to
sharing the thread-unsafe structures between threads. The compiler will simply
reject your code, through the magic of the [`Sync`][sync] trait.

| | Static | Dynamic | Dynamic,<br>thread-safe |
| --- | --- | --- | --- |
| **Direct ownership** | `T` | | |
| **Ownership via heap** | [`Box<T>`][box] | | |
| **Shared ownership** | | [`Rc<T>`][rc] | [`Arc<T>`][arc] |
| **Get, set, compare<br>& swap, etc.** | [`&mut T`][mut-ref] | [`Cell<T>`][cell] | [`AtomicFoo`][atomic] |
| **Borrow immutably** | [`&T`][ref] | | |
| **Borrow mutably,<br>or single reader** | | | [`Mutex<T>`][mutex] |
| **Borrow mutably,<br>or multiple readers** | [`&mut T`][mut-ref] | [`RefCell<T>`][refcell] | [`RwLock<T>`][rwlock] |
| **Borrow mutably,<br>unsafe** | [`static mut`][static mut] | [`UnsafeCell<T>`][unsafecell] | [`UnsafeCell<T>`][unsafecell] |


## Numeric types

| | Unsigned integer | Signed integer | Floating-point |
| --- | --- | --- | --- |
| **8 bits** | [`u8`][u8] | [`i8`][i8] | |
| **16 bits** | [`u16`][u16] | [`i16`][i16] | |
| **32 bits** | [`u32`][u32] | [`i32`][i32] | [`f32`][f32] |
| **64 bits** | [`u64`][u64] | [`i64`][i64] | [`f64`][f64] |
| **128 bits** | [`u128`][u128]<sup>α</sup> | [`i128`][i128]<sup>α</sup> | |
| **Pointer-sized** | [`usize`][usize] | [`isize`][isize] | |

<sup>α</sup> Nightly-only, as of rustc 1.18.


## Strings

| Format | Borrow<sup>β</sup> | Borrow substr? | Mutate | Copy on write | Owned, in heap |
| --- | --- | --- | --- | --- | --- |
| Any bytes | [`&[u8]`][slice] | yes | [`&mut`&nbsp;`[u8]`][slice] | [`Cow<[u8]>`][cow]<br> | [`Vec<u8>`][vec] |
| [UTF-8] | [`&str`][str] | yes | [`&mut`&nbsp;`str`][str]<sup>α</sup> | [`Cow<str>`][cow]<br> | [`String`][string] |
| Platform-dependent | [`&OsStr`][osstr] | no | | [`Cow<OsStr>`][cow] | [`OsString`][osstring] |
| Filesystem path | [`&Path`][path] | no | | [`Cow<Path>`][cow] | [`PathBuf`][pathbuf] |
| `NUL`-terminated, safe | [`&CStr`][cstr] | no | | [`Cow<CStr>`][cow] | [`CString`][cstring] |
| `NUL`-terminated, raw | [`*const`<br>`c_char`][c_char]<sup>γ</sup> | yes<sup>δ</sup> | [`*mut`<br>`c_char`][c_char]<sup>γ</sup> | | [`*mut`<br>`c_char`][c_char]<sup>γε</sup> |

<sup>α</sup> Nearly useless, because most mutations could change the length of a UTF-8
             codepoint. One exception is [ASCII-only case conversion][make_ascii_lowercase].

<sup>β</sup> In most cases, you can borrow static memory (e.g. a string literal) with a type
             like `&'static str`.

<sup>γ</sup> With raw pointers, you are on your own regarding ownership / borrowing
             semantics. Any good C library will document its expectations.

<sup>δ</sup> You can slice off the front of a `NUL`-terminated string, but not the end.

<sup>ε</sup> On the general principle that if you own something you can mutate
             it. But you could use [`*const c_char`][c_char] instead.


## Functions and closures

The *captures* or *free variables* of a closure are the variables used in a
lambda expression which are not defined in the lambda or its arguments list.
The captures come from the surrounding environment of a lambda expression.

Rust infers which trait(s) a closure can implement from how the captures are
used. You can force values to be moved into a closure by prefixing [the `move`
keyword][move-closure].

Each lambda and `fn` has its own unique, un-nameable type (Voldemort type?).
This enables static dispatch and inlining. Each of these un-nameable types can
be coerced to the appropriate `fn` / `Fn` / `FnMut` / `FnOnce`.

| | Is a | Can mutate captures? | Can move out of captures? |
| --- | --- | --- | --- |
| [`fn(A) -> B`][fnptr] | type | no captures | no captures |
| [`Fn(A) -> B`][fn] | trait | no | no |
| [`FnMut(A) -> B`][fnmut] | trait | yes | no |
| [`FnOnce(A) -> B`][fnonce] | trait | yes | yes |
| [`FnBox(A) -> B`][fnbox]<sup>α</sup> | trait | yes | yes |

<sup>α</sup> Nightly-only, as of rustc 1.18.


## Sizes

This table describes the sizes of some common types.

A *word* is a pointer or a pointer-sized integer.

The first size is the size of the value itself: the stuff that ends up on the
stack if you put it in a `let` variable. The second size is the size of any
*owned* data in the heap.

We assume `T`, `A`, `B`, `C` are [`Sized`][sized].

| Type | Value size | Contents | Heap size |
| --- | --- | --- | --- |
| [`bool`][bool] | 1 byte | 0 or 1 | |
| [`()`][tuple] | empty! | | |
| [`(A, B, C)`][tuple]<br>[`struct`][struct] | sum of `A`, `B`, `C` + pad / align | values of type `A`, `B`, `C` | anything owned by `A`, `B`, or `C` |
| [`enum`][enum] | size of tag<br>+ max of variants<br>+ pad / align | tag + one variant | anything owned by variant |
| [`[T; n]`][array] | `n` × size of `T` | `n` elements of type `T` | anything owned by `T` |
| [`&T`][ref], [`&mut T`][mut-ref]<br>[`*const T`][rawptr], [`*mut T`][rawptr] | 1 word | pointer | |
| [`Box<T>`][box] | 1 word | pointer | size of `T` |
| [`Option<T>`][option] | 1 word + size of `T` + pad / align (but see below) | tag + optionally `T` | anything owned by `T`, if `Some` |
| [`Option<&T>`][option]<br>[`Option<&mut T>`][option] | 1 word<sup>β</sup> | pointer or `NULL` | |
| [`Option<Box<T>>`][option] | 1 word<sup>β</sup> | pointer or `NULL` | size of `T`, if `Some` |
| [`[T]`][slice], [`str`][str] | dynamic size | elements or codepoints | |
| [`&[T]`][slice] | 2 words | pointer, length (in elements) | |
| [`&str`][str] | 2 words | pointer, length (in bytes) | |
| [`Box<[T]>`][box] | 2 words | pointer, length (in elements) | length × size of `T` |
| [`Box<str>`][box] | 2 words | pointer, length (in bytes) | length (bytes) |
| [`Vec<T>`][vec] | 3 words | pointer, length, capacity | capacity × size of `T` |
| [`String`][string] | 3 words | pointer, length, capacity | capacity (bytes) |
| [`Trait`][trait-object] | dynamic size | fields of concrete type | anything owned by fields |
| [`&Trait`][trait-object] | 2 words | pointer to concrete value, pointer to vtable | |
| [`Box<Trait>`][trait-object] | 2 words | pointer to concrete value, pointer to vtable | size of concrete value |
| [Specific `fn` used as a value][fnptr]<sup>α</sup> | empty! | | |
| [Specific lambda][closure] | depends on captures,<br>but known statically | captures | anything owned by captures |
| [`fn(A) -> B`][fnptr]<br>[`unsafe fn(A) -> B`][fnptr]<br>[`extern fn(A) -> B`][fnptr] | 1 word<sup>α</sup> | pointer to code | |
| [`PhantomData<T>`][phantomdata] | empty! | | |
| [`Rc<T>`][rc]<br>[`Arc<T>`][arc] | 1 word | pointer | 2 words + size of `T` + pad / align |
| [`Cell<T>`][cell] | size of `T` | `T` | anything owned by `T` |
| [`AtomicT`][atomic] | size of `T` | `T` | |
| [`RefCell<T>`][refcell] | 1 word + size of `T` + pad / align | borrow flag, `T` | anything owned by `T` |
| [`Mutex<T>`][mutex]<br>[`RwLock<T>`][rwlock] | 2 words + size of `T` + pad / align | poison flag, pointer to OS mutex, `T` | anything owned by `T` + OS mutex |

<sup>α</sup> These are function pointers. Technically, they can have a different size
             from a data pointer, but this does not happen on common architectures.

<sup>β</sup> This optimization actually applies to any `Option`-shaped enum which contains,
             somewhere, a field which cannot be 0.


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
