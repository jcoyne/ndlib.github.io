---
author: jeremyf
category: practices
filename: 2016-07-18-implementing-an-adapter-for-library-sponsored-collections.md
layout: post
tagline: You Put Your Left Foot In
title: Implementing an Adapter for Library Sponsored Collections
tags: 'ruby, rails, design, coherent-development'
---

This is the sixth part in a series of posts regarding [CurateND's](https://curate.nd.edu/) Library Collection development. The series dives into the details of implementing a new type of collections within our [Hydra ecosystem](https://projecthydra.org/).

Before reading this post, please read [the previous post regarding the complete solution](/posts/2016-07-15-complete-solution-for-library-sponsored-collections).

# And Now for Something Completely Different

With the business logic of Curate::Indexer complete (and shareable with other implementors), its time to move into implementing [CurateND's own adapter](https://github.com/ndlib/curate_nd/blob/fb23ba489bc2872610cd1245da6da4a41f8299ce/lib/curate/library_collection_indexing_adapter.rb).

As a refresher, my intention in writing Curate::Indexer as a gem was three fold:

1) I wanted a way to quickly test multiple graph transformation scenarios 2) I wanted the code to be shareable as its something Hydra may not yet have fully implemented (in a modular way)

With the business logic of graph traversal encapsulated to my satisfaction, I needed to implement the adapter. What follows is a snap-shot of that implementation.

It may look like a lot of code, but I was able to focus on implementing the interface. Had it been feasible, I could've handed off the implementation details much easier than if it were the entire reindexing logic.

