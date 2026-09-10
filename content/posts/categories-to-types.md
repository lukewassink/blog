+++
draft = false
date = 2026-09-06T15:33:45-04:00
title = "From Categories to Types"
description = "Explains how category theory maps onto types"
authors = ['Luke Wassink']
tags = ["math", "scala", "functional programming"]
series = []
toc = true
+++

## Who is this for?

At this point, there are, by conservative estimate, 80,000 blog posts and
Youtube videos that explain how monads fit into functional programming. It seems
to be mandated by federal law that 99% of them begin by reassuring you that,
unlike all the other explanations out there, *this explanation* won't use any
fancy math or require any knowledge of category theory. This blog post belongs
to the remaining 1%.

I'm writing it for me from two months ago. Two months ago me liked functional
programming and had actually taken a whole course on category theory back in the
day, but he just couldn't seem to see exactly how the two fit together. He
trawled through video after blog post, only to be reassured once again that
"*This explanation* won't involve any category theory." Finally, he ventured
forth into the shadowy depths of Wikipedia and other dark corners of the
internet, to return bearing this hidden knowledge. So now I'm writing the blog
post that two-months-ago me wished someone else had written. A post that
explains a little bit about category theory (but with lots of links to other
places that say more) and explains even less about functional programming
(Google it-there's stuff out there) but does its best to lay out the connection
between the two as clearly and concretely as possible.

So it might be a post written only for its author. But maybe not. Read on if:

- you want a high level, and hopefully somewhat friendly, introduction to
    category theory
- you want to know what category theory has to do with types, and exactly which
    types are functors
- you want a discussion of monads that does it's very best to clearly explain
    exactly what a monoid in the category of endofunctors is, and why it might
    be nice thing for your programs.


## There are monoids and monoids

First, a disambiguation. If you Google monoids, you'll quickly learn that a
[monoid](https://en.wikipedia.org/wiki/Monoid) is a set with a binary,
associative operation with identity. For those in the know, it's just a group
that doesn't have to have inverses, or a semigroup that does have an identity
element.

More concretely, it's a set of things with some operation we can write $*$ that
has to satisfy:

1. $a * (b * c) = (a * b) * c$ for any elements $a, b, c$,
1. there is an element $e$ such that $e * a = a * e = a$ for any element $a$.

Examples: non-negative integers under addition (zero is the identity); $n\times
n$ matrices under multiplication (the identity matrix is the identity).

Now the million dollar question: what does this have to do with types?
Absolutely nothing! Or at least, not directly. To see the connection we'll have
to go through category theory, and meet another definition of monoid. (Or is it
secretly the same definition?.  Let's find out.)


## Abstract nonsense

Mathematicians love to generalize. You work with enough things that have some
notion of multiplication, and you start to notice common features: Most of them
are associative. Lots of them have inverse elements. And so the notion of a
[group](https://en.wikipedia.org/wiki/Group_(mathematics)) is born. Similarly,
there are lots of structures where you can add and multiply, and multiplication
distributes over addition, and so you have
[rings](https://en.wikipedia.org/wiki/Ring_(mathematics)).

Not content to stop there, mathematicians have generalized yet further. Once you
study groups and rings, and maybe also
[modules](https://en.wikipedia.org/wiki/Module_(mathematics)) and some other
things, you start to notice common patterns in the ways they fit together and
the things you can do with them. And that's [category
theory](https://en.wikipedia.org/wiki/Category_theory).

More precisely, category theory groups together various structures in
*categories*. In a category, we have:

- **Objects:** the things in the category. These objects could be sets, or
    something fancier like groups or rings.
- **Morphisms:** just think of morphisms as arrows. Each morphism points from
    one object to another. In the category of sets, for example, morphisms are
    just functions. We also need a notion of composition of morphisms denoted
    $\circ$. If I have morphisms $f: A\to B$ and $g:B\to C$, then I get a
    morphism $g\circ f: A\to C$.

There are a few other conditions the morphisms have to satisfy, but this is as
far as I'll go. Follow the links for details.

The example to keep in mind is the category of sets. Objects are sets, morphisms
are functions, and composition of morphisms is just ordinary composition of
functions.

It's important to note that morphisms are first class citizens in this
definition, and while there might be a natural choice for some set of objects,
nothing forces us to make that choice. Keep the objects but change the
morphisms, and you get a different category.

At first we had concrete things and we could get from one thing to another.
Then we got whole sets of things and maps from one set to another. Now we have
categories of sets of things[^1], and naturally we want maps from one category
to another. These maps are called [functors](https://en.wikipedia.org/wiki/Functor).

Say we have categories $\mathcal{C}$ and $\mathcal{D}$. A functor
$F:\mathcal{C}\to\mathcal{D}$ gives you an object $F(A)$ in $\mathcal{D}$ for
every object $A$ in $\mathcal{C}$. But that's not enough. We also need to say
what happens to morphisms. If we have a morphism $f$ from $A$ to $B$, then we
should get a morphism $F(f)$ from $F(A)$ to $F(B)$. And finally, $F$ had better
play nice with composition[^functor-of-morphism]. Again, what happens to the
morphisms is up to us. There may be a most natural or useful choice, but just
choosing what happens to the objects doesn't force that choice on us.

Let's make this more concrete. Consider the category of sets. A functor $F$
would need to do two things:

1. for each set $A$, give us another object $F(A)$,
2. for each function $f$, give us a morphism $F(f)$.

First a very simple example, then a more interesting one. How can we get
something for each set? Why not just the set itself? And what about morphisms?
Let's do the same thing: the morphism itself. So $F(A) = A$ and $F(f) = f$ for
all sets and all functions.  This is called the identity functor.

Now something more interesting. Consider the functor $W$ that takes in a set and
returns that set wrapped in another set: $F(A) = \{A\}$. So:

$$
\begin{align*}
F(\emptyset) &= \{\emptyset\} \\
F(\{1, 2\}) &= \{\{1, 2\}\}
\end{align*}
$$

What should $F$ do to functions? A logical choice would be: just apply them to
the wrapped set. So $F(f)(F(A)) = \{f(A)\}$.

Here's something interesting about both the functors we mentioned: they both map
from the category of sets, *back to the category of sets*. Not all functors have
to do that. We can have a functor $F:\mathcal{C}\to\mathcal{D}$ where
$\mathcal{C}$ and $\mathcal{D}$ are distinct, but a functor
$F:\mathcal{C}\to\mathcal{C}$ has a special name. It's called an *endofunctor*
of $\mathcal{C}$.

That's right, *that* endofunctor. We've got the first hint of monads! How do we
get from here to a monoid in the category of endofunctors? We start with
*monoidal categories*.

A [monoidal category](https://en.wikipedia.org/wiki/Monoidal_category) is a
category where we can take the product of objects. For any two objects $A$ and
$B$, the category must also contain an object $A\otimes B$. To be a bit more
precise, $\otimes$ has to be what's called a *binary functor*, so we can also
take the product of morphisms, and the product has to be associative.  There
also needs to be an identity object $I$ with the property that $I\otimes A$ is
[isomorphic](https://en.wikipedia.org/wiki/Natural_transformation) to $A$. These
conditions on $\otimes$ are sometimes called *coherence conditions*. Remember
that!

If this seems like a lot, there's no need to sweat the details in general
(unless you want to, in which case go for it!). The situation in the category of
sets, which is our focus, is much simpler. The monoidal product $\otimes$ is
just the ordinary product: the set of all pairs of elements from the two sets.
For example:

$$
\{1, 2\} \times \{a, b, c\} = \{(1, a), (1, b), (2, a), (1, b), (2, b), (1, c), (2, c)\}.
$$

For the identity object, just pick any set with one element. You get an
invertible mapping from $A\times \{x\}\to A$ by just mapping $(a, x)$ to $a$;
the inverse maps $a$ to $(a, x)$.

Now, inside a monoidal category, we can have a [monoid
object](https://en.wikipedia.org/wiki/Monoid_(category_theory)), and it turns
out this is the monoid we are looking for. To be a monoid, an object $M$ needs
two special morphisms:

$$
\begin{align*}
\mu&: M\otimes M\to M \\
\eta&: I\to M
\end{align*}
$$

Perhaps you see why we need to define $M$ inside a monoidal category? It's so
$M\otimes M$ exists.  Of course, $\mu$ and $\eta$ have to satisfy some
constraints additional constraints that, confusingly, are also called
*coherence conditions*. We'll spell them out a bit more in the context of
types. For now, follow the links if you want more details.

Now what does a monoid object look like in our trusty category of sets The
multiplication morphism, as $\mu$ is often referred to, is going to be a map
from $M\times M$ to $M$, so it takes in a pair of elements of $M$ and returns a
single element of $M$. Let's use our example of non-negative integers from the
section on monoids. We can define $\mu(a, b) = a + b$ and $\eta(x) = 0$, where
$x$ is the lone element in your favorite single-element set.

This has the nice consequence that $\mu(\eta(x), a) = \mu(0, a) = 0 + a = a$.
This turns out to be one of those coherence conditions we were talking about a
couple paragraphs ago, so non-negative integers along with this choice of $\mu$
and $\eta$ turn out to be a monoid object in the category of sets.

Let's take stock. We've defined monoids. We've defined endofunctors. Now we're
almost ready for, you guessed it, **monoids in the category of endofunctors!**
You see, mathematicians are perverse and masochistic. They had things like
numbers, and ways to get from one thing to another. Then they built sets out of
those things, and gave some of those sets structure to make groups and rings and
vector spaces.  Each of those has their own preferred kind of map that let you
get from one structure to another. Then mathematicians took all the sets, all
the groups, all the rings, and gathered each of them together to make the
category of sets, and the category of groups, and the category of rings. And
they defined maps between categories, and called them functors. 

Well, why stop there? We can now consider all the functors from a category to
itself—all the endofunctors—and *those* form a category. What are the morphisms
in this insane category? They're things called [natural
transformations](https://en.wikipedia.org/wiki/Natural_transformation).  This
means that a natural transformation must somehow map one functor to another. How
can it do that. Let's pick two endofunctors $F$ and $G$. Take any
object $X$. A natural transformation $\varphi$ from $F$ to $G$ needs to give us a
morphism $\varphi_X: F(X)\to G(X)$[^2]. Think of this as mapping the effect $F$
has on objects to the effect $G$ has on objects.

It also turns out that the category of endofunctors is always a monoidal
category. The product of two endofunctors is just given by composition:

$$F\times G = F\circ G$$

Composition is associative, and the identity object is just the identity functor
we discussed earlier[^monoid-product-morphisms].

Let's put the pieces together. A *monad* is a *monoid* in the category of
*endofunctors*. Let $F$ be a monad. A monoid object needs two special morphisms.
First, $\mu: F\times F\to F$. Second, $\eta: I\to F$. For endofunctors the
product is just composition, morphisms are natural transformations, and the
identity object is the identity functor, which we'll call $Id$. A natural
transformation gives us a morphism for each object. So for each object $X$ we
need morphisms:

$$
\begin{align*}
\mu_X:& F(F(X)) \to F(X) \\
\eta_X:& X \to F(X)
\end{align*},
$$

where the second line follows from $Id(X) = X$. If the coherence conditions hold
for $\mu$ and $\eta$, than $F$ is a monad! Let's take a brief detour, and then
we can finally connect all this to programming.


## A brief detour

You may have a lingering question: what of the humble, concrete monoid we
mentioned in the first section? Was that really just a red herring, thrown in
our path by Google to keep us from discovering the real monoid we wanted all
along: the monoid object?

**Warning:** in the previous section I tried to give a somewhat accessible
explanation of everything, even if it was often a fuzzy explanation that
elided lots of important details. I'm not saying I succeeded, but at least
I tried. For the remainder of this section I won't even do that. If you don't
have some prior exposure to abstract algebra and particularly category theory,
you'll probably find this part pretty inaccessible. If you want to skip to the
next section where there are nice friendly blocks of code, you have my blessing!

The answer to the question from two paragraphs ago is, of course, no. The two
concepts of monoid are *not* unrelated. In fact, they are quite closely
connected in at least two ways.

First, recall that a *small* category is a category whose objects form a set,
and not a proper class. The set of equivalence classes of objects under
isomorphism of a small monoidal category *is* a monoid. The identity
element is the identity object, and multiplication is given by the product. The
coherence conditions on the monoidal product guarantee that multiplication is
associative and that the identity behaves correctly.

Conversely, any monoid is a small monoidal category with objects given by the
elements of the monoid, only the identity morphism for each object product,
monoidal product given by multiplication in the monoid, and the identity object
given by the identity element in the monoid.

Second, any monoid is a monoid object in the category of sets. The
multiplication morphism $\mu$ is given by multiplication in the monoid:
$\mu(a\otimes b) = a*b$. Pick any singleton set $\{x\}$. Then $\eta$ is
given by $\eta(x) = e$ where $e$ is the identity in the monoid.
Associativity and identity properties of the monoid guarantee that the coherence
conditions on $\mu$ and $\eta$ are satisfied.

Conversely, suppose $(M,\mu,\eta)$ is a monoid object in the category of sets.
Then $M$ is a monoid if we define multiplication and identity by $a*b =
\mu(a,b)$ and $e = \eta(x)$. Thus we can view monoid objects as a generalization
of the concrete notion of a monoid.


## Types, at last

At last, we come to types. In this section, I'll lay out the connection between
types and category theory, and in the next I'll zoom in on monads and show how
their usual definition matches category theoretic definition we've unpacked. Of
course, there will be plenty of examples along the way.

For a long time I would hear about functors and monads in Scala and Haskell, but
I was never sure exactly how this matched up to category theory. What's the
category here? What are the objects? What are the morphisms?

Before we answer these questions, here's another. What is a type? We will model
a type simply as a set. That might seem a little odd if you're used to
unconsciously thinking of a type as something like a description. The type `Int`
means "integer", or maybe it means a description of 32 bit integers. Modeling
types as sets can accommodate this: given a description, the type is all the
values that match that description. So:

- `Int` is the set of integers
- `Double` is the set of double precision floating point numbers
- `String` is the set of strings

This model can also easily accommodate union and intersection types:
they're just the union and intersection of sets. It can also handle singleton
types, which some languages allow, and which hold only a single element. For
example, in Scala you can write:

```scala
type One = 1
val x: One = 1 // this compiles
val y: One = 2 // this throws a compiler error
```

Types like these might seem strange, but if a type is just a set then there's
not problem.

It's important to note that this is only a model. Thinking of a type as a set
captures something important about the type, but a type is not actually a set.
We further note that two types can represent the exact same data and still be
distinct types. For example, take:

```scala
case class MyInt(value: Int)
```

The type `MyInt` can represent the exact same data as `Int`, and there is a
bijective mapping given by sending `x: Int` to `MyInt(x)`, but `Int` and `MyInt`
are not the same types.

One final note: we often ignore the limitations of the computer when they won't
matter for our analysis. For example, we often act as though `Int` is the
actual, infinite set of integers. But of course it isn't, and it's operations
aren't even designed the same way. They have builtin logic to handle overflow.
Integers in math can't overflow. You could model `Int` precisely, but when
you're asking things like, is this type a product of these other two types? or,
is this parametric type a functor? it really doesn't matter. So a lot of the
time we keep things simple and pretend that `Int` really does represent the
ordinary integers, that `String`s can be arbitrarily long, and so on.

So what category are we interested in? It's the **category of types**, modeled
essentially as the category of sets. In this category:

- **Objects are types**. So objects are things like `Int`, `String`, and any
    other types you might define. Instances of those types, such as `1` or
    `"abc"` are not objects, just as elements of sets are not objects in the
    category of sets (unless those elements themselves are also sets).
- **Morphisms are functions from one type to another**. For example `val f = x => x + 2`
    is a morphism from `Int` to `Int`, and `val g = s => s.length()` is a
    morphism from `String` to `Int`.

And that's almost all there is to it. The only missing piece is functors. A
functor does two things: maps objects to other objects, and maps morphisms to
other morphisms. Let's start with objects. What are you familiar with that takes
in a type and gives out another type? That's right, a parametric type! Consider
the humble `List`. It takes in a type, and gives you back a new type: it
turns `Int` into `List[Int]`, `String` into `List[String]`, etc.

But that's not enough. A functor also needs to tell us to what to do with morphisms.
Given functor `F` and a morphism $f:A\to B$, we should get a morphism
$F(f):F(A)\to F(B)$. So suppose we have a function `f: A => B`. How do we get a
function `functoredF: List[A] => List[B]`? Right again, `map`! In code:

```scala
val f: A => B = ???
val functoredF: List[A] => List[B] = l => l.map(f)
```

So, in general, a parametric type `F[T]` is a functor if it also has a `map`
method[^functor-types]:

```scala
def map[S](f: T => S): F[S] = ???
```

Note that the target category of `F` is also the category of
types. Since it maps objects from a category back to the same category, `F` is
an *endofunctor*.


### An interesting detour

This section isn't necessary for anything that follows. But it does highlight
and interesting point at which the connection between actual Scala types and
objects in a category breaks down. Feel free to skip it if you just want to get
to monads.

I took this detour when I asked: can we find an example of a functor that isn't
an endofunctor? All the functors we are considering go from types to types, so
to avoid being an endofunctor, we need a functor whose domain and range are
distinct subcategories. Here's a nice simple example: consider the subcategory
of all types of the form `List[A]` and map it to the subcategory of all types of
the form `Vector[A]`. Since we're identifying a type with the set of all
instances of that type, our functor needs to send each instance of `List[A]` for
a given `A` to an instance of `Vector[A]`, and it also needs to map functions.
Here's a first shot:

```Scala
object ListToVec:
  def apply[T](list: List[T]): Vector[T] = list.toVector

  def apply[T, S](f: List[T] => List[S]): Vector[T] => Vector[S] =
    v => f(v.toList).toVector
```

We can write `ListToVec(List(1, 2))` to get a vector, and we can also map
functions: `ListToVec(f)`, but have you spotted the problem? The problem is that
`ListToVec` isn't a functor at all. It doesn't actually map one type to another.
It maps instances of one type to instances of another type, but for a functor we
really do need an actual parametric type so we can genuinely map one type to
another.

That is, we need a type `F[A]` such that 1) the typechecker knows that `A` isn't
allowed to be any old type. It must be of the form `List[T]` for some type `T`;
2) the type checker knows that`F[A]` is equal to the type `Vector[T]`.
Unfortunately, as far as I know, implementing `F` in Scala isn't possible. Or if
it is, it's quite complicated, and something in the neighborhood is certainly
impossible. After all, think of what we're asking of the type checker.  We're
asking it to:

1. Restrict `A` to only types of the form `List[T]`
2. Identify `T` given such an `A`
3. Prove to itself that `F[A]` is of the `Vector[T]` for the same `T`

It's disconcerting that this is out of reach when it seems so tantalizingly
close. After all, `ListToVec` provides exactly the logic that `F` should need.
The fact that implementing `F` is difficult at best even in Scala and impossible
in main many mainstream statically typed languages, while being almost trivial
in the category of sets, highlights a key point at which our model of types as
sets breaks down.

Sets really are defined in terms of their members. Do two sets have the same
elements? Then they're the same set. In the model we're considering, a type is
identified with the set of instances of that type, but of course no type carries
around a list of all its possible instantiations—think of the memory overhead!
Instead, the actual implementing of a type is more like a symbolic description
that all instantiations of that type must satisfy. To convince the type checker
that `A` and `B` are the same type, you have to convince it that every instance
of `A` is an instance of `B` and vice versa. The compiler avoids false
positives. If it says `A = B` then they really do have the same instances, but
it's very possible for the type checker to be unable to prove two types are the
same even when they are. This shows us why our model is a good one: the type
checker really does *want* to think of types as sets. It only identifies two
types when it sees they represent the same set. But it also shows a point at
which the model breaks down: sometimes the type checker just can't see that they
represent the same set even when they do.

Further, the Scala type system is very flexible, but it still isn't easy to use
just any description for a type. Per our current example, how do you actually
describe a type `A` that has to be of the form `List[T]`? How do you extract
`T` from it? Can you? I honestly don't know the answers to these questions.

A final important point about the set model: in category theory we often are
interested in objects only up to isomorphism[^isomorphism]. In programming, we
care what type something actually is, not what it's types isomorphism class is.
Consider `MyInt` from earlier. It may be isomorphic to `Int`, but that doesn't
mean I'm allowed to pass it to a function that's looking for an `Int`. This is
why we need things like traits and interfaces to convince Scala that two objects
of distinct types really can be treated the same way for some purpose.

Keeping these distinctions in mind will help you avoid loading the connection
between types and sets down with more weight than it can actually bear.


## A monoid in the category of endofunctors

We know that some of these functors are something more. Some of them are
*monads*. Let's collect a stock of examples:

- `List` ([docs](https://www.scala-lang.org/api/3.x/scala/collection/immutable/List.html))
- `Option` ([docs](https://www.scala-lang.org/api/current/scala/Option.html))
- `Try` ([docs](https://www.scala-lang.org/api/current/scala/util/Try.html))

are all monads. So are other containers like `Set`. At the end of this section
we will implement our own example from scratch. 

What elevates these functors to monads? They are functors because they are
parametric types with `map`. Now they also need some version of $\eta$ and
$\mu$. Take `List`. For each type `A`, its $\eta$ must provide a morphism from
`A` to `List[A]`. But a morphism is just a function, so we need a function that
takes in instances of `A` and gives out instances of `List[A]`. We can simply use
the `apply` method `List(a)`. That is, we map `Int` to `List[Int]` by:

```scala
1 => List(1)
2 => List(2)
...
```

and we do the same for other types. What about $\mu$? It needs to provide a
function `List[List[A]] => List[A]` for each `A`. It appears `flatten` is just
the method for the job:

```scala
List(List(1), List(2, 3)).flatten == List(1, 2, 3)
```
and in fact this works! Once can check the coherence conditions and confirm that
`List()` and `flatten` do indeed make `List` a monad.

Our other monad types are similar:

- List: $\mu$ is `flatten`, $\eta$ is `List`
- Option: $\mu$ is `flatten`, $\eta$ is `Some`
- Try: $\mu$ is `flatten`, $\eta$ is `Success`

Now, what is this doing for us? Why should we care that these types are monads?
We can think of a monad as representing a domain of computation. The
transformation $\eta$ allows us to bring new values into that domain, while
$\mu$ allows us to compose instances of computation.

The type `List` represents multiple items. We can take any object and wrap it in
a list, and if each of those items is itself multiple items, we can move from
the level of multiple-multiple items back down to multiple items.

Similarly, `Option` represents a value that may not be present. Any value can be
viewed as a value that may not be present, and if the value itself may not be
present, we can collapse (or _flatten_, see what I did there ;) back down to the
level of values that may not be present. A `Try` represents a computation that
might fail, and you can probably fill in the details analogous to `List` and
`Option` yourself.

There is a little more. We never explicitly stated the coherence conditions,
but in this context they are easier to spell out. They work out to requiring
that the following equalities always hold:

```scala
val list: List[A] = ???
List(list).flatten == list
list.map(x => List(x)).flatten == list

val nested: List[List[List[A]]] = ???
nested.flatten.flatten == nested.map(_.flatten).flatten
```

Stare at that code for a minute, and if it doesn't become obvious, work it out
for a simple example. Set `list = List(1, 2)` and `nested = List(List(List(1),
List(2, 3)))` and check that both sides are equal.

The point is that we don't need to worry about the order we apply `flatten` and
`List()`. We'll always get the same thing. You can think of this as saying that
`flatten` is associative and that `List()` cancels with `flatten`. I leave it
to you to check that these identities always hold for `List`, and that analogous
ones hold for our other examples of monads.

This may not seem like such a big deal. After all, for the examples we're
considering, it's fairly clear what the ultimate result should be. And who would
write `nested.map(_.flatten).flatten` anyway? You would always just write
`nested.flatten.flatten`, right? Two points:

1. `List` is a fairly simple monad. Not all situations will be this easy to
   reason about, and you may be glad to rely on these identities.
2. You might not write `nested.map(_.flatten).flatten` in so many words, but you
   could easily end up with code that effectively does just that. Suppose you
   have `val y = x.map(_.doSomeStuff.flatten)`, and then, maybe hundreds of lines
   later, you have `val z = y.flatten`. Later `y` gets refactored to
   `x.map(_.doSomeStuff).flatten`. Will `z` still be the same? You bet it will!
   Monads to the rescue!

We can sum this up as: **monads allow us to safely compose computation**.


### What about flatMap

We'll go on to implement our own monad, but first, you may have been wondering
about our choice of methods. Why so much emphasis on `flatten`? Aren't monads
all about `flatMap`? It turns out that the two are equivalent. We can define
`flatten` and `flatMap` in terms of each other, and while `flatten` is closer to
the category theory (it directly corresponds to $\mu$) `flatMap` is often more
convenient for programming.

The equivalence is given by:

```scala
l.flatMap(f) == l.map(f).flatten
l.flatten == l.flatMap(identity)
```

So it's really up to us whether to focus on `flatten` or `flatMap`. If we have
one, we can define the other. There are two reasons `flatMap` is often more
convenient.

The first is simply that it's flexible. It saves us having to call `map` and
`flatten` separately. We can even define `map` as well as `flatten` in terms of
`flatMap`:

```scala
l.map(f) == l.flatMap(x => List(f(x)))
```

The second, and more important reason, is that `flatMap` is usually easier to
define. It is defined on all instances of the monad, while `flatten` should only
be defined on nested instances. For example, have you ever wondered what happens
if you call `flatten` on a non-nested list such as `List(1)`? Before writing this
post, I assumed it would just return the list unchanged. But it doesn't:
`List(1).flatten` causes a compiler error.

This actually makes sense given that $\mu$ is only supposed to be defined on
$F\circ F$. But how is it achieved? We already noted that it's hard to define a
method on `List[A]` while restricting `A` to be a list itself. What would you
do? Think about it, then take a look at the [docs for
flatten](https://www.scala-lang.org/api/3.x/scala/collection/immutable/List.html#flatten-5d3).
It takes an implicit argument of type `A =>
IterableOnce[B]`[^pure-function-syntax] that allows it to iterate through each
item in the list to get all its elements and add them to the new, flattened
list. If the list isn't nested to begin with, the implicit argument won't be
found, and you'll get a compiler error.

So to implement `flatten` you actually basically need to add an extra typeclass.
The signature for `flatMap`, on the other hand, is [exactly what you'd
expect](https://www.scala-lang.org/api/3.x/scala/collection/immutable/List.html#flatMap-fffff812).
No need for type classes or any other fanciness. No need to restrict to
`List[A]` for only some `A`. Thus, if you want to define your own monad, it's
often easier to just define `flatMap` and not worry about `flatten`.


### Source

Let's consider one more example monad that will illustrate some potential
pitfalls. We will create a class `Source` that contains a computation that
returns a value:

```scala
class DeferredSource(source: () => T):
  def map[S](f: T => S): Source[S] = Source(() => f(source()))

  def run: T = source()
```

Note that `source` can be anything. It could read from a database. It could make
a network call. And those actions could have side effects. Maybe it advances an
iterator, or pops an element off of a queue. Whatever `source` does, `Source`
should hold onto it and defer it's execution until we call `run`.

To make `Sourc` a monad we also need to define `flatMap`.  Here's the first
thought you might come up with:

```scala
def flatMap1[S](f: T => S): Source[S] = f(run)
```

The type signatures all work out. It seems to satisfy the coherence conditions.
But if you're a student of tropes, you can probably tell another shoe is going
to drop. The problem is, it doesn't actually deffer computation. It runs the
computation, and then simply wraps it up in another `Source`. And because the
computation can have side effects, we loose the benefits given to us by all the
category theory we did.

This is because `flatMap1` violates what is known as *referential transparency*. An
expression is referentially transparent if we can replace it with it's evaluated
value without affecting the program. Mathematical functions are just recipes for
turning input into output. They don't *do* anything, so evaluating a
mathematical function is always referentially transparent. When we did math
earlier in this post, we assumed that if evaluating two functions always
returned the same value, they were the same function. But if a function violates
referential transparency, then applying this mathematical model is no longer
valid, and all the nice guarantees given by the category theory we did no longer
hold. So how can we make `flatMap` referentially transparent? It takes a bit of
staring, but the solution is:

```scala
def flatMap2[S](f: T => Source[S]): Source[S] = Source(() => f(run).run)
```

If source had no side effects, `flatMap1` and `flatMap2` would have the same
behavior, though we might still prefer `flatMap2` because computing `source()`
might be very resource intensive and we want to control when that computation
runs. If `source` does have side effects, the behavior can be quite different.

For example, suppose we want want to read a file path from a database and use
that path to read more data. We've set up our database methods to return
`Source`s. You might try:

```scala
val pathSource: Source[String] = DB.readPath

// The next line throws an error because we haven't populated the path yet.
val dataSource: Source[String] = pathSource.flatMap1(path => DB.readData(path))

DB.writePath("path/to/data")

val data = dataSource.run
```

It doesn't work because `flatMap1` isn't actually deferring computation, but if
you replace `flatMap1` with `flatMap2`, referential transparency is restored,
and no error is thrown because we actually defer reading the path until we call
`run`.

Well, that's about it. It is often remarked that if you truly want to learn
something, you should explain it. That has never been more true for me that in
the writing of this post. It ballooned from my original plan for a brief
description of the connection between categories and types, to over 6000 words.
If you made it this far, thanks! I hope it was helpful!


## Some optional homework

If you've read this far, hopefully you have a deeper understanding of
categories, how they can model types, and particularly of monads, and how they
can safely model computation. If you want to go use monads without worrying any
further about the math, go for it! Just in case you want to further solidify your
understanding of the theory of monad, here's an outline of some material you can
try to work through for yourself. It should fill in most of the gaps we passed
over in the post. Here are the steps:

1. Look up the definitions for
   [catgory](https://en.wikipedia.org/wiki/Category_theory),
   [functor](https://en.wikipedia.org/wiki/Functor), [natural
   transformation](https://en.wikipedia.org/wiki/Natural_transformation),
   [monoidal category](https://en.wikipedia.org/wiki/Monoidal_category), and
   [monoid object](https://en.wikipedia.org/wiki/Monoid_(category_theory)) and
   COPY THEM. This is very important. Reading a definition simply isn't the same
   as writing it out yourself and being forced to actually read every character.

1. Prove that the category of endofunctors with morphisms given by natural
   transformations, $\otimes$ given by composition of endofunctors, and $\rho$,
   $\lambda$, and $\alpha$ given by the identity, is a
   monoidal category. The tricky part is to work out how to define
   $\varphi\otimes\psi$ for two natural transformations. Try to figure out
   what it should be—you'll need it to check the coherence conditions. If you get
   stuck click below for the answer.
{{< details summary="reveal answer" >}}
Suppose $F, G, H, K$ are endofunctors with $\varphi:F\to G$ and $\varphi:H\to K$.
Let $X$ be an object. Then:

$$
(\varphi\otimes\psi)_X = \varphi_{K(X)} \circ F(\psi_X).
$$

It may help to consider the diagram:
$$
\begin{equation*}
\begin{CD}
F(H(X)) @>\varphi_{H(X)}>> G(H(X)) \\
@VF(\psi_X)VV @VVG(\psi_X)V \\
F(K(X)) @>>\varphi_{K(X)}> G(K(X))
\end{CD}
\end{equation*}
$$
Now you can check the coherence conditions.
{{< /details >}}

1. Let $F$ be a monad. Show that the coherence conditions for a monoid object
   reduce to:
   $$
   \begin{align*}
   \mu_X\circ\eta_{F(X)} &= F \\
   \mu_X\circ F(\eta_X) &= F \\
   F(\mu_x) &= \mu_{F(X)}
   \end{align*}
   $$

1. Pick a monad: `List`, `Option`, etc. Check that it is a functor. Then check
   that it is a monad.


[^1]: Actually, not all categories have objects that can be viewed as sets, but
  we needn't worry about that here.

[^functor-of-morphism]: In particular, $F(f\circ g) = F(f)\circ F(g)$ for
  covariant functors and $F(f\circ g) = F(g)\circ F(f)$ for contravariant
  functors.

[^2]: We also need $\varphi$ to play nicely with the way $F$ and $G$ map
  morphisms.


[^monoid-product-morphisms]: We also need to spell out how the monoidal product
  maps morphisms in this category. This is actually a bit tricky. See the final
  section of this post for more details.

[^functor-types]: Two points: 1) naming the method `map` is purely convention. You can call
  it whatever you like. 2) to be a functor `map` must also factor over
  composition of morphisms. For covariant functors, this works out to `a.map(x
  => f(g(x))) == a.map(g).map(f)`, and for contravariant functors it's `a.map(x
  => f(g(x))) == a.map(f).map(g)`. Confirming that these conditions are
  satisfied for most common functors is straightforward.

[^isomorphism]: An isomorphism in a category is a morphism $f: A\to B$ that has
  an inverse morphism $f^{-1}:B\to A$ such that $f\circ f^{-1} = id_A$ and
  $f^{-1}\circ f = id_B$.

[^pure-function-syntax]: Actually, because of an experimental language feature,
  the type is `A -> IterableOnce[B]`. See
  [here](https://docs.scala-lang.org/scala3/reference/experimental/purefuns.html)
  for details.
