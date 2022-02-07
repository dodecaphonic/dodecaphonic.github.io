---
layout: post
title: Confident Ruby in the presence of nil
date: 2022-02-02 12:34
comments: true
categories: blog
tags:
  - haskell
  - functional
  - functionalprogramming
  - ruby
  - maybe
---

When people talk about working with Ruby, the usual _spiel_ is about joy, humans at the center, [MINASWAN][minaswan], kittens, and all that sparkles. However, they rarely address the role of anxiety in our day to day. Or rather, of one specific concern: will I get a `NoMethodError` due to an unexpected `nil`?

Think about it. If you ask an Array for a value that's not there, you get `nil`. If you call a method that produces a side-effect, you often get `nil`. If you call another method and it stops when a precondition fails, it might return `nil`. Even writing this list made me sweat a little.

That inescapable reality often leads to patterns like:

```ruby
do_something if something
x = y if z
foo = bar ? bar.size : -1
```

We get used to it, but it's hardly the most fluent or confident way to write code. Consider the following:

```ruby
def rejigger_oldest_widget(widgets, max_age_years)
  oldest_widget = widgets
                  .select(&:working?)
                  .find { _1.age_in_years <= max_age_years }

  if oldest_widget
    oldest_widget
      .rejigger 
      .shuffle
      .shine
  end
end
```

It's not awful, but it's hardly remarkable. What if `rejigger`, `shuffle`, or `shine` also return `nil` on occasion? Or worse, what if they _all_ might produce one? Do we despair?

## Our knight in shiny armor

Like many other languages, the solution Ruby (since 2.3) has found for this problem is the safe navigation operator, represented by `&.`. It's neither only `&` nor simply `.`, but `&.`, a worthy unit that allows us to change `rejigger_oldest_widget` to the following:

```ruby
def rejigger_oldest_widget(widgets, max_age_years)
  widgets
    .select(&:working?)
    .find { _1.age_in_years <= max_age_years }
    &.rejigger
    &.shuffle
    &.shine
  end
end
```

Since `Enumerable#find` might return a `nil`, it will only be followed by `rejigger` when it produces a value. When it doesn't, the computation stops at that point, and `rejigger_oldest_widget` returns `nil`.

Conveniently, it works with anything that could be called as a method, no matter the receiver (even `nil`). So, for example, say you had the following:

```ruby
def ruffle_feathers(random_joe)
  return if (feathers = random_joe.feathers).nil?

  ruffled_feathers = annoyobot.ruffle_feathers(feathers)
  annoyometer.measure(ruffled_feathers)
end
```

You could easily write that as:

```ruby
def ruffle_feathers(bird)
  random_joe
    .feathers
    &.then { annoyobot.ruffle_feathers(_1) }
    &.then { annoyometer.measure(_1) }
end
```

I tend to favor this style. I don't care about `ruffled_feathers` other than as an argument to `measure`. The chain of calls with the safe navigation operator makes that more evident (i.e., we want to go from a `random_joe` whose `feathers` can be ruffled to how ruffled their feathers were).

## _Maybe_ we can discuss further

Suppose you're familiar with functional languages in the ML family. In that case, you're probably making the connection I (and [many][andand] [others][drymonadsmaybe]) have made: this is a lot like using the Maybe/Option monad. `&.` is like `map`/`fmap`. You can have the short-circuiting capabilities of `flatMap`/`bind` by producing a `nil` at any step of your computation.

You can easily find comments to that effect about any language with a safe navigation operator — often with a verdict like _"this is a hack"_ or _"this is not as good as having a Monad."_ And they're not wrong! Indeed, if you learn how Functors, Applicatives, and Monads work and what they make possible, you'll probably be a little disappointed with how little `&.` accomplishes.

However, when terms like "monads," "laws," and "categories" show up in conversation, they mean a lot to the initiated and next to nothing to programmers with other backgrounds. So why do Haskell snobs poo-poo something which looks so helpful?

Since this is not a treatise on the narcissism of minor differences, nor another Monad tutorial, let's focus on a practical scenario you can't conveniently handle with `&.` alone.

## Too many nils