What I found was that testing was very fast. I could follow the documentation (and specs) that explained what the expected inputs and outputs were. It also made [the specs](https://github.com/ndlib/curate_nd/blob/fb23ba489bc2872610cd1245da6da4a41f8299ce/spec/lib/curate/library_collection_indexing_adapter_spec.rb) much easier to write and conceive.

```ruby
module Curate
  # An implementation of the required methods to integrate with the Curate::Indexer gem.
  # @see Curate::Indexer::Adapters::AbstractAdapter
  module LibraryCollectionIndexingAdapter
    # @api public
    # @param pid [String]
    # @return Curate::Indexer::Documents::PreservationDocument
    def self.find_preservation_document_by(pid)
      # Not everything is guaranteed to have library_collection_ids
      # If it doesn't have it, what do we do?
      fedora_object = ActiveFedora::Base.find(pid, cast: true)
      if fedora_object.respond_to?(:library_collection_ids)
        parent_pids = fedora_object.library_collection_ids
      else
        parent_pids = []
      end
      Curate::Indexer::Documents::PreservationDocument.new(pid: pid, parent_pids: parent_pids)
    end

    # @api public
    # @yield Curate::Indexer::Documents::PreservationDocument
    def self.each_preservation_document
      query = "pid~#{Sufia.config.id_namespace}:*"
      ActiveFedora::Base.send(:connections).each do |conn|
        conn.search(query) do |object|
          next if object.pid.start_with?(ReindexWorker::FEDORA_SYSTEM_PIDS)
          # Because I have a Rubydora object, I need to find it via ActiveFedora, thus the reuse.
          yield(find_preservation_document_by(object.pid))
        end
      end
    end

    # @api public
    # @param pid [String]
    # @return Curate::Indexer::Documents::IndexDocument
    def self.find_index_document_by(pid)
      solr_document = find_solr_document_by(pid)
      coerce_solr_document_to_index_document(solr_document)
    end

    # @api public
    # @param document [Curate::Indexer::Documents::IndexDocument]
    # @yield Curate::Indexer::Documents::IndexDocument
    def self.each_child_document_of(parent_document, █)
      # Need to find all documents that have ancestors equal to one or more of the given parent_document's pathnames
      pathname_query = parent_document.pathnames.map do |pathname|
        "_query_:\"{!raw f=#{SOLR_KEY_ANCESTOR_SYMBOLS}}#{pathname.gsub('"', '\"')}\""
      end.join(" OR ")
      results = ActiveFedora::SolrService.query(pathname_query)
      results.each do |solr_document|
        yield(coerce_solr_document_to_index_document(solr_document))
      end
    end

    # @api public
    # @param attributes [Hash]
    # @option pid [String]
    # @return Hash
    def self.write_document_attributes_to_index_layer(attributes = {})
      # As much as I'd love to use the SOLR document, I don't believe this is feasable as not all elements of the
      # document are stored and returned.
      fedora_object = ActiveFedora::Base.find(attributes.fetch(:pid), cast: true)
      solr_document = fedora_object.to_solr

      solr_document[SOLR_KEY_PARENT_PIDS] = attributes.fetch(:parent_pids)
      solr_document[SOLR_KEY_PARENT_PIDS_FACETABLE] = attributes.fetch(:parent_pids)
      solr_document[SOLR_KEY_ANCESTORS] = attributes.fetch(:ancestors)
      solr_document[SOLR_KEY_ANCESTOR_SYMBOLS] = attributes.fetch(:ancestors)
      solr_document[SOLR_KEY_PATHNAMES] = attributes.fetch(:pathnames)

      ActiveFedora::SolrService.add(solr_document)
      ActiveFedora::SolrService.commit
      solr_document
    end

    SOLR_KEY_PARENT_PIDS = ActiveFedora::SolrService.solr_name(:library_collections).freeze
    SOLR_KEY_PARENT_PIDS_FACETABLE = ActiveFedora::SolrService.solr_name(:library_collections, :facetable).freeze
    SOLR_KEY_ANCESTORS = ActiveFedora::SolrService.solr_name(:library_collections_ancestors).freeze
    # Adding the ancestor symbol as a means of looking up relations; This is cribbed from our current version of ActiveFedora's
    # relationship
    SOLR_KEY_ANCESTOR_SYMBOLS = ActiveFedora::SolrService.solr_name(:library_collections_ancestors, :symbol).freeze
    SOLR_KEY_PATHNAMES = ActiveFedora::SolrService.solr_name(:library_collections_pathnames).freeze

    def self.coerce_solr_document_to_index_document(solr_document, pid = solr_document.fetch('id'))
      parent_pids = solr_document.fetch(SOLR_KEY_PARENT_PIDS, [])
      ancestors = solr_document.fetch(SOLR_KEY_ANCESTORS, [])
      pathnames = solr_document.fetch(SOLR_KEY_PATHNAMES, [])
      Curate::Indexer::Documents::IndexDocument.new(pid: pid, parent_pids: parent_pids, pathnames: pathnames, ancestors: ancestors)
    end
    private_class_method :coerce_solr_document_to_index_document

    def self.find_solr_document_by(pid)
      query = ActiveFedora::SolrService.construct_query_for_pids([pid])
      ActiveFedora::SolrService.query(query).first
    end
    private_class_method :find_solr_document_by
  end
end
```

# And a Little Bit More

There were other things to add:

1. Configuring the `Curate::Indexer.adapter`
2. I needed some background jobs for:

  - initializing all of the relationships in the index
  - updating a single relationship in the index

3. Wiring in when the relationships were updated

## Configuring the `Curate::Indexer.adapter`

In an initializer, I added:

```ruby
Curate::Indexer.configure do |config|
  require 'curate/library_collection_indexing_adapter'
  config.adapter = Curate::LibraryCollectionIndexingAdapter
end
```

This replaced the default in memory adapter (a non-starter for production code).

## Background Jobs

These are rather straight-forward (and you can read the code):

- [ObjectRelationshipReindexerWorker](https://github.com/ndlib/curate_nd/blob/fb23ba489bc2872610cd1245da6da4a41f8299ce/app/workers/object_relationship_reindexing_worker.rb#L2)
- [AllRelationshipsReindexerWorker](https://github.com/ndlib/curate_nd/blob/fb23ba489bc2872610cd1245da6da4a41f8299ce/app/workers/all_relationships_reindexing_worker.rb)

## Wiring in when the Relationships Were Updated

This was a bit more tricky; Through my testing I found that I couldn't add this as an `after_save` event on an ActiveFedora object. So I instead needed to wire it into the CurationConcernActor concept ([see code example](https://github.com/ndlib/curate_nd/blob/fb23ba489bc2872610cd1245da6da4a41f8299ce/app/services/curation_concern/base_actor.rb#L64)).

In either case, I knew I didn't want to fire what could be a VERY expensive operation during tests. So I created some [configuration options](https://github.com/ndlib/curate_nd/blob/master/lib/curate/configuration.rb#L23-L49):

```ruby
module Curate
  class Configuration
    def all_relationships_reindexer
      @all_relationships_reindexer || default_all_relationships_reindexer
    end
    attr_writer :all_relationships_reindexer

    def default_all_relationships_reindexer
      @default_all_relationships_reindexer ||= lambda {
        require 'all_relationships_reindexing_worker'
        Sufia.queue.push(AllRelationshipsReindexerWorker.new)
        true
      }
    end
    private :default_all_relationships_reindexer

    def relationship_reindexer
      @relationship_reindexer || default_relationship_reindexer
    end
    attr_writer :relationship_reindexer

    def default_relationship_reindexer
      @default_relationship_reindexer ||= lambda { |pid|
        require 'object_relationship_reindexing_worker'
        Sufia.queue.push(ObjectRelationshipReindexerWorker.new(pid))
        true
      }
    end
    private :default_relationship_reindexer
  end
end
```

Then [in testing](https://github.com/ndlib/curate_nd/blob/master/config/environments/test.rb#L48-L50), I inject a simple lambda for those two methods:

```ruby
Curate.configuration.relationship_reindexer = lambda { |pid| true }
Curate.configuration.all_relationships_reindexer = lambda { true }
```

This prevents the cascade of many asynchronous jobs.

## Wrapping Up for Now

In the next, and final installment, I'm going to attempt to synthesize the entire process.
