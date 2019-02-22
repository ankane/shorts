# Scaling the Monolith

Many companies start out with a single web application. As the team and codebase grow, things feel less organized and common tasks like booting the app and running the test suite take longer and longer. It can be tempting to turn to microservices to alleviate some of this pain. However, distributed systems add a significant amount of complexity and mental overhead.

Before you decide to split apart your app, there are a number of tactics you can use to scale it [majestically](https://m.signalvnoise.com/the-majestic-monolith-29166d022228#.bst5vwy6r). Spend a significant amount of time trying to solve your existing problems before making big changes.

The topics we’ll cover are:

- [Code](#code)
- [Errors](#errors)
- [Boot Times & Memory](#boot-times-memory)
- [Testing](#testing)
- [Databases](#databases)
- [Stability](#stability)

The examples are geared towards Rails apps, but the principles apply to any codebase.

## Code

Rails models and controllers tend to get larger and larger. Rails introduced [concerns](https://signalvnoise.com/posts/3372-put-chubby-models-on-a-diet-with-concerns) as one way to address this. Concerns allow you to pull out related logic into a separate file.

Service objects are another nice pattern for this. Here’s [an example](https://hackernoon.com/service-objects-in-ruby-on-rails-and-you-79ca8a1c946e) of a service object. There’s not a standard way to create service objects, but it’s a good idea to decide on a convention for your app. You can use gems like [Interactor](https://github.com/collectiveidea/interactor) to establish one.

Use namespaces to organize code.

```ruby
class Admin::UsersController < Admin::BaseController
end
```

Some teams also prefer to use Rails engines, although I’m not a fan of this approach. Here’s a [good comparison](https://stackoverflow.com/a/29641532/1177228) of the pros and cons of each.

## Errors

As the team grows, it’s important that errors get routed to the right place. You can use the [ownership](https://github.com/ankane/ownership) gem to help with this. Add it to controllers, jobs, and rake tasks.

```ruby
class WelcomeJob < ApplicationJob
  owner :growth
end
```

`git blame` can help with assigning initial owners.

## Boot Times & Memory

As your app accumulates more gems and files, its boot time and memory usage grow. There have been a number of projects over the years to speed up boot time. [Spring](https://github.com/rails/spring) was introduced in Rails 4.1 and keeps your app running in the background so it doesn’t have to boot every time you run a new command.

Last year, Shopify released [Bootsnap](https://github.com/Shopify/bootsnap), which caches expensive loading computations. It’s now part of Rails 5.2 and can be used with earlier versions of Rails as well. With Bootsnap, “the core Shopify platform - a rather large monolithic application - boots about 75% faster, dropping from around 25s to 6.5s.”

Another tactic is lazy loading files. Instead of incurring a speed and memory penalty at startup to load files, you can incur it the first time a request or job requires it. If it’s never needed, it’s never loaded. You can specify which gems to load in your Gemfile.

```rb
gem 'groupdate', require: false
```

You can also use different Bundler groups to selectively load gems for different environments.

```rb
group :web do
  gem 'rack-attack'
end

group :admin_web do
  gem 'activeadmin'
end

group :worker do
  gem 'premailer-rails'
end
```

Read how to [set it up here](https://engineering.harrys.com/2014/07/29/hacking-bundler-groups.html).

Use [Bumbler](https://github.com/nevir/Bumbler) to see how long each gem takes to load and [Derailed Benchmarks](https://github.com/schneems/derailed_benchmarks) to see memory usage. Focus on the top ones and leave the rest.

If a gem is slow, there’s a chance it may be doing a lot of work upfront. You can try to debug the gem and fix it. Here’s an [example](https://github.com/ankane/area/commit/2c8cc47d151828ebdcce0e7060b7ac77a4c2f9ce) of speeding up initial load time by only reading a CSV file when it’s needed.

## Testing

As the number of tests grow, the test suite can become slow. [TestProf](https://test-prof.evilmartians.io) provides a number of tools to profile and optimize your tests. You can also use a library like [Database Cleaner](https://github.com/DatabaseCleaner/database_cleaner) to quickly clean the database after tests.

In development, you can use Guard for [Minitest](https://github.com/guard/guard-minitest) or [RSpec](https://github.com/guard/guard-rspec) to automatically run tests when relevant files are modified. Also make sure it’s easy to manually run common subsets of tests. You can use tags in RSpec for this.

```sh
rspec --tags growth
```

The key to speeding up the entire test suite is parallelization. Stripe has a [great post](https://stripe.com/blog/distributed-ruby-testing) about how they were able to get three hours of tests to run in three minutes. With continuous integration, split tests across multiple machines. Both [Travis](https://docs.travis-ci.com/user/speeding-up-the-build/#parallelizing-your-builds-across-virtual-machines) and [Circle](https://circleci.com/docs/2.0/parallelism-faster-jobs/) support this. You can use [ParallelTests](https://github.com/grosser/parallel_tests) in development to use all the cores on your machine. Rails 6 will run tests in parallel by default.

Another way to speed up tests is to change your schema dump format to SQL.

```ruby
config.active_record.schema_format = :sql
```

This allows you to load the database schema for tests without booting the Rails app. With Postgres, you can use:

```sh
psql < db/structure.sql
```

To prevent slow tests from being added, automatically fail tests that take too long. With RSpec, you can do:

```ruby
RSpec.configure do |config|
  config.around(:each) do |example|
    duration = Benchmark.realtime(&example)
    raise "Test took over 2 seconds to run" if duration > 2
  end
end
```

Start with a higher value and ratchet it down as you fix tests that are slow. You can see the slowest tests with:

```sh
rspec --profile
```

As the number of tests grows, there’s a higher chance of a random network issue causing an individual test to fail. Automatically retry failing tests to cut down on noise. With RSpec, you can use [RSpec::Retry](https://github.com/NoRedInk/rspec-retry) for this.

```ruby
require "rspec/retry"

RSpec.configure do |config|
  config.around(:each) do |example|
    example.run_with_retry retry: 2 # must be 2 to retry once (shrug)
  end
end
```

For test failures, make sure they get routed to the committer. You can use webhooks from your CI platform to do this.

## Databases

Modern relational databases can scale extremely well if you follow best practices.

One of the most important things you can do is set a [statement timeout](https://github.com/ankane/the-ultimate-guide-to-ruby-timeouts#statement-timeouts-1) to prevent bad queries from taking too many resources.

```yml
production:
  variables:
    statement_timeout: 250 # ms
```

It’s also good to track which queries consume the most CPU time. With Postgres, you can use [PgHero](https://github.com/ankane/pghero) for this.

<p style="text-align: center;"><img src="/images/query-stats.png" alt="PgHero" /></p>

Use [Marginalia](https://github.com/basecamp/marginalia) to make it easy to identity the origin of queries. This adds a comment to the end of queries like `/*application:Datakick,controller:items,action:edit*/` so you can see where they’re coming from.

Add defensive measures as well. For instance, pause low priority job queues automatically when the database CPU gets too high.

```ruby
Sidekiq::Queue.new("low").pause!
```

As the team grows, so does the chance of someone accidentally running a migration that takes down the site. [Strong Migrations](https://github.com/ankane/strong_migrations) can help prevent downtime due to database migrations. It raises an error if you try to run an unsafe operation and gives instructions for a better way to do it.

<p style="text-align: center;"><img src="/images/strong-migrations.png" alt="Strong Migrations" /></p>

Some tables can accumulate a lot of columns. You can split them into multiple tables based off concern that have a 1-to-1 relationship.

Scale reads by fixing N+1 queries and caching frequent queries. [Bullet](https://github.com/flyerhzm/bullet) can help you identify N+1 queries. If you still have high load after spending a good amount of time on these, use [Distribute Reads](https://github.com/ankane/distribute_reads) for replicas.

Scale writes and space with additional databases. Use [Multiverse](https://github.com/ankane/multiverse) to manage them. This can also be good if you have business domains with different workloads. It adds complexity and removes the ability to join certain tables, but can increase stability.

Partitioning is another strategy for space for tables that only need recent data. You can use [pgslice](https://github.com/ankane/pgslice) for Postgres.

While Rails has built-in connection pooling, connections can become an issue when you have a lot of servers. With Postgres, use a connection pooler like [PgBouncer](https://ankane.org/pgbouncer-setup) when you start to hit 500 connections.

Be hesitant to introduce new data stores. Most of the time you can [just table data](https://ankane.org/just-table-it). It’s often not worth having another technology to manage if your current stack can do the job.

## Stability

Your monolith is one codebase, but you can increase stability by isolating different parts of the app in production. Have separate load balancers and web servers for your customer site and admin site so customers aren’t impacted if the admin site goes down. Use separate workers for different groups of queues so a backed up queue or bad job won’t affect the whole system.

You can separate by business domain, which will be aligned with teams if you have vertical teams. This also allows you to scale different parts of your app independently as if they were different services.

## Conclusion

As you’ve seen, there are a number of things you can do to scale your monolith. Focus on developer happiness and productivity as well as system stability. Keep track of metrics over time that impact developers, like boot time, test suite time, and deploy time. It’s also good to invest in projects that make it comfortable to ship code fast, like quick rollbacks. Overall, spend a decent amount of time trying to solve your exact pain points before breaking your app apart to solve them.
