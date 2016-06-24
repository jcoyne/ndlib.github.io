---
author:   jeremyf
category: practices
filename: 2016-06-24-solving-library-sponsored-collections.md
layout:   post
tagline:  Rapid Prototyping
title:    Solving Library Sponsored Collections
tags:     ruby, rails, design, coherent-development
---

This is the third part in a series of posts regarding [CurateND's](https://curate.nd.edu/) Library Collection development.
The series will dive into the details of implementing a new type of collections within our [Hydra ecosystem](https://projecthydra.org/).

Before reading this post, please read [the previous post for planning information](/practices/2016-06-23-planning-library-sponsored-collections).

[Here is a link to the final repository state for this solution](https://github.com/ndlib/curate-indexer/tree/298746c).

## Building a Gem for Rapid Testing

As I stated earlier, fast tests were a requirement. So instead of attempting to incorporate this logic into the existing CurateND code-base, I opted to create a [small gem](https://github.com/ndlib/curate-indexer) to iterate through these solutions.

One metric to consider is that simply loading the test suite for CurateND requires 10 seconds.
Whereas loading the test suite for Curate::Indexer requires about 1/10th of a second.

With the gem in place, I wired up Rubocop and SimpleCov to help ensure that I'm writing consistent code and that its tested.

## Test Driving Lap #1

I knew that I was going to need different objects:

* Preservation object (i.e. the Fedora object)
* Index object (i.e. the SOLR document)

I also suspected, based on our initial modeling that we would need a Processing Object; Something that could track the state of what had been visited.

Here was my initial guiding scenario (its a bit chatty):

```ruby
module Curate
  RSpec.describe Indexer do
    before { Indexer::Persistence.clear! }

    context 'Graph Scenario 1' do
      let!(:collection_a) { Indexer::Persistence::Collection.new(pid: 'a') }
      let!(:collection_b) { Indexer::Persistence::Collection.new(pid: 'b', is_member_of: [collection_a.pid, collection_d.pid]) }
      let!(:collection_c) { Indexer::Persistence::Collection.new(pid: 'c', is_member_of: [collection_b.pid]) }
      let!(:collection_d) { Indexer::Persistence::Collection.new(pid: 'd') }
      let!(:collection_e) { Indexer::Persistence::Collection.new(pid: 'e') }
      let!(:collection_f) { Indexer::Persistence::Collection.new(pid: 'f') }
      let!(:collection_g) { Indexer::Persistence::Collection.new(pid: 'g') }
      let!(:work_1) { Indexer::Persistence::Work.new(pid: '1', is_member_of: [collection_a.pid, collection_e.pid]) }
      let!(:work_2) { Indexer::Persistence::Work.new(pid: '2', is_member_of: [collection_b.pid]) }
      let!(:work_3) { Indexer::Persistence::Work.new(pid: '3', is_member_of: [collection_c.pid]) }
      let!(:work_4) { Indexer::Persistence::Work.new(pid: '4', is_member_of: [collection_d.pid]) }
      let!(:work_5) { Indexer::Persistence::Work.new(pid: '5', is_member_of: [collection_f.pid]) }
      let!(:work_6) { Indexer::Persistence::Work.new(pid: '6') }

      context 'when building index for Work 2' do
        it 'will be direct in Collection C and transitive in B, A, D' do
          # Do the work
          response = Indexer.reindex(pid: work_2.pid)

          expect(response.is_member_of).to eq([collection_b.pid])
          expect(response.is_transitive_member_of).to eq([collection_b.pid, collection_a.pid, collection_d.pid])
          expect(response.has_collection_members).to eq([])
          expect(response.has_transitive_collection_members).to eq([])

          indexed_collection_b = Indexer::Index::Query.find(collection_b.pid)
          expect(indexed_collection_b.is_transitive_member_of).to eq([collection_a.pid, collection_d.pid])
          expect(indexed_collection_b.is_member_of).to eq([collection_a.pid, collection_d.pid])
          expect(indexed_collection_b.has_collection_members).to eq([work_2.pid])
          expect(indexed_collection_b.has_transitive_collection_members).to eq([work_2.pid])

          indexed_collection_a = Indexer::Index::Query.find(collection_a.pid)
          expect(indexed_collection_a.is_transitive_member_of).to eq([])
          expect(indexed_collection_a.is_member_of).to eq([])
          expect(indexed_collection_a.has_collection_members).to eq([collection_b.pid])
          expect(indexed_collection_a.has_transitive_collection_members.sort).to eq([work_2.pid, collection_b.pid].sort)

          indexed_collection_d = Indexer::Index::Query.find(collection_d.pid)
          expect(indexed_collection_d.is_transitive_member_of).to eq([])
          expect(indexed_collection_d.is_member_of).to eq([])
          expect(indexed_collection_d.has_collection_members).to eq([collection_b.pid])
          expect(indexed_collection_d.has_transitive_collection_members.sort).to eq([work_2.pid, collection_b.pid].sort)
        end
      end
    end
  end
```

From my [initial working commit](https://github.com/ndlib/curate-indexer/tree/235a066) to the [final state of this solution](https://github.com/ndlib/curate-indexer/tree/298746c), there were lots of collaborating objects that I leveraged to break apart the problem:

* `Reindexer`: Coordinates the reindexing of the entire direct relationship graph
* `IndexingDocument`: Responsible for representing an index document
* `RuntimeError`: Namespacing for common errors
* `ReindexingReachedMaxLevelError`: An exception thrown when a possible cycle is detected in the graph.
* `Queue`: An assistive class in the breadth first search.
* `Index`: Represents the interaction with the index
  * `Index::Rebuilder`: Responsible for co-ordinating the rebuild of the index
  * `Index::Document`: Responsible for representing an index document (extends IndexingDocument)
  * `Index::Query`: Contains the Query interactions with the Index
* `Processing`: Responsible for coordinating all of the building process of the new index data.
  * `Processing::Builder`: Responsible for building a processing document by "smashing" together a persisted document and its index representation.
  * `Index::Document`: Represents a document under processing (extends IndexingDocument)
* `Persistence` Responsible for being a layer between Fedora and the heavy lifting of the reindexing processor. It has aspects that will need to change.
  * `Persistence::Document`: This is a disposable intermediary between Fedora and the processing system for reindexing. I believe it is a good idea to keep separation from the persistence layer and the processing. Unlike the IndexingDocument, the Persistence Document should only have the direct relationship. (extends ProcessingDocument)
  * `Persistence::Collection`: (extends Persistence::Document)
  * `Persistence::Work`: (extends Persistence::Document)
* `CachingModule`: There are several layers of caching involved, this provides some of the common behavior.

There are a lot of objects to consider; but the fundamental thing to remember is that we are:

* Loading a Fedora object and its ancestors
* Retrieving from SOLR each loaded Fedora object
* Processing the subgraph
* Then writing back to SOLR with the updated relationship information.
* Recursive methods
* Possible cycles in the graph
* Writing bi-directional graph information to each index document

Along the way Rubocop and SimpleCov were providing guidance.
There were points in which methods were too long or complex.
I spent a bit of time refactoring those, and what emerged was more descriptive method (see ["Refactoring to appease rubocop" commit](https://github.com/ndlib/curate-indexer/commit/424924b1b42e1d24bc2314c7369290f4c880dad5)).

## Key Abstraction

The key abstraction in this process was the Index cache and Persistence cache. Instead of interacting with CurateND's full Persistence layer (i.e. Fedora) and Index layer (i.e. SOLR), I wrote an in memory document store that I could run my tests against.

I had custom documents with what I thought would be the bare interface for interaction (i.e. a PID and relationships attributes).

I found this abstraction key in focusing on the harder problem of graph traversal and reindexing. It allowed for very rapid feedback cycles.

At its slowest the entire build suite – Rubocop, RSpec, and SimpleCov - takes about 1.5 seconds of elapsed time (on my machine). This meant I could refactor, extend, refine, and explore in very rapid fire.

It also meant, when I hit a brick wall with this solution, my code was not already entwined in the application. And did I hit a brick wall.

## Up Next

In the next installment, I'll talk about the Pivot.
