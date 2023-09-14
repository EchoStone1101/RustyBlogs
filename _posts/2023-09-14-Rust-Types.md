---
title: "How I Implemented a Type Equivalence System in Rust"
date: 2023-09-14
author: EchoStone
layout: post
tags:
   - Rust
---

**Disclaimer**: what I am about to describe is not necessarily limited to Rust in any way. I just happened to have implemented it in Rust. Plus I also happened to like Rust.

Also, it is possible that this problem was solved long ago by other clever computer people, and I was just too impatient to dig for it. The truth is: I spent so long getting *my* solution to work, that now I am just proud of it. At least I get to be one of the "clever computer people" :)

## Why a Type System?

So I have been working on this compiler-like project for a while (it's actually a symbolic execution engine, but that's irrelevant to the topic), and one of the early design decisions was to enforce a *strict type system* throughout the project. 

By that, I mean every expression that my compiler handles will be associated with a `Type`, a separate object containing type information for the expression, in the context of my custom IR. According to my brief experience with LLVM, this is fairly common practice in a compiler system.

For instance, the `Type` of an `Exp::PointerLiteral` (a pointer literal, as an expression) is always `Type::Pointer`, which should never be used like an `Exp::IntLiteral`, whose `Type` may be one of `Type::Int` or `Type::BitVec`. (Yes, both `Exp` and `Type` are Rust enums. God knows how I would write code without Rust enums.)

`Type`s are the runtime safety line that gives me confidence, that my code will not break deep into some parsing or optimization pass. Better still, `Type`s look much less desperate than a Java programmer spamming `instanceof` everywhere. `Type`s are really nice and all. But there's one problem with it - how do I compare two `Type`s?

## Higher-Order Types, and Named Types, and Hash Failure

What's difficult about comparing `Type`s? 

Well first, remember that `Type`s are Rust enums. And they are almost never unit variants. For example, `Type::BitVec` looks like this:
```rust
pub enum Type {
    BitVec(BitVecKind),
}

pub struct BitVecKind {
    width: usize,
}
```
so comparing `Type`s, conceptually, requires per-field comparison of the `Type` object. What's worse, some of the `Type`s are *higher-order*, which is my fancy word for "they refer to other `Type`s":
```rust
pub enum Type {
    Tuple(TupleKind),
}

pub struct TupleKind {
    fields: Vec<Rc<Type>>,
}
```
Hope the problem is clear: to tell apart two `Type`s, not only do we need per-field comparison, we need *recursive* comparison! Considering how pervasive `Type` checks are in my code, this is obviously not favorable.

And as if I was a poor guy facing a code review question, I smashed `HashMap` at the problem.

### Hash for the Win

Note how `TupleKind` is referring to `Type`s of fields with `Rc<Type>`. For unfamiliar readers, `Rc` stands for "(R)eference (C)ountered pointer". Like shared smart pointers in C++ or objects in Java, `Rc<X>` in Rust holds the ownership of `X`, while allowing for cloning the pointer to `X` as `Rc<X>` instances, which can be used like immutable references. 

Now my intuition was: make sure any identical `Type`s are actually the same `Type` instance. I already had a `TypeManager` struct that manages all created `Type`s centrally, via factory-style methods:
```rust
let mut tman = TypeManager::new();
let field1 = tman.mk_tuple(vec![tman.mk_bitvec(32)]);
let field2 = tman.mk_array(5, tman.mk_bitvec(64));
let field3 = tman.mk_pointer(tman.mk_bool());
let ty = tman.mk_tuple(vec![field1, field2, field3]);
// `ty` should be ( ( i32 ), [5 x i64], i1* )
```
It would seem sufficient to just keep a bunch of internal `HashMap`s, one for each kind of `Type`, within the `TypeManager`. Each `HashMap` effectively collects all the `Type`s of the same kind made so far. The key would be the `Type`'s fields, hashed appropriately, and the value would be the `Rc<Type>` instance.
Whenever the `TypeManager` is consulted to make a new `Type`, just check the corresponding `HashMap`, using the provided components to query whether an identical `Type` has been created already. If not, create and register a new `Type`; otherwise just return a clone of the old `Rc<Type>`.

