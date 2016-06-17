---
author:   jeremyf
category: practices
filename: 2016-06-17-complete-solution-for-library-sponsored-collections.md
layout:   post
tagline:  A Second Iteration
title:    Complete Solution for Library Sponsored Collections
tags:     ruby, rails, design, coherent-development
---

* Solution II
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
