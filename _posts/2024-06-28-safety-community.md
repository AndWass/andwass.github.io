---
layout: post
title: "Safety and community"
category: rust
tags: rust safety c++
---

When talking about alternatives to C and C++, Rust is often mentioned. Rust is considered a safer alternative
that can prevent both memory and threading issues. In discussions it is fairly common to hear that safe Rust
does not exhibit undefined behaviour, which then leads to safer and more correct code. As an escape hatch
Rust has the `unsafe` keyword, with the intention that `unsafe` is used to shift responsibility from the
compiler to the programmer. The programmer is responsible for ensuring that no undefined behaviour exists in the
unsafe parts. If used correctly, safe Rust can then build safe abstractions around the `unsafe` portions. The
property of no undefined behaviour in safe Rust is then upheld. If the programmer fails to create a fully safe wrapper, so that safe Rust
can cause undefined behaviour, the code is said to be unsound.

This is in contrast to C and C++, where there is no such marker or distinction, and so it becomes much harder to
isolate the code that can cause undefined behaviour, and find code that could potentially cause undefined behaviour.
When discussing C and C++ code the discussion does not mention soundness in the same way. It is generally
more acceptable for code to exhibit undefined behaviour in some corner cases, as long as it is documented.

One thing I think is missing in these discussions is the huge role that the community plays. I would even go so
far as to say that Rusts safety properties are not technically enforced, but are enforced by the community.
To give a small example of what I mean, let's consider the following Rust function:

```rust
pub fn deref_ptr(ptr: *const i32) -> i32 {
    unsafe { *ptr }
}
```

From the language perspective this is valid code, no language rule prevents me from writing this
code, or using it, or even publishing to `crates.io`. Using it with an invalid pointer is wrong though, so in Rust it is not a
socially acceptable function. The Rust community will tell you that it is unsound since `deref_i32_ptr`
isn't marked `unsafe`. I should also note that `cargo clippy` does warn on this function, but change it to

```rust
fn deref_ptr_priv(ptr: *const i32) -> i32 {
    unsafe { *ptr }
}

pub fn deref_ptr(ptr: *const i32) -> i32 {
    deref_ptr_priv(ptr)
}
```

and `clippy` no longer complains.

I would argue that in C, and to a certain extenct C++, the equivalent functions are generally considered acceptable
by the community, as long as any pre-conditions are mentioned in the documentation. The lack of an `unsafe` equivalent keyword
really hinders the community from imposing restrictions.

Within Rust these restrictions are enforced entirely by the community though.

## What does this mean for Rust?

The `unsafe` keyword and its intended usage relies on crate authors to buy
in to the division between safe and unsafe code and the requirement that safe code never exhibits undefined behaviour.
So far that has generally succeeded.
But as the Rust community grows there is a risk that nonchalance creeps in and unsoundness takes root.
For instance I have personally observed the naked dereference of pointers, equivalent to the code above, in crates
(I won't link to them).

I think the likelihood of this increases as the Rust community grows, and in worst case scenario
Rust could end up being lumped together with C and C++ as an unsafe language, because at the very
core the Rust safety properties are upheld by the community.

One way to combat this is to shift left, make the compiler enforce rules that today is enforced by the community.
This is harder than it looks though, because there are instances when it is perfectly safe for a safe function to
dereference a pointer. But if we can identify unsafe patterns that are allowed today, but are never truly safe, and make the
compiler error on those, we are left with a higher safety margin.

### What about other tools

Cargo, and the various tools that can be run by a simple `cargo <tool>` invocation is awesome, it makes it dead simple to
run static analysis tools and enables the usage of a whole slew of different tools. But when it comes to safety the best
tool is the compiler. Because that will always run. Shifting responsibility to static analysis tools is something that
C and C++ has historically done as well, but I that might be slowly changing as well.

I am wondering if `crates.io` could play a role to increase the safety margins as well? Together with existing or new tools?
I only have very vague thoughts on the subject, but I think there is value in figuring out which crates are in most need
to be (continuesly) audited, due to them needing a large amount of unsafe code. And the more they can be audited automatically
on a new release to `crates.io` the better.

## What about C/C++

I will only talk about C++ since that is what I am following the most. Recently there has been an upswing in the safety and security
talk within the C++ community. Almost all conferences has a talk that talks about safety, and there are early papers being discussed
in the ISO meetings that attempts to tackle the safety and security issues.

I am a bit worried though that the committee is seeking a purely technical solution to a problem that needs community buy-in and
community tools as well. There are tools available today that increases the safety and security, but they must be used. And the same
would go for any technical solution; it must be used.

