# Steal borrow

To explain myself I'll use a simple, silly `struct` to calculate an average.

I defined a method to "add" values and another to "obtain" the average of these values.

It could be like that.

```rust,ignore
struct Avg {
    count: u64,
    sum: u64,
}

impl Avg {
    fn add(&mut self, item: u8) {
        self.count += 1;
        self.sum += u64::from(item);
    }

    fn avg(&self) -> f32 {
        (self.sum as f32) / (self.count as f32)
    }
}
```

An example using it

```rust,ignore
fn main() {
    let mut data = Avg { count: 0, sum: 0 };
    data.add(1);
    data.add(2);
    data.add(3);

    println!("{}", data.avg());
}
```

Inconvenient, abuse of mutability.

We define a mutable instance, and then we call functions that might or might not mutate it. They could not mutate it today but they could mutate it tomorrow.

It is not clear that `data.avg` does not change `data' internally, in fact, today it could not mutate it and tomorrow it could.

This excess of mutability makes the code less readable.

I don't go any further in defense of the reduction of mutability and I will continue with the idea I want to propose.

Mutability is usually viral (in most languages), but in Rust, we have a way to avoid it.

> By viral mutability, I mean that using mutability at a point, it forces us to extend that mutability in the code to callers.

In Rust we have `borrow`, `mutable borrow` and `ownership`.

- You can borrow something with the condition you will not modify it (see but not touch).
- You can borrow something with permission to be modified.
- Or you can "give it away," so the receiver can decide whether or not to change it (it's all yours).

Thanks to the last point, we can control and prevent mutability from spreading virally.

The code would look like this:

```rust,ignore
struct Avg {
    count: u64,
    sum: u64,
}

impl Avg {
    fn add(mut self, item: u8) -> Self {
        self.count += 1;
        self.sum += u64::from(item);
        self
    }

    fn avg(&self) -> f32 {
        (self.sum as f32) / (self.count as f32)
    }
}
```

Now we can use the `Avg` class as follows:

```rust,ignore
fn main() {
    let data = Avg { count: 0, sum: 0 };
    let data = data.add(1);
    let data = data.add(2);
    let data = data.add(3);

    println!("{}", data.avg());
}
```

Or even fluent API style...

```rust,ignore
fn main() {
    let data = Avg { count: 0, sum: 0 }.add(1).add(2).add(3);

    println!("{}", data.avg());
}
```

We have the same result, but in our program we use **zero** mutability.

```rust,ignore
    let data = Avg { count: 0, sum: 0 };
    let data = data.add(1);
    let data = data.add(2);
    let data = data.add(3);
```

This reassignment of the variable data is SSA (Static Single Assignment) (cool)

Here it is clear that `add` generates a new value (internally with mutation, internal implementation detail for the API user) and `avg` reads but does not mutate.

It is not necessary to look at the signature or the source code of the functions used to know that.

As our vars are not mutable, we know nobody can modify them stealthily.

The mutability is not viral, the code is more readable.

**BUT...**

There are several difficulties on the horizon to use this model.

I propose next two.

- We receive a mutable reference from `Avg` in a function.
- `Avg` is part of a mutable structure.

In both cases, since they are mutable, it seems reasonable to want and be able to add elements, but it is not easy.

Case mutable reference in a function:

```rust,ignore
fn mut_avg(avg: &mut Avg) {
    avg.add(1);
}
```

This doesn't work (because that's what we wanted to avoid).

`Avg` case in a structure:

```rust,ignore
fn struct_avg1(savg: &mut StructAvg) {
    savg.avg.add(1);
}
```

It's not working.

This gets ugly, when we receive it in a structure, we can't even use the ownership pattern that I'm proposing.

```rust,ignore
fn struct_avg2(mut savg: StructAvg) -> StructAvg{
    savg.avg.add(1);
    savg
}
```

:\_(

The simplest solution is to do it the other way.

Just in case, and to avoid problems, we can create an API based on mutable references, and from these, it is very simple to create the API with ownership.

But that's not good. We will end up not doing the second step and not using the API with ownership with the evident consequence of mutability flood (viral), as in the rest of programming languages that support mutability control.

To avoid this, I propose to add a new model to those already mentioned. It would look like this.

- Borrow with the condition that you do not touch it
- Borrow allowing to touch
- Gift with all the consequences (ownership)
- **Provisional borrow steal**

Let me show this new option in the previous examples

```rust,ignore
fn mut_avg(avg: &mut Avg) {
    steal_borrow(avg, &|avg| {
        let avg = avg.add(1);
        let avg = avg.add(2);
        let avg = avg.add(3);
        avg
    });
}

fn struct_avg1(savg: &mut StructAvg) {
    steal_borrow(&mut savg.avg, &|avg| {
        let avg = avg.add(1);
        let avg = avg.add(2);
        let avg = avg.add(3);
        avg
    });
}

fn struct_avg2(mut savg: StructAvg) -> StructAvg {
    steal_borrow(&mut savg.avg, &|avg| {
        let avg = avg.add(1);
        let avg = avg.add(2);
        let avg = avg.add(3);
        avg
    });
    savg
}
```

And `steal_borrow` function could be implemented as...

```rust,ignore
pub fn steal_borrow<T>(target: &mut T, f: &Fn(T) -> T) {
    let mut fake = unsafe { std::mem::zeroed() };
    std::mem::swap(&mut fake, target);
    let mut fake = f(fake);
    std::mem::swap(&mut fake, target);
    std::mem::forget(fake);
}
```

Bellow, on references, there is a link with a better solution on crates.io

It's not `zero cost abstraction` but it's not expensive.

> Attention!!!, to avoid that `zeroed` incurs in a cost in big structures, those data should be put in a `Box<...>`

What do you think???

## References

- Here, the boys of fpcomplete propose the same idea that is the origin of the problems that I explain and I propose a solution.
  - [https://www.fpcomplete.com/blog/2018/10/is-rust-functional]()
- One of the places where I explain the proposal

  - [https://www.reddit.com/r/rust/comments/a4x00h/thought_for_reducing_mutability_in_rust/]()

- CornedBee point to two great links with similar proposal and good implementations

  - [https://crates.io/crates/take_mut]()
  - [https://crates.io/crates/replace_with]()

- The way to solve this kind of problems has been proposed as RFC, but not approved
  - [https://github.com/rust-lang/rfcs/pull/1736]()
