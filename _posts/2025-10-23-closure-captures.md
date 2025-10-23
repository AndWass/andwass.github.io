---
layout: post
title:  "Closure captures"
category: rust
---

There have been some discussions on how to make closure captures more ergonomic. For a background
on motivation and a discusson on what I perceive to be the current way forward I recommend reading
[this](https://smallcultfollowing.com/babysteps/blog/2025/10/07/the-handle-trait/) blog post.

In various Reddit discussions the thought on explicit captures came up, and a blog post discussing some aspects of this
was published [here](https://smallcultfollowing.com/babysteps/blog/2025/10/22/explicit-capture-clauses/).

In this post I want to explore how C++ handles this same issue, and see if this could be solved in a similar manner
in Rust.

# The issue(s)

Rusts closures are pretty much all or nothing, you either capture everything by reference, or you capture everything by move. You cannot grant access to values,
you effectively have access to everything in the outside scope.

If you want to capture some things by reference and some things by move you need to set this up outside of the closure.
If you want to clone things into the closure you can only do so by first clone, and then move into the closure.

```rust
let some_value = Arc::new(something);

// task 1
let _some_value = some_value.clone();
tokio::task::spawn(async move {
    do_something_with(_some_value);
});

// task 2
let _some_value = some_value.clone();
tokio::task::spawn(async move {
    do_something_else_with(_some_value);
});
```

If you need to clone a lot of things into the closure you are left with in a very unergonomic situation

```rust
let _some_a = self.some_a.clone();
let _some_b = self.some_b.clone();
let _some_c = self.some_c.clone();
let _some_d = self.some_d.clone();
let _some_e = self.some_e.clone();
let _some_f = self.some_f.clone();
let _some_g = self.some_g.clone();
let _some_h = self.some_h.clone();
let _some_i = self.some_i.clone();
let _some_j = self.some_j.clone();
tokio::task::spawn(async move {
  	// do something with all the values
});
```

And given that there is no way to control what values a closure can make use of, it is easy to accidentally
refer to the wrong values from inside a closure.

## Requirements

So these issues gives us these requirements that we must fulfill:

  1. It shall be possible to easily clone values into a closure.
  2. It shall be possible to restrict the set of outside values that a closure can make use of.

I also think that the following requirement should make it to the list:

  * It shall be possible to switch between full control, and ergonomic cloning, in different parts in the same source file.

The last requirement is based on the fact that context matters. In some part of the program it might be ok to clone a `Vec` or a `String`
or whatever else big datastructure you have. In other parts this must never be done.

# Rust vs C++

Incidentally this is one area where I think C++ got things exactly right.

Lets look at various scenarios and see how C++ and Rust compares.

<table>
  <tr>
    <th>Rust</th>
    <th>C++</th>
    <th>C++ alt.</th>
  </tr>
  <tr>
    <td>
      <pre>
let mul_2 = |x| x*2;
      </pre>
    </td>
    <td>
      <pre>
auto always_2 = [](int x) { return x*2; };
      </pre>
    </td>
  </tr>
  
  <tr>
    <td>
      <pre>
let mut y = 3i32;
let mut add_one = || y += 1;
      </pre>
    </td>
    <td>
      <pre>
int y = 3;
auto add_one = [&y] { y += 1; };
      </pre>
    </td>
    <td>
      <pre>
int y = 3;
auto add_one = [&] { y += 1; };
      </pre>
    </td>
  </tr>
  
  <tr>
    <td>
      <pre>
let mut y = 3i32;
let mut add_one = move || y += 1;
      </pre>
    </td>
    <td>
      <pre>
int y = 3;
auto add_one = [y] mutable { y += 1; };
      </pre>
    </td>
    <td>
      <pre>
int y = 3;
auto add_one = [=] mutable { y += 1; };
      </pre>
    </td>
  </tr>

  <tr>
    <td>
      <pre>
let some_arc = Arc::new(something);
let _some_arc = some_arc.clone();
let do_something = move || do_something_impl(_some_arc);
      </pre>
    </td>
    <td>
      <pre>
auto some_data = std::make_shared<something>();
auto do_something = [some_data] { do_something_impl(some_data); };
      </pre>
    </td>
    <td>
      <pre>
auto some_data = std::make_shared<something>();
auto do_something = [=] { do_something_impl(some_data); };
      </pre>
    </td>
  </tr>
  
</table>

We can see that C++ gives both full control, and shorthands. Essentially in this case C++ allows for the best of both worlds.

In terms of our requirements we tick both boxes....almost.

## This is weird

In C++, the `this` pointer makes things more awkward.

```c++
#include <cstdio>

struct Data {
  int x = 5;

  auto get_closure() {
    return [=]() {
      x += 1;
    };
  }
};

int main() {
    Data d;
    auto clos = d.get_closure();
    std::printf("%d\n", d.x);  // prints 5
    clos();
    std::printf("%d\n", d.x);  // prints 6
}
```

In the above example the implicit `this` pointer gets captured by value, but since it is a pointer, `x` is actually referred to by reference.
That is why `d.x` gets modified within `clos()`, even though we captured everything by value.

Now C++ has deprecated implicit capture of `this` when using the `[=]` capture form, but it still makes `this` slightly weird.

To solve this in C++ one has to do the following:

```c++
#include <cstdio>

struct Data {
  int x = 5;

  auto get_closure() {
    // Capture this->x and assign it to the closure-local variable x.
    return [x=x] mutable {
      x += 1;  // Only the closure-local variable x is modified.
    };
  }
};

int main() {
    Data d;
    auto clos = d.get_closure();
    std::printf("%d\n", d.x);  // prints 5
    clos();
    std::printf("%d\n", d.x);  // prints 5
}
```

# C++ approach in Rust

So could we do the same approach in Rust as C++? Lets play with the thought and see where it brings us.

This would be an empty closure that cannot refer to variables in the outer scope. Doing so would become a compiler error:
```rust
let x = || 5;
// Could become
let x = []|| 5;
```

Here we can either explicitly control the captures, or capture whatever we refer to by ref:
```rust
let my_vec = Vec::new();

let x = || my_vec.len();
// Could become
let x = [&my_vec]|| my_vec.len();
// alt
let x = [&]|| my_vec.len();
```

Again, we can either explicitly control the captures, or capture whatever we refer to by move:
```rust
let my_vec = Vec::new();

let x = move || my_vec.len();
// Could become
let x = [my_vec]|| my_vec.len();
// alt
let x = [=]|| my_vec.len();
```

Same for clone:
```rust
let my_vec = Vec::new();

let _my_vec = my_vec.clone();
let x = move || _my_vec.len();
// Could become
let x = [+my_vec]|| my_vec.len();
// alt
let x = [+]|| my_vec.len();
```

We can even mix and match:
```rust
let my_vec = Vec::new();
let my_string = String::new();
let my_arc = Arc::new(whatever);
let my_ref = &some_var;

// All of this
let _my_string = &my_string;
let _my_arc = my_arc.clone();
let x = move || something(my_vec, _my_string, _my_arc, some_var);
// Could become
let x = [my_vec, &my_string, +my_arc, some_var]|| something(my_vec, my_string, my_arc, some_var);
```

## Self is wild

The same problem that C++ has with `this`, Rust will have with `self`.

Consider a struct like this:

```rust
struct Data {
  some_a: Arc<A>,
  some_b: Arc<B>,
  some_c: Arc<C>,
}
```

To refer to `self.some_a` we need to decide how to capture `self` and then further on how to capture `some_a`.

For now I think my suggestion lands on allowing wildcards in the capture clause:

```rust
fn some_f(&self) {
  // Captures the &self reference
  do_something(|| self.some_a.something());
  do_something([self]|| self.some_a.something());
  // Do not allow self to be captured through a shorthand

  // Allow cloning a single value
  do_something({
    let some_a = self.some_a.clone();
    move || some_a.something()
  });
  do_something([some_a=self.some_a.clone()]|| some_a.something());
  // Allow a slight shorthand:
  do_something([+self.some_a]|| self.some_a.something());
  // Allow a wildcard:
  do_something([+self.*]|| self.some_a.something());
}
```

# Did we solve anything?

We set out to solve the following:

  1. It shall be possible to easily clone values into a closure.
  2. It shall be possible to restrict the set of outside values that a closure can make use of, and how they are captured by the closure.

Number 2 is definately fulfilled. We can clearly restrict what values a closure can make use of and we can control how they are captured.

Lets look at the two examples in the issue chapter and see how they would be solved using this more refined capture control:

```rust
let some_value = Arc::new(something);

// task 1
tokio::task::spawn(async [+some_value]|| {
    do_something_with(some_value);
});

tokio::task::spawn(async [+]|| {
    do_something_else_with(some_value);
});
```

Now the second one is more dramatic, so lets look at the original first:

```rust
let _some_a = self.some_a.clone();
let _some_b = self.some_b.clone();
let _some_c = self.some_c.clone();
let _some_d = self.some_d.clone();
let _some_e = self.some_e.clone();
let _some_f = self.some_f.clone();
let _some_g = self.some_g.clone();
let _some_h = self.some_h.clone();
let _some_i = self.some_i.clone();
let _some_j = self.some_j.clone();
tokio::task::spawn(async move {
  	// do something with all the values
});
```

This gets reduced to
```rust
tokio::task::spawn(async [+self.*] {
  	// do something with all the values
});
```

I don't think this can be made much simpler to be honest. So requirement number 1 is also fulfilled.

Now this is all bikeshed syntax, but I think this can give some ideas and directions where to go.
