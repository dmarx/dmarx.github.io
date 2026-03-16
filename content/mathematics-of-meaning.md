---
layout: post
title: "Concept Geometry"
excerpt: ""
tags: [philosophy]
date: 2026-03-15
created: 2026-03-15
comments: true
---

## Concept Vectors

A recent paper intersected with ideas I have been developing independently, concurrently. In it, they
connect the Linear Representation Hypothesis (LRH) (which I would have probably just referred to as the
Distributional Hypothesis, from semantics) to Formal Concept Analysis (FCA). They call their thing the
"Lattice Representation Hypothesis" (LatRH), and cache their claims specifically in the operation of LLMs. I agree
with the authors on a lot of their work: it's very nearly a paper I've been meaning to write myself and 
have annoyingly left distributed across piles of notes instead of polishing it into a publishable form.
But that's neither here nor there. The authors leave a lot of value on the table: by connecting the LRH
to FCA, we're able to make inferences about mathematical/geometric properties that concepts must necessarily
be subject to.

As a concrete example: the LatRH paper points out how ReLU induces a half-space, which is how they
characterize concepts. This is a minor implementation detail in the neural network: literally any decision
boundary induces a half space. The broader and more interesting connection here is that FCA concepts are
mappable to 0-1 test functions, i.e. predicates.

The real meat left on the bone here, imho, is by failing to recognize the significance of mapping concepts to
*vectors*. The half space isn't the concept, it's a *judgement*. For example, `is_dog` is an example of a
predicate that easily maps to a binary test function of the kind FCA would map to the "concept" of "a dog". 
My contention though is that the "concept" being invoked here isn't actually "a dog" but rather "dog-*ness*".
The concept induces an ordering. The half space characterizes a particular judgement wrt that concept. 

What's strictly more powerful here is that by treating concepts as vectors, we can differentiate between
complementation and negation. This is an issue FCA struggles with, but is trivial when we reason about concepts
as vector spaces.


## Vector Geometry

A vector is a direction. It's usually modeled visually as an arrow. One way you can think about this is as an 
ordering: less to more, negative to positive, etc.

Adopting the predicate perspective, every non-trivial vector induces a tangent space. The consequence of this is that
you can always slice vector space by a separating hyperplane that characterizes a decision threshold relative to 
the concept ordering. This hyperplane gives us the half-space version of a "concept" described in the LatRH paper.
Proximity to this hyperplane gives us magnitude in the concept's vector space, and is the reason we necessarily have
a poset here: worst case, all items are equidistant from the boundary, but we will always have some notion of proximity 
to this boundary even if the items in our space are permutation invariant relative to it.

In a one dimensional vector space, we only have magnitude: a number line. This is the simplest case for understanding a concept 
relative to a decision threshold. When the concept's predicate form is satisfied, we have the positive half space, and the complement
of the predicate is the negative half space. simple enough.

* pie chart
* similarity
* complement
* 3D
* cone emeanating from origin to surface
* double-cone
* complement *contains* "anti-region": quotient vs negation
