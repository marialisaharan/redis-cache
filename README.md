How Caching Works?

Traditionally, accessing disk has been expensive. Trying to access data from the disk frequently will have an adverse impact on performance. To counter this, we can implement a caching layer in between your application and the database server.

A caching layer doesn’t hold any data at first. When it receives a request for the data, it calls the database and stores the result in memory (the cache). All subsequent requests will be served from the cache layer, so the unnecessary roundtrip to the database server is avoided, improving performance.
Why Redis?

Redis is an in-memory, key-value store. It’s blazingly fast and data retrieval is almost instantaneous. Redis supports advanced data structures like lists, hashes, sets, and can persist to disk.

While most developers prefer Memcache with Dalli for their caching needs, I find Redis very simple to setup and easy to administer. Also, if you are using resque or Sidekiq for managing your background jobs, you probably have Redis installed already. For those who are interested in knowing when to use Redis, this discussion is a good place to start.
Prerequisites

I’m assuming you have Rails up and running. The example here uses Rails 4.2.rc1, haml to render the views, and MongoDB as the database, but the snippets in this tutorial should be compatible with any version of Rails.

You also need to have Redis installed and running before we get started. Move into your app directory, and execute the following commands:


```
$ wget http://download.redis.io/releases/redis-2.8.18.tar.gz
$ tar xzf redis-2.8.18.tar.gz
$ cd redis-2.8.18
$ make


```

The command is going to take a while to complete. Once it has completed, just start the Redis server:

```
$ cd redis-2.8.18/src
$ ./redis-server

```
Initialize Redis

There is a Ruby client for Redis that helps us to connect to the redis instance easily:

```
gem 'redis'
gem 'redis-namespace'
gem 'redis-rails'
gem 'redis-rack-cache'

```
Once these gems are installed, instruct Rails to use Redis as the cache store:

# config/application.rb


```

#...........
config.cache_store = :redis_store, 'redis://localhost:6379/0/cache', { expires_in: 90.minutes }
#.........

```
The redis-namespace gem allows us to create nice a wrapper around Redis:

# config/initializers/redis.rb

```

$redis = Redis::Namespace.new("site_point", :redis => Redis.new)

```
All the Redis functionality is now available across the entire app through the ‘$redis’ global. Here’s an example of how to access the values in the redis server (fire up a Rails console):
```
$redis.set("test_key", "Hello World!")
```
This command will create a new key called “test_key” in Redis with the value “Hello World”. To fetch this value, just do:
```
$redis.get("test_key")
```
Now that we have the basics, let’s start by rewriting our helper methods:

# app/helpers/category_helper.rb
```
module CategoryHelper
  def fetch_categories
    categories =  $redis.get("categories")
    if categories.nil?
      categories = Category.all.to_json
      $redis.set("categories", categories)
    end
    @categories = JSON.load categories
  end
end
```

The first time this code executes there won’t be anything in memory/cache. So, we ask Rails to fetch it from the database and then push it to redis. Notice the to_json call? When writing objects to Redis, we have a couple of options. One option is to iterate over each property in the object and then save them as a hash, but this is slow. The simplest way is to save them as a JSON encoded string. To decode, simply use JSON.load.

However, this comes in with an unintended side effect. When we’re retrieving the values, a simple object notation won’t work. We need to update the views to use the hash syntax to display the categories:

# app/views/category/index.html.haml
```
%h1
  Category Listing
%ul#categories
  - @categories.each do |cat|
    %li
      %h3
        = cat["name"]
      %p
        = cat["desc"]
```

Nope, this change is not reflected in our views. This is because we’ve bypassed accessing the database and all values are served from the cache. Alas, the cache is now stale and the updated data won’t be available until Redis is restarted. This is a deal breaker for any application. We can overcome this issue by expiring the cache periodically:

```
# app/helpers/category_helper.rb

module CategoryHelper
  def fetch_categories
    categories =  $redis.get("categories")
    if categories.nil?
      categories = Category.all.to_json
      $redis.set("categories", categories)
      # Expire the cache, every 3 hours
      $redis.expire("categories",3.hour.to_i)
    end
    @categories = JSON.load categories
  end
end
```
This will expire the cache every 3 hours. While this works for most scenarios, the data in the cache will now lag the database. This likely will not work for you. If you prefer to keep the cache fresh, we can use an after_save callback:


# app/models/category.rb
```
class Category
  #...........
  after_save :clear_cache

  def clear_cache
    $redis.del "categories"
  end
  #...........
end
```

Conclusion

Lower level caching is very simple and, when properly used, it is very rewarding. It can instantaneously boost your system’s performance with minimal effort.


