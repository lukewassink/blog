+++
draft = true
date = 2026-09-06T15:33:45-04:00
title = "From Categories to Types"
description = "Explains how category theory maps onto types"
authors = ['Luke Wassink']
tags = ["math", "scala", "functional programming"]
series = []
+++

# Who is this for?

At this point, there are approximately 80,000 blog posts and Youtube videos that
explain how monads fit into functional programming. It seems to be a fundamental
natural law of the universe that 99% of them begin by reassuring you
that, unlike all the other explanations out there, *this explanation* won't use
any fancy math or require any knowledge of category theory. This blog post
belongs to the remaining 1%.

I'm writing it for me from two months ago. Two months ago me liked functional
programming and had actually taken a whole course on category theory back in the
day, but he just couldn't seem to see exactly how the two fit together. He
trawled through video after blog post, only to be reassured once again that
"*This explanation* won't involve any category theory." Eventually he made his
way down into the depths of Wikipedia, and other dark corners of the internet to
dredge up this hidden knowledge. So now I'm writing the blog post that
two-months-ago me wished someone else had written. A post that explains a little
bit about category theory (but with lots of links to other places that say more)
and explains even less about functional programming (Google it-there's stuff out
there) but does it's best to lay out the connection between the two as clearly
and concretely as possible.

So it might be a post written only for its author. But maybe not. Read on if:

- you want a very high level, and hopefully somewhat friendly, introduction to
    category theory
- you want to know what category theory has to do with types, and exactly which
    types are functors
- you want a discussion of monads that does it's very best to clearly explain
    what it means that monads are monoids in the category of endofunctors, and
    why that's a nice thing for your programs.


## There are monoids and monoids

First, a disambiguation. If you Google monoids, you'll quickly learn that a
[monoid](XXX) is a set with a binary, associative operation with identity. For those in
the know, it's just a group that doesn't have to have inverses, or a semigroup
that does have an identity element.

More concretely, it's a set of things with some operation we can write $*$ that
has to satisfy:

1. $a * (b * c) = (a * b) * c$ for any elements $a, b, c$,
1. there is an element $e$ such that $e * a = a * e = a$ for any element $a$.

Examples: non-negative integers under addition (zero is the identity); $n\times
n$ matrices under multiplication (the identity matrix is the identity).

