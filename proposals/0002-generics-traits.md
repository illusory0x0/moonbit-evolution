# Generics Traits

* Proposal: [ME-0002](https://github.com/illusory0x0/moonbit-evolution/blob/0002-generics-traits/proposals/0002-generics-traits.md)
* Author: [illusory0x0](https://github.com/illusory0x0)
* Status: Under review
* Review and discussion: [Github issue](https://github.com/moonbitlang/moonbit-evolution/pull/3)

## Introduction

We use type to constrain the value, and trait to constrain the type, but now supports generic data structures, but not generic traits which leads to a lot of API consistency issues when using generic data structures.

On the other hand, JavaScript has many APIs that require generic traits, the underlying DOM, and other web front-end frameworks make heavy use of generic classes, such as `react.js`, `vue.js`.

Although using `cast` function can solve some problems, it is not good practice to use `cast`  function heavily.


## Motivation

### API consistency

Right now Moonbit's trait doesn't support generic parameters, 
which can lead to a lot of API inconsistencies when using generic data structures.

for example, 
[Moonbit core library](https://github.com/moonbitlang/core/issues?q=is%3Aissue%20label%3A%22consistency%20review%22) has 
**29** consistency review issues, if we have **Generics Traits**, the compiler can check API consistency without human to code review.

### DOM API

Many JavaScript APIs use [Generic Classes](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-classes) 
or [Generic Interfaces](https://www.typescriptlang.org/docs/handbook/2/generics.html#generic-types).

Using the most commonly used API querySelector as an example. We have to write `DOM_Element::downcast` to cast.

```moonbit 
///|
#external
type Element

///|
trait DOM_Element {
  querySelector(Self, String) -> Element = _
  downcast(Element) -> Self = _
  to_element(Self) -> Element
}

///|
extern "js" fn Element::querySelector(
  self : Element,
  selectors : String
) -> Element = "(self,selecter) => self.querySelector(selecter)"

///|
fn[A, B] coerce(x : A) -> B = "%identity"

///|
impl DOM_Element with querySelector(self, selectors) {
  Element::querySelector(coerce(self), selectors)
}

///|
impl DOM_Element with downcast(self) = "%identity"

///|
#external
type HTMLCanvasElement

///|
#external
type HTMLDivElement

///|
#external
type HTMLAnchorElement

///|
#external
type HTMLImageElement

///|
impl DOM_Element for HTMLCanvasElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLDivElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLAnchorElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLImageElement with to_element(self) = "%identity"

///|
test {
  let x : HTMLDivElement = {
    ...
  }
  let e : HTMLImageElement = x.querySelector("x") |> DOM_Element::downcast

}
```




## Proposed solution


### Functor Example 


This makes it easy to use the compiler to check for API consistency.

Actually this `Functor` more similar to [traverse](https://hackage.haskell.org/package/base-4.21.0.0/docs/Prelude.html#v:traverse) when passing callback function which raise error.

```moonbit 
trait Functor[A] {
  fn[A,B] map(Self[A], f : (A) -> B raise?) -> Self[B] raise?
}

trait Monad[A] {
  fn[A,B] bind(Self[A], f : (A) -> Self[B] raise?) -> Self[B] raise?
  fn[A] pure(A) -> Self[A]
}

impl[A] Functor for FixedArray with map(self, f) { ... }
impl[A] Monad for FixedArray with[A,B] bind(self, f) {
  self.map(f).flatten()
}
impl[A] Monad for FixedArray with pure(a) {
  [a]
}

impl[T] Functor for Result with[T,F] map(self, f) { ... }
impl[T] Monad for Result with[T,F] bind(self, f) {
  match self {
    Ok(value) => f(value)
    Err(err) => err(err)
  }
}
impl[T] Monad for FixedArray with[T,F] pure(a) {
  Ok(a)
}
```

with generic traits, we can list the list of such traits that types are meant to belong, and thus improve API consistency.

### querySelector Example

support generics method in traits can reduce many `downcast` call.

```moonbit 
///|
#external
type Element

///|
trait DOM_Element {
  fn[E : DOM_Element] querySelector(Self, String) -> E = _
  to_element(Self) -> Element
}

///|
extern "js" fn Element::querySelector(
  self : Element,
  selectors : String
) -> Element = "(self,selecter) => self.querySelector(selecter)"

///|
fn[A, B] coerce(x : A) -> B = "%identity"

///|
impl DOM_Element with querySelector(self, selectors) {
  coerce(Element::querySelector(coerce(self), selectors))
}

///|
#external
type HTMLCanvasElement

///|
#external
type HTMLDivElement

///|
#external
type HTMLAnchorElement

///|
#external
type HTMLImageElement

///|
impl DOM_Element for HTMLCanvasElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLDivElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLAnchorElement with to_element(self) = "%identity"

///|
impl DOM_Element for HTMLImageElement with to_element(self) = "%identity"

///|
test {
  let x : HTMLDivElement = {
    ...
  }
  let e : HTMLImageElement = x.querySelector("x") 

}
```


## Possible alternatives

use [virtual packages](https://docs.moonbitlang.com/en/latest/language/packages.html#virtual-packages) to check API consistency, 
like OCaml [module signature](https://ocaml.org/manual/5.3/moduleexamples.html#s:signature), but module signature only can check module, can not check for type.
