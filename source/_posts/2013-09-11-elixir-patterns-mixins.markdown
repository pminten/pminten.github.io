---
layout: post
title: "Elixir Patterns: Mixins"
date: 2013-09-11 13:48
categories: elixir
---

Elixir's macros are a powerful way to add language features without encoding
them in the language core itself. One of such features is a pattern one could
call "mixin" (by analogy to the
[Mixin concept](https://en.wikipedia.org/wiki/Mixin) in class based
object-oriented languages.

In Ruby mixin modules are used for example to define the `==`, `<`, `>=`, etc
operators for a class if the `<=>` (compare) operator is defined. The compare
operator returns -1 if the first argument is smaller than the second, 0
if it is equal to the second and 1 if it is greater than the second. Obviously
if you have that it's easy to define `<` and friends. By including the
`Comparable` mixin you get most of the comparison operators for free, just
remember to define `<=>`.

## Default functions for protocols

In Elixir we don't have classes but we do have similar situations where you
generally want to define something in terms of something else. Take for example
`Enumerable`. The `Enumerable` protocol has three methods: `reduce/3`, `count/1`
and `member?/2`. However you can always define `count` and `member?` in
terms of `reduce`, they're just there so that you can override them with a more
efficient implementation.

Because protocols don't support default definitions for a method you always have
to define all three, even if you don't do anything special for `count` and
`member?`. That is to say for a simple binary tree you have to write:

{% highlight elixir %}
defrecord BinTree, value: nil, left: nil, right: nil

defimpl Enumerable, for: BinTree do
  def reduce(nil, acc, _), do: acc
  def reduce(BinTree[value: value, left: left, right: right], acc, fun) do
    acc1 = fun.(value, acc)
    acc2 = reduce(left, acc1, fun)
    reduce(right, acc2, fun)
  end

  def count(BinTree[] = t) do
    reduce(t, 0, fn (_, acc) -> acc + 1 end)
  end
  def member?(BinTree[] = t, x) do
    reduce(t, false, fn (v, acc) -> acc or x == v end)
  end
end
{% endhighlight %}

Using a mixin we can simplify this. A mixin in Elixir is generally defined as a
module with a `__using__` macro.

{% highlight elixir %}
defmodule Enumerable.Mixin do
  defmacro __using__(_) do
    quote location: keep do
      def count(e) do
        reduce(e, 0, fn (_, acc) -> acc + 1 end)
      end
      def member?(e, x) do
        reduce(e, false, fn (v, acc) -> acc or x == v end)
      end
    end
  end
end
{% endhighlight %}

By simply writing `use Enumerable.Mixin` we now get `count` and `member?`
defined in our module. The argument to `__using__` we're ignoring is a list of
keywords that you can use with `use`, for example `use ExUnit.TestCase, async:
true`.

With the mixin our code becomes simpler:

{% highlight elixir %}
defimpl Enumerable, for: BinTree.Tree do
  use Enumerable.Mixin
  
  def reduce(nil, acc, _), do: acc
  def reduce(BinTree[value: value, left: left, right: right], acc, fun) do
    acc1 = fun.(value, acc)
    acc2 = reduce(left, acc1, fun)
    reduce(right, acc2, fun)
  end
end
{% endhighlight %}

Certainly an improvement. But what if we want to define one of the methods we
generated. For example if our binary tree has the invariant "left.value < value
< right.value" we can use that for faster member testing.

To support this we'll mark the mixed in functions as overridable. That means
that if the compiler comes across a new definition for the function (i.e.
function clauses that aren't right next to previous clauses of the function) it
will not complain but forget about the old definition of the function and use
the new one.

{% highlight elixir %}
defmodule Enumerable.Mixin do
  defmacro __using__(_) do
    quote location: keep do
      def count(e) do
        reduce(e, 0, fn (_, acc) -> acc + 1 end)
      end
      def member?(e, x) do
        reduce(e, false, fn (v, acc) -> acc or x == v end)
      end
      defoverridable [count: 1, member?: 2]
    end
  end
end
{% endhighlight %}

The arguments to `defoverridable` are the function names and arities of the
functions you want to be overridable. Now that `member?` is overridable we can
define a new custom `member?`:

{% highlight elixir %}
defimpl Enumerable, for: BinTree.Tree do
  use Enumerable.Mixin
  
  def reduce(nil, acc, _), do: acc
  def reduce(BinTree[value: value, left: left, right: right], acc, fun) do
    acc1 = fun.(value, acc)
    acc2 = reduce(left, acc1, fun)
    reduce(right, acc2, fun)
  end

  def member?(nil, _), do: false
  def member?(BinTree[value: value, left: left, right: right], x) do
    cond do
      x == value -> true
      x < value  -> member?(left, x)
      x > value  -> member?(right, x)
    end
  end
end
{% endhighlight %}

Now our `member?` is much faster, assuming a balanced tree of 1024 nodes it only
takes 10 steps instead of 1024 steps.

## Behaviours

Another place where mixins come in handy is in OTP style behaviours. Take a look
at a (slightly edited) bit of GenServer.Behaviour:

{% highlight elixir %}
defmodule GenServer.Behaviour
  defmacro __using__(_) do
    quote location: :keep do
      @behavior :gen_server

      def init(args) do
        { :ok, args }
      end

      def handle_call(_request, _from, state) do
        { :noreply, state }
      end

      # functions ommitted

      defoverridable [init: 1, handle_call: 3, handle_info: 2,
        handle_cast: 2, terminate: 2, code_change: 3]
    end
  end
end
{% endhighlight %}

This declares the module that uses `GenServer.Behaviour` to be a gen\_server
callback module (`@behavior :gen_server`) and declares all the callbacks with
simple no-op implementations, which are overridable. The idea is that you define
just the callbacks you need without being forced to declare all of them just to
keep the compiler happy.

## Caveats

Mixins are a great tool. But like all tools they have their specific strengths
and weaknesses. Ever tried to put a nail in a wall with a screw driver?

Mixins are macro's and as such they tend to obscure the true meaning of the
code. If you refer to a function that was introduced by a mixin people might
wonder where that function came from. Luckily `use` is fairly easy to spot so
this isn't such a big problem.

Mixins also bloat the code by placing copies of definitions in multiple files.
It's best to keep mixed in definitions simple, if you need something more
complicated consider factoring out the part of a function that doesn't need to
be in the target module into a separate function in some common module and call
that.

Finally badly documented mixins make understanding what goes on in the target
module much harder than it needs to be. If you write a mixin make sure you
include documentation on what functions it adds and what they do.

These small quibbles aside mixins are a great tool for reducing the amount of
boilerplate in modules.
