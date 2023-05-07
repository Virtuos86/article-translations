##Вступление от автора

OK, I gave it a shot by just rewriting each section in order. But honestly, I'm still not happy with it. (Probably I wouldn't be happy with the book either.)

In short I think a dedicated, freestanding section on borrowing and reborrows is needed to not be sort of woefully incomplete or misleading. But guides like this are aimed at letting readers "just get started", and I have yet to see something that strikes the balance of

* summarizing "everything"
* being brief / not overwhelming
* not being inaccurate

At least, I haven't figured out a way. Maybe it just has to be along the lines of "look here's a taste so you're not blindsided, but just get started for now and come back to this tutorial (link) when you hit a wall or just want to understand more." I.e. give up on summarizing "everything" (and own up to "there's too much [detail/nuance/just too much] to explain everything here").

---

I sort of suspect the diff is more elucidating than the before or the after 😅.

Anyway here's my shot at rewriting 10.3.

##[Validating References with Lifetimes](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#validating-references-with-lifetimes-1)

Every value in Rust has a *scope* - a point in the code where the value is created, and a point in the code where the value is destroyed. Every reference in Rust has a *lifetime*, which is a region where the reference is valid. When inferred within a function body, this region is typically the spans of code where the reference is used. The lifetime of the reference obviously needs to be shorter than the scope of the value it points to, as otherwise the reference would point to freed memory -- and such a reference cannot be valid.

In addition to the value going out of scope, there are other situations which can make a reference invalid. Moving the value would also cause a reference to dangle, for example. A `&mut` is an exclusive reference, so taking a `&mut` to the value also invalidates any pre-existing borrows. This constraint avoids undefined behavior such as data races, and related problems such as iterator invalidation.

The Rust compiler has a *borrow checker* which infers the lifetimes of references based on their use and any other annotations or constraints in the code (which we explore below). It also checks these inferred lifetimes against the scopes, moves, and other uses of values. If there are any constraints that cannot be met, or any uses of lifetimes that conflict with the use of values, they are reported as errors.

> Notes on this section
> 
> Mainly here I make a distinction between scopes and lifetimes. I also mention a couple of common ways to invalidate a reference besides things going out of scope. A constraint violation would be a useful code addition, which could be as simple as "you said you'd return a `&'static` but didn't".
> 
>> Just as rustc infers the type of many of our parameters, in most cases Rust can infer the lifetime of a reference (usually from when it is created until it's last use in a function). Just as we can explicitly annotate a variable's type, we can also explicitly annotate the lifetime of a reference in cases where the compiler can't infer what we want.
> 
> For intra-body lifetimes, there's no way to directly annotate the inferred lifetimes as they have no names. You have to use tricks like the helper functions at the top of this thread. It's a niche technique and usually can just make things more restrictive.

##[Preventing Dangling References with Lifetimes](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#preventing-dangling-references-with-lifetimes-2)

One of Rust's memory safety guarantees is that references never dangle -- never point at memory that has been freed. Here's an example:

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;           // --+ 'r
                          //   |
    }                     // --💥--------------- `x` is freed
                          //   |
    println!("r: {}", r); // --+ last use of 'r
}
```

This won't compile. The variable `r` is used after the inner block, but it's a reference to `x` which will be dropped when we reach the end of the inner block. After we reach the end of that inner block, `r` is now a reference to freed memory, so Rust's *borrow checker* won't let us use it.

More formally, we can say that the scope of `x` is shorter than the lifetime of `r`, which we've labeled as `'r` in the comments of this example (a strange name, but this is actually a bit of foreshadowing). The borrow checker sees that `x` goes out of scope before the end of `'r`, and the borrow checker won't allow this.

This version fixes the bug:

```rust
fn main() {
    let x = 5;
    let r = &x;           // --+-- 'r
                          //   |
    println!("r: {}", r); // --+ last use of 'r
}                         // ------------------ `x` is freed
```

Here `x` goes out of scope after `'r`, so `r` can be a valid reference to `x`.

