+++
date = '2018-08-13T00:31:27-05:00'
draft = false
title = 'The Typestate Pattern: Where the Rust Type System Shines'
+++

If you've been writing Rust for a while, you've probably encountered a peculiar
but elegant pattern: structs that consume themselves to become different
structs, encoding a state machine directly into the type system. This is the
**typestate pattern**, and it's one of those patterns that makes Rust special.

## What Is the Typestate Pattern?

The typestate pattern encodes the states of a state machine as distinct types.
Instead of tracking state with an enum field or boolean flags, each state
becomes its own type, and transitions consume the old state to produce the new
one.

Here's a classic example—a door that can be locked or unlocked:

```rust
use std::marker::PhantomData;

struct Locked;
struct Unlocked;

struct Door<State> {
    state: PhantomData<State>,
}

impl Door<Locked> {
    fn new() -> Self {
        Door { state: PhantomData }
    }
    
    fn unlock(self) -> Door<Unlocked> {
        println!("Door unlocked");
        Door { state: PhantomData }
    }
}

impl Door<Unlocked> {
    fn open(&mut self) {
        println!("Door opened!");
    }
    
    fn lock(self) -> Door<Locked> {
        println!("Door locked");
        Door { state: PhantomData }
    }
}
```

> **Note about PhantomData**: Rust requires generic type parameters to actually
> be _used_ in a struct. `PhantomData` is a zero-sized marker type that tells 
> the compiler "This struct behaves as if it owns a value of type `T`", 
> satisfying the compiler with zero runtime cost.

Now try to open a locked door:

```rust
let door = Door::<Locked>::new();
door.open(); // Compile error! No method `open` on `Door<Locked>`
```

The compiler simply won't let you. The `open()` method doesn't exist for
`Door<Locked>`—only for `Door<Unlocked>`. Your state machine is enforced at
compile time.

## Why Typestates Work So Well in Rust

The concept of typestates isn't new; [it dates back to the 1980s](https://en.wikipedia.org/wiki/Typestate_analysis).
While the *concept* is old, actually using typestates as a practical
programming pattern has historically been awkward or verbose in most languages.
Rust makes them practical.

Rust's particular combination of features creates a perfect environment for 
typestate programming:

### **1. Move Semantics**

Rust's ownership system naturally enforces state transitions. When you call
`unlock(self)`, the `Door<Locked>` is moved and consumed. You literally cannot
use the old state anymore—the compiler won't let you:

```rust
let locked_door = Door::<Locked>::new();
let unlocked_door = locked_door.unlock();
// locked_door is gone now—try to use it and get a compile error
```

This is crucial. In languages with only reference semantics, preventing use of
the old state requires runtime checks or careful discipline. Rust's move
semantics make it automatic and free.

### **2. Zero-Cost Abstractions**

Those state marker types (`Locked`, `Unlocked`) are zero-sized types (ZSTs).
They contain no data and exist only at compile time. After optimization,
there's no runtime overhead—no enum tags, no extra bytes, no performance cost.
You get all the safety with none of the runtime penalty.

### **3. Expressive Type System**

Rust's generics and trait system let you express complex state relationships
clearly. You can share common behavior across states, implement state-specific
methods, and let the type system guide users toward correct usage.

```rust
impl<State> Door<State> {
    fn material(&self) -> &str {
        "oak"  // available in all states
    }
}
```

### **4. Compile-Time Guarantees**

Instead of discovering state machine violations at runtime (or worse, in
production), you find them while writing code. The compiler catches these bugs
before your code ever runs.

### **5. No Garbage Collection**

In garbage-collected languages, even if you can encode typestates, you often
can't guarantee the old state is truly gone—references might linger. Rust's
ownership guarantees that when a value is moved, it's genuinely inaccessible.

## The Bigger Picture

The typestate pattern exemplifies what makes Rust special: it doesn't just give
you tools for memory safety—it gives you tools for *correctness*. It takes the
philosophy of "if it compiles, it works" and extends it beyond memory to your
domain logic.

In other languages, you might use runtime checks, extensive testing, or just
hope developers read the documentation. In Rust, you make invalid states
unrepresentable. The code that compiles is code that respects your state
machine.
