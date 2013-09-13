---
layout: post
title: "Elixir Patterns: Abstract Data Structures"
date: 2013-09-12 10:45
categories: [elixir, "Elixir Patterns"]
---

Wouldn't you like to define a data structure as a module, with the internals
hidden? Obviously you can, but because modules are not records the data
structure then can't be used with protocols. You could define a record and in
the `do` block put functions to manipulate it but then you're making it easy for
your users to use the record directly, and trample on the invariants in the
process.

## Record tags

Luckily there is a trick. It's not very obvious but if you read the
documentation for `Kernel.defrecordp/3` closely there is a tag field which is
used in an example to set the tag to the name of the enclosing module. What's
the tag? Well in Erlang, and Elixir, records are stored as tuples with the first
field being a "tag" that distinguishes different records and the other fields
being the data of the record. With `defrecord` the tag is always the name of the
record module, with `defrecordp` it is the record name.

{% codeblock lang:elixir %}
defrecordp :foo, a: nil, b: nil
# name = :foo, tag = :foo, foo(a: 1, b: 2) ==> { :foo, 1, 2 }
defrecordp :foo, :bar, a: nil, b: nil
# name = :foo, tag = :bar, foo(a: 1, b: 2) ==> { :bar, 1, 2 }
{% endcodeblock %}

When defining implementations of protocols the name you pass to `for: ` is the
tag of the record. This makes perfect sense as the protocol doesn't need to know
anything about your record except how to recognize it, and that's what the tag
is for.

## An abstract binary tree

Using record tags you can write an abstract binary tree module like this:

{% codeblock An abstract binary tree lang:elixir %}
defmodule BinTree do
  # Invariant: left.value <= value <= right.value
  defrecordp :tree, __MODULE__, value: nil, left: nil, right: nil

  def new() do
    tree()
  end

  def from_enum(e) do
    Enum.sort(e) |> do_from_enum()
  end

  defp do_from_enum(e) do
    count = Enum.count(e)
    if count > 0 do
      middle = div(count, 2)
      { left, rest } = Enum.split(e, middle)
      { [value], right } = Enum.split(rest, 1)
      tree(value: value, left: do_from_enum(left), right: do_from_enum(right))
    else
      nil
    end
  end

  def reduce(nil, acc, _) do
    acc
  end
  def reduce(t, acc, f) do
    [value, left, right] = tree(t, [:value, :left, :right])
    acc1 = reduce(left, acc, f)
    acc2 = f.(value, acc1)
    reduce(right, acc2, f)
  end
end

defimpl Enumerable, for: BinTree do
  def reduce(t, acc, f), do: BinTree.reduce(t, acc, f)
  def count(t), do: BinTree.reduce(t, 0, fn _, n -> n + 1 end)
  def member?(t, x), do: BinTree.reduce(t, false, fn y, found -> found || x == y end)
end
{% endcodeblock %}

The `__MODULE__` is a special variable that expands to the name of the module. I
could have written `BinTree` in it's place, but `__MODULE__` stands out and is a
good reminder of what trick is used here.

The module works as you'd expect:

{% codeblock lang:elixir %}
BinTree.from_enum(1..4)    
# {BinTree, 3, {BinTree, 2, {BinTree, 1, nil, nil}, nil}, {BinTree, 4, nil, nil}}
BinTree.from_enum(1..4) |> Enum.count()
# 4
{% endcodeblock %}

We're still showing the guts of the record through `inspect` though, but that's
easily changed.

{% codeblock lang:elixir %}
defimpl Inspect, for: BinTree do
  def inspect(t, _), do: "#<BinTree #{inspect(Enum.to_list(t))}>"
end
{% endcodeblock %}

{% codeblock lang:elixir %}
BinTree.from_enum(1..4)
# #<BinTree [1, 2, 3, 4]>
{% endcodeblock %}

Just remember that we're only hiding the representation when pretty printed, if
you pass `raw: true` to inspect you'll still see the internal representation.
Due to how Erlang (and thus Elixir) works this is unavoidable, but the pretty
printing serves as a powerful reminder to the user that the internal
representation should not be relied upon.

