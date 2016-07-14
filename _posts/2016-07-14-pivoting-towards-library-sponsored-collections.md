---
author: jeremyf
category: practices
filename: 2016-07-14-pivoting-towards-library-sponsored-collections.md
layout: post
tagline: Its Just a Step to the Right
title: Pivoting Towards Library Sponsored Collections
tags: 'ruby, rails, design, coherent-development'
---

This is the fourth part in a series of posts regarding [CurateND's](https://curate.nd.edu/) Library Collection development. The series dives into the details of implementing a new type of collections within our [Hydra ecosystem](https://projecthydra.org/).

Before reading this post, please read [the previous post regarding an initial solution](/posts/2016-06-24-solving-library-sponsored-collections).

# Change is the Only Constant

Its Thursday, I'm almost done with my initial Solution. But there are storm clouds.; The current solution works only for additions to the graph structure. It does not account for removing `isMemberOfCollection` edges.

And my colleague - Don - shoots me the following HipChat:

> The tree code is just trying to index an item into solr, right? It might be easier than we thought because we could just assign "tree numbers" to items. And then it will behave like the hierarchal facets.

From there, he describes an alternate strategy from the one I had implemented. Instead of storing Ancestors or Descendants, we would store the ancestry path.

Building on our proposed proposed starting state scenario:

![Library Collections Starting State](/images/collections.png)

We would need to build the following index entries:

node | parents | ancestors                                   | pathnames
---- | ------- | ------------------------------------------- | -----------------------------
a    | []      | []                                          | [a]
b    | [a]     | [a]                                         | [a/b]
c    | [a, b]  | [a, a/b]                                    | [a/c, a/b/c]
d    | [b, c]  | [a, a/b, a/c, a/b/c]                        | [a/b/d, a/b/c/d, a/c/d]
e    | [b, c]  | [a, a/b, a/c, a/b/c]                        | [a/b/e, a/b/c/e, a/c/e]
f    | [e]     | [a, a/b, a/b/e, a/c, a/c/e, a/b/c, a/b/c/e] | [a/b/e/f, a/b/c/e/f, a/c/e/f]

# This Is a Big Change

I'm thinking to myself; _I just spent a decent chunk of time on the previous solution, can I retrofit that solution? I could see that the current solution was still incomplete._

I wasn't going to be walking up the ancestor path, but instead down the descendants, so some of the previous objects didn't make sense. But others continued to make sense (a representation of a Fedora object and a Solr document).

I set aside the problem for a week, focusing on other issues. And on the following Friday morning, Don and I spent a considerable amount of time white boarding and discussing different graphs and scenarios.

# Lets Get Down to Business

I went to lunch, and when I came back, I decided to "start over" within the same code-base. My plan was as follows:

- Create a feature spec that describes the starting state, change, and expected end state
- Create the requisite classes inside that spec file so I could quickly iterate on the same file

[Here is the first commit of the second solution](https://github.com/ndlib/curate-indexer/blob/483fa08c10c853ab00e66dadd1f59c3fd8e09f27/spec/features/reindex_descendants_spec.rb). It is a single spec file with the self-contained logic with a feature spec. **In keeping the implementation close to the spec, I was able to move quickly on the code. During this time, I still had the other partial solution that was also being tested.**

I was able to work through this solution in short order because I had explored the previous problem and knew some of the likely collaborators. I also leveraged several micro-libraries to aid in the rapid building of these new objects (e.g. [dry-initializer](https://github.com/dry-rb/dry-initializer) , [dry-equalizer](https://github.com/dry-rb/dry-equalizer), and [dry-types](https://github.com/dry-rb/dry-types)).

My tests remained very fast (0.5 seconds to load and execute everything).

The important "discovery" was encoding the initial state, change, and expected state in a tabular format:

```ruby
{
  name: 'A semi-complicated graph with diamonds and triangle relationships',
  starting_graph: {
    parent_pids: { a: [], b: ['a'], c: ['a', 'b'], d: ['c', 'e'], e: ['b'] },
    ancestors: { a: [], b: ['a'], c: ['a/b', 'a'], d: ['a', 'a/b', 'a/b/c', 'a/b/e', 'a/c'], e: ['a', 'a/b'] },
    pathnames: { a: ['a'], b: ['a/b'], c: ['a/c', 'a/b/c'], d: ['a/c/d', 'a/b/c/d', 'a/b/e/d'], e: ['a/b/e'] }
  },
  preservation_document_attributes_to_update: { pid: :c, parent_pids: ['a'] },
  ending_graph: {
    parent_pids: { a: [], b: ['a'], c: ['a'], d: ['c', 'e'], e: ['b'] },
    ancestors: { a: [], b: ['a'], c: ['a'], d: ['a', 'a/b', 'a/b/e', 'a/c'], e: ['a', 'a/b'] },
    pathnames: { a: ['a'], b: ['a/b'], c: ['a/c'], d: ['a/c/d', 'a/b/e/d'], e: ['a/b/e'] }
  }
}
```

The above Ruby structure is a representation of the above image. The `preservation_document_attributes_to_update` shows that we are going to change node `c`'s `isMemberOfCollection` from `['a', 'b']` to `['a']`.

By formalizing the structure of state change, I have an opening to write more scenarios.

# Up Next

In the next installment, I'll talk about the second, more complete solution.