Now the million dollar question: what does this have to do with types?
Absolutely nothing! Or at least, not directly. To see the connection we'll have
to go through category theory, and meet another definition of monoid. (Or is it
secretly The same definition?.  Let's find out.)


## Abstract nonsense

Mathematicians love to generalize. You work with enough things that have some
notion of multiplication, and you start to notice common features: Most of them
are associative. Lots of them have inverse elements. And so the notion of a
[group](XXX) is born. Similarly, there are lots of structures where you can add
and multiply, and multiplication distributes over addition, and so you have
[rings](XXX).

Not content to stop there, mathematicians have generalized yet further. Once you
study groups and rings, and maybe also [modules](XXX) and some other things, you
start to notice common patterns in the ways they fit together and the things you
can do with them. And that's [category theory](XXX).

More precisely, category theory studies categories. In a category, we have:

- **Objects:** the things in the category. These objects could be sets, or
    something fancier like groups or rings.
- **Morphisms:** just think of morphisms as arrows. Each morphism points from
    one object to another. In the category of sets, morphisms are just
    functions, for example. We also need a notion of composition of morphisms
    denoted $\circ$. If I have morphisms $f: A\to B$ and $g:B\to C$, then I get
    a morphism $g\circ f: A\to C$.

There are a few other conditions the morphisms have to satisfy, but this is as
far as I'll go. What do you think all these links are for?

The example to keep in mind is the category of sets. Objects are sets, morphisms
are functions, and composition of morphisms is just ordinary composition of
functions.

It's important to note that morphisms are first class citizens in this
definition, and while there might be a natural choice for some set of objects,
nothing forces us to make that choice. Keep the objects but change the
morphisms, and you get a different category.

At first we had sets of concrete things and functions from one thing to another.
Then we got whole sets of things (say, groups) and maps from one set to another
(say, group homomorphisms). Now we have categories of sets of things[^1], and
naturally we want maps from one category to another. These maps are called
[functors](XXX).

Say we have categories $\mathcal{C}$ and $\mathcal{D}$. A functor
$F:\mathcal{C}\to\mathcal{D}$ gives you an object $F(A)$ in $\mathcal{D}$ for
every object in $\mathcal{C}$. But that's not enough. We also need to say what
happens to morphisms. If we have a morphism $f$ from $A$ to $B$, then we should
get a morphism $F(f)$ from $F(A)$ to $F(B)$. And finally, $F$ had better play
nice with composition (it's left as an exercise to the reader to work out
exactly what this should require). Again, what happens to the morphisms is up to
us. There may be a most natural or useful choice, but just choosing which
objects go to which doesn't force that choice on us.

Let's make this more concrete. Consider the category of sets. A functor $F$
would need to do two things:

1. for each set $A$, give us another object $F(A)$,
2. for each function $f$, give us a morphism $F(f)$.

First a very simple example, then a more interesting one. How can we get
something for each set? Why not just the set itself? And what about morphisms?
Let's do the same thing: the morphism itself. So $F(A) = A$ and $F(f) = f$ for
all sets and all functions.  This is called the identity functor.

Now something more interesting. Consider the functor $W$ that takes in a set and
returns that set wrapped in another set: $F(A) = \\{A\\}$. So:

$$
\\begin{align*}
F(\\emptyset) &= \\{\\emptyset\\} \\\\
F(\\{1, 2\\}) &= \\{\\{1, 2\\}\\}
\\end{align*}
$$

What should $F$ do to functions? A logical choice would be just apply them to
the wrapped set. So $F(f)(F(A)) = \\{f(A)\\}$.

Here's something interesting about both the functors we mentioned: they both map
from the category of sets, *back to the category of sets*. Not all functors have
to do that. We can have a functor $F:\\mathcal{C}\\to\\mathcal{D}$, but a
functor $F:\\mathcal{C}\\to\\mathcal{C}$ has a special name. It's called an
*endofunctor* of $\\mathcal{C}$. It's starting to come together. How do we get
from here to a monoid in the category of endofunctors? We start with monoidal
categories.

A [monoidal category](XXX) is a category where we can take the product of
objects. For any two objects $A$ and $B$, there is an object $A\times B$. To be
a bit more precise, $\times$ is a binary functor, so we can also take the
product of morphisms, and the product has to be associative, and everything has
to play nicely together. There also has to be an identity object $I$ with the
property that $I\times A$ is [isomorphic](XXX) to $A$.

Don't sweat the details (unless you want to, in which case go for it!). The
product in the category of sets which is our focus, is just the ordinary
product: the set of pairs of elements from the two sets. For example:

$$
\\{1, 2\\} \times \\{a, b, c\\} = \\{(1, a), (1, b), (2, a), (1, b), (2, b), (1, c), (2, c)\\}.
$$

For the identity object, just pick any set with one element. You get an
invertible mapping from $A\times \\{x\\}\to A$ by just mapping $(a, x)$ to $a$;
the inverse is $a$ maps to $(a, x)$.

Now, inside a monoidal category, we can have a [monoid object](XXX) (I know). A
monoid object $M$ needs two special morphisms:

$$
\\begin{align*}
\\mu&: M\times M\to M \\\\
\\eta&: I\to M
\\end{align*}
$$

Perhaps you see why we need the monoidal category? It's so $M\\times M$ exists.
Of course, $\\mu$ and $\\eta$ have to satisfy some constraints that we'll say a
bit more about later so that everything works nicely together.

Now let's translate this to our trusty category of sets. The multiplication
morphism, as $\\mu$ is often referred to, is going to be a map from $M\times M$
to $M$, so it takes in a pair of elements of $M$ and returns a single element of
$M$. For old times sake, let's use our example of non-negative integers from the
section on monoids. We can define $\\mu(a, b) = a + b$ and $\\eta(x) = 0$, where
$x$ is the lone element in your favorite single-element set.

This has the nice consequence that $\\mu(\\eta(x), a) = \\mu(0, a) = 0 + a =
a$. This turns out to be one of those constraints we were talking about a couple
paragraphs ago, so non-negative integers along with this choice of $\\mu$ and
$\\eta$ turns out to be a monoid object in the category of sets.

Let's take stock. We've defined monoids. We've defined endofunctors. Now it's
time for, you guessed it, **monoids in the category of endofunctors!** You see,
mathematicians are perverse and masochistic. They had things like numbers, and
ways to get from one thing to another. Then they built sets of those things, and
gave some of those sets structure to make groups and rings and vector spaces.
Each of those has their own preferred kind of map that let you get from one
structure to another. Then mathematicians took all the sets, all the groups, all
the rings, and gathered each of them together to make the category of sets, and
the category of groups, and the category of rings. And they defined maps between
categories, and called them functors. 

Well, why stop there? We can now consider all the functors from some category to
itself-all the endofunctors-and *those* form a category. What are the morphisms
in this insane category? They're things called [natural transformations](XXX).
To do this job, a natural transformation must map one functor to another. How
can we do that. Let's consider two endofunctors $F$ and $G$. Take a particular
object $Z$, to make things more concrete, and set $Y = F(X)$ and $Z = G(X)$. We
can picture the situation as:

$$
\\begin{tikzcd}
	& X & \\\\
	\\\\
	Y && Z
	\\arrow["F", from=1-2, to=3-1]
	\\arrow["G"', from=1-2, to=3-3]
\\end{tikzcd}
$$


[^1]: actually, not all categories have objects that can be viewed as sets, but
  we needn't worry about that here.
