---
layout: post
title: "Concurrent I/O using dry-monads' Task"
date: 2022-02-08 10:53
comments: true
categories: blog
tags:
  - haskell
  - functional
  - functionalprogramming
  - ruby
  - dry-monads
  - task
  - gvl
  - gil
---

[dry-monads][drymonads] is a _beautiful_ gem. It brings a few valuable Monads to Ruby and strives to make them fit snuggly among the other constructs we usually employ when building our software. One such Monad is [Task][drymonadstask].

`Task` is a lot like [PureScript's `Aff`][psaff], or [fp-ts's `Task`][fptstask]: it represents an asynchronous computation that creates side-effects, sometimes resulting in failures. By being a Monad, we can rely on the same techniques and infrastructure available to siblings like `Maybe` and `Result`.

Its implementation is, ultimately, dependent on `Thread`. Suppose you're familiar with concurrency and parallelism in Ruby. In that case, you're probably aware that `Thread` doesn't give you parallelism in CRuby (JRuby is another story) due to the _global VM lock_ (also called _global interpreter lock_, or GIL, a relic of Ruby pre-1.9). Concurrency is a different matter.

Threaded code can be run concurrently in the CRuby VM when a `Thread` is waiting on I/O. If you've ever heard Ruby's performance is not relevant in a world of network calls and file operations, understanding that particularity can be very relevant to building faster programs. That is why Sidekiq, Puma, and many other projects are very successful (and often fast), even without parallelism.

`Task` offers you a principled way to build complex call chains that might fail without sacrificing readability or introducing complex error handling code. If you've ever worked with Promises in JavaScript, it should be very familiar.

## The basics

You can promote (or, using the FP jargon, _lift_) any computation to a `Task` by using its constructor:

```ruby
$ irb
irb(main):001:0> require "dry/monads"
=> true
irb(main):002:0> include Dry::Monads[:task]
=> Object
irb(main):003:0> Task { "success" }
=> Task(?)
```

Since it's an asynchronous computation, it doesn't block or immediately have a value (unless you run it in the `:immediate` thread pool — see the documentation). That's why you see `Task(?)`. The computation _does_ start immediately; if your `Task` consists of an HTTP call, it's in-flight as soon as a thread is available for scheduling. This is different from other languages and libraries, which often separate describing a computation from running it.

When you ask for a Task's value (with `Task#value!`), it blocks until it can produce one:

```ruby
irb(main):004:0> def measure; t0 = Time.now; yield; t1 = Time.now; t1 - t0; end
:measure
irb(main):005:0> measure { Task { sleep 10; "success" }.value! }
=> 10.000725254
```

Like other Functors, it's possible to alter (with `Task#fmap`) its value:

```ruby
irb(main):006:0> Task { "success" }.fmap { "#{_1}: it might be fleeting" }.value!
=> "success: it might be fleeting"
```

Like other Monads, it's possible to chain (with `Task#bind`) computations:

```ruby
irb(main):007:0> success = Task { "success" }
=> Task(?)
irb(main):008:0> success.bind { |a| Task { "fame" }.fmap { |b| "#{a} and #{b}" } }.value!
"success and fame"
```

As well as use _do notation_:

```ruby
irb(main):009:0> include Dry::Monads[:do]
=> Object
irb(main):010:1* def doing_well
irb(main):011:1*   success = yield Task { "success" }
irb(main):012:1*   fame = yield Task { "fame" }
irb(main):013:1* 
irb(main):014:1*   Task { "#{success} and #{fame}" }
irb(main):015:0> end
=> :doing_well
irb(main):016:0> doing_well.value!
=> "success and fame"
```

As previously mentioned, `Task` models the possibility of your computation failing. That means that if the block raises an error, the `Task` will wrap it and behave accordingly downstream. So, when you call `value!`, it will raise the error; when you call `map` or `bind`, nothing will happen:

```ruby
irb(main):017:0> Task { fail "Oh no!" }.value!
# ...
RuntimeError (Oh no!)
irb(main):017:0> Task { fail "Stop!" }.fmap { 10 }.value!
# ...
RuntimeError (Stop!)
```

`Task` also defines a few other combinators, which can be very useful when building more intricate computations.

## When the thing you need is not a Monad

Imagine making three HTTP requests to gather all the data you need to build a report. On average, the first takes 320ms, the second takes 2000ms, and the third takes 1500ms. If you made the calls sequentially, you would need (again, on average) at least 3820ms to have all the data before you could start any processing; doing them concurrently will take at most 2000ms (i.e., the duration of the slowest request).

Monads are, however, inherently sequential. That's why they're commonly called ["programmable semicolons"][semicolons]: the fate of the second operation is bound to what happens in the first. This characteristic is at odds with what we want to achieve. `Task` being a Monad is not what will solve our problem.