To reiterate: this hashing scheme effectively **makes `Type`s become singleton with respect to the identity relation**. This stone kills two birds. (1) The *recursive* problem is fixed, since we can now inductively argue that identical `Type`s must be clones of the same `Rc<Type>`; hence a type comparison is reduced to **one pointer comparison**, and hashing an aggregate `Type`, like `Type::Tuple`, needs only a shallow loop over its fields. (2) The memory overhead is also greatly reduced, considering how extensively `Type`s are used throughout the code.

### Named Types: The Bad Apple Spoiling The Bunch

Seems nice? Well, now comes the *but* part of the story, and the gist of it is *named types*.

The language my compiler is parsing contains type declarations, and these declarations come with *names*. Like so:
```llvm
%StructType.0 = type { %_type.0, %IPST.46 }
%_type.0 = type { i64, i64, i32, i8, i8, i8, i8, ptr, ptr, ptr, ptr, ptr }
%IPST.46 = type { ptr, i64, i64 }
```
(Yes, I am parsing LLVM IR.) Note how the *name* of a type can be used interchangeably with its verbose form. In other words, `%StructType.0`, `{ %_type.0, { ptr, i64, i64 } }`, and `{ { i64, i64, i32, i8, i8, i8, i8, ptr, ptr, ptr, ptr, ptr }, %IPST.46 }` should actually be *the same* `Type`.

One may want to expand all such named type references to avoid ambiguities. Unfortunately, that's not always possible. One simple example would be the type of a singly linked list node:
```llvm
%ListNode = type { i32, %ListNode* }
```
where the type declaration may contain *recursion*.

With named types, the hashing scheme suddenly falls apart. To represent the named type references, there was a `Type::Named` variant, which only contained the type name, which in turn must be resolved by the `TypeManager` into its verbose, *true* `Type`. This workaround, apart from being messy, is fundamentally flawed, because it breaks the **invariant that identical `Type`s are clones of the same `Rc<Type>`**. Before and after resolving a `Type::Named`, `TypeManager` can give different answers to a `Type` comparsion.

Previously, I actually managed to make this scheme work, for the parsing part of my compiler only, by invoking the resolve function at very specific points. No need to say it was unfavorable. Worse still, it simply breaks when I started executing the expressions, which made me really reconsider how to design a scheme that works for the named types.

## Identity, Equivalence, and Union-Find

In hindsight, the old scheme got it *almost* right. The one thing it fails on is **confusing identity with equivalence**.

Let's look at an example. Consider the following type declarations:
```llvm
X = type { A, i32 }
Y = type { i8, B }
Z = type { i8, i32 }
A = type i8
B = type i32
```
Having distinguished **identity** from **equivalence**, we note that the types `X`, `Y`, and `Z` are still *equivalent* to each other, while each pair of them is no longer deemed *identical*. 

Expanding on this view, `Type::Named` should never be considered an "incomplete" `Type` that must be "resolved". `Type::Named` *should be its own type*, unique by its name. A type declaration which binds a type name and a verbose `Type`, should be viewed as an **aliasing** relation, where a `Type::Named` aliases to the verbose `Type`, and vice versa.

And I don't know about you, but the word "equivalence" just rings a bell for me, and the name of the ringtone is **union-find**, which is the standard solution to determine equivalence relations.

Combining all these (plus some staring into the void, murmuring to myself and pulling of my hair), I was able to come up with the final algorithm.

### The Algorithm

