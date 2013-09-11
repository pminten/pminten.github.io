---
layout: post
title: "Elixir's Enumerable" 
description: "The reduce-based logic behind Enum and Stream"
category: elixir
---
Sometimes Elixir can be quite a pain. Often it's because the libraries don't yet
have what you need or because a feature that looks like it's going to do exactly
what you want turns out to be more limited than you would like (`|>` I'm looking
at you). Sometimes however the pain is of a different kind, the kind you get
from trying to understand why that seemingly logical Go move is the exact
opposite of what's good, the kind you get from reading about category theory,
the kind you get from trying to use the newest kind of lenses in Haskell. In
other words the kind of pain that comes from desperately trying to understand
something that seems to make no sense at all.

And then, after beating your head against the proverbial desk enough time, you
suddenly realize that things are starting to make sense and that it's actually
all quite clear.

Today I had one of those moments. After trying to produce a better stream design
based on functions that return a value and a replacement for themselves (think
Continuation Passing Style) I ran into a problem where my `Iter.to_list` was
much slower than `Enum.to_list`. When I asked on IRC @ericmj told me that it's
just not possible to make my iterator approach work fast enough and that that's
the reason Elixir switched to a reduce based approach. He also gave me the
excellent tip to look at Clojure's [reducers](http://clojure.org/reducers),
which was the inspiration for Elixir's design.

While the blog posts linked from Clojure's documentation are informative they
are (naturally) about Clojure and not everybody understands that. So here's an
attempt to explain things in an Elixir context.

This post consists of four main parts.

