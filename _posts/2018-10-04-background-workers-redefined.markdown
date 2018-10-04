**DISCLAIMER:** The idea comes from a blog post that appeared at [saturnflyer.com](https://www.saturnflyer.com/blog/working-later-bridging-your-code-with-the-background). Following either post should give you roughly the same information; mine is a little more detailed.

## Preface
Every web application at some point will require a background worker of some sort. Let it be sending emails, processing large sets of data in the background, making external API calls.

And let's face - since most of Ruby devs associated with web development comes from Rails with very, very strong Convention over Configuration movement not much of us really pay attention to how to implement that which renders `mkdir app/workers` and `gem install sidekiq` our best friends. Or aren't they?

## Set a stage
Picture that - your app has a feature called Stations. The app operates in a multi-tenant environment, so each tenant (or organization) has its own set of stations.

At the end of the day, the tenant can push all of their stations as CSV into an external system.
The following code more or less depicts that:

```ruby
# app/services/export_stations.rb
class ExportStations
  def initialize(tenant)
    @tenant = tenant
    @delivery_types = fetch_delivery_types
    @collection_types = fetch_collection_types
  end

  def call
    # there is a very long processing going on
    # not to mention sending data through the wire across the world
  end

  def fetch_delivery_types
    # here we call DB for some data
  end

  def fetch_collection_types
    # here we call DB for some data
  end
end
```

Everything works within a web thread pretty well but at some point, a large number of stations make the request last longer (vital point especially when you host your app on Heroku, which timeouts any request over 30 seconds long). Or the external API might be unavailable for a while. Or there is just no need to stop a user from interacting with our app further (and blocks available web thread for a longer period of time).

Here comes the background job.

## Location  and code splitting

Let's deal first with the location of our background job we are about to create. By following the de facto standard with Sidekiq you'll probably end up with something like that:
```ruby
# app/workers/export_stations_worker.rb
class ExportStationsWorker
  include Sidekiq::Worker

  def perform(tenant)
    ExportStations.new(tenant).call
  end
end
```
which is quite ok, but has two problems.
1. You have to change every `ExportStations.new(tenant).call` to `ExportStationWorker.perform_async(tenant)`.
2. It creates a somehow distant layer. Should you or a new developer check what the `ExportStationWorker` does, you have to open that file first, find what service it calls, and then open up that service file.

Some will agree with me on that. Some will not and start grumbling the usage is perfectly valid because it follows single responsibility and by splitting into two files it takes off the burden with reading those (in fact it is contrary because you have to keep a mental model of two files...).

The thing is that making the decision whether something should be processed in background or foreground should be as easy as possible. As easy as deciding between `UserMailer.welcome(@user).deliver_now` vs `UserMailer.welcome(@user).deliver_later` which everyone in Rails world uses. I really see no point in invoking totally different objects. If we could only incorporate the worker into our service...

And sure we can. There are many ways to skin a cat and in this particular situation probably the easiest and most naive is as follows:
```ruby
# app/services/export_stations.rb
class ExportStations
  def initialize(tenant)
    @tenant = tenant
    @delivery_types = fetch_delivery_types
    @collection_types = fetch_collection_types
  end

  def call
    # there is a very long processing going on
  end

  def call_later
    ExportStationsWorker.perform_async(@tenant)
  end

  def fetch_delivery_types
    # here we call DB for some data
  end

  def fetch_collection_types
    # here we call DB for some data
  end
end
```
which does the job because it solves aforementioned problems/pain points, and you can use change using your old service in sync `ExportStations.new(tenant).call` or async mode `ExportStations.new(tenant).call_later` really easily but still, you are required to create that worker class which is just a wiring. Not to mention that lengthy DB calls in `fetch_collection_types` and `fetch_delivery_types` are called twice. Once on init of the service, and 2nd time... on init of the service, but this time from the worker.

Let's leave that for a moment and move to the next point which is...

## Sidekiq. A de facto standard

We were told that choosing an appropriate tool for a job is vital, and choosing an appropriate worker is no different, yet Sidekiq is a default solution without giving any thoughts to it. Because I'm referring to the specific requirements let me write them down.
1. The jobs have to be stored in any persistence tool; memory is out of the question here.
2. The worker has to be able to operate on multiple threads (aka workers) at the same time.
3. Should be efficient.

   So far, so good, Sidekiq got us covered. Unless...

4. Low cost on resources.
With the low cost, I mean bill on Heroku (if using) or overall maintenance cost on your own infrastructure.

That's where things start to fall apart because Sidekiq requires Redis, which is another moving part that can break. The part that you have to maintain. And backup. Or simply pay.

And this is where [Que steps in, a lovely queue library](https://github.com/chanks/que/tree/master/docs) which solves the aforementioned problems.
1. It stores data in PostgreSQL (which is already on board. If it's not Que probably is not for you, but read on), so whenever a process dies your jobs are safe and sound.
2. It uses advisory locks, so workers do not block each other while trying to work the jobs.
3. Locks are held in memory, thus no disk-write are fired. Under a heavy load, CPU would be a bottleneck, not IO. To give some numbers and compare Que to Sidekiq which is considered fast - compared to DelayedJob Sidekiq is 21x times faster while Que is 20x. One can say Sidekiq and Que are of comparable speeds. :)
4. There is no additional cost. You already have PostgreSQL with backups. Your jobs and data can't get out of sync because your jobs either commit or rollback with rest of your data. And you can run Que WITHIN your web process, so should you want e.g. minimize Heroku bill you don't have to spin up another dyno with a worker.

Sample Que worker class looks very alike Sidekiq one

```ruby
# app/workers/export_stations_worker.rb
class ExportStationsWorker < Que::Job
  def run(tenant)
    ExportStations.new(tenant).call
  end
end
```
And to use it `ExportStationsWorker.enqueue("a_tenant")`.

## Put things together

So far I've stressed about two things - easiness of calling a service in the background or foreground without having to rewrite much of the app, and about Que. Let's put those two things together.

What I like about Sidekiq is that you can transform any object into a worker. All you have to do is to include a module. That way you can get rid of the proxy class. Something very similar could be done with Que. :)

Let me start with the usage to hook you first.
```ruby
# app/services/export_stations.rb
class ExportStations
  include ProcessLater

  def initialize(tenant)
    @tenant = tenant
    @delivery_types = fetch_delivery_types
    @collection_types = fetch_collection_types
  end

  def initializer_arguments
    [@tenant] # array with arguments required to re-init the service
  end

  def call
    # there is a very long processing going on
  end

  # rest omitted
end
```
That piece of code allows you to use either `ExportStations.new("a_tenant").call` or `ExportStations.new("a_tenant").later(:call)`. Cool, isn't it? Should you have more methods in the object you could use `later(:method_name)` to call any of them in the background.
Let's take a peek into the `ProcessLater` module I've included in the service.

```ruby
module ProcessLater
  def later(which_method)
    # later_class is a Que class under the namespace of our service
    # where we push arguments required to initialize the service
    # and which method we want to call in the background
    later_class.enqueue(*initializer_arguments, which_method)
  end

  def self.included(base)
    later_class = Class.new(::ProcessLater::Later)
    base.const_set(:Later, later_class)
    later_class.class_to_run = base
  end

  private

  def later_class
    self.class.const_get(:Later)
  end

  class Later < Que::Job
    class << self
      attr_accessor :class_to_run
    end

    def class_to_run
      self.class.class_to_run
    end

    def run(*args)
      which_method = args.pop
      class_to_run.new(*process_args(args)).send(which_method)
    end

    def process_args(args)
      # this allows us to use "plain" arguments and keyword arguments as well.
      # If an arg is a hash it is retrieved from DB with string keys,
      # which had to transformed into symbols should we want to
      # support keyword arguments.
      args.map do |arg|
        arg.is_a?(Hash) ? Hash[arg.map{ |k, v| [k.to_sym, v] }] : arg
      end
    end
  end
end
```

## Summary

The above solution is rather a non-standard one, though I found it super comfortable to work with. In the project where I introduced that approach, the background workers came relatively late and quite a few services were required to run in the background. This approach allowed to introduce background with a low cost both in terms of additional resources and changing current codebase.

In addition, Que allows creating cron-like jobs, which is an additional feature (you can find more advanced stuff in [their docs](https://github.com/chanks/que/tree/master/docs))

There is one thing left. Remember the initializer?
```ruby
class ExportStations
  include ProcessLater

  def initialize(tenant)
    @tenant = tenant
    @delivery_types = fetch_delivery_types
    @collection_types = fetch_collection_types
  end
  # rest omitted
end
```

It calls for delivery and collection types, which is a total waste if we are to `later(:call)` that. That renders a question - what is the purpose of an initializer? Should it only set the stage for the class or perform some heavy-load operations up-front? That's an interesting question, but for another time. In the service, I end up lazy-instantiating the types.
