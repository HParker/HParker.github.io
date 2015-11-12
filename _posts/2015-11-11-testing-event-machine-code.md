---
layout: post
title: Testing Event Machine Code
date: 2015-11-11 12:39:14 -0700
categories: ruby event machine testing rspec
---

> Credit  goes  in  large  to  [Nick  Gauthier's](http://ngauthier.com/)
> [2011 post](http://ngauthier.com/2011/09/how-i-test-eventmachine.html). However,
> it took me longer than I would like to admit to understand why testing
> event machine this  way is useful. I  hope that here I  will make that
> clear for others who think like me.

I have a  event machine project, and,  when testing it, I  ran into an
issue  I didn't  fully understand.  I  was able  to able  to test  the
methods I was using in event  machine, but the results seemed somewhat
random. Every 1 in 3 test runs would just fail. I wasn't sure what the
issue was,  and I searched around  the internet for tools  for testing
event  machine code,  but at  the time  I was  very unsure  of what  I
actually needed. Here  are some of the things learned  while trying to
test reactor code in ruby.

> for use in this article we will be using
> [sock-drawer](https://github.com/HParker/sock-drawer). It is a
> [gem](https://rubygems.org/gems/sock-drawer) for providing socket
> connections to browsers to receive real time events over.

Are you really done?
============================

Turns out the  issue I was having  is that because lines  in a reactor
are not  guaranteed to return before  executing the next line.  So, my
test assertions where  happening before my code completed.  Here is an
example:

{% highlight ruby %}
it 'will send incoming webhooks to a channel' do
  EM.run do
    server.socket_start_listening
    conn.callback do
      conn.send_msg('LGTM')
      server.channel('incoming-hook').subscribe { |msg|
        expect(msg).to eq('LGTM')
      }
    end
  end
end
{% endhighlight %}

We want  to make sure  that messages sent from  the browser go  to the
`incoming-hook` event machine channel. The  issue here is that despite
the  callbacks, we  still can't  really be  sure that  the events  are
happening before we are running our assertion.

Lets get some help
---------------------

To start, lets replace the `EM.run` with a version that has a time out
and handles unexpected behavior a bit more robustly.

{% highlight ruby %}
def event_block
  Timeout::timeout(5) do
    EM.epoll
    EM.run do
      yield
    end
  end
rescue Timeout::Error
  fail "Eventmachine ran out of time."
end
{% endhighlight %}

Cool, so now  that we have a error  state if a test takes  more than 5
seconds, lets look at checking off the things that we have done.

{% highlight ruby %}
def steps(*steps)
  @_em_steps = *steps
end

def complete(step)
  @_em_steps.delete(step)
  EM.stop if @_em_steps.empty?
end
{% endhighlight %}

Great! now that we have steps,
lets make the error message on our timeout block more informative.

{% highlight ruby %}
def event_block
  Timeout::timeout(5) do
    EM.epoll
    EM.run do
      yield
    end
  end
rescue Timeout::Error
  fail "Eventmachine was not stopped with #{@_em_steps} left"
end
{% endhighlight %}

Now we can delete events as they complete.
Lets apply these two helpers to our code.

{% highlight ruby %}
it 'will send incoming webhooks to a channel' do
  steps :open, :receive
  event_block do
    server.socket_start_listening
    conn.callback do
      conn.send_msg('LGTM')
      complete :open
      server.channel('incoming-hook').subscribe { |msg|
        expect(msg).to eq('LGTM')
        complete :receive
      }
    end
  end
end
{% endhighlight %}

Now our test implicitly has two very useful tests.
- Code takes longer than 5 seconds to run (which in event machine is an eternity)
- Did I complete all the steps that I expected to complete.

I found this super helpful, and  I totally recommend that you checkout
[Nick Gauthier's](http://ngauthier.com/). His post seriously helped me
out when I really had no idea what I was doing.
