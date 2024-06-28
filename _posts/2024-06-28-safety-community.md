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

This usage of `unsafe` gives us a very nice property that if your code, and all your dependencies, is written without using `unsafe`
then your program cannot exhibit undefined behaviour. And even if you or your dependencies use `unsafe` code, as long as that code
is correct your program still cannot exhibit undefined behaviour.

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
socially acceptable function. The Rust community will tell you that it is unsound since `deref_ptr`
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

Within Rust these restrictions are enforced entirely by the community though. Is there room for improvement though?
And can this pose a long-term risk of trust and safety erosion?

## Growing community == erosion?

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

Cargo, and the various tools that can be run by a simple `cargo <tool>` invocation is awesome! It makes it dead simple to
run static analysis tools, and many other useful tools. But when it comes to safety the best
tool is the compiler. Because that will always run. Shifting responsibility to static analysis tools is something that
C and C++ has historically done as well, but I think that might be slowly changing as well. There are pushs towards
moving diagnostics to the compiler instead of relying on additional tools.

I am also wondering if `crates.io` could play a role to increase the safety margins as well? Together with existing or new tools?
I only have very vague thoughts on the subject but I think there is value in figuring out which crates are in most need
of being (continuesly) audited, due to them needing a large amount of unsafe code. And the more they can be audited automatically
on a new release to `crates.io` the better. Could there be other useful metrics to guide users in determining if a crate is sound
or not which could be displayed?

## What about C/C++

I will only talk about C++ since that is what I am following the most. Recently there has been an upswing in the safety and security
talk within the C++ community. Almost all conferences has a talk that talks about safety, and there are early papers being discussed
in the ISO meetings that attempts to tackle the safety and security issues.

I am a bit worried though that the committee is seeking a purely technical solution to a problem that needs community buy-in and
community tools as well. There are tools available today that increases the safety and security, but they must be used. And the same
would go for any technical solution; it must be used.

## Conclusion

Rusts safety story relies, at least partially, on the community adhering to the praxis that safe code can never exhibit undefined behaviour.
As the community grows I think there is a risk that this adherence might gradually decline. If it declines enough it might spell trouble
for Rusts reputation as a safe language.

To combat this I think it would be interesting to see if there are any unsafe patterns that should error at compile time. Utilizing `crates.io`
as an auditing point and figuring out useful metrics to guide users could also be interesting.