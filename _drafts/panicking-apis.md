---
layout: post
title:  "Panicking APIs"
category: rust
---

Some functions in Rust will panic if a precondition isn't met. Examples of this includes indexing
into a `Vec` outside its bounds

```rust
let vec = Vec::new();
vec[0]; // Panics
```
or unwrapping an empty `Optional`

```rust
let opt = None;
let val = opt.unwrap();
```

In my opinion these are perfectly reasonable choices, but why do I think they are reasonable?
And why do I consider other choices less reasonable?