---
layout: post
title: "Thinking in Elixir: Hide Your Messages"
date: 2013-09-14 13:39
categories: [elixir, "Thinking in Elixir"]
published: false
---

In this new blog series I will try to explain some of the concepts behind
programming in Elixir. This will be less practical oriented than my Elixir
Patterns series and more focussed on the big ideas of functional concurrent
programming as supported by Elixir.

## Interface, messages and implementation

Elixir is a language in which concurrency is done by passing messages between
processes. However in practical code it's actually pretty rare to see explicit
message passing. For example say we're working on a chat server. A chat room
could be written somewhat like this, using
[otp\_dsl](https://github.com/pragdave/otp_dsl):

{% codeblock lang:elixir %}
defmodule Chatroom do
  use OtpDsl.GenServer, initial_state: HashDict.new()

  defcall enter(name), users do
    send_all(users, "#{name} has entered the room")
    # _from is secretly filled in by defcall and contains the PID of the caller
    reply(:ok, Dict.put(users, name, _from))
  end

  defcall leave(name), users do
    d = Dict.delete(users, name)
    send_all(users, "#{name} has left the room")
    reply(:ok, d)
  end

  defcall message(name, message) do
    send_all(users, message)
    reply(:ok, d)
  end

  defp send_all(users, message) do
    Enum.each(Dict.values(users), User.send_line(&1, message)) 
  end
end
{% endcodeblock %}

<!-- more -->

This is a simple gen\_server which has an API of three calls:

* `enter(name)`
* `leave(name)`
* `message(name, message)`

When `enter` is called the chatroom adds the user to the list of users. The PID
of the sender is associated with the name (used in `send_all` to send a message
to all users). Because of how otp\_dsl currently works the process gets the
local name `chatroom`.

If you look at the above code do you notice something? There are no messages in
sight. Of course this is a bit of a trick since the whole point of otp\_dsl is
to simplify implementing OTP behaviours like gen\_server. This is how the code
would look when using the normal Elixir `GenServer.Behaviour`:

{% codeblock lang:elixir %}
defmodule Chatroom2 do
  use GenServer.Behaviour
  
  ## Interface

  def enter(name) do
    # Normally I'd allow a PID of the gen_server to call to be passed along but
    # in order to keep this code compatible with the otp_dsl variant (which
    # does't currently support that) we'll have to write it this way.
    :gen_server.call(:chatroom, { :enter, name })
  end

  def leave(name) do
    :gen_server.call(:chatroom, { :leave, name })
  end

  def message(name, message) do
    :gen_server.call(:chatroom, { :message, name, message })
  end

  ## Callback implementations

  @doc false
  # @doc false stops this showing up in ex_doc, it's a way of hiding
  # functions from documentation.
  def init(_args) do
    { :ok, HashDict.new() }
  end

  @doc false
  def handle_call({ :enter, name }, from, users) do
    send_all(users, "#{name} has entered the room")
    { :reply, :ok, Dict.put(users, name, from) }
  end

  def handle_call({ :leave, name }, _from, users) do
    d = Dict.delete(users, name)
    send_all(users, "#{name} has left the room")
    { :reply, :ok, d }
  end

  def handle_call({ :message, name, message }, _from, users) do
    send_all(users, message)
    { :reply, :ok, users }
  end

  ## Private functions
  
  defp send_all(users, message) do
    Enum.each(Dict.to_list(users), fn { user, pid } ->
      User.send_line(user, message)
    end)
  end
end
{% endcodeblock %}

Here the messages aren't immediately obvious (unless you're used to OTP) but
they're more explicit. The `{ :enter, name }`, `{ :leave, name }` and `{
:message, name, message }` are the messages, or well, part of them. When you
call `:gen_server.call(pid, msg)` it actually sends `msg` inside a wrapper
message that contains information that this is a gen\_server call and what the
PID of the sender is. It's all rather complicated stuff intended to make
everything work as it should. Be glad OTP is here to help you, you really don't
want to do this by hand.

Anyway, to get back on the subject, we can see what goes into the messages in
this version. Notice though that the general API hasn't changed. There are still
API functions (the top section) which do nothing but put their arguments in a
message and call `:gen_server.send/2` to send it.

What has changed however is that where otp\_dsl gives you the illusion that
calls are one magic function (`defcall enter`) here it's clear that there are
two functions: one to send a message to the process and one to handle the
received message. Do note that these run in different processes, the `enter`
function runs in the process that calls it while `handle_call` runs in the
chatroom process, I remember it took me a long while before I got used to this
different-functions-in-the-same-module-run-in-different-processes idea when I
was learning Erlang/OTP.

We could send messages from a different process to our chatroom but this is
brittle, if we support this then we can't change the communication protocol
(those `{ :enter ...}` tuples) because we can't know if some other process is
using them. Having an explicit API simplifies things, other modules and
processes need to know nothing about our code except our API. Sure, if you want
to change the API you're still going to have a bit of a headache, but that's
going to be true of any API in your program and you can apply the same
techniques as everywhere else to compensate for it.

Our modules keep their communication protocol internal to the module. This is
the recommended technique when using OTP. It turns processes into a kind of
active objects (active because they have their own "thread"), with the external
API defining the methods. In general hiding knowledge about the details of
message passing behind a "normal" API makes code easier to maintain, which is
why this is a very big thought in Elixir.

## A picture of how a gen\_server works 

To needlessly drive the point home I have made a picture which I don't want to
waste, so here it is:

![Overview of communication to an OTP process](/assets/images/process_interface.svg)

This is a picture of a module that implements a gen\_server with the usual
external API and internal handles. The blue dots on the outer ring represent API
functions (`enter` and friends) while the green dots on the inner ring represent
message handlers (`handle_call`). This picture shows all three types of
communication gen\_server supports: calls (to get a reply), casts (fire and
forget, don't wait for a reply) and infos (all other messages). It shows the
generally useful forms of these: calls and casts come from the module itself
while other messages come from outside the module (for example a message with
some data from a socket). Note that there is no law that forbids you from having
two API functions that would trigger the same message handler, this can be quite
handy at times (for example if you have to keep a legacy API function).

## Abstracting the user

Our chatrooms code sends messages to the user by taking the PIDs that sent the
`enter` message and passing it to `User.send_line` (which will presumably send a
message to the user processes that they should pass the line to the user). We
know this is a PID because we got it from the OTP system. In the `Chatroom`
module we don't _need_ to know this is a PID though, we just need to know it's
something we can pass to `User.send_line`. We honestly don't care if it's a PID,
some integer, a reference (guaranteed unique value) or something else entirely.

By using the `from` information from OTP we force a particular pattern: each
user must be a separate process and the user processes must register themselves,
another process can't do that (well, it can, but it's hard and hacky).

There is another way to write `enter`. Instead of just asking for a name we can
ask for some identifying token and a name. This will work just as well as long
as `User.send_line` accepts the token. Now it's possible for some other process
to enter a user into a chatroom (not sure if that's ever needed, but it's nice
to have the option). We can also do things like having multiple types of users
each with their own communication protocol and having `User.send_line` handle
the differences based on information in the token. Again, not something that's
immediately useful, but the flexibility is nice to have.

Note that all it took to add that flexibility is that `enter` API function,
message and handler got an extra parameter to replace the use of the `from`
field. Sometimes flexibility is cheap.

{% comment %}
vim:ft=octopress
{% endcomment %}