Every _Monad_ is also an _Applicative Functor_ (and a _Functor_). [Applicative Functors][apfunctors], in very simplified terms (_don't @ me_), allow us to run an _effectful_ computation (basically, a _structure-building operation_, like, say, constructing a `Maybe`) by lifting a function into a structure and applying it to values in that same structure. Being an Applicative Functor does not imply sequencing, and this means code that relies on it also does not require operations to run sequentially.

`dry-monads` implements that notion for its structures. We wrap the function by calling `pure` on the structure and provide it with arguments using `apply`:

```ruby
irb(main):018:0> Task.pure { |a, b| a + b }.apply(Task { 10 }).apply(Task { 20 }).value!
30
```

Being an Applicative gives us many tools to build code that isn't overly restricted just because we want it to short-circuit in some situations. The one that's most relevant to our use case is `traverse`.

## traverse 

People often joke that the answer is `traverse`, no matter the question. In Haskell, it relies on a structure being `Traversable`, which means you can go element by element and perform an action on each, collecting the results as it goes. By that description, you might be thinking it's a lot like `fmap`. And you're right! The main difference is that instead of building individual effects (e.g., `fmap`ping each element of an Array into a `Maybe`), it considers them in the aggregate.

I know that's very vague. So let's look at it like this: if you have an `Array` and you want to `traverse` it by producing a `Maybe` for each element, you will end up with a `Maybe` with an `Array` inside it. If any of the `Maybe`s you create is a `None`, you'll have a `None`; otherwise, you'll have a `Some([...])` (like `Promise.all`, if you know JavaScript, only more general).

`dry-monads` defines only two `Traversable` structures; we'll use one called [`List`][drymonadslist]. If you ever wondered why that exists, it's time to let it shine:

```ruby
irb(main):019:0> include Dry::Monads[:list]
=> Object
irb(main):020:0*> List[3, 2, 1] \
irb(main):021:0*>   .typed(Task) \
irb(main):022:0*>   .traverse { |n| Task { sleep n; puts "Finished sleeping #{n}"; n * 100 } } \
irb(main):023:0*>   .value!
Finished sleeping 1
Finished sleeping 2
Finished sleeping 3
=> List[300, 200, 100]
```

Let's parse this. With `List[1, 2, 3]`, we build a `Dry::Monads::List`. Then comes an odd detail: `typed(Task)`. Unlike Haskell or PureScript, which know what `Applicative` we'll be using for our effect at compile-time, `dry-monads` needs us to specify how to perform the switcheroo (i.e., which structure to call `pure` on when beginning the aggregation). Then we specify the _structure-building operation_ by providing a block to `traverse`, which will construct a `Task` for each number. 

At this point, we have a `Task[List[Integer]]`. Calling `value!` will block until that `Task` is done and return the `List[Integer]`.

## Putting it together

Let's go back to the HTTP call scenario we described earlier. When using `Task` as our effect, relying on `traverse` means the calls will be run _concurrently_, like we wanted, being only as slow as the slowest call. We can see that in practice:

```ruby
$ irb
irb(main):001:0*> require "net/http"
true
irb(main):002:0*> require "dry/monads"
true
irb(main):003:0*> def measure; t0 = Time.now; yield; t1 = Time.now; t1 - t0; end
:measure
irb(main):004:1*> def fetch_example; Net::HTTP.get("example.com", "/index.html"); end
:fetch_example
irb(main):005:1*> include Dry::Monads[:task, :list]
=> Object
irb(main):006:1*> def sequential
irb(main):007:1*>   (0..9).reduce(Task.pure([])) do |acc, _| 
irb(main):008:1*>     acc.bind { |vs| Task { fetch_example }.fmap { |v| vs + [v] } }
irb(main):009:1*>   end.value!
irb(main):010:1*> end
:sequential
irb(main):011:1*> def concurrent
irb(main):012:1*>   List[*(0..9)]
irb(main):013:1*>     .typed(Task)
irb(main):014:1*>     .traverse { Task { fetch_example } }
irb(main):015:1*>     .value!
irb(main):016:1*> end
:concurrent
irb(main):017:0> measure { sequential }
=> 3.134079812
irb(main):018:0> measure { concurrent }
=> 0.300010159
```

One order of magnitude faster while keeping every guarantee `dry-monads` provides in your toolbelt. That's pretty great!

## Wrapping up

This technique can easily provide your software with a speed boost. In addition, the fact that it creates a grammar for how you structure problems that is very consistent and robust is a great benefit that can be a positive change for any codebase.

There are, of course, potential problems. Thread starvation. Team buy-in. Having to onboard new developers. No static analysis support ([unless you're using Sorbet][drymonadssorbet]; even so, barely) to let you know you're using `fmap` and you should be using `bind` or vice-versa. Still, I believe the _pros_ easily beat the _cons_. Hopefully, after reading this article, you do too.

[drymonads]: https://dry-rb.org/gems/dry-monads
[drymonadstask]: https://dry-rb.org/gems/dry-monads/1.3/task/
[concurrentrubypromise]: https://ruby-concurrency.github.io/concurrent-ruby/master/Concurrent/Promise.html
[psaff]: https://pursuit.purescript.org/packages/purescript-aff/6.0.0
[fptstask]: https://gcanti.github.io/fp-ts/modules/Task.ts.html
[semicolons]: http://book.realworldhaskell.org/read/monads.html
[apfunctors]: http://www.staff.city.ac.uk/~ross/papers/Applicative.pdf
[drymonadslist]: https://dry-rb.org/gems/dry-monads/1.3/list/
[rubyasync]: https://github.com/socketry/async
[drymonadssorbet]: https://github.com/bellroy/dry-monads-sorbet
