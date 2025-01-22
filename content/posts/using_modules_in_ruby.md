+++
date = '2025-01-22T09:39:08-07:00'
draft = true
title = 'Using Modules in Ruby'
+++

The idea for this post came when I was reading the source code of some Gems that I have used in the past, something I have been making a point to do while I am in between roles. I make a point of noting any methods, patterns, or paradigms I am unfamiliar with for later research. I was looking through the code for `acts_as_tenant` when I came across the `prepend` method, which I had not seen before. `prepend` is related to Modules, but I didn't find the docs for that method very intuitive, so I felt like I needed some background information first, and what follows is an exploration of what Modules are, what they do, how they are used, and where `prepend` fits in. 

## Definition
The official documentation for Ruby defines modules as collections of methods and constants, where methods can be either:
- instance: methods that will become available in a class that includes this module or
- module: methods that will not be available to the including class but can be called directly through the module. 

```ruby
module SomeModule
  def an_instance_method
    'can be used from classes including this module'
  end
  def self.a_module_method
    'can only be used in this module'
  end
end

class SomeClass
  include SomeModule
end

sc = SomeClass.new
puts sc.an_instance_method 
# => can be used from classes including this module

begin
  puts sc.a_module_method
rescue => e
  puts e 
  # => undefined method `a_module_method' for #<SomeClass:123>
end

begin
  puts SomeClass.a_module_method
rescue => e
  puts e 
  # => undefined method `a_module_method' for SomeClass:Class
end

puts SomeModule.a_module_method 
# => can only be used in this module
```

Modules can also be used to add class methods to Classes by extending the module:

```ruby
module SomeOtherModule
  def this_method
    'this is a class method now'
  end
end

class SomeOtherClass
  extend SomeOtherModule
end

puts SomeOtherClass.this_method 
# => this is a class method now
```

## How modules really work
Understanding how modules work requires understanding how method lookup is implemented as well. Ruby uses something called the ancestor chain to determine if an object responds to a given message and including/extending a module in a class adds the module to the ancestor chain.
When including a module in a class that module is after the class in the ancestor chain, but using `prepend` makes the module come before the class, so the module could override an identically named method in the class.
```ruby
class SomeClass; end
module SomeModule; end
class OtherClass; end

SomeClass.include SomeModule
puts SomeClass.ancestors 
# => [SomeClass, SomeModule, Object, Kernel, BasicObject]
OtherClass.prepend SomeModule
puts OtherClass.ancestors 
# => [SomeModule, OtherClass, Object, Kernel, BasicObject]
```
## Applications of modules 

The 2 main uses of modules I would like to focus on are namespacing and code reuse.

### Namespacing
Empty modules are used for organization and managing identically named classes. Often seen in Rails code:
```ruby
# app/controllers/admin/users_controller.rb
module Admin
  class UsersController < ApplicationController
    # ...
  end
end

# app/controllers/users_controller.rb
class UsersController < ApplicationController
  # ...
end
```

### Code reuse / Composition
Real production Ruby code often groups similar methods into modules so that classes can be composed using them. An example of this is in Ruby on Rails' [`ActionController::Base`](https://github.com/rails/rails/blob/a28d2e4193c3a73cc166706c404f3e5398395602/actionpack/lib/action_controller/base.rb) class, which Rails applications inherit their controllers from. This class includes modules like [`Caching`](https://github.com/rails/rails/blob/a28d2e4193c3a73cc166706c404f3e5398395602/actionpack/lib/action_controller/caching.rb), [`Cookies`](https://github.com/rails/rails/blob/a28d2e4193c3a73cc166706c404f3e5398395602/actionpack/lib/action_controller/metal/cookies.rb), [`Logging`](https://github.com/rails/rails/blob/a28d2e4193c3a73cc166706c404f3e5398395602/actionpack/lib/action_controller/metal/logging.rb), etc that encapsulate those concepts. In fact, the base class contains little code and all the functionality comes from the modules it includes.
For example, the Logging module adds a class method called [`log_at`](https://github.com/rails/rails/blob/86312f5dc05b96d9c1c71ef03d257c155622f00e/actionpack/lib/action_controller/metal/logging.rb#L18) to [classes that include it](https://github.com/rails/rails/blob/86312f5dc05b96d9c1c71ef03d257c155622f00e/actionpack/lib/action_controller/base.rb#L302).
The [`ActionController::API`](https://github.com/rails/rails/blob/a28d2e4193c3a73cc166706c404f3e5398395602/actionpack/lib/action_controller/api.rb) class uses a subset of the modules that the [`Base`](https://github.com/rails/rails/blob/a28d2e4193c3a73cc166706c404f3e5398395602/actionpack/lib/action_controller/base.rb) class includes, displaying a use of composition.

## Sources
- https://ruby-doc.org/core/Module.html
- https://ruby-doc.org/core/Object.html#method-i-extend
- https://ruby-doc.org/core/Module.html#method-i-include
- https://ruby-doc.org/core/Module.html#method-i-prepend
- https://veerpalbrar.github.io/blog/2021/11/26/Include,-Extend,-and-Prepend-In-Ruby
- https://github.com/rails/rails/blob/main/actionpack/lib/action_controller/base.rb
- https://github.com/rails/rails/blob/main/actionpack/lib/action_controller/metal/logging.rb
- https://github.com/rails/rails/blob/main/actionpack/lib/action_controller/api.rb