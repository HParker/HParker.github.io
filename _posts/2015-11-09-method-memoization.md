---
layout: post
title: Caching methods with Ruby 2.1's method definition return value
date: 2015-11-06 16:26:43 -0700
categories: ruby performance metaprogramming
---

> You can see the final product of this article [here](https://github.com/HParker/ruby_memoize_method)

as of [Ruby 2.1.0](https://bugs.ruby-lang.org/issues/7998), method definitions grew a return value. They now return the method name as a symbol. This allow for some interesting things like inline `private` method declaration.

{% highlight ruby %}
private def foo
  ...
end
{% endhighlight %}

Replaces,

{% highlight ruby %}
def foo
  ...
end
private :foo
{% endhighlight %}

> **Note:** Inline private method declarations only works for instance methods. Because `def` is returning a symbol representing the method name, it does not know what the method was defined on.


so this got me thinking, what else could you use in front of a class definition.



Cache method
--------------

> **Note:** Throughout this post I used `cache` and `memoize` interchangably. In reality, `memoization` is a special case of caching. In this article, all caching is memoization caching.


I see the `@expensive_value ||= expensive_computation` pattern far too much. I wonder if we can do that automatically by using the return value of `def`.

{% highlight ruby %}
cache def foo
  expensive_computation
end
{% endhighlight %}

I kinda like the look of this and after exploring this problem some, turns out the implementation is not that ridiculous either.

{% highlight ruby %}
module MemoizeMethod
  def cache(method)
    recompute_method = "recompute_" + method.to_s
    cache_var = "@" + method.to_s + "_cache"
    alias_method recompute_method, method
    define_method(method) { |*args|
      instance_eval(cache_var) ||
        instance_eval("#{cache_var} ||= #{send(recompute_method, *args)}")
    }
  end
end
end

{% endhighlight %}

> **NOTE:** I dislike the double `instance_eval`, but I couldn't find a better way of sending arguments along to the `recompute` method.

To get this new behavior just `extend MemoizeMethod` in your class and enjoy method caching.

Lets walk through the code

- we alias the method we have just defined to the `recompute_` prefixed version of itself
- name the variable we will be storing the expensive computation in
- then, redefine the method we just aliased to first check the instance variable, then do the computation if it isn't already saved.

an added advantage of this approach is that we have access to the `recompute_method` method else where in the class. This lets us bust our own cache should we choose. There are tons of uses for having a return value from function definition. you can store source anotations in in the ruby process, or use it to add register the method with a callback. I can't honestly say that this is code I would use in production. After all we made simple code slightly simpler and introduced a big glob of magic in the middle. I would have to get a lot more iritated with `@this ||= computation` than I already currently.
