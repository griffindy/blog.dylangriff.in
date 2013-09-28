---
layout: post
title: "Using Sidekiq's Redis Connections in Other Places"
date: 2013-09-28 10:07
comments: true
categories: [code, ruby, sidekiq]
---

At [Shareaholic](https://shareaholic.com) we make heavy use of [Sidekiq](http://sidekiq.org/) for
background and asynchronous jobs. Because it uses threads it consumes far fewer
resources than its most similar sibling, [Resque](https://github.com/resque/resque),
which uses a separate process for each job. We are able to run one Sidekiq
process with 28 threads for each core on our worker machines. Sidekiq wraps all
of its connections in a connection pool using the
[`connection_pool`](https://github.com/mperham/connection_pool) gem as can be
seen [here](https://github.com/mperham/sidekiq/blob/master/lib/sidekiq/redis_connection.rb#L16-L18).
Sidekiq gives the user access to its connections to redis with the
`Sidekiq.redis` method, to which you pass a block and are yielded a connection:

``` ruby
Sidekiq.redis do |connection|
  # get all the keys in redis
  connection.keys('*')
end
```

This is advantageous if you ever wanted to do your own things in redis, but
didn't want to duplicate the logic to connect. We do in fact make other
connections to redis to coordinate locks on resources (mostly domains so that we
don't point 1000+ threads at a single domain) using the
[`redis-semaphore`](https://github.com/dv/redis-semaphore/) gem. This gem allows
you to pass in your own object for connecting to redis in case you want to use
something other than the defaults or something like `Redis::Namespace`. In
order to do so, you pass it in as a key when instantiating a new
`Redis::Semaphore` instance:

``` ruby
Redis::Semaphore.new(:name, redis: redis)
```

I figured I could pass `Sidekiq.redis` and the semaphores would use the
same connections as Sidekiq. However, I got an unsatisfactory result:

``` ruby
semaphore = Redis::Semaphore.new(:name, redis: Sidekiq.redis)
ArgumentError: requires a block
```

Since ruby will try to evaluate any and all arguments, we end up calling
`Sidekiq.redis` without a block. Clearly we're going to have to do something
slightly more involved. I came up with two solutions, one of which is simpler,
but only reuses the connection information and does not take part in the pooling
Sidekiq offers. Because `Sidekiq.redis` returns the
last return from the block we can use `Sidekiq.redis { |conn| conn }`:

``` ruby
def redis
  Sidekiq.redis { |conn| conn }
end

Redis::Semaphore.new(:name, redis: redis)

=> #<Redis::Namespace:0x007fea5e233f70
 @namespace="app_name_sidekiq",
 @redis=#<Redis client v3.0.4 for redis://localhost:6379/12>,
 @warning=false>
```

Huzzah, now we don't need to define our redis connections in two places!

But as I said before, this merely returns a connection to redis, and so does not
take into account any connection pooling. To be 100% honest, I'm not even sure
of the performance characteristics of this and whether this is actually removing
a connection from the pool. I opted for this second, slightly more complicated
method, which preserves how `Sidekiq.redis` is supposed to be used, namely with
a block so that the connections can be synchronized and shared.

What we really want is something that will delegate a method to inside of the
`Sidekiq.redis {...}` block. For example:

``` ruby
class SidekiqRedisConnectionWrapper
  def get(key)
    Sidekiq.redis do |conn|
      conn.get(key)
    end
  end

  def set(key, value)
    Sidekiq.redis do |conn|
      conn.set(key, value)
    end
  end
end

wrapper = SidekiqRedisConnectionWrapper.new
wrapper.set('key', 'value')
wrapper.get('key')
# => 'value'
```

This works fine, but would it would be very tedious and duplicative to redefine
the Redis API inside of our `SidekiqRedisConnectionWrapper` class. Fortunately
ruby gives us an easy way to delegate a whole bunch of methods without even
knowing their names. Consider the following:

``` ruby
def method_missing(meth, *args, &block)
  Sidekiq.redis do |connection|
    connection.send(meth, *args, &block)
  end
end
```

This is in fact all we really need to proxy the redis commands
`Redis::Semaphore` needs through `Sidekiq`. Here is the entirety of the
`SidekiqRedisConnectionWrapper` I wrote:

``` ruby
# Because +Sidekiq.redis+ requires passing a block,
# we can't pass it straight to +Redis::Semaphore+.
# This class simply delegates every method call to
# the Sidekiq connection.
class SidekiqRedisConnectionWrapper
  def method_missing(meth, *args, &block)
    Sidekiq.redis do |connection|
      connection.send(meth, *args, &block)
    end
  end

  def respond_to_missing?(meth)
    Sidekiq.redis do |connection|
      connection.respond_to?(meth)
    end
  end
end
```

Like a [one ought to when defining `method_missing`](http://robots.thoughtbot.com/post/28335346416/always-define-respond-to-missing-when-overriding)
I have also provided a definition of `respond_to_missing?` With this code I can
use `Sidekiq.redis` and still pass it around as a redis object that other gems
can use. Double win!

I also toyed with the idea of abstracting the call to `Sidekiq.redis` into a
lambda that the class uses, but decided that this class solves a very specific
problem, though if I ever come across an object I need to pass around that
requires a block, I know where to start. Let me know if I've done anything wrong
or how you would do it in the comments.