Sometimes you need to perform a computation with two or more values, and none of them can be `nil`. If any are `nil`, the final product will also be `nil`. So you write something like this:

```ruby
def fix_global_warming(committed_nations, ethical_companies, conscientious_citizens)
  return unless committed_nations && ethical_companies && conscientious_citizens
  
  fix_it!(committed_nations, ethical_companies, conscientious_citizens)
end
```

Presume you want something as declarative as chaining with `&.`. "Well, why not try `&.` itself?", you think, ending up with:

```ruby
def fix_global_warming(committed_nations, ethical_companies, conscientious_citizens)
  committed_nations&.then do |nations| 
    ethical_companies&.then do |companies| 
      conscientious_citizens&.then do |citizens| 
        fix_it!(nations, companies, citizens) 
      end
    end
  end
end
```

It completely hides what the method does, and you might have trouble understanding that this is all for checking for the presence of values. The guard clause in the original version is _markedly_ better.

How does Haskell solve this? After all, if writing code in that language were anything like the sample above, people wouldn't think it's so great. Fortunately, there are a few ways to make the core of the code clearer. We'll go over some of them.

The first requires special syntax called ["Do notation"][donotation]:

```haskell
-- Starting with something like the Ruby code above
fixGlobalWarming :: Maybe ComittedNations -> Maybe EthicalCompanies -> Maybe ConscientiousCitizens -> Maybe BetterPlanet
fixGlobalWarming committedNations ethicalCompanies conscientiousCitizens = 
  comittedNations >>= 
    \nations -> ethicalCompanies >>=
      \companies -> conscientiousCitizens <&>
        \citizens -> pure (fixIt nations companies citizens)
      
-- A version with do notation
fixGlobalWarming committedNations ethicalCompanies conscientiousCitizens = do
  nations <- committedNations
  companies <- ethicalCompanies
  citizens <- conscientiousCitizens
  
  pure (fixIt nations companies citizens)
```

Since it's syntax, we can only approximate it ([dry-monads][drymonadsmaybe] has its take), which requires a bit of infrastructure. Considering that might prove excessive, we can also rely on it being an `Applicative Functor`, as well as on combinators that remove the boilerplate:

```haskell
-- As an Applicative Functor
fixGlobalWarming committedNations ethicalCompanies conscientiousCitizens =
  fixIt <$> committedNations <*> ethicalCompanies <*> conscientiousCitizens
  
-- With a combinator
fixGlobalWarming = liftM3 fixIt
```

The Applicative version would never go well with Ruby. An operator soup is not most Rubyists' favorite dish (even though it can be delicious!). So that leaves us with `liftM3`.

According to its documentation, what it does is "promote a function to a monad." That means that you can take any function that takes three values and make it work with values in any monad, including Maybe. In that same vein, if you had a function that took two values, you would use `liftM2`; four, `liftM4`. And so on. 

The reason for so many `liftM*` functions is that Haskell needs to account for the possibility each parameter is of a different type. That is, of course, not an issue we have with Ruby. We can, thus, define a similar combinator that allows us to work on any number of arguments:

```ruby
def with_all_present(*values)
  yield *values if values.none?(&:nil?)
end
```

This gives us a way to rewrite `fix_global_warming` as follows:

```ruby
def fix_global_warming(committed_nations, ethical_companies, conscientious_citizens)
  with_all_present(committed_nations, ethical_companies, conscientious_citizens) do
    fix_it(_1, _2, _3)
  end
end
```

## Is it better?

As programmers, we're always trying to find a good balance between conciseness and readability. It's easy to go too far, and each individual cares about different things. Knowing that, it would be foolish to be assertive about any technique.

In my book, both `&.` and custom combinators give us tools to end up more confident and expressive code, unmarred by the fear of `nil`. They're significant steps in the direction of robustness. And even if it's true that neither will kill every `NoMethodError`, they'll go a long way.

[andand]: https://github.com/raganwald/andand
[drymonadsmaybe]: https://dry-rb.org/gems/dry-monads/1.0/
[donotation]: https://en.m.wikibooks.org/wiki/Haskell/do_notation
[minaswan]: https://www.wordsense.eu/MINASWAN/
