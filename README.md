# Deterministic

[![Gem Version](https://badge.fury.io/rb/deterministic.png)](http://badge.fury.io/rb/deterministic)

Deterministic is to help your code to be more confident, it's specialty is flow control of actions that either succeed or fail.

This is a spiritual successor of the [Monadic gem](http://github.com/pzol/monadic). The goal of the rewrite is to get away from a bit to forceful aproach I took in Monadic, especially when it comes to coercing monads, but also a more practical but at the same time more strict adherence to monad laws.


## Usage

### Success & Failure

```ruby
Success(1).to_s             # => "1"
Success(1) << Success(2)    # => Success(2)
Success(Success(1))         # => Success(1)
Success(1).map { |v| v + 1} # => Success(2)
Success({a:1}).to_json      # => '{"Success": {"a":1}}'

Failure(1).to_s             # => "1"
Failure(1) << Failure(2)    # => Failure(1)
Failure(Failure(1))         # => Failure(1)
Failure(1).map { |v| v + 1} # => Failure(2)
Failure({a:1}).to_json      # => '{"Failure": {"a":1}}'
```

Chaining successful actions

```ruby
Success(1).and Success(2)            # => Success(2)
Success(1).and_then { Success(2) }   # => Success(2)

Success(1).or Success(2)             # => Success(1)
Success(1).or_else { Success(2) }    # => Success(1)
```

Chaining failed actions

```ruby
Failure(1).and Success(2)            # => Failure(1)
Failure(1).and_then { Success(2) }   # => Failure(1)

Failure(1).or Success(1)             # => Success(1)
Failure(1).or_else { |v| Success(v)} # => Success(1)
```

The value or block result must always be a `Success` or `Failure`



### Either.attempt_all
The basic idea is to execute a chain of units of work and make sure all return either `Success` or `Failure`.

```ruby
Either.attempt_all do
  try { 1 }
  try { |prev| prev + 1 }
end # => Success(2)
```
Take notice, that the result of of unit of work will be passed to the next one. So the result of prepare_somehing will be something in the second try.

If any of the units of work in between fail, the rest will not be executed and the last `Failure` will be returned.

```ruby
Either.attempt_all do
  try { 1 }
  try { raise "error" }
  try { 2 }
end # => Failure(RuntimeError("error"))
```

However, the real fun starts if you use it with your own context. You can use this as a state container (meh!) or to pass a dependency locator:

```ruby
  class Context
    attr_accessor :env, :settings
    def some_service
    end
  end

  # exemplary unit of work
  module LoadSettings
    def self.call(env)
      settings = load(env)
      settings.nil? ? Failure('could not load settings') : Success(settings)
    end

    def load(env)
    end
  end

  Either.attempt_all(context) do
    # this unit of work explicitly returns success or failure
    # no exceptions are catched and if they occur, well, they behave as expected
    # methods from the context can be accessed, the use of self for setters is necessary
    let { self.settings = LoadSettings.call(env) }

    # with #try all exceptions will be transformed into a Failure
    try { do_something }
  end
```

### Pattern matching
Now that you have some result, you want to control flow by providing patterns.
`#match` can match by

 * success, failure, either or any
 * values
 * lambdas
 * classes

```ruby
Success(1).match do
  success { |v| "success #{v}"}
  failure { |v| "failure #{v}"}
  either  { |v| "either #{v}"}
end # => "success 1"
```
Note1: the inner value has been unwrapped! 

Note2: only the __first__ matching pattern block will be executed, so order __can__ be important.

The result returned will be the result of the __first__ `#try` or `#let`. As a side note, `#try` is a monad, `#let` is a functor.

Values for patterns are good, too:

```ruby
Success(1).match do
  success(1) {|v| "Success #{v}" }
end # => "Success 1"
```

You can and should also use procs for patterns:

```ruby
Success(1).match do
  success ->(v) { v == 1 } {|v| "Success #{v}" }
end # => "Success 1"
```

Also you can match the result class

```ruby
Success([1, 2, 3]).match do
  success(Array) { |v| v.first }
end # => 1
```

Combining `#attempt_all` and `#match` is the ultimate sophistication:

```ruby
Either.attempt_all do
  try { 1 }
  try { |v| v + 1 }
end.match do
  success(1) { |v| "We made it to step #{v}" }
  success(2) { |v| "The correct answer is #{v}"}
end # => "The correct answer is 2"
```

If no match was found a `NoMatchError` is raised, so make sure you always cover all possible outcomes.

```ruby
Success(1).match do
  failure(1) { "you'll never get me" }
end # => NoMatchError
```

A way to have a catch-all would be using an `any`:

```ruby
Success(1).match do
  any { "catch-all" }
end # => "catch-all"
```

## core_ext
You can use a core extension, to include Either in your own class or in Object, i.e. in all classes.

In a class, as a mixin

```ruby
require "deterministic/core_ext/either" # this includes Deterministic in the global namespace!
class UnderTest
  include Deterministic::CoreExt::Either
  def test
    attempt_all do
      try { foo }
    end
  end

  def foo
    1
  end
end

ut = UnderTest.new
ut.test # => Success(1)
```

To add it to all classes

```ruby
require "deterministic/core_ext/object/either" # this includes Deterministic  to Object

class UnderTest
  def test
    attempt_all do
      try { foo }
    end
  end

  def foo
    1
  end
end

ut = UnderTest.new
ut.test # => Success(1)
```

or use it on built-in classes

```ruby
require "deterministic/core_ext/object/either"
h = {a:1}
h.attempt_all do
  try { |s| s[:a] + 1}
end # => Success(2)
```

## Maybe
The simplest NullObject wrapper there can be. It adds `#some?` and `#none?` to `Object` though.

```ruby
require 'deterministic/maybe' # you need to do this explicitly
Maybe(nil).foo        # => None
Maybe(nil).foo.bar    # => None
Mmaybe({a: 1})[:a]     # => 1

Maybe(nil).none?      # => true
Maybe({}).none?       # => false

Maybe(nil).some?      # => false
Maybe({}).some?       # => true
```

## Mimic

If you want a custom NullObject which mimicks another class.

```ruby
class Mimick
  def test; end
end

null = Maybe.mimick(Mimick)
null.test             # => None
null.foo              # => NoMethodError
```

## Inspirations
 * My [Monadic gem](http://github.com/pzol/monadic) of course
 * `#attempt_all` was somewhat inspired by [An error monad in Clojure](http://brehaut.net/blog/2011/error_monads)
 * [Pithyless' rumblings](https://gist.github.com/pithyless/2216519)
 * [either by rsslldnphy](https://github.com/rsslldnphy/either)
 * [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html)
 * [Naught by avdi](https://github.com/avdi/naught/)
 * [Rust's Result](http://static.rust-lang.org/doc/master/std/result/enum.Result.html)

## Installation

Add this line to your application's Gemfile:

    gem 'deterministic'

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install deterministic

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Add some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request
