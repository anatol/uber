# Uber

_Gem-authoring tools like class method inheritance in modules, dynamic options and more._

## Installation

Add this line to your application's Gemfile:

    gem 'uber'

Ready?

# Inheritable Class Attributes

This is for you if you want class attributes to be inherited, which is a mandatory mechanism for creating DSLs.

```ruby
require 'uber/inheritable_attr'

class Song
  extend Uber::InheritableAttr

  inheritable_attr :properties
  self.properties = [:title, :track] # initialize it before using it.
end
```

Note that you have to initialize your attribute which whatever you want - usually a hash or an array.

You can now use that attribute on the class level.

```ruby
Song.properties #=> [:title, :track]
```

Inheriting from `Song` will result in the `properties` object being `clone`d to the sub-class.

```ruby
class Hit < Song
end

Hit.properties #=> [:title, :track]
```

The cool thing about the inheritance is: you can work on the inherited attribute without any restrictions, as it is a _copy_ of the original.

```ruby
Hit.properties << :number

Hit.properties  #=> [:title, :track, :number]
Song.properties #=> [:title, :track]
```

It's similar to ActiveSupport's `class_attribute` but with a simpler implementation resulting in a less dangerous potential. Also, there is no restriction about the way you modify the attribute [as found in `class_attribute`](http://apidock.com/rails/v4.0.2/Class/class_attribute).

This module is very popular amongst numerous gems like Cells, Representable, Roar and Reform.


# Dynamic Options

Implements the pattern of defining configuration options and dynamically evaluating them at run-time.

Usually DSL methods accept a number of options that can either be static values, symbolized instance method names, or blocks (lambdas/Procs).

Here's an example from Cells.

```ruby
cache :show, tags: lambda { Tag.last }, expires_in: 5.mins, ttl: :time_to_live
```

Usually, when processing these options, you'd have to check every option for its type, evaluate the `tags:` lambda in a particular context, call the `#time_to_live` instance method, etc.

This is abstracted in `Uber::Options` and could be implemented like this.

```ruby
require 'uber/options'

options = Uber::Options.new(tags:       lambda { Tag.last },
                            expires_in: 5.mins,
                            ttl:        :time_to_live)
```

Just initialize `Options` with your actual options hash. While this usually happens on class level at compile-time, evaluating the hash happens at run-time.

```ruby
class User < ActiveRecord::Base # this could be any Ruby class.
  # .. lots of code

  def time_to_live(*args)
    "n/a"
  end
end

user = User.find(1)

options.evaluate(user, *args) #=> {tags: "hot", expires_in: 300, ttl: "n/a"}
```

## Evaluating Dynamic Options

To evaluate the options to a real hash, the following happens:

* The `tags:` lambda is executed in `user` context (using `instance_exec`). This allows accessing instance variables or calling instance methods.
* Nothing is done with `expires_in`'s value, it is static.
* `user.time_to_live?` is called as the symbol `:time_to_live` indicates that this is an instance method.

The default behaviour is to treat `Proc`s, lambdas and symbolized `:method` names as dynamic options, everything else is considered static. Optional arguments from the `evaluate` call are passed in either as block or method arguments for dynamic options.

This is a pattern well-known from Rails and other frameworks.

## Evaluating Elements

If you wanna evaluate a single option element, use `#eval`.

```ruby
options.eval(:ttl, user) #=> "n/a"
```

## Single Values

Sometimes you don't need an entire hash but a dynamic value, only.

```ruby
value = Uber::Options::Value.new(lambda { |volume| volume < 0 ? 0 : volume })

value.evaluate(context, -122.18) #=> 0
```

Use `Options::Value#evaluate` to handle single values.

## Performance

Evaluating an options hash can be time-consuming. When `Options` contains static elements only, it behaves *and performs* like an ordinary hash.


Uber::Options.new volume: 9, track: lambda { |s| s.track }


dynamic: true

only use for declarative assets, not at runtime (use a hash)

# Undocumented Features

(Please don't read this!)

* You can enforce treating values as dynamic (or not): `Uber::Options::Value.new("time_to_live", dynamic: true)` will always run `#time_to_live` as an instance method on the context, even thou it is not a symbol.

# License

Copyright (c) 2014 by Nick Sutterer <apotonick@gmail.com>

Roar is released under the [MIT License](http://www.opensource.org/licenses/MIT).