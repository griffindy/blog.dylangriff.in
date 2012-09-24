---
layout: post
title: "Using Guard and Jasmine to Make CoffeeScript Testing Fast"
date: 2012-09-23 14:21
comments: true
categories: [code, coffeescript, javascript, guard]
---

When I'm developing in Ruby and Rails using RSpec, I have a mapping in my vim
configuration for `,l` that runs the current test in a separate tmux window.
This is incredibly useful for testing early and often and doing TDD to the
fullest. So when I started working my [dominion
app](http://github.com/griffindy/dominion), which uses CoffeeScript and
Backbone, I wanted to set up a similar system of quickly being able to run tests.
[Jasmine](http://pivotal.github.com/jasmine/) is one of the more widely used
frameworks for testing JavaScript applications, and is heavily influenced by
RSpec:

```javascript
describe("A suite", function() {
  it("contains spec with an expectation", function() {
    expect(true).toBe(true);
  });
});
```

In CoffeeScript, with it's simpler syntax, this looks even closer to RSpec:

```coffeescript
describe 'A suite', ->
  it 'contains spec with an expectation', ->
    expect(true).toBe(true)
```

So this is all awesome, but Jasmine doesn't run on the command line (well, it
does, but I started off wanting to use the browser). Furthermore, I had to make
sure that all of my CoffeeScript files were being compiled to the right places.
Fortunately there is a tool perfect for this that I've heard a lot about, but
never used, [Guard](https://github.com/guard/guard). From the documentation:
"Guard is a command line tool to easily handle events on file system
modifications." It uses a DSL to watch files and act when they are updated, and
looks like this:

```ruby
guard :coffeescript, :input => 'coffeescripts', :output => 'javascripts'
```

You can also pass a block to the `guard` method in order to watch multiple files
using regular expressions:

```ruby
guard :jessie do
  watch(%r{^spec/.+(_spec|Spec)\.(js|coffee)})
end
```

Where guard became especially useful for me is you can use it to automatically
refresh a browser. Similarly, you pass `'livereload'` and a block of files, and
guard will automatically reload the browser when the change. Here is the full
contents of my Guardfile, which boths compiles my CoffeeScript and reloads my
browser:

```ruby
guard :coffeescript, :output => 'javascripts' do
  watch(%r{^src/(.*)\.coffee})
end

guard :coffeescript, :output => 'spec/javascripts' do
  watch(%r{^spec/src/(.*)\.coffee})
end

guard 'livereload' do
  watch(%r{^spec/javascripts/.*/(.*)\.js})
  watch(%r{^spec/javascripts/(.*)\.js})
  watch(%r{^javascripts/.*/(.*)\.js})
  watch(%r{^javascripts/(.*)\.js})
end
```

These are all different gems, so this is what my Gemfile consists of (so I can
easily recreate this environment using `bundle install`):

```ruby
source 'https://rubygems.org'

gem 'jasmine'
gem 'guard'
gem 'guard-coffeescript'
gem 'guard-livereload'
```

That's all I need, and now when I am writing a test in CoffeeScript, if I save
the spec orthe file it's testing, it will all be compiled and then my browser
refreshed so I can see whether or not the test passed. It's two fewer keystrokes
than I would have done writing and running RSpec tests.
