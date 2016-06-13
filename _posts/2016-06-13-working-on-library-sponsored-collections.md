---
author:   jeremyf
category: practices
filename: 2016-06-13-working-on-library-sponsored-collections.md
layout:   post
tagline:
title:    Working on Library Sponsored Collections
tags:     ruby, rails, design, coherent-development
---

* Background
  * What we already have
  * What we needed
  * An observation regarding implementation details
  * Our digital library collections librarian would be administering the collection building
  * Build from specification documentations
* Planning
  * Whiteboarding session for graphs
    * Discuss likely graph structure
    * Discuss likely snags or emergent structures to account for
    * Pictures or it didn't happen
  * Graphs
    * DOT Notation
    * Recursion
    * Cycle Avoidance via Time to Live
* Foundation
  * Need for Fast Tests
    * Didn't want to fire up Fedora and SOLR, persist documents, then reindex multiple documents
  * Modeling basic document needs
  * Modeling processing needs
  * Stubbing both Index and Preservation Layer
  * Aside for Software Mentoring
    * Breadth First Search
* The Solution
  * TODO: Find the final snap shot and aggregate that into a single file
  * Bottom up design
    * Test the stubs for Index and Preservation storage
    * Write the types of data structures
    * Extracting small methods and collaborating classes
  * Index from node up through ancestors
  * Adding bi-directional information to the index document
  * Handling the "add to the graph" behavior
  * Wrote a "compact" graph notation in Ruby
  * Discovery that updating the index when an item is removed from the graph is rather challenging
* Pivot
  * Friday afternoon email: "What if we stored the tree information?"
  * Monday morning design session
    * Whiteboarding with scenarios and requisite information
    * Throwing out a few options
    * Index from node down through descendants
    * For now ignore the "rebuild the entire index"
* Solution II
  * Monday afternoon implementation session
  * Leverage lessons already learned from first iteration
    * Preserved the existing implementation
    * Wrote new implementation within the spec file
      * Allowed for faster iteration
  * Focus on describing the before and after state of an event
    * Established a structured description of states and events so I could create additional scenarios
* Cleanup
  * Describe the public facing elements via the `@api public` documentation declaration
    * This is my contract to the outward facing behavior
    * Why `@api public` is important
  * Privatize classes, constants, and methods to reduce surface area of obligation
    * This is a natural extension of `@api public`
  * Write documentation
  * Refactor spec methods to remove repetition and better describe what is happening
* Bootstrapping to Solution II
  * Explore how to create the initial index elements
  * Describe with similar specifications
* Integration
  * Account for an object's index has changed between retrieval and update
    * Reconcile with ETags
  * Build an adapter to execute the indexing strategy against the Fedora model and SOLR document
  * Create an asynchronous job to handle the heavy lifting
  * Tightening private methods by introducting public adapter methods
    * Example: `Curate::Indexer.find_preservation_object_for_indexing(pid)`
