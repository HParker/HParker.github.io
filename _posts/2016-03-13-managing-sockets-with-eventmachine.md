---
layout: post
title: Managing Socket connections with Eventmachine
date: 2016-03-13 13:44:13 -0700
categories: ruby eventmachine websockets
---

With the anouncement of
[Action Cable](https://github.com/rails/actioncable) I wanted to see
if I could create a lightweight websocket manager for sending live
events to and from a browser.


Starting small
-----------------


Client
========

Just to begin I wanted to be able to send messages to the browser when
an event happens on the server. I just wanted to solve for the "New
Message!" case. This means that I can really limit the
interface. First, we need a something we can call from the server when
an event happens.

The initial interface looks something like,

{% highlight ruby %}
class Client
    def initialize(name: 'sock-drawer')
      @name = name
    end

    def pub(msg, postfix)
      Redis.new.publish(channel(postfix), msg)
    end

    def channel(postfix)
      "#{@name}/#{postfix}"
    end
  end
end
{% endhighlight %}

and with this we have added our first external dependency

{% highlight ruby %}
spec.add_dependency "redis"
{% endhighlight %}

In rails, add `Client.new.pub('new model!', 'model_created')` to an
after create hook on your model and send events to clients... Of
course, we are missing a key piece. We have an event publishing to
redis, but nothing is forwarding that on to the client. This is where
we have to start handling the publish events from redis and sending
them on to a websocket.

Server
========

There are a couple Things we need to get started. We are going to add
a couple libraries to handle the events that come from redis.

{% highlight ruby %}
spec.add_dependency 'eventmachine'
spec.add_dependency "em-hiredis"
spec.add_dependency "em-websocket"
{% endhighlight %}

Event machine gives us concurrency and has couple good libraries for
receiving events from redis and send information to a client.

So lets start with taking subscriptions from the server. The receiver
of the events we sent into redis in our client.

{% highlight ruby %}
def subscribe(subscription)
  EM::Hiredis.connect.pubsub.psubscribe(subscription + "*") do |c, msg|
    channel(c).push(msg)
  end
end

def channel(channel_name)
  @channels || = {}
  @channels[channel_name] ||= EM::Channel.new
end
{% endhighlight %}

With this we can use Event Machine's channels to push events from
different redis publish events. We are using `psubscribe` which allows
us to pattern match events beginning with the name of our
subscription. I decided to go with a internal convention for channel
names. start with the namespace for sock-drawer events, then the group
that would want to listen, followed by the event name seperated by
`/`s. so the final event would confirm to `<namespace>/<group>`. With
our model create example from above, the event would look like,
`sock-drawer/model_create` and along with this we would send the
message `'new model!'`.

So we have pushed the from redis, into an event machine process, we
still need to send it to the browser if anyone is listening. So again,
trying to do the minimum, lets just use our new subscribe method, then
make sure that all of our subscribed browsers receive whatever message
is pushed on to that event machine channel.

{% highlight ruby %}
def start!
  EM.run do
    subscribe('sock-drawer')
    EventMachine::WebSocket.start({ host: '0.0.0.0', port: 8020 }) do |ws|
      ws.onopen { |handshake|
         sid = channel(@name + handshake.path).subscribe { |msg| ws.send(msg) }
        ws.onclose { channel(@name + handshake.path).unsubscribe(sid) }
      }
    end
  end
end
{% endhighlight %}

So, now when we call `start!` it will start running the event machine
and subscribe to our default namespace `sock-drawer`. After that we
need to get event machine's `WebSocket` library responding to events
on localhost. There are two events that websockets can receive that we
care about right now. on opening a new connection, we need to
subscribe it to the event machine channel that we created earlier and
when it receives an event send the message back across to the
browser. we also want to clean up after ourselves and when an open
websocket connection closes, we want to remove the subscription to the
event machine channel.

browser
-------

At this point I want the very next thing I want to do is plug it in
and see what happens. for that we need a browser with a connection to
the server and some helper scripts to get things running is just a
little output to console.

{% highlight javascript %}
var webSocket = new WebSocket("ws://" + location.hostname + ":8081/" + "my-channel");

webSocket.onmessage = function(event) {
  console.log(event.data);
}
{% endhighlight %}

Cleaning up
--------------

Now that we have the major components in place, lets refine the interface a bit.

### client

first target is the `pub` method on the client. I don't like the two
mystery arguments. I have already forgot if the channel name comes
before or after the channel, so lets see if we can improve this. Also,
if I am going to use this for real, I want to be able to plug in a
logger.

{% highlight ruby %}
module Sock
  class Client
    def initialize(name: DEFAULT_NAME,
                   logger: Logger.new(STDOUT),
                   redis: Redis.new)
      @logger = logger
      @name = name
      @redis = redis
    end

    def pub(msg, channel: '')
      @logger.info "sending #{msg} on channel: #{channel_name(channel)}"
      @redis.publish(channel_name(channel), msg)
    end

    private

    def channel_name(postfix)
      "#{@name}/#{postfix}"
    end
  end
end
{% endhighlight %}

Now, that I can get behind a little more. Overall largely small
changes, but the naming is much better and we now have easy interfaces
into the logger, the redis connection and the name_space we are using.

> you can see an  encorporating what we have done up until now [here](https://github.com/HParker/sock-drawer/tree/master/examples/simple).

Ruby to Ruby events
------------------------

So, I was curious if I could receive events pushed into redis from
ruby as well as firing them across a websocket. I experimented with a
couple ways of registering events. I thought about some kind of DSL
that takes a lambda or maybe a `register` method on the client. The
immediate ran into issues because any event on the client side has to
be serialized through redis to reach the server. Then even with that
extra complexity, after deserializing, we can't guarantee that things
called on the server side are in scope. It also makes things harder to
test.

### The interface

Ultimately I landed on something similar to a router. There is a bit
of magic here and we have to make some changes to the server to get it
to work, but lets start with the final interface.

First we need a class that tells us what to do when an event comes
in. Lets do the echo example here.

{% highlight ruby %}
class MyListener
  include Sock::Subscriber

  on 'echo' do |msg|
    msg
  end
end
{% endhighlight %}

Now when we create our server we just need to pass in the listener.

{% highlight ruby %}
Sock::Server.new(listener: MyListener)
{% endhighlight %}

### The DSL
{% highlight ruby %}
module Subscriber
  def self.included(base)
    base.extend(SubscriberDSL)
  end

  module SubscriberDSL
    def on(channel, &block)
      channels[channel] = block
    end
   def channels
      @_channels ||= {}
    end
  end
end
{% endhighlight %}

### Making it work on the server

Lets start by doing the same thing we did with the client to the
`initialize` method on the server.

{% highlight ruby %}
def initialize(name: DEFAULT_NAME,
               logger: Logger.new(STDOUT),
               socket_params: { host: HOST, port: PORT },
               mode: 'default',
               listener: nil)
  @name = name
  @socket_params = socket_params
  @channels = {}
  @logger = logger
  @mode = mode
  @listener = listener
end
{% endhighlight %}

So, it might take a few too many arguments to be
[metzian](https://gist.github.com/henrik/4509394), but I like this
kind of interface for configuration. You aren't required to provide
anything, but you have access to just about everything you would want
to configure on the class.

The notable addition here is the `listener` keyword. So lets see how
we can send messages to ruby processes.

Ruby to Ruby
---------------

Sending messages from ruby to ruby is not really the purpose of this
library, but it is possible. A more realistic example is receiving a
message from the browser and forwarding it on to the server. Anyways,
this is also possible.

{% highlight ruby %}
require 'sock/drawer'
class MyListener
  include Sock::Subscriber

  on 'my-channel' do |msg|
    puts '--------------------'
    puts msg
    puts '--------------------'
  end
end
{% endhighlight %}

This will just echo whatever comes through redis back out to STDOUT.

and all we have to do is register this listener with the server before starting it.

{% highlight ruby %}
Sock::Server.new(listener: MyListener)
{% endhighlight %}

Now ruby can listen to events just like websockets can!

Conclusion
------------

I had a lot of of fun working on this library. I learned quite a bit
about writing evented code in ruby enough to get moving with
websockets. I hope to start using this for something soon.

In the future, I think I might have written this as a part of a larger
project, then extracted it. I Have read a number of tweets recently
from [DHH](https://twitter.com/dhh) about how rails really is an
extraction of Basecamp. I think that is a good approach because you
have to validate your abstractions first.
