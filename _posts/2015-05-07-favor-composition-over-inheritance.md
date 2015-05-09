---
author:   jeremyf
category: practices
filename: 2015-05-07-favor-composition-over-inheritance.md
layout:   post
tagline:  And Avoid ActiveSupport::Concern as Its Encourages a Leaky Ship
title:    Favor Composition over Inheritence
tags:     ruby, rails, design, coherent-development
---

> Favor object composition over class inheritence
>
> "Design Patterns: Elements of Reusable Object-Oriented Software" by Gamma, Erich; Helm, Richard; Johnson, Ralph; Vlissides, John

This much quoted phrase has been written about by lots of people.
A Google search yields a plethora of results.

Bill Venner's asks the following question of Erich Gamma:

> "Favor object composition over class inheritance." What does that mean, and why is it a good thing to do?

Erich Gamma's response:

> Inheritance is a cool way to change behavior. But we know that it's brittle, because the subclass can easily make assumptions about the context in which a method it overrides is getting called. There's a tight coupling between the base class and the subclass, because of the implicit context in which the subclass code I plug in will be called. Composition has a nicer property. The coupling is reduced by just having some smaller things you plug into something bigger, and the bigger object just calls the smaller object back. From an API point of view defining that a method can be overridden is a stronger commitment than defining that a method can be called.
>
> ["Design Principles from Design Patterns: A Conversation with Erich Gamma, Part III" - Bill Venners & Erich Gamma](http://www.artima.com/lejava/articles/designprinciples4.html)

There are lots of blog posts about favoring object composition over class inheritance, but I want to go into a few ways to do it in Ruby on Rails.

The example that I'm building is inspired from a [real-world instance](https://github.com/ndlib/sipity/commit/3cce37579df96b4ae3f783605f1ba1d191013cc7#diff-3d600fac39c68ac9be0e872b98c2cb82).

It highlights the many moving parts.

```ruby
class ResearchPaper
  def initialize(title:, abstract:)
    self.title = title
    self.abstract = abstract
  end

  attr_accessor :title, :abstract

  def abstract_to_html
    ConvertToHtml.call(abstract)
  end
end
```

Along comes another class with an abstract.

```ruby
class ConferencePresentation
  def initialize(title:, abstract:, presenter:)
    self.title = title
    self.abstract = abstract
    self.presenter = presenter
  end

  attr_accessor :title, :abstract, :presenter

  def abstract_to_html
    ConvertToHtml.call(abstract)
  end
end
```

There is a duplication of knowledge that is occurring.
So lets refactor.

Options available to us:

* Extract a CreativeWork abstract class (or [Wortem](https://github.com/projecthydra-labs/hydra-works/issues/8))
* Extract a module to Mixin the appropriate behavior
* Extract a collaborating class concerning abstracts then compose it back in

Humor me, as this example is trivial. But just remember how long does code stay trivial?

## CreativeWork as an Abstract Class

```ruby
class CreativeWork
  def initialize(title:, abstract:)
    self.title = title
    self.abstract = abstract
  end

  attr_accessor :title, :abstract

  def abstract_to_html
    ConvertToHtml.call(abstract)
  end
end

class ResearchPaper < CreativeWork
end

class ConferencePresentation < CreativeWork
  def initialize(presenter:, **keywords)
    super(**keywords)
    self.presenter = presenter
  end
  attr_accessor :presenter
end
```

This is great, but does it make sense for `ConferencePresentation` and `ResearchPaper` to have a common direct ancestor? To rephrase that, is there a concept between `ConferencePresentation` and `CreativeWork`? `Presentation` perhaps? Do we want to be in the business of managing huge inheritance trees?

I don't. And you shouldn't either.

## Mixin Module

Up next, the ubiquitous Module mixin, made super easy by ActiveSupport::Concern.

```ruby
module WithAbstract
  def abstract_to_html
    ConvertToHtml.call(abstract)
  end
end

class ResearchPaper
  include WithAbstract

  def initialize(title:, abstract:)
    self.title = title
    self.abstract = abstract
  end

  attr_accessor :title, :abstract
end

class ConferencePresentation
  include WithAbstract

  def initialize(title:, abstract:, presenter:)
    self.title = title
    self.abstract = abstract
    self.presenter = presenter
  end

  attr_accessor :title, :abstract, :presenter
end
```

Better, we no longer have an inheritance tree for our ConferencePresentation and ResearchPaper.
However we now have a subtle coupling; What does `WithAbstract` depend upon?
We can push the example even further leveraging `ActiveSupport::Concern`.

```ruby
module WithAbstract
  extend ActiveSupport::Concern
  included do
    def initialize(**keywords, :abstract)
      super(**keywords)
      self.abstract = abstract
    end
    attr_accessor :abstract
  end
  def abstract_to_html
    ConvertToHtml.call(abstract)
  end
end

class ResearchPaper
  include WithAbstract

  def initialize(title:)
    self.title = title
  end

  attr_accessor :title
end

class ConferencePresentation
  include WithAbstract

  def initialize(title:, presenter:)
    self.title = title
    self.presenter = presenter
  end

  attr_accessor :title, :presenter
end
```

While we have encapsulated the dependencies of the `WithAbstract`, there is a confusing interaction with the initialize method.

## Composition Class

```ruby
module Compositions
  class AbstractComposition
    def new(base)
      self.base = base
    end

    attr_accessor :abstract

    def abstract_to_html
      ConvertToHtml.call(abstract)
    end

    private

    attr_accessor :base
  end
end

class ResearchPaper
  def initialize(title:, abstract:)
    self.title = title
    self.abstract_extension = Compositions::AbstractComposition.new(self)
    self.abstract = abstract
  end

  delegate :abstract, :abstract=, :abstract_to_html, to: :abstract_extension
  attr_accessor :title, :presenter

  private

  attr_accessor :abstract_extension
end

class ConferencePresentation
  def initialize(title:, presenter:, abstract:)
    self.title = title
    self.presenter = presenter
    self.abstract_extension = Compositions::AbstractComposition.new(self)
    self.abstract = abstract
  end

  delegate :abstract, :abstract=, :abstract_to_html, to: :abstract_extension
  attr_accessor :title, :presenter

  private

  attr_accessor :abstract_extension
end
```

This is my current preferred method for a few reasons:

* The dependencies are explicit
* I can test the Compositions::AbstractComposition in true isolation

Yes, it is extra lines of code, but it does not hide the dependencies.
It draws attention to the exact methods you are using.

Now lets say that we want a formatted abstract with title?

## Updating Composition Class

In the case of the composition class method, there are a few adjustments but they
again remain explicit.

```ruby
# reopening from Composition class example
class ResearchPaper
  delegate :title_and_abstract_for_presentation, to: :abstract_extension
end

class ConferencePresentation
  delegate :title_and_abstract_for_presentation, to: :abstract_extension
end

module Compositions
  class AbstractComposition
    def title_and_abstract_for_presentation
      ConvertToHtml.call("<h1>#{base.title}</h1>\n#{abstract}")
    end

    # Guarding the API
    def base=(a_base)
      raise unless a_base.respond_to?(:title)
      @base = title
    end
  end
end
```

## Updating Mixin Module

```ruby
module WithAbstract
  extend ActiveSupport::Concern
  included do

    # I suppose this can be a guard, but what about method_missing, respond_to_missing?
    # But what if this is included before the other module that adds :title as method?
    raise unless self.instance_methods.include?(:title)

    def initialize(**keywords, :abstract)
      super(**keywords)
      self.abstract = abstract
    end
    attr_accessor :abstract
  end
  def title_and_abstract_for_presentation
    # Where did the title method come from?
    ConvertToHtml.call("<h1>#{title}</h1>\n#{abstract}")
  end
end
```

On the upside I only need to re-open one class.
But that is a small thing compared to the obfuscated dangers.
I hope you can agree with me when I say: Yuck!

## Updating Inheritance

This example is trivial and offers the least invasive changes.
But does someone now realize that their `ConferencePresentation` has a `#title_and_abstract_for_presentation` method?

```ruby
class CreativeWork
  def title_and_abstract_for_presentation
    # Where did the title method come from?
    ConvertToHtml.call("<h1>#{title}</h1>\n#{abstract}")
  end
end
```

For a real world example:

* [Crafting a small collaborating object](https://github.com/ndlib/sipity/commit/3cce37579df96b4ae3f783605f1ba1d191013cc7#diff-3d600fac39c68ac9be0e872b98c2cb82) instead of using the ubiquitous ActiveSupport::Concern kludge.

* [Converting mixins to collaborators to encourage decomposition](https://github.com/OregonDigital/oregondigital_2/compare/feature/cleanup)

Other examples are welcome.