Let's cut straight to the algorithm itself. First, all variants of `Type` has been updated to contain an *alias pointer*:
```rust
pub enum Type {
    Array(ArrayKind),
}

pub struct ArrayKind {
    len: usize,
    elem_ty: Rc<Type>,
    alias: AliasPointer,
}

type AliasPointer = RefCell<Weak<Type>>;
```
which is intended to be carefully set up, so that it points to a `Type` that is an alias of the current `Type`. The `RefCell<_>` wrapper is used so that we can mutate the alias pointer, even when the current `Type` is behind multiple `Rc<_>` pointers. This is called **interior mutability** in Rust, a topic I will not elaborate here. Just know that it enables us to mutate `alias`. `Weak` replaces `Rc` because alias pointers may form loops (and thus self-reference).

The algorithm has two procedures, one for computing the correct aliasing relation given a set of type declarations, the other for setting the alias pointer with the current declarations in effect.

> Algorithm I. Computing alaising relation, given type declarations.

**Input**: All `Type`s created so far; a set of declarations, in the form of `(name, Rc<Type>)` pairs.

**Output**: All alias pointers are correctly set up.

1. Clear existing alias relation by resetting the alias pointer of all `Type`s to point to the `Type` itself.

2. For each `(name, ty: Rc<Type>)` pair in the declarations, first make a `Type::Named` with `name` if non-existent. Then, find the alias root (by which we mean traversing the alais pointer, until it points to itself; namely the "find" operation in the union-find) of `ty`, and set its alias pointer to point at the `Type::Named`.

3. Sort all `Type`s by their *order*, in increasing order. The *order* of a `Type` is defined as 0 for zero-order `Type`s (`Type`s that do not reference other `Type`s;  note that `Type::Named` is zero-order), and 1 plus the maximal *order* of all component `Type`s for higher-order (`Type`s that do reference other `Type`s).

4. In the order determined by (3), update the alias pointer of a `Type`. For a zero-order `Type`, nothing needs to be done. For a higher-order `Type`, denote it as `ty` and its current alias root as `ultimate`. Make a `Type` of similar kind, using the alias roots of the components of `ty`, and find its alias root, which we denote as `reduced`.\
Point the alias pointer of `ty` to `reduced`. Then, if the order of `ultimate` is less than that of `reduced`, point `reduced` to `ultimate`. Otherwise, point `ultimate` to `reduced`.

> Algorithm II. Given an `Rc<Type>`, determine the alias pointer, with current declarations in effect.

**Input**: A `Rc<Type>` to be set up.

**Output**: The given `Type`'s alias pointer is correctly set up.

1. Denote the given `Rc<Type>` as `ty`. Find `reduced` like in step (4) of Algorithm I.
2. Point the alias pointer of `ty` to `reduced`.

### Example

Sort of complicated? Here is a demonstration, using the running example
```llvm
X = type { A, i32 }
Y = type { i8, B }
Z = type { i8, i32 }
A = type i8
B = type i32
```

**Alg.1 (1)**

We have `Type`s `i8`, `i32`, `A`, `B`, `X`, `Y`, `Z`, `{A, i32}`, `{i8, B}`, and `{i8, i32}`. Reset their alias pointers, producing

||`i8`| `i32`| `A`| `B`|`X`| `Y`| `Z`| `{A, i32}`|`{i8, B}`| `{i8, i32}`|
|---|---|---|---|---|---|---|---|---|---|---|
|alias|`i8`| `i32`| `A`| `B`|`X`| `Y`| `Z`| `{A, i32}`|`{i8, B}`| `{i8, i32}`|

**Alg.1 (2)**

||`i8`| `i32`| `A`| `B`|`X`| `Y`| `Z`| `{A, i32}`|`{i8, B}`| `{i8, i32}`|
|---|---|---|---|---|---|---|---|---|---|---|
|alias|`A`| `B`| `A`| `B`|`X`| `Y`| `Z`| `X`|`Y`| `{i8, i32}`|

**Alg.1 (3)**

