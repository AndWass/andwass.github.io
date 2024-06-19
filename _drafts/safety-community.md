---
layout: post
title: "Safety and community"
category: rust
tags: rust safety c++
---

Rust is by many considered a memory and thread-safe programming language. It has been mentioned as
an alternative to C and C++ in reports from for instance [CISA] and the [White House].
In these same reports there has been a general recommendation to move away from memory-unsafe
languages, of which C and C++ has been specifically called out. This has caused a bit of a stir
in the C++ community with rebuttals posted by for instance [Bjarne Stroustrup], blogs posts by
people like [Herb Sutter] and discussions on places like Reddit and Hacker news.

[CISA]: https://media.defense.gov/2023/Dec/06/2003352724/-1/-1/0/THE-CASE-FOR-MEMORY-SAFE-ROADMAPS-TLP-CLEAR.PDF
[White House]: https://www.whitehouse.gov/wp-content/uploads/2024/02/Final-ONCD-Technical-Report.pdf
[Bjarne Stroustrup]: https://www.infoworld.com/article/3714401/c-plus-plus-creator-rebuts-white-house-warning.html
[Herb Sutter]: https://herbsutter.com/2024/03/11/safety-in-context/

Parts of these discussion revolves around how C++ has no clear distinction between code
that can cause undefined behaviour and code which can't, or how some default behaviours in C++ makes it easy
to invoke undefined behaviour. Is that all there is to it though? Or is there something else at play?

One thing I think is missing in these discussions is the huge role that the community plays. I would even go so
far as to say that Rusts safety properties are not technically enforced, but are enforced by the community.
To give a small example of what I mean, let's consider the following Rust function:

```rust
fn deref_i32_ptr(ptr: *const i32) -> i32 {
    unsafe { *ptr }
}
```

This is a technically valid function, no language rule prevents me from writing this
function, or using it. Using it with an invalid pointer is wrong though, so in Rust it is not a
socially acceptable function. The Rust community will tell you that it is unsound since `deref_i32_ptr`
isn't marked `unsafe`. I should also note that `cargo clippy` does warn on this function.

So if it is technically possible to create "safe" functions that can invoke undefined behaviour,
what is the point? Why have the `unsafe` keyword at all? The `unsafe` keyword is a marker for
"here be dragons" and, **if used correctly**, it makes it much easier to distinguish between sound and potentially unsound code.
The `unsafe` keyword is really what enables Rust to reason about the soundness of a function or crate at all without
having to reason about the soundness of the entire function or crate.

Contrast this with C and C++, where there is no such marker. In C and C++ there is generally much less talk about
soundness, at least in my experience, and I believe the missing syntax marker is the root cause of this. It is much harder to find the spots that requires extra attention so you get sweeping guidelines like "pointers are nullable, use references
for never-null. Never pass ownership via pointers or reference" but still no way to mark those spots in the source file.
Which means it is much harder to distinguish between sound/potentially-unsound functions.

## Community as safety foundation

For Rust, the important bit in the text above is **if used correctly**. `unsafe` relies on crate authors to buy
in to the sound/potentially-unsound, and safe/unsafe, division. So far that has succeeded.
But as the Rust community grows there is a risk that nonchalance creeps in and unsoundness takes root.
If that happens Rust could very well end up being lumped together with C and C++ as an unsafe language,
because at the very core the Rust safety properties are upheld by the community.




