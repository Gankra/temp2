% Understanding Unsafe in Rust

## Alexis Beingessner - May 22, 2015 - Rust 1.0.0

Rust tries to strike a hard dichotomy between code that is guaranteed to be memory-safe and code that isn't. It does this using the `unsafe` keyword. This keyword is the source of much anxiety and debate. This is because its meaning and semantics are not as well-defined as some would like (myself included). `unsafe` has no actual language semantics. This is not a mistake. This is *by design*. `unsafe` is for when you need to do something the compiler or language is unable to sufficiently reason about. That said, this means you can use `unsafe` however you want. Codegen has no notion of safety. All the compiler cares about is that `unsafe` is "handled", for a very trivial definition of handled.

## Trying to Define Unsafe

So what is `unsafe`? There are four primary places `unsafe` appears: on blocks, on functions, on trait declarations, and on trait implementations:

```rust
unsafe fn foo() { .. }

fn bar() { 
    unsafe {
        foo();
    }
}

unsafe trait Foo { .. }
unsafe impl Foo for Bar { .. }
```

Already our woes begin as `unsafe` means four completely different things:

* On functions, `unsafe` is declaring a function to be "unsafe" to call. Users of the function must check the documentation to determine what this means, and then have to write `unsafe` somewhere to identify that they're aware of the danger.
* On blocks, `unsafe` is declaring any unsafety from an "unsafe" operation to be "handled". Somehow. (more on this below)
* On trait declarations, `unsafe` is declaring that *implementing* it is an "unsafe" operation. Generally this is so that other unsafe code can "trust" it.
* On trait implementations, `unsafe` is declaring that the unsafety of the `unsafe` trait has been "handled". Roughly this means you believe you're upholding its trusted invariants.

Some examples of unsafe functions and traits:

* `slice::get_unchecked` will perform unchecked indexing, allowing memory safety to be freely violated.
* `ptr::offset` invokes undefined behaviour if it is not "in bounds" as defined by LLVM (god help you figuring out what that means...).
* `mem::transmute` reinterprets some value as having the given type, by-passing type safety in arbitrary ways.
* All FFI functions are `unsafe` because they can do arbitrary things. C being an obvious culprit, but generally any language can do something that Rust isn't happy about.
* `Send` is a marker trait (it has no actual API) that promises implementors are safe to send to another thread.
* `Sync` is a marker trait that promises that threads can safely share implementors through a shared reference. 

I put some terms in scare-quotes because "unsafe" and "handled" are, strictly speaking, up to your imagination. The compiler doesn't know or care. For functions, all the compiler cares about is that calls to `unsafe` functions are captured by an `unsafe` block or that the parent function is marked as `unsafe` itself, with the caveat that `main` can't be `unsafe`. For traits, all it cares about is that implementors have to write `unsafe`.

The Rust developers intend a very specific meaning: An `unsafe` function can violate memory safety. Interestingly, this is equivalent to C's "Undefined Behaviour" due to a circular interaction: violating memory safety causes Undefined Behaviour, and causing Undefined Behaviour can violate memory safety. 

Undefined Behaviour in Rust generally includes many of the classic C bogeymen (non-exhaustive):

* Accessing invalid memory (null ptr, dangling ptr, unallocated, uninitialized, freed)
* Causing a data race (not to be mistaken with a general race condition)

But includes a few Rust-specific (non-exhaustive):

