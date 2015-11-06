---
layout: post
title: Setting up Backbonejs
date:   2015-11-06 11:23:18 -0700
categories: backbone js tutorial
---

# Setting up backbonejs

I was surprised to not find an easy, _setting up backbone with npm_. The popular tutorials seem to take some shortcuts to get you in the code quickly. I really wanted to know how to get set _fo' reals_. after all, I want to start working with backbone not because I think it will be good for me, but because **I want to make something.**

# Getting npm

I am not an authority on setting up npm, but there are [good instruction here](https://nodejs.org/download/) and npm is available via [Homebrew](http://brew.sh/) which, for a mac user, is by far the easiest.

# Setting up package.json

npm will use a JSON file called `package.json` to express dependencies and provide information about the file. for Ruby programmers `package.json` is the equivalent of a `Gemfile`.

## Getting the basics

easiest way to get started is to use the `npm init` command to get the very basics into the `package.json` file.
`npm init` will walk you through the steps to setup your app, after that you have a basic project.json that looks something like,

`package.json`

{% highlight javascript %}
{
  "name": "example",
  "version": "0.0.0",
  "description": "Example project",
  "main": "index.js",
  "scripts": {
    "test": "grunt dev"
  },
  "keywords": [
    "Example"
  ],
  "author": "Adam",
  "license": "MIT"
  }
{% endhighlight %}

## Adding dependencies

So now we have the basic npm package setup, now we need to express our dependencies. The conventions are important and specific. npm is going to look for the `dependencies` key at the top level. after that the format is `"<package>": "<version>"`

So the simplest thing that works for a simple backbone project just adds underscore, jquery and backbone.

`package.json`

{% highlight javascript %}
{
  ...

  "dependencies": {
    "underscore": "1.8.3",
    "jquery": "2.1.4",
    "backbone": "1.2.1"
  }
}
{% endhighlight %}

Additionally, npm can do the work for you. if you want something in your package json just type `npm install jsdom --save` into your console and npm will add it for you and install the dependency


# Where is the structure?

Backbone does not decide a structure for you. Backbone only really cares that you use a REST interface and have underscore.js. So the rest is up to you. This is a great chance to shoot yourself in the foot, but also part of what makes backbone great.

## some sane defaults

I think that it makes sense to have dir for each of the _what is a_ sections on the [cdnjs docs](https://cdnjs.com/libraries/backbone.js). So the simple template I would start with is something like:

`Tree`

{% highlight bash %}

$ tree src -L 1
src
├── app.js
├── collections
├── models
├── routers
├── templates
└── views
{% endhighlight %}

This way each folder is responsible for a particular type of file you are going to need in your backbone app

# Grunt for minifying and running tests

Now when I started with Backbone I wasted far too much time on my grunt file. What I ended up with was not that complicated.

First things first, we have our little magic incantation at the top of the file,

{% highlight javascript %}
module.exports = function(grunt) {

  grunt.initConfig({
    pkg: grunt.file.readJSON('package.json'),
{% endhighlight %}


Then we setup concat to join all my files together,

{% highlight javascript %}
    concat: {
      options: {
        separator: ';'
      },
      dist: {
        src: [
          'src/templates/*.js',
          'src/models/*.js',
          'src/collections/*.js',
          'src/views/*.js',
          'src/routers/*.js'
        ],
        dest: 'dist/<%= pkg.name %>.js'
      }
    },
{% endhighlight %}

After that we can uglify our script,

{% highlight javascript %}
    uglify: {
      options: {
        banner: '/*! <%= pkg.name %> <%= grunt.template.today("dd-mm-yyyy") %> */\n'
      },
      dist: {
        files: {
          'dist/<%= pkg.name %>.min.js': ['<%= concat.dist.dest %>']
        }
      }
    },
{% endhighlight %}

Then setup jshint and testing,

{% highlight javascript %}
    jshint: {
      files: ['gruntfile.js', 'src/**/*.js', 'test/**/*.js'],
      options: {
        // options here to override JSHint defaults
        globals: {
          jQuery: true,
          console: true,
          module: true,
          document: true
        }
      }
    },

    jasmine: {
      testAll: {
        src: 'src/**/*.js',
        options: {
          version: '2.0.1',
          specs: 'spec/*Spec.js'
        }
      }
    },
{% endhighlight %}

For extra credit we can setup jshit to watch our files and do whatever we would like when they change,

{% highlight javascript %}
    watch: {
      files: ['<%= jshint.files %>'],
      tasks: ['jshint']
    }
{% endhighlight %}


# conclusion

Now the rest is up to you. This is what I needed to do to get javascript tested and compiled into a single file I could include in another app. So hopefully you can spend less time bouncing around the internet trying to set this up and get straight to the good part and write something interesting.
