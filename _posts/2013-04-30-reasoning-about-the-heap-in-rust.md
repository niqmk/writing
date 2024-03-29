---
layout: post
title: "Reasoning About the Heap in Rust"
tags:
- rust
- math
status: published
type: post
published: true
listed: true
vote: http://news.ycombinator.com/
---

If you follow programming languages or web technologies closely it's likely that you've heard of [Rust](http://www.rust-lang.org). Rust is one part of a larger effort by [Mozilla Research](http://www.mozilla.org/en-US/research/) to build a new browser engine in [Servo](http://www.mozilla.org/en-US/research/projects/#servo), but its value as a development tool certainly extends beyond that initial goal. In particular it has received attention for its memory model which, "encourages efficient data structures and safe concurrency patterns, forbidding invalid memory accesses that would otherwise cause segmentation faults" [[1](#footnotes)].

In this post we'll take a look at the basics of Hoare logic and an extension Separation logic which aid in reasoning about imperative program behavior and memory state. Then we'll apply those tools to examine the impact that Rust's [memory ownership system](http://static.rust-lang.org/doc/0.6/tutorial.html#ownership) has on the heap.

*While I was writing this post John Reynolds passed away. He was an incredible force in PL research and Separation logic was one of his most important works. There's much more to it than is covered here so if you're curious and you want to learn more please explore the links in the footnotes.*

## Hoare logic

In the late 1960s [Tony Hoare](http://en.wikipedia.org/wiki/C._A._R._Hoare) proposed a formal system for reasoning about programs which would eventually be referred to as [Hoare logic](http://en.wikipedia.org/wiki/Hoare_logic). The central feature of Hoare logic is a triple `{P} C {Q}` where `P` and `Q` are predicate logic assertions, referred to as the pre/post conditions, and `C` is a command (reads: program/program fragment). The idea is that, outside of any guarantee of termination [[2](#footnotes)], if `P` is true before `C` and `Q` is true after `C`, the triple proves partial correctness for `P` and `Q`. That is, it proves some property asserted by `P` and `Q` about `C`.

A simple example using C with the assertions in comments:

```c++
// { x == n }
x = x + 1
// { x == n + 1 }
```

Here the triple `{P} C {Q}` asserts that if `x` is equal to some `n` before `x = x + 1` then `x` will be equal to `n + 1` afterward.

In addition to this basic structure it's possible to define axioms for common programming constructs like assignment, branching, while loops, and for loops that allow for more general reasoning and manipulation of assertions. For assignment it takes the form `{P[E/V]} V=E {P}`. That is, substituting `E` for `V` in `P` before the assignment should hold and `P` should hold afterward [[3](#footnotes)].

```c++
// P[E/V]
x = x + 1
// P
```

Borrowing `Q` from the earlier example here, `P` is `x == n + 1`. Substituting `x + 1` for `x` in `P` gives `x + 1 == n + 1`, or `x == n`, for the precondition.

```c++
// The result of substituting `x + 1` for `x` in `x == n + 1`:
// { x == n }
x = x + 1
// { x == n + 1 }
```

With the help of this and other axioms, established for each programming environment, Hoare logic allows the wielder to write specifications for programs. For most domains (especially those that my usual reader works in) the approach might be heavy handed, but there are many domains where this type of specification is necessary. In particular it's often important to specify the behavior of a program with regards to memory.

## Separation Logic

[Separation logic](http://en.wikipedia.org/wiki/Separation_logic) is an extension to Hoare logic that provides tools for specifying memory use and safety with new assertions for how a program will interact with the heap and stack [[4](#footnotes)].

The four assertions that Separation logic adds for describing the heap are:

* `emp` - for the empty heap. It asserts that the heap is empty, and it can be used to extend assertions about programs that don't interact with the heap.
* `x |-> n` - for the singleton heap. It asserts that there is a heap that contains one cell at address `x` with contents `n`.
* `P * Q` - as a replacement for `∧` with disjoint heaps. It asserts that, if there is a heap where `P` holds and a *separate* (disjoint) heap where `Q` holds, both `P` and `Q` hold in the conjunction of those two heaps.
* `P -* Q` - as a replacement for implication, `=>`. It asserts that if there is a heap in which `P` holds then `Q` will hold in the current heap extended by the heap in which `P` holds. An important point of clarity: `P -* Q` holds for the *current heap* and *not* the current heap extended by the heap in which `P` holds.

There are also some shortcuts for common heap states that are built on top of these four assertions:

* `x |-> n, o, p` is equivalent to `x |-> n * x + 1 |-> o * x + 2 |-> p`. That is, `x` points to a series of memory cells that can be accessed by using `x` and pointer arithmetic.
* `x -> n` is a basic pointer assertion. It is equivalent to `x |-> n * true`, that suggests there is a heap where `n` is the value at `*x` which is a part of a larger heap about which we can't make any assertions.

Again we'll turn to C to demonstrate how these assertions fit with common programs [[5](#footnotes)].

```c++
// { emp }
int *ptr = malloc(sizeof(int));
*ptr = 5;
// { ptr |-> 5 }
```

The first assertions states that the heap is empty (`emp`). After the `malloc` call and assignment there exists a singleton heap with a single cell containing the value `5` that is pointed to by `ptr`.


```c++
// { emp }
int *ptr = malloc(sizeof(int));
*ptr = 5;
// { ptr |-> 5 }
int *ptr2 = malloc(sizeof(int));
*ptr2 = 5;
// { (ptr |-> 5) * (ptr2 |-> 5) }
```

Adding `ptr2` means the addition of another singleton heap and the connective `*`. It's possible to write this as `{ (ptr -> 5) ∧ (ptr2 -> 5) }`, but this assertion provides no information about how the heap is arranged. It simply says that there are two pointers to the value 5 somewhere in a heap. They might be pointing to the same memory location. By using the singleton heap pointer `|->` and the connective `*`, the new assertion makes clear that the two pointers are *not* pointing to the same memory location.

```c++
// { emp }
int *arry = calloc(3, sizeof(int));
*arry = 1;
*(arry + 1) = 2;
*(arry + 2) = 3;
// { arry |-> 1,2,3 }
```

Here the comma separated list of values following the singleton pointer in `{ arry |-> 1,2,3 }` denotes contiguous memory. It's simply a shorthand notation for the pointer arithmetic.

```c++
// { arry |-> 1 * (arry + 1) |-> 2 * (arry + 2) |-> 3 }
```

It's worth noting that separating implication, `P -* Q` doesn't appear to have any particularly useful or clear concrete examples. This seems to be the consequence of its relationship to logical implication in that the whole assertion is only false when `Q` is false. Borrowing from Reynolds [[6](#footnotes)], something like `{ x |-> 1 -* Q }` for some assertion `Q` can be extended with the separating implication to show:

```c++
// { x |-> 0 * (x |-> 1 -* Q) }
*x = 1
// { Q }
```

The precondition here says that there are two disjoint heaps. One in which `x |-> 0` holds and one in which `(x |-> 1 -* Q)` holds. The implication on the right is, if second heap was extended so that `x` *was* pointing to `1`, `Q` would hold. After the assignment `*x` is no longer `0` but rather the second heap has been extended so that `*x` is `1` and as a result `Q` holds [[7](#footnotes)].

## Rust Ownership

Rust provides two new type modifiers for dealing with pointers and memory management. Both have very specific semantics that are checked at *compile time* to help prevent memory leaks.

* `~` - provides a lexically scoped allocation on the heap. That is, when the newly assigned pointer variable goes out of scope the memory is freed.
* `@` - provides a garbage collected allocation on the heap. In Rust each [task](http://static.rust-lang.org/doc/tutorial-tasks.html) has its own garbage collector responsible for handling this type of heap allocation.

A few simple examples borrowed in part from Rust's tutorials will illustrate when the memory for each of these type modifiers is freed.

{% highlight rust %}
fn main() {
    {
        let a = ~0;
    } // a is out of scope and *a is freed
}
{% endhighlight %}

With the tilde, the memory at `*a` on the heap is freed when the variable to which it is assigned goes out of scope. Since `a` is declared inside an explicit block it goes out of scope at the end of the block and the associated memory is freed.

{% highlight rust %}
fn main() {
    let a;

    {
        let b = ~1;

        // move ownership of *b to a
        a = b;

        io::println(fmt!("%?", *b)); // error: use of moved value: `b`
        io::println(fmt!("%?", *a)); // => 1
    }
} // *a is destroyed here
{% endhighlight %}

When the ownership of memory is transferred between variables the compiler prevents further reference to the original owner. In this example `a` is the new owner and the compiler will prevent any further reference to `b`. This concept of single ownership is the reason that the memory can be deallocated safely when the current owner goes out of scope.

Alternately, the `@` type modifier can be used to request that the run-time manage the allocated memory on a per-task basis. This presents some interesting issues when creating pointers to memory allocated as part of a record.

{% highlight rust %}
struct X { f: int, g: int }

fn main() {
    let mut x = @X { f: 0, g: 1 };
    let y = &x.f;

    x = @X { f: 2, g: 3 };

    // original *x is now subject to gc

    io::println(fmt!("%?", *x)); // => {f: 2, g: 3}
    io::println(fmt!("%?", *y)); // => 0
}
{% endhighlight %}

Note that `x` is declared as a mutable variable (Rust's default is immutability). When a pointer is made to the field `f` with `&x.f` and then `x` is reassigned, the memory at `*x` would be subject to garbage collection. Luckily Rust does a bit of work for you and inserts a lexically scoped reference to the original record to prevent deallocation by the garbage collector.

{% highlight rust %}
struct X { f: int, g: int }

fn main() {
    let mut x = @X { f: 0, g: 1 };
    let x1 = x;
    let y = &x.f;

    x = @X { f: 2, g: 3 };

    io::println(fmt!("%?", *x)); // => {f: 2, g: 3}
    io::println(fmt!("%?", *y)); // => 0
}
{% endhighlight %}

You might also wonder how Rust handles references to lexically scoped record fields. In this case the compiler raises an error and (rather nicely) highlights the discrepancy in scoping expectations.

{% highlight rust %}
struct X { f: int, g: int }

fn main() {
    let a;

    {
        let mut b = ~X { f: 1, g: 2 };

        // move ownership of *b to a
        a = &b.f;
    }

    // error: illegal borrow: borrowed value does not live long enough
    io::println(fmt!("%?", *a));
}
{% endhighlight %}

Here, `a` is assigned the memory location of the field `f` of `b`, but the scope of `a` is larger than the scope of `b` which means that `*b` will be freed long before `*a`.

## Formalizing Ownership

Rust's memory management facilities exist mostly at compile time to prevent users from shooting themselves in the foot, but it's still worth applying Separation logic to get a feel for what's happening to the heap.

{% highlight rust %}
fn main() {
    // { emp }
    {
        // { emp }
        let a = ~0;
        // { a |-> 0 }
        let b = ~1;
        // { a |-> 0 * b |-> 1 }
    }
    // { emp }
}
{% endhighlight %}

Both `a` and `b` are scoped to the explicit block and exist in disjoint parts of the heap. When they go out of scope the memory associated with both is deallocated leaving an empty heap.

{% highlight rust %}
fn main() {
    let a;
    // { emp }
    {
	    // { emp }
        let b = ~1;
        // { b |-> 1 }
        let a = b;
        // { a -> 1 ∧ b -> 1 }
    }
    // { a |-> 1 }
}
{% endhighlight %}

In this case if there was a reference to `b` after the ownership of `*b` was transferred to `a` the compiler would complain. Since there is no reference we assume that `b` is pointing to the same location until the pointer is removed at the close of the explicit block.

{% highlight rust %}
struct X { f: int, g: int }

fn main() {
    // { emp }
    let mut x = @X { f: 0, g: 1 };
    // { x |-> 0, 1 }
    let x1 = x;
    // { x1 -> 0, 1 ∧ x -> 0, 1 }
    let y = &x1.f;
    // { x1 -> 0, 1 ∧ x -> 0, 1 ∧ y -> 0 }
    x = @X { f: 2, g: 3 };
    // { (x1 -> 0, 1 ∧ y -> 0) * x |-> 2, 3 }

    io::println(fmt!("%?", *x)); // => {f: 2, g: 3}
    io::println(fmt!("%?", *y)); // => 0
} // { emp }
{% endhighlight %}

Here we've re-purposed the managed memory example from earlier with the explicit addition of the reference that would otherwise be inserted by Rust to prevent GC, `x1`. Let's examine each expression and the associated assertions in turn.

{% highlight rust %}
// { emp }
let mut x = @X { f: 0, g: 1 };
// { x |-> 0, 1 }
{% endhighlight %}

As before the allocation of managed memory creates a singleton heap pointer to the memory containing the record values [[8](#footnotes)].

{% highlight rust %}
// { x |-> 0, 1 }
let x1 = x;
// { x1 -> 0, 1 ∧ x -> 0, 1 }
{% endhighlight %}

Adding an additional reference to that same memory means that we have two pointers to the same memory. As a result we *cannot* use the `*` connective or the the singleton heap pointer `|->` to represent the heap.

{% highlight rust %}
// { x1 -> 0, 1 ∧ x -> 0, 1 }
let y = &x1.f;
// { x1 -> 0, 1 ∧ x -> 0, 1 ∧ y -> 0 }
{% endhighlight %}

A new reference to the memory location of the first record field in `x` and `x1` adds another pointer that overlaps with the existing heap. It's important to keep in mind that the basic conjunction simply says that the heap *may* overlap. To the reader it may be obvious, but in terms of specifying program behavior it's a weaker assertion that the equivalent made with the `*` connective.

{% highlight rust %}
// { x1 -> 0, 1 ∧ x -> 0, 1 ∧ y -> 0 }
x = @X { f: 2, g: 3 };
// { (x1 -> 0, 1 ∧ y -> 0) * x |-> 2, 3 }
{% endhighlight %}

Finally with the allocation of a wholly new record and pointer for `x` we can employ the more powerful connective because the new record lives in a newly allocated section of memory on the heap. The remaining pointers to the original record and its first field remain ambiguous.

## Further Reading

Not covered here for brevity's sake is an important part of Separation logic, the Frame Rule. The Frame Rule provides for rigorous local reasoning about the heap without concern for other possibly overlapping references to the same memory locations. That is, it allows each assertion to be correct in spite of the fact that the program fragments they pertain to are often operating in a larger application that manipulates the heap.

Also, the concept of borrowed pointers is important reading if you're interested in Rust. A common but effective memory efficiency is achieved in C by passing pointers to data structures instead of using the default pass-by-value semantics. Similarly one can "borrow" a pointer to a data structure in Rust, but because of the type level restrictions it's both safer and more complex [[9](#footnotes)]. The borrowed pointers [tutorial](http://static.rust-lang.org/doc/tutorial-borrowed-ptr.html) makes those complexities clear.

## Conclusion

Hopefully this post has given you an initial sense of a portion of Rust's memory management facilities and also the formalism of Separation logic.

Special thanks goes to [@steveklabnik](https://twitter.com/steveklabnik), [@evanphx](https://twitter.com/evanphx), and Matt Brown for reviewing various bits of my post.

### Footnotes
<a name="footnotes"></a>

1. http://static.rust-lang.org/doc/0.6/tutorial.html#introduction
2. When the program fragment can be shown to terminate the triple proves total correctness.
3. There are actually two forms of the assignment axiom. The second proposed by Floyd is more complex but addresses issues that exist in common imperative languages the first cannot.
4. More information on Separation logic https://wiki.mpi-sws.org/star/cpl
5. The examples that follow assume that `malloc` is always operating by allocating fresh memory not pointed to elsewhere.
6. John C. Reynolds: [Separation Logic: A Logic for Shared Mutable Data Structures.](http://www.cs.cmu.edu/~jcr/seplogic.pdf)
7. It certainly feels convoluted but the separating implication is used as a powerful tool when reasoning about the execution of programs moving backwards (among other things). All the examples in this post start from some initial state and then move forward following the execution of the program. Doing this can produce useless assertions if the thing you are trying to prove isn't affected by some of the program snippets. Many times it's easier to reason from the end goal, that is from the final result of a set of commands/expressions/program snippets and work backwards. As you can see `{ x |-> _ * (x |-> 1 -* Q) } *x = 1 { Q }` the final assertion can be anything. This means that you can start with some assertion you want to show for a program and move backward "over" a memory mutation!
8. This assumes the memory allocated for a struct in Rust is sequential (for [reference](http://static.rust-lang.org/doc/tutorial-borrowed-ptr.html#borrowing-unique-boxes)). It also assumes that `x.f` and `x.g` aren't explicitly scoped pointers but rather tranlate into an operation on the pointer `x` at a low level (they aren't represented in the assertions).
9. Obviously this depends on your perspective and how complex the code is that uses the pointer. That is, it may be exceptionally hard to get the memory freed properly as result of passing around pointers in which case the compiler might be extremely valuable when writing the Rust equivalent.
