---
author: jfriesen
category: practices
filename: 2017-02-23-a-dependency-injection-pattern.md
layout: post
tagline: One of Many Ways
title: A Dependency Injection Pattern
tags: 'ruby, design'
---

# Dependency Injection Setup

The `MyObject` class has a default collaborator; Pretend that the collaborator performs an expensive function.
We don't always want to call it. And in some cases we don't know the implementation details of the collaborator.

```ruby
class MyObject
  def initialize(collaborator: default_collaborator)
    @collaborator = collaborator
  end

  # @return [#call] A lambda with an arity 1 method signature
  def default_collaborator
    -> (object) { :do_expensive_processing_here }
  end
  private :default_collaborator

  # In collaboration with the given @collaborator, do it!
  def do_it!
    @collaborator.call(self)
  end
end
```

## Example 1

Injecting another lambda:

```ruby
mock_collaborator = -> (object) { :done }
my_object = MyObject.new(collaborator: mock_collaborator)
expect(my_object.do_it!).to eq(:done)
```

## Example 2

Injecting another object's method as a lambda:

```ruby
class CheapCollaborator
  def process_an_object(object)
    :done
  end
end
cheap_collaborator = CheapCollaborator.new
my_object = MyObject.new(collaborator: cheap_collaborator.method(:process_an_object))
expect(my_object.do_it!).to eq(:done)
```

## Example 3

Injecting an object that responds to call:
```ruby
module ModuleMethodCollaborator
  def self.call(object)
    :done
  end
end
my_object = MyObject.new(collaborator: ModuleMethodCollaborator)
expect(my_object.do_it!).to eq(:done)
```

In each of the above cases, we are passing a collaborator that is a "call"-able object. I have found this to be an effective method to:

* speed up tests
* isolate a decision that is not yet finalized
* clearly communicate interfaces

# Additional Consideration

In some cases, I have went a step further and guarded the interface with a setter method. In doing so, I'm ensuring that the object enforces the interface during initialization.

With the below example, there is clear guidance on how to configure the collaborator (at least in regards to its input).

```ruby
class MyObject
  def initialize(collaborator: default_collaborator)
    self.collaborator = collaborator
  end

  # @return [#call] A lambda with an arity 1 method signature
  def default_collaborator
    -> (object) { :do_expensive_processing_here }
  end
  private :default_collaborator

  def collaborator=(input)
    raise "Expected #{input} to respond to :call" unless input.respond_to?(:call)
    raise "Expected #{input}#call to have arity 1" unless input.method(:call).arity == 1
    @collaborator = input
  end
  private :collaborator=

  def do_it!
    @collaborator.call(self)
  end
end
```