Let's say we are updating exactly in the order of `i8`, `i32`, `A`, `B`, `X`, `Y`, `Z`, `{A, i32}`, `{i8, B}`, and `{i8, i32}`. The last three `Type`s have *order* of 1.

**Alg.1 (4)**

The first interesting case appears when updating for `{A, i32}`. Following the algorithm, the `ultimate` would be `X`. The `reduced` would be the alias root of `{A, B}` (since `i32` points to `B`). Note that `{A, B}` is a new `Type` that we jus created, whose alias pointer always points to itself. Hence `reduced` is `{A, B}`. By the last step, we point `reduced`, that is `{A, B}`, to `ultimate`, that is `X`.

Next, updating for `{i8, B}` also creates `Type` `{A, B}`. This time it already exists, and aliases to `X`. Hence `reduced` is `X` and `ultimate` is `Y`, where we point `Y` to `X`.

Finally for `{i8, i32}`, its `ultimate`, i.e. itself, becomes pointed directly to `X`, which is the `reduced`.

Now the alias relation looks like

||`i8`| `i32`| `A`| `B`|`X`| `Y`| `Z`| `{A, i32}`|`{i8, B}`| `{i8, i32}`|
|---|---|---|---|---|---|---|---|---|---|---|
|alias|`A`| `B`| `A`| `B`|`X`| `X`| `X`| `X`|`X`| `X`|

Note how all three aggregate `Type`s now alias to `X`. To determine whether two `Type`s are *equivalent*, it suffices to run a normal union-find query, which, with path compression, is practically `O(1)`. 

In other words, the new scheme should still be efficient - obviously slower than a mere pointer comparison, but of the same complexity in principle - while solving the named problem elegantly.

I will not bore you with a rundown of the second procedure. Hopefully the algorithm is now making sense intuitively. If not, then you might be asking, just like I was:

### (Why) Does it Work?

The step (1) and (2) of Algorithm.I is fairly standard union-find business, which I don't think needs defending. The step (3) and (4) contains two uncommon details, but first I'd like to throw out the main idea when designing the algorithm: 

> **Alias pointers should always point from a higher-order `Type`, to a lower-order one.**

* The update order in step (3), assuming the principle above, is actually making an *inductive* guarantee: that, given all lower-order `Type`s are correctly aliased, the current `Type` can be correctly aliased as well, only by finding the roots of component `Type`s.

* The comparison of *order* towards the end of step (4) is simply enforcing the principle above.

Informally speaking, the algorithm is essentially doing *best effort reduction* over all `Type`s, rather than the intuitive idea of "expanding" that we mentioned earlier. The running principle makes sure that each move on the alias tree never increases the *order*, so all `Type`s in the end should be aliased to a `Type` that is as low-ordered as possible.

The principle has yet one more important aspect, and that is better handling for *recursive* type declarations mentioned above. The exact impact on a recursive declaration is, as it would be in other great informative writings, left for the curious reader to think about themselves.

In truth, the implementation contains several more subtlties in terms of updating the alais pointer, so that we don't accidentally go on an endless recursion. I would like to show the real code, but it is kind of convoluted with other stuff. Anyway, I believe that the description so far has captured the gist of the design.

## After-Math

Designing and implementing the algorithm is easily the most *algorithmic*, *math-y* experience for me as a system programmer within a long while. It was surprisingly quite fun and fulfilling.

Arriving at the final design of my type system took around a week. Nearly half of it was spent in making the transition from the old scheme to the new one. The fact that Rust's `HashMap` uses a random hashing algorithm by default made it extra difficult to debug the algorithm, especially when the bugs are occurring only 1% of the time. I really should have replaced the hasher with a deterministic one, but somehow I was just lazy.

And all that for what? I now have a type system that enforces type-singleton on a type identity basis, while also providing `O(1)` equivalence checks using alias pointers. If you can't tell, I was secretly kind of happy with it. Now that I have bragged about it here, I can offcially conclude that I am now publicly happy with it :)