> Notes on this section
> 
>> The variable `r` is in scope for the entire `main()` function,
> If you don't take a reference to the `r` or entwine its lifetime with something else, it's scope doesn't really matter, because the destruction of a reference is a no-op.

##[Generic Lifetimes in Functions](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#generic-lifetimes-in-functions-3)

Now for an example that doesn't compile, for what might not at first be obvious reasons. We're going to pass two string slices to a `longest()` function, and it will return back whichever is longer:

```rust
fn main() {
    let string1 = String::from("abcd");
    let string2 = "xyz";
    let result = longest(string1.as_str(), string2);
    println!("The longest string is {}", result);
}

// This doesn't work!
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

If we try to compile this, we get an error. When the Rust compiler checks a call to a function, it doesn't look at the contents of the function, only at the signature; the signature is a contract that both the function caller and the function body must uphold. Additionally, when you elide the lifetimes of input arguments as in this example, they are interpreted to be two distinct generic input lifetimes.

So the root of the problem here is that in this function, it is ambiguous from the signature alone whether the function is meant to return `x` or `y` or potentially either one. Since these have different lifetimes, the compiler considers the signature to be unacceptably ambiguous. Consider this example of calling this function:

```rust
fn main() {
    let string1 = String::from("abcd");
    let result;
    {
        let string2 = String::from("uvwxyz");
        result = longest(&string1, &string2);
    }
    // This shouldn't compile if a reference to `string2` can be
    // returned.  (But how can we convey that possibility to Rust?)
    println!("The longest string is {}", result);
}
```

Here if `longest()` returned the reference to `string1`, it would still be valid by the time we get to the `println!`, but if it returned the reference to `string2` it would not.

We need a way to let the compiler -- and other humans who read our code! -- whether our function returns a reference with the same lifetime as the first argument, the second argument, or potentially either argument.

> Notes on this section
> 
>> When the Rust compiler checks a call to a function, it doesn't look at the contents of the function, only at the signature.
> This is true, [mumble mumble except return position `impl` trait] but the example doesn't compile even if we never call it. The API (signature) of a function is a contract that the compiler enforces both on callers, but also on the function body. The API itself is just considered inherently ambiguous here. I don't think it gets as far as borrow-checking, or rather if it does, it's a "we'll just assume `'static` I guess so we can spew more errors that might or might not make sense" situation.

##[Lifetime Annotation Syntax](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#lifetime-annotation-syntax-4)

We fix this problem by telling the compiler about the relationship between these references. We do this with *lifetime annotations*. Lifetime references are of the form `'a`:

```rust
&i32        // a shared reference with an anonymous or inferred lifetime
&'a i32     // a shared reference with an explicit lifetime
&mut i32    // an exclusive reference with an anonymous or inferred lifetime
&'a mut i32 // an exclusive reference with an explicit lifetime
```

##[Lifetime Annotations in Function Signatures](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#lifetime-annotations-in-function-signatures-5)

We can declare a lifetime annotation for a function in much the same way we add generic types. The lifetime annotation must start with a `'`. Typically they are single characters, much like generic types. And just like generic types, these will be filled in with a real lifetime for each call to the function.

We can fix the `longest()` function in our previous example with:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

`longest` a generic function (the syntax is similar for a good reason). We're saying here that to call this function, there exist some lifetime which we're going to call `'a`, and the input references `x` and y both have the lifetime `'a`, and we're going to return a `&str` with that same lifetime. Under this signature, it would be valid to return either `x` or `y`.

It's important to note that it's still possible to pass variables to the function which don't have the same lifetime at the call site. This is possible because shared references can be copied, and can also coerce from a long lifetime to a short lifetime. When a function takes a shared reference as an argument, this happens automatically if required. For example, when calling `longest`, a lifetime no longer than the shorter of the two argument lifetimes will be inferred. Depending on how the return value is used, a lifetime that is shorter than *both* arguments could be inferred as well.

