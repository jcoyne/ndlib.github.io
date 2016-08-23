---
author:   jeremyf
category: practices
filename: 2014-10-06-coherent-development-part-1.md
layout:   post
tagline:  We Are Having a Conversation
title:    Coherent Development (Part 1)
tags:     rails, design, projecthydra, coherent-development
---

Last week during [Hydra Connect 2014](https://wiki.duraspace.org/display/hydra/Hydra+Connect+2+Program) at [Case Western Reserve University](http://www.case.edu/), I presented on [Taming Hydra and Rails](https://speakerdeck.com/jeremyf/taming-hydra-and-rails). The presentation was going to be 30 minutes but I condensed it into 5 minutes.

Over the next few weeks I hope to write several blog posts delving into how I hope to write more shareable code.
These posts are from my observations on developing applications by myself and developing applications that are to be shared by multiple institutions.
The posts will leverage sources that I've read and followed.

My general conjecture is that Rails provides you tools for writing **your** application very fast.
It does not provide solid guidance on:

* maintaining your application
* decomposing your application into smaller parts
* sharing your application with others in a configurable manner

I have written code that was inflexible and painful to use.
I have also written code that is a joy to work with, as it remains flexible yet cohesive.

Topics that I'm going to explore are:

1. [Code is a Conversation](/practices/code-is-a-conversation)
1. [Never Unprepared for Collaboration](/practices/never-unprepared-for-collaboration)
1. [ActiveRecord Pattern and Alternatives](/practices/active-record-pattern-and-alternatives)
1. [Rails: Come for the Active Record, Stay for the Testing](/practices/rails-come-for-the-activerecord-stay-for-the-testing)
1. [Test Driven Development](/practices/my-developoing-philosophy-of-test-driven-development)
1. [Modes of Development](/practices/modes-of-development)
1. [Pay Attention to Pain Points](/practices/pay-attention-to-pain-points)
1. [Leveraging Instance Variables in Ruby Classes](/practices/leveraging-instance-variables-in-ruby-classes)
1. Fast Tests
1. S.O.L.I.D. Design Principles
1. Command / Query Separation
1. Controller Action's as Objects
1. [Composition over Inheritance](/practices/favor-composition-over-inheritance)

I don't have comments enabled for this blog, but I want this to be an ongoing conversation.
So if you want to engage with me consider:

* Submitting an [Issue or Pull Request to the ndlib.github.io blog](https://github.com/ndlib/ndlib.github.io).
* Asking for [clarity on Twitter](https://twitter.com/jeremyfriesen)
* Reaching out on hydra-tech@googlegroups.com
