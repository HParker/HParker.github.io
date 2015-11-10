---
layout: post
title: Experimenting with Mutant
date: 2015-11-06 16:26:43 -0700
categories: ruby experiment testing rspec
---


I noticed a couple of interesting things when I experimented with [Mutant](https://github.com/mbj/mutant). If you are unfamiliar with Mutant, basically it parses your ruby code into an [AST](https://en.wikipedia.org/wiki/Abstract_syntax_tree) and does modifications to your code to see if your code breaks. The idea is that with tools like [SimpleCov](https://github.com/colszowka/simplecov), you can see what lines your tests run, but it does not show that you are actually testing the behavior correctly. Mutant modifies your programs AST to create valid ruby that is likely to not be covered by your tests.


The project
===============


I am going to be using a little movie management app as my test app. You can find [caketop](https://github.com/XanderStrike/caketop-theater) and look at the code if you like, but really I only expect you to understand what a movie to understand what I learned using mutant.


Discoveries
-------------


lazy tests
==============


### Found


Some tests are just lazy, I found a couple examples of tests like this:

{% highlight ruby %}
it 'should parse text with markdown and put it in contents on save' do
  page = create(:page)
  expect(page.content).to_not eq(nil)
end
{% endhighlight %}


### Solution


This kind of test has a very simple solution. just expect something. In this case I went with a regular expression. This should be more robust than trying to match the whole page content which is factory generated and could change.

{% highlight diff %}
 it 'should parse text with markdown and put it in contents on save' do
   page = create(:page)
-  expect(page.content).to_not eq(nil)
+  expect(page.content).to match(/This\ is\ a\ test\ page!/)
 end

{% endhighlight %}

I noticed some changes that I was making in my code while looking through the things that broke with Mutant. I was expecting to be making changes in my tests, but in reality I ended up making at least equal amounts of changes in my code. and I think the code is better for it.

Missing special cases
-----------------

### Found


I feel like this is totally the intention of Mutant. one of my favorite finds was a very non-dry method that was missing a test. So Mutant threw up on this in a big was.

The method was,

{% highlight ruby %}
def self.update(params)
-    s = Setting.get(params[:setting])
-    s.update_attribute(:content, params[:content])
-    s.update_attribute(:boolean, (params[:boolean] == 'true'))
-    Setting.get('admin-pass').update_attributes(content: Digest::SHA256.hexdigest(params[:admin_pass])) if params[:setting] == 'admin'
end
{% endhighlight %}

In addition to not fully really understanding ruby's `self` it also has this special case for setting the admin password. This case where the

### Solution


so First thing I did was extract the special behavior around admin_pass and the unnecissary references to `Setting`

{% highlight diff %}
+    def update(params)
+      s = get(params[:setting])
+      s.update_attribute(:content, params[:content])
+      s.update_attribute(:boolean, (params[:boolean] == 'true'))
+      admin_pass(params[:admin_pass]) if params[:setting] == 'admin'
+    end
+
+    private
+
+    def admin_pass(new_password)
+      get('admin-pass').update_attributes(content: Digest::SHA256.hexdigest(new_pass))
+    end

-  def self.update(params)
-    s = Setting.get(params[:setting])
-    s.update_attribute(:content, params[:content])
-    s.update_attribute(:boolean, (params[:boolean] == 'true'))
-    Setting.get('admin-pass').update_attributes(content: Digest::SHA256.hexdigest(params[:admin_pass])) if params[:setting] == 'admin'
end
{% endhighlight %}

Then I added the missing test case,

{% highlight ruby %}
  describe 'admin' do
it 'will sha the admin_pass' do
  setting = create(:setting, name: 'testsetting', boolean: false)
  Setting.update(setting: 'admin_pass', content: 'mypwd')
  expect(Setting.get('testsetting').content).to_not be nil
end

{% endhighlight %}



Constants over static methods
=========================================


### Found

First thing I ran into was a a fun little method for on the model that listed the different sort orders you could use when sorting movies.

{% highlight ruby %}
class Movie < ActiveRecord::Base

...

def self.sort_orders
  [
    ['Title (asc)', 'title asc'],
    ['Title (desc)', 'title desc'],
    ['Release Date (asc)', 'release_date asc'],
    ['Release Date (desc)', 'release_date desc'],
    ['Runtime (asc)', 'runtime asc'],
    ['Runtime (desc)', 'runtime desc'],
    ['TMDB Rating (asc)', 'vote_average asc'],
    ['TMDB Rating (desc)', 'vote_average desc'],
    ['Revenue (asc)', 'revenue asc'],
    ['Revenue (desc)', 'revenue desc'],
    ['Added (asc)', 'added asc'],
    ['Added (desc)', 'added desc']
  ]
end
{% endhighlight %}

Clearly, first thing I thought was, "wow this method should be a constant." the tests that Mutant ran on this file inserted `nil` or `self` throughout the this array of arrays. I realize that I didn't care about testing that because this array will likely always be exactly like this as long as those are the fields on a Movie you can sort.

### Solution

{% highlight diff %}
 class Movie < ActiveRecord::Base
+  SORT_ORDERS = [
+    ['Title (asc)', 'title asc'],
+    ['Title (desc)', 'title desc'],
+    ['Release Date (asc)', 'release_date asc'],
+    ['Release Date (desc)', 'release_date desc'],
+    ['Runtime (asc)', 'runtime asc'],
+    ['Runtime (desc)', 'runtime desc'],
+    ['TMDB Rating (asc)', 'vote_average asc'],
+    ['TMDB Rating (desc)', 'vote_average desc'],
+    ['Revenue (asc)', 'revenue asc'],
+    ['Revenue (desc)', 'revenue desc'],
+    ['Added (asc)', 'added asc'],
+    ['Added (desc)', 'added desc']
+  ]
+
   has_many :genres

   has_many :encodes
@@ -15,23 +30,6 @@ class Movie < ActiveRecord::Base
     "/backdrops/#{id}.jpg"
   end

-  def self.sort_orders
-    [
-      ['Title (asc)', 'title asc'],
-      ['Title (desc)', 'title desc'],
-      ['Release Date (asc)', 'release_date asc'],
-      ['Release Date (desc)', 'release_date desc'],
-      ['Runtime (asc)', 'runtime asc'],
-      ['Runtime (desc)', 'runtime desc'],
-      ['TMDB Rating (asc)', 'vote_average asc'],
-      ['TMDB Rating (desc)', 'vote_average desc'],
-      ['Revenue (asc)', 'revenue asc'],
-      ['Revenue (desc)', 'revenue desc'],
-      ['Added (asc)', 'added asc'],
-      ['Added (desc)', 'added desc']
-    ]
-  end
-
{% endhighlight %}

When I realized that there really isn't a practical way to change this array without changing the code, I moved the method into a constant. Next step would normally be to add the method back and just reference the constant to maintain the interface, but I found only one use of this method, so I just referenced it directly there.
