+++
draft = true
date = 2026-09-18T20:59:12-05:00
title = "Ad Hoc Polymorphism in Rust and Scala"
description = "A comparison of Rust traits and Scala type classes."
authors =  ["Luke Wassink"]
tags = ["scala", "functional programming"]
series = []
+++

It is common when learning a new language to find the one's understanding of
one's first language is deepened. The same, of course, is true of programming. 
This post will compare Scala type classes with Rust traits as alternative
methods of implementing ad hoc polymorphism. I will suggest that, on a certain
understanding, both can be viewed as methods of implementing type classes,
though Scala's are somewhat more flexible.

My experience is that comparing different approaches helps to conceptually strip
away the implementation details to reveal the truly fundamental functionality
that a language provides. Rust traits are also, in my opinion, somewhat simpler
than Scala type classes, since they do not depend on the machinery of implicit
parameters. Thus, since they provide similar functionality, this comparison may
be helpful for anyone struggling to understand type classes in Scala.

I will first explain Scala type classes and Rust traits, then conclude with a
more general discussion of ad hoc polymorphism and type classes and a comparison
of the approaches of these two languages.


# Scala Type Classes

Let's start with 