The net result is that the return value of `longest()` will live no longer than the shorter lifetime of the references passed at the call site, and thus could be derived from either input. When the rust compiler analyzes a call to `longest()` it can now mark it as an error if the return value is used in a way incompatible with this constraint.

Returning to this example:

```rust
fn main() {
    let string1 = String::from("abcd");
    let result;
    {
        let string2 = String::from("xyz");
        result = longest(&string1, &string2);
    }
    // This doesn't compile!
    println!("The longest string is {}", result);
}
```

Here the compiler now knows that the return value of `longest()` can only be as long as the shorter of `&string1` and `&string2`, so it knows that the use of `result` in the `println!` macro is invalid -- `&string2` cannot be valid at that location.

> Notes on this section
> 
> Rewritten to highlight that, yes, the lifetimes really are the same, and that this works due to lifetime shortening coercion and copy. Still missing: Explaining reborrows for the `&mut` case.

##[Thinking in Terms of Lifetimes](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#thinking-in-terms-of-lifetimes-6)

The way we annotate lifetimes depends on what the function is meant to do. If we changed `longest()` to only ever return the first parameter, we could annotate the lifetimes as:

```rust
fn longest<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

This tells rustc that the lifetime of the return value is the same as the lifetime of the first parameter.

The lifetime of the return value will generally have the same annotation as at least one of the input parameters (or be `'static`, which we'll discuss in a moment).

The concrete lifetimes chosen for function lifetime parameters are determined by the call site and as a result, are always longer than the function body. Therefore you cannot create references to local variables with the lifetime of a function parameter at all, since local variables drop at the end of the function. And naturally you cannot return references to local variables either, as they would immediately dangle.

> Notes on this section
> 
>> The lifetime of the return value must have the same annotation as at least one of the parameters
> It can be independent in function signatures (but not function pointer types or `dyn Fn` types). Such cases are pretty niche and many of them could be replaced by `'static`.

##[Lifetime Annotations in Struct Definitions](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#lifetime-annotations-in-struct-definitions-7)

So far all the structs we've created in this book have owned all their types. If we want to store a reference in a struct, we can, but we need to explicitly annotate it's lifetime. Just like a function, we do this with the generic syntax:

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().expect("Could not find a '.'");
    let i = ImportantExcerpt {
        part: first_sentence,
    };
}
```

Again, it's helpful to think about this like we would any other generic declaration. When we write `ImportantExcerpt<'a>` we are saying "there exists some lifetime which we'll call `'a`" -- we don't know what that lifetime is yet, and we won't know until someone creates an actual instance of this struct. When we write `part: &'a str`, we are saying "when someone reads this ref, it has the lifetime `'a`" (and if someone later writes a new value to this ref, it must have a lifetime of at least `'a`). At compile time, the compiler will fill in the generic lifetimes with real lifetimes from your program, and then verify that the constraints hold.

Here this struct has only a single reference, and so it might seem odd that we have to give an explicit lifetime for it. However, having to declare all lifetime parameters avoids some ambiguities and footguns in the same way having to declare all variables does. Additionally, knowing whether or not a struct may borrow something is important information for those reading your code. It would be easy to miss that a field is a reference if the lifetimes could be completely elided in the definition.

