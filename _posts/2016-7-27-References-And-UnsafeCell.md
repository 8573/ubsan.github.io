---
layout: post
title: References and UnsafeCell
---

I've seen a lot of confusion about references (`&T` and `&mut T`), as well as
`UnsafeCell`, and I'd like to clear it up.

Please note: despite being on the unsafe guidelines team, these are rules which
haven't yet been fully written. This is how I see it, however, and I think
probably how it'll shake out in the long run (as in, this is how I think it
should go).

First of all, let's look at `&` and `&mut` by themselves.

#### (Before that, a bit of terminology)

  * Pointers

    * References

      * Shared Reference - `&T`

      * Mutable Reference - `&mut T`

    * Raw Pointer

      * Const Raw Pointer - `*const T`

      * Mutable Raw Pointer - `*mut T`

    * Smart Pointer

      * Box - `Box<T>`

      * Reference Counted Pointer - `Rc<T>`

      * etc.

# References

`&T` and `&mut T` are two of the most important types in Rust, perhaps the most
important types. They are similar to `T*` and `T const*` in C, but with a few
extra guarantees; most importantly for today, there shall be no aliasing writes.

If you look at the [nomicon][nomicon-alias], aliasing is defined in an...
interesting way, with liveness and paths and all that fun stuff. I don't suggest
reading it unless you want to have a confusing time. Let's go over what
"aliasing writes" really means:

First of all, we must define a pointer, if we're to understand what "aliasing"
is. A pointer is a handle to a specific "lvalue", or block of memory. There are
many ways to get a block of memory like this; these are two common ones:

```rust
fn main() {
  let x: u32 = 0; // on x86_64, x is an lvalue with size 4, alignment 4
  let y = Box::new(x); // on x86_64, y is an lvalue with size 8, alignment 8
                       // while *y is an lvalue with size 4, alignment 4
}
```

Importantly, you can also have sub-lvalues:

```rust
struct Foo { x: u32, y: u32 }

let foo = Foo { x: 0, y: 1 }; // on x86_64, foo is an lvalue of size 8, align 4

let f = &foo; // f points to the entirety of Foo.
let x = &f.x; // x points to *just* the "x" part of foo,
              // or an lvalue with size 4, align 4
let y = &f.y; // y points to just the "y" part of foo,
              // also an lvalue with size 4, align 4
```

One important thing to see is that `x` and `y` are disjoint; they don't point at
the same lvalue. `f` covers both `x` and `y`; it points to an lvalue which holds
both `x` and `y`.

```
|   f   |
| x | y | (you'll notice that f covers both x and y, but x and y are separate)
```

So, pointers are *really* handles to lvalues; and you aren't allowed to alias
writes with references. What does this really mean?  It means that if you have
any type of pointer (`*const T`, `*mut T`, `&T`, `&mut T`, `Box<T>`, etc.), and
a reference (`&T`, `&mut T`), which alias (with the definition of aliasing seen
above), you can't write to the aliased lvalue through one of the pointers,
and read the changed lvalue through the other pointer.

## Examples

```rust
// This is an example of a write through a pointer, and a read through the
// other.
// You may not alias ptr1 and ptr2, because you're writing through ptr1, and one
// of these is a reference (technically, both of them are)
fn foo(ptr1: &mut u32, ptr2: &u32) {
  *ptr1 = *ptr2 + 5;
}

struct Bar { x: u32, y: u32 };
let mut bar = Bar { x: 0, y: 1 };
foo(&mut bar.x, &bar.y); // totally fine! bar.x and bar.y are disjoint lvalues
foo(&mut *(&mut bar.x as *mut _), &bar.x); // undefined behavior, because bar.x
                                           // is aliased to itself
// (the &mut *(... as *mut _) is to get around the borrow checker)

// This is a similar example; if ptr1 is aliased to ptr2.x, it's UB
fn baz(ptr1: &mut u32, ptr2: &Bar) {
  *ptr1 = ptr2.x;
}

let mut bar = Bar { x: 0, y: 1 };
baz(&mut 0, &bar); // fine, they're not aliased ("disjoint")
baz(&mut *(&mut bar.x as *mut _), &bar); // UB, you write through ptr1, and 
                                         // they're aliased as above
baz(&mut *(&mut bar.y as *mut _), &bar); // not UB in my opinion, although 
                                         // definitely not open-and-shut. this
                                         // is a read from bar.x, and a write
                                         // through bar.y, which *shouldn't* be
                                         // UB, but no promises
// of course, if you delete the function calls, then it's totally not UB
// UB follows use.
```

There are also rules about reborrows: if you mutably reborrow a pointer, then
writes through any pointers down the line will be visible through each pointer
up the line, which isn't Undefined Behavior. Maybe next time, I'll cover these
in more detail.

```rust
let original_ref = &mut 0;
{
  let mutably_reborrowed = &mut *original_ref;
  *mutably_reborrowed = 1;
}
println!("{}", *original_ref); // 1
{
  let raw_pointer: *mut _ = original_ref;
  unsafe { *raw_pointer = 2; }
}
println!("{}", *original_ref); // 2
{
  let raw_pointer: *mut _ = original_ref;
  let ref_from_raw = unsafe { &mut *raw_pointer };
  *ref_from_raw = 3;
  println!("{}", *raw_pointer); // 3
  // can't write to ref_from_raw after this point, because you've read from
  // raw_pointer now
}
println!("{}", *original_ref); // 3
// can't write to either, because you've read from original_ref
```

These are all examples of mutable reborrows.

# UnsafeCell, and how it fits into all of this

First thing you must know: `UnsafeCell` is special. Really, really special.
`&UnsafeCell<T>` means that you can mutably alias that `T`, and is in fact the
*only* way to do that in Rust with references. (note: this doesn't apply to
`&mut UnsafeCell<T>`; still no mutable aliasing there!). How does it do this
magical thing? By being built into the language.  Basically, having a shared
reference to an `UnsafeCell<T>` shall take away that shared reference's powers
when it comes to aliasing.

```rust
unsafe fn foo(x: &UnsafeCell<u32>, y: *mut u32) {
  *x.get() = *y + 3;
  println!("{}", *x.get() + *y);
} // Rust is not allowed to assume that x and y do not alias, and therefore,
  // this is well defined even if x and y point to the same lvalue.
```

You can write through shared references to an `UnsafeCell<T>`, and you can alias
an `&UnsafeCell<T>` with writes to other non-aliasing pointers.

Let's look at it another way:

```rust
struct Foo {
  x: u32,
  y: UnsafeCell<u32>,
}

let foo = Foo { x: 3, y: UnsafeCell::new(1) };
let y1: &UnsafeCell<u32> = &foo.y;
let y2: *mut u32 = foo.y.get();
unsafe { *y1.get() += *y2 + foo.x }; // this is totally fine

/*
 * |          foo            |
 * | foo.x     | foo.y       |
 * |           | y1          |
 * |           | y2          |
 *               ^ these three lvalue handles are allowed to be mutably aliased
 *                 unlike normal references
 */
```

Basically, `&UnsafeCell<T>` and `*mut T` are equivalent with regard to aliasing
and mutability, although it still has the same non-nullability and lifetime of
`&T`. The other important ability of `UnsafeCell<T>` is that you can place it
inside a structure; this is put to good use in `RefCell<T>`. This means that
only that sub-lvalue of the structure is considered aliasable, which will
hopefully eventually be good for optimizations, even if we aren't using it now
:)


[nomicon-alias]: https://doc.rust-lang.org/nomicon/references.html