## Dispatching on the record tag

There is another trick I'd like to show you. If you look at the source of the
`Set` module from the standard library most functions look like this:

{% codeblock A Set function lang:elixir https://github.com/elixir-lang/elixir/blob/87be134a5b91773643ea292b521c9a8ec8167894/lib/elixir/lib/set.ex#L77-L79 %}
def difference(set1, set2) do
  target(set1).difference(set1, set2)
end
{% endcodeblock %}

Huh? Object-oriented programming. Nope, just plain old `apply/3` (if the first
argument is not a literal symbol Elixir translates this to runtime apply call).
The magic is in the `target` macro:

{% codeblock target macro lang:elixir https://github.com/elixir-lang/elixir/blob/87be134a5b91773643ea292b521c9a8ec8167894/lib/elixir/lib/set.ex#L39-L47 %}
defmacrop target(set) do
  quote do
    if is_tuple(unquote(set)) do
      elem(unquote(set), 0)
    else
      unsupported_set(unquote(set))
    end
  end
end
{% endcodeblock %}

Remember that the first field of a record is the tag, so `elem(rec, 0)` extracts
the record tag. If you call `Set.difference(s1, s2)` (where `s1` and `s2` are
HashSets) `target(s1).difference(s1, s2)` gets called, which expands to
(ignoring the tuple check) `elem(s1, 0).difference(s1, s2)`, which effectively
translates to `apply(HashSet, :difference, [s1, s2])`.

When I first saw that code I spent some time scratching my head over why they
didn't use a protocol. I'm still not 100% sure, but I strongly suspect
performance is one of the main reasons. As a test I made a very simple benchmark
to compare the overhead of calling through a protocol vs calling through
`apply`.

{% gist 2b125e2165713ec49c86 %}

Running this a number of times gives a pretty consistent result after the first
few runs: the protocol calls are about 14-15 times slower than the dispatch
calls (which is how I named the "through apply" calls, not sure if it's the
right term).

Now protocol calls are known to be slow (that's a big part of why Elixir has
reduce based collections, those only require a few protocol calls even for
large amounts of data). There are
[ideas](https://github.com/elixir-lang/elixir/issues/950) on how to fix that and
once those get implemented things may change for released code. For now however
indirect calls through `apply` are significantly faster.

{% codeblock Set's callbacks lang:elixir https://github.com/elixir-lang/elixir/blob/87be134a5b91773643ea292b521c9a8ec8167894/lib/elixir/lib/set.ex#L20-L37 %}
use Behaviour

@type value :: any
@type values :: [ value ]
@type t :: tuple

defcallback delete(t, value) :: t
defcallback difference(t, t) :: t
defcallback disjoint?(t, t) :: boolean
defcallback empty(t) :: t
defcallback equal?(t, t) :: boolean
defcallback intersection(t, t) :: t
defcallback member?(t, value) :: boolean
defcallback put(t, value) :: t
defcallback size(t) :: non_neg_integer
defcallback subset?(t, t) :: boolean
defcallback to_list(t) :: list()
defcallback union(t, t) :: t
{% endcodeblock %}

If you look at `Set` again notice that it's a behaviour with callbacks of the
same types and arity for each exposed function. All the implementation modules
(just `HashSet` in the standard library) implement the `Set` behaviour and the
compiler makes sure that all the functions are indeed implemented. If you're
going to do the same protocol replacing trick as `Set` it's a good idea to
define a behaviour as well.

## Caveats

So what are the downsides of the abstract data structure technique? Well you
lose the nice record syntax and decomposition and have to work with the macros
`defrecordp` generates for you. And it's a bit more work getting everything set
up.

But let's face it, those are minor issues. What you get is the ability to
offer a data structure without exposing it's representation. You get the ability
to make a data structure that (ugly hacks aside) can only be manipulated through
the API you provide, an API that maintains the internal invariants.

If you define a public record stop and think for a moment why you are defining
the record that way, because often hiding it in a module results in cleaner,
more maintainable code.

{% comment %}
vim:ft=octopress
{% endcomment %}