**Info**: The original ["The Rust Programming Language"](https://doc.rust-lang.org/stable/book/ch10-03-lifetime-syntax.html#lifetime-annotations-in-struct-definitions) here said that "this annotation means an instance of `ImportantExcerpt` can't outlive the reference it holds in its part field," but I found that not a helpful way to think about this -- of course a struct can't outlive any references stored inside it. I found [this answer on Stack Overflow](https://stackoverflow.com/questions/27785671/why-can-the-lifetimes-not-be-elided-in-a-struct-definition/27785916#27785916) to be a lot more illuminating.

Here's an example where a struct requires two different lifetime annotations (borrowed from [this Stack Overflow discussion](https://stackoverflow.com/questions/29861388/when-is-it-useful-to-define-multiple-lifetimes-in-a-struct/66791361#66791361) which has some other good examples too):

```rust
struct Point<'a, 'b> {
    x: &'a i32,
    y: &'b i32,
}

fn main() {
    let x = 1;
    let v;
    {
        let y = 2;
        let f = Point { x: &x, y: &y };
        v = f.x;
    }
    println!("{}", *v);
}
```

The interesting thing here is that we're copying a reference out of a struct and then using it after the struct has been dropped. This is okay because in this case the lifetime of the reference is longer than the scope of the struct. This is only possible because the `x` and `y` fields have distinct lifetimes (because `Point` has two lifetime parameters). If we had defined Point with only one lifetime parameter:

```rust
struct Point<'a> {
    x: &'a i32,
    y: &'a i32,
}
```

[Then you will get a borrow check error](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=3319b70c6b2965acc82d39b994f36176), as the lifetime of `Point` cannot be greater than the scope of the inner block (due to taking a reference to `y`).

**tip**: Similar to trait bounds, we can add a [lifetime bound](https://doc.rust-lang.org/reference/trait-bounds.html#lifetime-bounds) to a lifetime annotation in a function or a struct.

```rust
struct Point<'a, 'b: 'a> {
    x: &'a f32,
    y: &'b f32,
}
```

You can read `'b: 'a` as "`'b` outlives `'a`", and this implies that `'b` must be at least as long as `'a`. There are very few cases where you would need to do such a thing, though.

> Notes on this section
> 
> I rewrote the part about not having lifetime declarations to a POV you may or may not agree with. Question though: why didn't you include the same argument about the compiler just figuring things out for type parameters?
> I rewrote the last example somewhat too.

##[Lifetime Elision](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#lifetime-elision-8)

Way back in [chapter 4](https://jasonwalton.ca/rust-book-abridged/ch04-ownership), we wrote this function:

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}
```

How come this compiles without lifetime annotations? Why don't we have to tell the compiler that the return value has the same lifetime as `s`? Actually, in the pre-1.0 days of Rust, lifetime annotations would have been mandatory here. But there are certain cases where Rust can now work out the lifetime on it's own. We call this *lifetime elision*, and say that the compiler *elides* these lifetime annotations for us.

What the compiler does is to assign a different lifetime to every reference in the parameter list (`'a` for the first one, `'b` for the second, and so on...). If there is exactly one input lifetime parameter, that lifetime is automatically assigned to all output parameters. If there is more than one input lifetime parameter but one of them is for `&self`, then the lifetime of `self` is assigned to all output parameters. Otherwise, the compiler will error.

In the case above, there's only one lifetime that `first_word` could really be returning; if `first_word` created a new `String` and tried to return a reference to it, the new `String` would be dropped when we leave the function and the reference would be invalid. The only sensible reference for it to return comes from `s`, so Rust infers this for us. (It *could* be a static lifetime, but if it were we'd have to explicitly annotate it as such.)

[You can find more details and examples in the Reference.](https://doc.rust-lang.org/reference/lifetime-elision.html)

##[Lifetime Annotations in Method Definitions](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#lifetime-annotations-in-method-definitions-9)

We can add lifetime annotations to methods using the exact same generic syntax we use for generic structs:

```rust
impl<'a> ImportantExcerpt<'a> {
    fn level(&self) -> i32 {
        3
    }

    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

Here `'a` refers to the lifetime of the struct itself, but thanks to lifetime elision, in `announce_and_return_part()`, the return value is automatically given the same lifetime as `self`, so we don't actually have to use it.

If the return will always be a copy or sub-borrow of your borrowed field, however, it may be more flexible for consumers of the method to receive the longer lifetime:

```rust
fn announce_and_return_part(&self, announcement: &str) -> &'a str {
        println!("Attention please: {}", announcement);
        self.part
    }
```

> Notes on this section
> Extended because in my experience, if you can return the longer lifetime, it's better for consumers of your type. This deserves an example but I haven't created one yet.

[The Static Lifetime](https://users.rust-lang.org/t/scopes-and-lifetimes/93222/13#the-static-lifetime-10)

[I didn't have any changes after here]
