---
author: jeremyf
category: practices
filename: 2016-07-15-complete-solution-for-library-sponsored-collections.md
layout: post
tagline: A Champion Emerges
title: Complete Solution for Library Sponsored Collections
tags: 'ruby, rails, design, coherent-development'
---

This is the fifth part in a series of posts regarding [CurateND's](https://curate.nd.edu/) Library Collection development. The series dives into the details of implementing a new type of collections within our [Hydra ecosystem](https://projecthydra.org/).

Before reading this post, please read [the previous post regarding the pivot](/posts/2016-07-14-pivoting-towards-library-sponsored-collections) to this solution.

# Formalized Scenarios

With a formalized scenario schema (and provided data), I was able to whip up a generalized spec that could run.

```ruby
module Curate::Indexer
  RSpec.describe 'Reindex pid and descendants' do
    before do
      # Ensuring we have a clear configuration each time; Also assists with code coverage.
      Curate::Indexer.configure { |config| config.adapter = Curate::Indexer::Adapters::InMemoryAdapter }
      Curate::Indexer.adapter.clear_cache!
    end

    # Begin formalized scenario schema
    [{
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
    }].each_with_index do |the_scenario, index|
      context "#{the_scenario.fetch(:name)} (Scenario #{index})" do
        let(:starting_graph) { the_scenario.fetch(:starting_graph) }
        let(:preservation_document_attributes_to_update) { the_scenario.fetch(:preservation_document_attributes_to_update) }
        let(:ending_graph) { the_scenario.fetch(:ending_graph) }
        it 'will update the graph' do
          # A custom test helper method that builds the starting graph in the indexing and persistence layer.
          # This builds the "initial" data state
          build_graph(starting_graph)

          # Logic that mirrors the behavior of updating an ActiveFedora object.
          write_document_to_persistence_layers(preservation_document_attributes_to_update)

          # Run the "job" that will reindex the relationships for the given pid.
          Curate::Indexer.reindex_relationships(preservation_document_attributes_to_update.fetch(:pid))

          # A custom spec helper that verifies the expected ending graph versus the actual graph as retrieved
          # This verifies the "ending" data state
          verify_graph_versus_storage(ending_graph)
        end
      end
    end
  end
end
```

The above may be a bit to digest. But take a bit to read the code and the comment.

## And You're Back

This was, in essence my first test. I had the hindsight of [the previous solution](/posts/2016-06-24-solving-library-sponsored-collections) to provide some insights. From this single test I worked at building out what was needed.

Lets expand the three methods:

## #build_graph

Due to the constraints of the data structure, its a bit more complicated. But I prefer the concise data structure, so here we go:

```ruby
def build_graph(graph)
  # Create the starting_graph
  graph.fetch(:parent_pids).keys.each do |pid|
    # Build the preservation document (analog to Fedora object)
    parent_pids = graph.fetch(:parent_pids).fetch(pid)
    Indexer.adapter.write_document_attributes_to_preservation_layer(pid: pid, parent_pids: parent_pids)

    # Build the index document (analog to the Solr object)
    Indexer.adapter.write_document_attributes_to_index_layer(
      pid: pid,
      parent_pids: graph.fetch(:parent_pids).fetch(pid),
      ancestors: graph.fetch(:ancestors, {})[pid],
      pathnames: graph.fetch(:pathnames, {})[pid]
    )
  end
end
```

### Indexer.adapter: An Aside

The `Indexer.adapter` object a late game that I knew I wanted from the begining. The adapter provides the interface between the boundaries of the business logic that is `Curate::Indexer` and the business logic of [Curate](https://curate.nd.edu) (or other Hydra Heads that wish to implement Library Collections).

The `Curate::Indexer` gem provides an in memory adapter [Curate::Indexer::Adapters::InMemoryAdapter](https://github.com/ndlib/curate-indexer/blob/3181287b4cfc4d395ad68e95377c189fbe47175b/lib/curate/indexer/adapters/in_memory_adapter.rb) that extends the documented [Curate::Indexer::Adapters::AbstractAdapter](https://github.com/ndlib/curate-indexer/blob/3181287b4cfc4d395ad68e95377c189fbe47175b/lib/curate/indexer/adapters/abstract_adapter.rb).

By working through the abstract adapter, the total time to run these specs (Rubocop, RSpec, and Simplecov) is `1.937s`. No spinning up Solr nor Fedora. Nor expensive writes to a persistence service. Just write to an in-memory data store.

## #write_document_to_persistence_layers

Similar to build_graph but acknowledges a different data structure.

```ruby
# Logic that mirrors the behavior of updating an ActiveFedora object.
def write_document_to_persistence_layers(preservation_document_attributes_to_update)
  Indexer.adapter.write_document_attributes_to_preservation_layer(preservation_document_attributes_to_update)
  Indexer.adapter.write_document_attributes_to_index_layer(
    { pathnames: [], ancestors: [] }.merge(preservation_document_attributes_to_update)
  )
end
```

## Curate::Indexer.reindex_relationships

The production code that gets run and its documentation. In our Curate implementation, this is fired as an "after_save" job for ActiveFedora objects.

```ruby
module Curate::Indexer
  # This assumes a rather deep graph
  DEFAULT_TIME_TO_LIVE = 15
  # @api public
  # Responsible for reindexing the associated document for the given :pid and the descendants of that :pid.
  # In a perfect world we could reindex the pid as well; But that is for another test.
  #
  # @param pid [String] - The permanent identifier of the object that will be reindexed along with its children.
  # @param time_to_live [Integer] - there to guard against cyclical graphs
  # @return [Boolean] - It was successful
  # @raise Curate::Exceptions::CycleDetectionError - A potential cycle was detected
  def self.reindex_relationships(pid, time_to_live = DEFAULT_TIME_TO_LIVE)
    RelationshipReindexer.call(pid: pid, time_to_live: time_to_live, adapter: configuration.adapter)
    true
  end
end
```

### @api public : Another aside

I first encountered the use of `@api public/private` in [Hanami code](http://hanamirb.org/). Its a letter to other developers indicating your maintenance and ownership intent for future versions.

In the case of `@api public`, my understanding is that you intend for that method to remain in the current major release of your project. If you want to get rid of it, its time to use deprecation notices. It also means documentation is much more important and should be provided. (HINT: keep the number of `@api public` declarations to a minimum).

In the case of `@api private`, my understanding, is this method is not something adopters should rely upon.

I keep forgetting to bring this up in conversations with Hydranauts, but will do so for the [Hydra Architecture Working Group](https://wiki.duraspace.org/display/hydra/Hydra+Architecture+Working+Group).

## #verify_graph_versus_storage

Now we need to compare the expected index documents with what was actually persisted. Again note the use of the `Indexer.adapter` method.

```ruby
def verify_graph_versus_storage(ending_graph)
  ending_graph.fetch(:parent_pids).keys.each do |pid|
    document = Documents::IndexDocument.new(
      pid: pid,
      parent_pids: ending_graph.fetch(:parent_pids).fetch(pid),
      ancestors: ending_graph.fetch(:ancestors).fetch(pid),
      pathnames: ending_graph.fetch(:pathnames).fetch(pid)
    )
    expect(Indexer.adapter.find_index_document_by(pid)).to eq(document)
  end
end
```

## Wrapping Up for Now

There is a lot more code that was written. I needed to [test bootstrapping an initial index](https://github.com/ndlib/curate-indexer/blob/3181287b4cfc4d395ad68e95377c189fbe47175b/spec/features/reindex_pid_and_descendants_spec.rb#L166-L227), because we would not be starting from an empty repository up.

In summary, the implementation was built out from a single high level feature scenario. The scenario was one of data transformation (and not UI interactions, which are painful to test). My understanding of the scenario came from extensive whiteboarding as well as a previous iteration through the problem space. From there I was able to write what I needed.

In the next installment, I will write about [implementing the adapter in CurateND](/posts/2016-07-18-implementing-an-adapter-for-library-sponsored-collections).