* [A bit about what `Enumerable`, `Enum` and `Stream` are useful for.](#uses)
* [An explanation of how `Enum` is implemented in terms of `Enumerable`.](#enum)
* [A short dicussion on writing your own `reduce` function.](#reduce)
* [An explanation of how streams are used to efficiently compose
  enumerables.](#stream)


## Enumerable, Enum and Stream <a name="uses">&nbsp;</a>

As the user of Elixir code you will often use `Enum`, sometimes use `Stream` and
rarely implement `Enumerable`. The
[`Enum`](http://elixir-lang.org/docs/stable/Enum.html) module is where most of
Elixir's collection manipulation code resides. It has your basic `map/2`,
`filter/2` and `reduce/3` functions as well as stuff like `at/2` / `fetch/2`
(look up an element given an index), `drop/2` and `min/1`. Whenever you want to
do something with a collection look in `Enum` first. Often the tools you need
are there.

Sometimes the methods from `Enum` are somewhat inefficient. Take for example
tripling some numbers and filtering out multiples of 7 afterwards
(`Enum.filter_map` doesn't help here, it filters before mapping). You could
write `Enum.map(l, &(&1 * 3)) |> Enum.filter(&(rem(&1, 7) != 0))` but there's an
inefficiency. It comes from `Enum.map` constructing a list which is immediately
"eaten" by `Enum.filter`.  This work is unnecessary if `Enum.map` could somehow
pass elements to `Enum.filter` one by one.

This is where streams come in. By simply writing `Stream.map(l, &(&1 * 3)) |>
Enum.filter(&(rem(&1, 7) != 0))` the inefficiency is removed. Instead of
returning a list `Stream.map/2` returns a stream for which there is an
`Enumerable` implementation so it can be used with `Enum`. Now what `Stream.map`
does is it creates a stream that wraps an enumerable and whenever an element is
requested it fetches the element from the enumerable but before passing it along
transforms it using the function given to `Stream.map`.

Because all the `Stream` functions return streams sooner or later you'll need to
use `Enum` functions to get a list or some other value (such as a sum) out of
the stream.

It's actually pretty easy to make your own data structures work with `Enum` and
`Stream`. For that you need to define your data structure as a record and add an
`Enumerable` implementation for it.


## Reduce and Enum <a name="enum">&nbsp;</a>

The `reduce` function, also known as (left) fold in other languages, is the most
direct translation of loops like this:

{% highlight ruby %}
a = 1
for i in 1..10 do
  a = a * i
end
# a is now 3628800
{% endhighlight %}

In Elixir we would write such a loop using `Enum.reduce/3` as:

{% highlight elixir %}
Enum.reduce(1..10, 1, fn (i, acc) ->
    acc * i
end)
# returns 3628800
{% endhighlight %}

Simple enough, isn't it. There's nothing complicated about reduce, well not
until you look at the implementation of streams.

Reducing works by walking down a data structure (typically a list) and to call a
function for each element. The function gets passed the element from the list
and a state value (called an accumulator) and is expected to return a new state
value, which gets passed to the function the next time it's called.  The initial
state value is what you pass to `reduce` as the second argument. From now on I
will refer to state values as accumulators.

All functions in `Enum` are conceptually based on this simple function (more on
that later). For example `map/2` can be very efficiently expressed using a reduce.
Here's the typical code for map:

{% highlight elixir %}
def map([], _), do: []
def map([h|t], f), do: [f.(h)|t]
{% endhighlight %}

Err, wait a minute. That can't be right. This function isn't tail recursive and
will thus eat much more stack memory than it needs to. This is more realistic:

{% highlight elixir %}
def map(l, f), do: do_map(l, f, [])

defp do_map([], f, acc), do: :lists.reverse(acc)
defp do_map([h|t], f, acc), do: do_map(t, f, [f.(h)|acc])
{% endhighlight %}

I've used `:lists.reverse` to avoid depending on `Enum` as the point of this
example is how you could do things without `Enum`. It's also slightly faster (no
overhead from calling a protocol function) and the fact that it's restricted to
lists doesn't matter as we're working on lists anyway.

Now here's how you would write `map/2` using `Enumerable`, so it works for every
enumerable:

{% highlight elixir %}
def map(e, f) do 
  Enumerable.reduce(e, [], fn (x, acc) -> [f.(x)|acc] end) |> :lists.reverse()
end
{% endhighlight %}

While superficially different this is actually a lot like the accumulator
version of map with a few differences. Firstly the recursive call to `do_map` is
gone, instead just the new accumulator is returned. Secondly there's no more
manual pattern matching to get the first element of the list. Instead the
`Enumerable.reduce` function passes our function an element, one at a time.

Read that last sentence again, it's key to understanding the power of reduce.
The `reduce` function is responsible for calling the function we supply with
each element. It is responsible for the how of iteration. Our function doesn't
need to know anything about the data structure we're iterating over as long as
there's a function that can call our function. Don't call us, we'll call you.

Another important function in `Enum` is `filter/2`. Again the translation is
quite obvious:

{% highlight elixir %}
def filter(l, f), do: do_filter(l, f, [])

defp do_filter([], f, acc), do: :lists.reverse(acc)
defp do_filter([h|t], f, acc) do 
  if f.(h) do
    do_filter(t, f, [h|acc])
  else
    do_filter(t, f, [acc])
  end
end
{% endhighlight %}

{% highlight elixir %}
def filter(e, f) do
  Enumerable.reduce(e, [], fn (x, acc) ->
    if h.(x) do
      [f.(x)|acc]
    else
      acc
    end
  end) |> :lists.reverse()
end
{% endhighlight %}

Just the same transformation as for map.

A function like `count/1` can be implemented as:

{% highlight elixir %}
def count(e) do
  Enumerable.reduce(e, 0, fn (_, acc) -> acc + 1 end)
end
{% endhighlight %}

In reality it's a call to `Enumerable.count/1`, one of the two extra
(non-reduce) functions on `Enumerable` (the other is `member?/2`) which are
probably there to support data structures that have a faster way of computing
them than using `reduce` (like sets). We'll talk a look at creating an
`Enumerable` implementation for a custom data structure in the next section.

As you have seen all the `Enum` functions can be easily and efficiently
implemented using `reduce`. `reduce` is quite general. It's also not all that
hard to implement, as we shall see.


## Writing a reduce function <a name="reduce">&nbsp;</a>

We've seen how you can use `reduce/3` to do quite a lot of things. But how do
you create a reduce function? Let's look at a few examples. First, a data
structure (a binary tree).

{% highlight elixir %}
defmodule BinTree do
  defrecord Tree, value: nil, left: nil, right: nil

  def reduce(nil, acc, _), do: acc
  def reduce(Tree[value: value, left: left, right: right], acc, fun) do
    acc1 = fun.(value, acc)
    acc2 = reduce(left, acc1, fun)
    reduce(right, acc2, fun)
  end
end
{% endhighlight %}

To reduce a binary tree we'll first give the value to the passed reducer
function and then call reduce on the left and right trees. Now that we have
`reduce` it's easy to define an `Enumerable` implementation:

{% highlight elixir %}
defimpl Enumerable, for: BinTree.Tree do
  def reduce(BinTree.Tree[] = t, acc, fun), do: BinTree.reduce(t, acc, fun)
  def count(BinTree.Tree[] = t) do
    BinTree.reduce(t, 0, fn (_, acc) -> acc + 1 end)
  end
  def member?(BinTree.Tree[] = t, x) do
    BinTree.reduce(t, false, fn (v, acc) -> acc or x == v end)
  end
end
{% endhighlight %}

The `count/1` and `member?/2` functions are required as part of the `Enumerable`
protocol, but you can always define them like I did. They're mostly useful for
when you have a faster way to compute them. For example if the tree has a
guarantee that left.value < value < right.value you could use that to avoid
examining the whole tree.

Let's test it:

{% highlight elixir %}
t = BinTree.Tree[value: 1, 
                 left: BinTree.Tree[value: 2],
                 right: BinTree.Tree[value: 3]]
Enum.to_list(t)    # returns [1, 2, 3]
Enum.count(t)      # returns 3
Enum.member?(t, 1) # returns true
Enum.member?(t, 4) # returns false
{% endhighlight %}

Ok, time for a more complicated scenario, which I'll also revisit in the
section about streams to demonstrate how enumerable can be used to create very
effective pipelines. Let's say you have a very big file and want to read it in
blocks of 512 bytes. There's stuff for this in the standard library
(`File.stream!/2`, `IO.binstream/2` and more). But for the sake of learning
let's pretend all that isn't there.

{% highlight elixir %}
defmodule BlockReader do
  defexception ReadError, message: nil

  def read!(device, block_size // 512) do
    fn acc, fun -> do_read!(device, block_size, acc, fun) end
  end

  defp do_read!(device, block_size, acc, fun) do
    case IO.read(device, block_size) do
      :eof                      -> acc
      {:error, reason}          -> raise ReadError, message: reason
      data when is_binary(data) -> do_read!(device, block_size, fun.(data, acc), fun)
    end
  end
end
{% endhighlight %}

Given an open file (i.e. an IO device) this will cut up a file in blocks.  It
can be used like this: 

{% highlight elixir %}
File.open!("/tmp/some_file") |> BlockReader.read() |> Enum.to_list()
{% endhighlight %}

Quick question to see whether you've been paying attention, did you notice that
`BlockReader.read!` works a bit differently than `BinTree.reduce`?

While `BinTree.reduce` calls the supplied function directly `BlockReader.read!`
returns a function that takes an accumulator and a reducer function. So how does
this work? Well `Enumerable` is implemented for functions. Any function that
takes two arguments, initial accumulator and function is a valid `Enumerable`.
The `reduce` implementation for functions very simple: 

{% highlight elixir %}
def reduce(function, acc, f), do: function.(acc, f)
{% endhighlight %}

In other words when a function gets passed to `Enum.reduce` all that happens is
that the function gets called with the other two arguments of `Enum.reduce`
(namely the accumulator and reducer function).

Besides manually writing reduce functions there are also a few functions in
`Stream` that generate them such as `Stream.iterate/2` and
`Stream.repeatedly/1`.

In the next section we'll see how streams allow us to put something between a
reduction function and the reducer.


## Streaming transformations <a name="stream">&nbsp;</a>

It's easy enough to write some code that computes the line count and character
count (grapheme count to be precise) of each line of a file:

{% highlight elixir %}
File.stream!("my_file") |> Enum.reduce({0, 0}, fn (line, {lines, chars}) -> 
  {lines+1, chars + String.length(line)}
end)
{% endhighlight %}

But what if we want something a little more complicated, like, say, ignoring
lines that start with '#'? We could of course complicate the reducer function
but there's an alterative.

Streams allow us to stick something (like a map or a filter) in the middle of a
reduction pipeline.

{% highlight elixir %}
File.stream!("my_file")
  |> Stream.reject(&(String.startswith(&1, "#")))
  |> Enum.reduce({0, 0}, fn (line, {lines, chars}) -> 
       {lines+1, chars + String.length(line)}
     end)
{% endhighlight %}

Here `Stream.reject/2` (reject is filter with the condition inverted) very
efficiently filters out the lines we don't want, without constructing a big
intermediate list of course.

Streams are easy to use, you just stick them between a generator (like
`File.stream!`) and some call to an `Enum` function.

Most, if not all, of the time the functions in `Stream` are all you need to work
with streams. But it's good to know how they work under the hood. Well, it's a
little bit complicated I'm afraid.

Let's start by looking at the `Stream.Lazy` record:

{% highlight elixir %}
defrecord Stream.Lazy, enumerable: nil, fun: nil, acc: nil
{% endhighlight %}

Three fields: `enumerable`, `fun` and `acc`. The `enumerable` field is due to a
pipeline `File.stream!("a") |> Stream.map(&some_fun/1) |> Enum.each(&IO.puts/1)`
being translated (by the `|>` macro) to `Enum.each(Stream.map(File.stream!("a"),
&some_fun/1), &IO.puts/1)`, meaning that `Stream.map` wraps the result of
`File.stream!` (which is a reducer calling function, which is an enumerable,
like in our implementation). The `enumerable` field stores the enumerable passed
to `Stream.map`.

The `acc` field stores the accumulator for your stream. It can be `nil`, if you
don't have an accumulator. The `fun` field stores your stream function. 

The stream function should always accept a reducer function (I'll call
this the "inner" reducer). If you're using an accumulator (the `acc` field is
specified and not `nil`) it should also accept and something commonly called
`nesting` that you only need if you want to stop streaming before the input is
done (more on that later). So a non-accumulator-using stream function should
accept an inner reducer and an accumulator-using stream function should accept
an inner reducer and `_nesting`.  If you consider this to be confusing, I fully
agree.

The stream function should return another function. If you're using an
accumulator the function should accept an entry (from the "outer"
generator/stream, so the input you're working on) and an accumulator for the
inner reducer function and return a new accumulator for it. For example here's
how you could implement a stream that adds 1 to each input before passing it
along.

{% highlight elixir %}
def add_one(e) do
  Stream.Lazy[enumerable: e,
              # acc is nil by default
              fun: fn(inner_fun) ->
                fn(entry, inner_acc) -> inner_fun.(entry + 1, inner_acc) end
              end]
end
{% endhighlight %}

If the accumulator is not `nil` you get a tuple `{ inner_acc, my_acc }` instead
of just `inner_acc` and should return a similar tuple. Take for example a
function that adds successive numbers to each value.

{% highlight elixir %}
def add_increasing(e) do
  Stream.Lazy[enumerable: e,
              acc: 1,
              fun: fn(inner_fun, _nesting) ->
                fn(entry, { inner_acc, my_acc }) -> 
                  { inner_fun.(entry + my_acc, inner_acc), my_acc + 1 }
                end
              end]
end
{% endhighlight %}

That's most of the magic of stream functions. One tiny detail remains. Remember
that weird `nesting` argument? It's used when you want to early abort a stream
(before the input is done). This is used in `Stream.take` and it depends on
throwing a tuple. Please don't write code that uses this, consider the internal
throw an implementation detail that's not supposed to leak. I'm only showing it
to explain how `Stream` works internally.

{% highlight elixir %}
def until_second_boom(e) do
  Stream.Lazy[enumerable: e,
              acc: 1,
              fun: fn(inner_fun, nesting) ->
                fn
                  (entry, { inner_acc, 0 }) when entry == "boom" ->
                    throw { :stream_lazy, nesting, inner_acc }
                  (entry, { inner_acc, my_acc }) ->
                    { inner_fun.(entry, inner_acc),
                      (if entry == "boom", do: my_acc - 1, else: my_acc) }
                end
              end]
end
{% endhighlight %}

When you run this code with `MyMod.until_second_boom([1,2,"boom",3,"boom"]) |>
Enum.to_list` the result is `[1,2,"boom",3]`.

That's it for `Stream`. Please note however that you're not supposed to write
your own streams. If you need something that's missing from `Stream` please
write a patch for it. That way you won't run the risk of something breaking if
the implementation of `Stream` is updated.

Wrapping it up
--------------

We've seen how `Enumerable` lets you work with all kinds of data structures and
custom stream sources (`File.stream!`) through a consistent interface. With
`Enum` and `Stream` pipelines of transformations can be both expressive and
efficient.  Yet at the same time there's great flexibility, you can define your
own stream sources and your own stream "consumers". You can even (but are not
advised to) define your own stream functions.

{% comment %}
vim:et:ts=2:sw=2:indentexpr=
{% endcomment %}
