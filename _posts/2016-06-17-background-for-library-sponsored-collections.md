---
author:   jeremyf
category: practices
filename: 2016-06-17-background-for-library-sponsored-collections.md
layout:   post
tagline:  There are many ways to collect things
title:    Background for Library Sponsored Collections
tags:     ruby, rails, design, coherent-development
---

This is the first part in a series of posts regarding [CurateND's](https://curate.nd.edu/) Library Collection development.
The series will dive into the details of implementing a new type of collections within our [Hydra ecosystem](https://projecthydra.org/).

## What We Have

CurateND is Notre Dame's institutional repository.
Within CurateND, Notre Dame's Intitutional Repository, there exists a concept of collections.
These collections are built upon the now deprecated [Hydra::Collections gem](https://github.com/projecthydra-deprecated/hydra-collections).

The current collections are implemented as user-defined collections – a user can create a collection then add any items that they can see into this collection.
For disambiguation I'll refer to these as User Collections.

Over the past few years, there have been ongoing conversations concerning collections:

* [Hydra Connect I notes on Collections](https://wiki.duraspace.org/display/hydra/Collections+and+Admin+Sets)
* [February 2016 Developers Congress notes on Collections](https://wiki.duraspace.org/display/hydra/User+Collections%2C+Admin+Sets%2C+Display+Sets)

## What We Need

In short, we need "Library Sponsored Collections" - objects that are curated into a logical group by Library staff.
Those curated objects would then be part of a library anointed collection.

User Collections have `hasCollectionMembers` predicates on the User Collection object.
The "collected" objects have no assertions regarding membership in User Collections.
A "collected" object's index document does have details of that object's membership.
*Remember, a user can collect any objects they can see; Even ones that they cannot edit.*

Library Collections have no assertions regarding their members.
Instead "collected" objects describe their membership in Library Collections via a `isMemberOfCollection` predicate.
An implementation detail is that in order for someone to assert that an object `isMemberOfCollection` they will need permission to update that object.

In the future we need both Library Collections and User Collections.
For now most of User Collections features have been disabled via the UI.

[Here is a link to our proposed Library Collections requirements for CurateND](https://docs.google.com/document/d/1ZjJz0tyEUsxpYo0oXsvb4lcoscFZ2qAsM2iJLdBvUyc/edit?usp=sharing).

### Tangential Aside

User Collections were implemented during the Curate/Hydramata 2014 sprints.
We, the developers and product owners, were building from a self-deposit mindset in which User Collections make sense.
However, our institutions have a strong desire for Library Collections, and for us, they are at a higher priority than User Collections.

One consideration is that the library would consider creating a custom page for a Library Collection.
This is not something we would do with User Collections.

I consider the development of User Collections to have been a failure in our prior specification process; I believe it is a requirement for our repository, but it just happens to be a much lower priority than Library Collections.

## An Observation Regarding Implementation Details

One of the major challenges of this development is that many of the collection concepts already exist.
They happen to be implemented as an inversion of what we want with Library Collections.
We were thinking that we could migrate and/or deprecate all existing collections.

As I was digging through [the code](https://github.com/ndlib/curate_nd), it became clear that the best approach would be to create a new model and new relations. (The User Collection logic is a labyrinth of module mixins upon mixins; a code-smell if ever there was one).

Also, Library Collections were different in predicate usage than User Collections.

We also wanted to account for [Cycles in a Graph](http://www.geeksforgeeks.org/detect-cycle-in-a-graph/). One example of a trivial cycle is:

* `A isMemberOfCollection B`
* `B isMemberOfCollection C`

[Our Digital Collections Librarian – Hello Ruth!](https://github.com/ruthtillman) - would be administering the initial Library Collections via our [Batch Ingestor](https://github.com/ndlib/curatend-batch). Any UI management is part of a future project.

## Up Next

In the [next installment](/posts/2016-06-23-planning-library-sponsored-collections), I'll talk about our planning.