* Aliasing a mutable pointer (&mut)
* Mutating something in a non-mutable slot (that isn't wrapped in an UnsafeCell)
* Transmuting between types that aren't `#[repr(C)]`
* Anything a library chooses to declare like invalid pointer offsets and non-utf8 strings

Things that *are* safe, but usually bad for a program to do (non-exhaustive):

* Arithmetic overflow (on a non-wrapping operation)
* Leaking memory 
* Failing to call a destructor
* Panicking
* Deleting the entire database and all the backups

Some people wish to broaden the formal definition of `unsafe` to include any kind of "danger" or to just mean "pay attention", but this risks watering down the meaning. What is dangerous is a matter of opinion. Some would argue anything short of total static verification of the program's inputs and outputs is dangerous. Certainly, there are industries where this could be a reasonable position to take. However this is not a problem Rust seeks to solve. We're largely interested in the problem of eliminating the specter of Undefined Behaviour from code that doesn't ask for it, while still giving the developer all the power and flexibility of C++. If `unsafe` is used to express any concerns one has with the API, it will be impossible to write *anything* safe, and we will become desensitized to it. `unsafe` is, ideally, something to respect.

Interestingly, `unsafe` is less about what it allows you to do, and more about what its absence *doesn't* let you do. The most basic guarantee of Safe Rust is as follows:

> It should be impossible to invoke Undefined Behaviour using only safe code.

Therefore a function that is safe makes a very bold claim: No matter what arguments you feed me, regardless of any other safe operations you perform, I will not cause Undefined Behaviour (unless I use `unsafe` internally and have a bug).

I find this claim particularly interesting because it is a *stateful* one. Not only must every possible choice of arguments be safe to use, but this must also hold under *any* state that *any other* safe code can produce. This gives the user of `unsafe` the dubious requirement of knowing *everything that every other safe API might do*. This includes all the fun little interactions between threads, unwinding, and destructors. It is also a property that can only be enforced by today's Rust *at the module boundary*.

As a concrete example, consider this mini implementation of an ArrayStack:

```rust
struct ArrayStack<T> {
    len: usize,
    cap: usize,
    ptr: *mut T, // Probably should be Unique<T>, but out of scope for this example
}


impl<T> ArrayStack<T> {
    pub fn new() -> Self {
        Vec { ptr: heap::EMPTY, len: 0, cap: 0 }
    }

    pub fn push(&mut self, elem: T) {
        if self.len == self.cap { self.realloc(); }
        unsafe {
            ptr::write(self.ptr.offset(self.len as isize), elem);
            // `checked` to guard against zero-sized types which will never OOM us. 
            // Ideally we should branch on the sizedness of `T` to make this check.
            self.len = self.len.checked_add(1).expect("len overflow"); 
        }
    }

    pub fn pop(&mut self) -> Option<T> {
        if self.len > 0 {
            unsafe {
                let elem = ptr::read(self.ptr.offset(self.len as isize - 1));
                self.len -= 1;
                Some(elem)
            }
        } else {
            None
        }
    }

}
```

This code is quite small, and as such it's fairly easy to "human verify" that the code is indeed correct and safe: no combination of calls in this API will produce UB (please gods of writing, let this be true). If I planned on using this code I would probably toss in some unit tests for some obvious corner cases and basic usage to be extra sure, though. 

> Full disclaimer: while writing this my first draft was wrong; I left out the `-1` in `pop`'s offset! Welcome to Unsafe Rust. :)

But let's say we add a new method:

```rust
fn make_some_room(&mut self) {
    self.len += 1; // *yawn*, I need some coffee...
}
```

As far as Rust is concerned, this function is totally safe. All we do is decrement an integer! However this has now violated the *implicit* contract that our apparently safe API implicitly relied on: that there are `len` initialized cells in the stack. Note that for *consumers* of our API, this wasn't a problem: the `len` field is private, and so they have no way to violate the contract. However we see that it's quite easy to violate the contract *internally* without any indication that we're doing something dangerous.

One response to this is fairly simple: mark the *fields* as `unsafe` to mutate. [And indeed there is a postponed RFC to this effect](https://github.com/rust-lang/rfcs/pull/80). If we accept and implement this RFC, then `make_some_room` would be `unsafe` and the unsafe statefulness would at least be alluded to. Of course this would in no way explain *what* the unsafe contract is. It just means we have to write `unsafe` more. One still has to properly document assumptions the code is making, and people still have to *read* those docs *and not screw up*. 

However this is just one case of unsafe contracts. Contracts might consist entirely of locals on the stack (e.g. don't do anything to invalidate `temp_ptr`). To get around this we could introduce a type with an unsafe constructor and unsafe `get` and `set`, but at some point one has to wonder if this becomes *worthwhile*. Are we preventing real bugs? Are we making other bugs *more likely*?

This gives rise to the alternative perspective: `unsafe` in a module means "you need to understand this module to correctly modify it". Basically, `unsafe` is a contaminant that infects the whole module. At the limit this interpretation means that you can actually play totally fast and loose with `unsafe` internally. You can mark private functions that are *really* unsafe as safe if you want, because *it doesn't matter and we can't do better*. This interpretation basically asserts that `unsafe` is only meaningful at a module's public API boundary. It also has the present-day advantage of being *usable*. The other interpretation requires new language features that may or may not be included for quite a while. And even then, it requires you to meticulously mark everything that matter as `unsafe`.

In the hopes of starting impromptu dance-off-street-fights, I will assert that the two groups are vicious rivals with sweet gang names and leather jackets. There's also definitely no middle ground. Definitely. Ultimately, though, both of these interpretations are valid. We have few community guidelines for how to use `unsafe` short of the following:

* Public APIs should be usable in a 100% safe way
* It's okay to have some unsafe APIs for low-level control, but those should be *optimizations*
* Traits shouldn't be `unsafe` just because it enables some optimizations (e.g. you can't currently "trust" implementors of Ord to have a total ordering; unsafe code must currently guard against this)
* `unsafe` should not be used to lint against "tricky" behaviours that are otherwise safe (`mem::forget` being the champion of this stance)

## Measuring and Controlling Unsafe

A related problem is that of *minimizing* `unsafe`. At a high level, this is clearly desirable: the less you write unsafe, the less places you need to worry about it. However as we've seen this might just be a lie where your "little" unsafe code actually relies heavily on contracts freely influenced by your safe code. API and abstraction boundaries generally mean *actually* not worrying about any `unsafe` on the other side. So at very least minimizing the number and size of modules that use `unsafe` seems desirable. However internally minimizing the number of lines or functions that are marked `unsafe` is a more dubious goal. For one, any metric you use for tracking unsafe usage within a module will be heavily influenced by your style and preference:

```rust
// "The whole routine is unsafe!"
fn do_stuff_1() {
    unsafe {
        step_1();
        step_2(); // the only actual "unsafe" function
        step_3();
    } 
}

// "The implementation has tons of unsafe functions!"
fn do_stuff_2() {
    step_1();
    unsafe { 
        step_2(); // actually relies on state from step_1 and step_3
    }
    step_3();
}

// "The implementation consists only of safe functions, with very little `unsafe`!"
fn do_stuff_3() {
    step_1();
    // just the code for step 2 inline'd
    let x = 1;
    ...
    unsafe {
        ...
    }
    ...
    unsafe {
        ...
    }
    ...
    frob();
    step_3();
}
```

The other problem is that *genuinely* reducing sources of unsafety ("You can't X with this design")may negatively impact other important properties of the code like maintainability and understandability. You may just be shuffling around the problem (instead of ensuring that "x isn't 0", you now have to "make sure that ZeroGuard can't outlive the ZeroShield"). In the extreme it may increase the rate of other bugs!

The best advice I can really give on writing `unsafe` code is *try not to be clever*. The most non-clever thing is of course to just use some safe API. Of course this may be impossible or inappropriate based on your usecase (e.g. you are *writing* that safe API, the overhead is too high, or it hides details that matter to you). So the next best thing is to just have runtime checks for all the bad stuff. `assert!(x != 0, "can't have zero frobs in the blarber")` can go a long way to keeping you sane (and is genuinely self documenting!). 

The *most* clever thing you can do is to try to encode stuff in the type system. This is a tricky game that I am all too guilty of playing. It's tempting, because if you can pull it off it's super cool! You can say your implementation is statically safe with no overhead (assuming the optimizer can figure out what the heck you're doing)! Unfortunately, it incurs a heavy mental overhead. At least, that's been my experience. As types and tricks proliferate, relatively simple logic becomes more obtuse. Maybe *you* understand it, but will the next person? God-forbid these crazy things spill out into your public API! I've also mostly seen it guarding against *bugs that weren't happening*. This is like premature optimization... but for safety. It also all falls apart if someone unwittingly makes a change that opens a hole in your design (is your type system safe against changes to the types?)!

## Reflections on Unsafe

So at this point some of you are probably wondering something along the lines of "if `unsafe` has so little meaning, and everything is built on top of `unsafe`, what exactly is Rust gaining us over C or C++?". This is an important question to ask! The most important difference, I think, is that of *defaults*. Rust is safe by default. You opt in to unsafety when you call clearly demarcated functions. Even then, you don't opt into *all* unsafety, you opt into *specific* unsafety. When you call `get_unchecked` on an array you don't suddenly have to worry about use after use-after-free or the array being a null pointer. You only have to worry about that index being in-bounds. In C and C++ the unsafety is ever-present. Even the modern abstractions like C++'s `std::unique_ptr` can be accidentally coerced into Undefined Behaviour without even thinking about it. 

But one thing is for certain: Rust is not perfect. It's trying to be practical. It does not statically guarantee that your program is correct. It is not statically guaranteed to be correct *itself*. All we have in Rust is a bunch of stuff that we think helps us avoid bugs and be productive, and you can feel free to tune that to your desires. Disagree with me about encoding things in types? Go nuts! [Push static verification to its limits](https://github.com/epsilonz/shoggoth.rs)! Really prefer direct code that "does what it says"? Sure thing! Here's your raw C-like pointers and bitshifts. But you're always coming from a place of safety, and always on your own terms. You're not trying to patch over the unsafety the language drops in your lap. 

Anyway, this is just *my feels* on what's up with `unsafe` in Rust. For the next couple months I'm going to be poking at what `unsafe` "means" and how it should be used. I'll also be working to solidify the -- *ahem* -- *fuzzier* rules around Safe Rust (e.g. how exactly pointer aliasing is supposed to work). If you have Opinions or some Key Insights into the whole manner, I'd love to hear them! 
