# Scaling Reads

One of the easier ways to scale your database is to distribute reads to replicas.

## Desire

Here’s the desired behavior:

```ruby
User.find(1)                  # master

distribute_reads do
  # use replica for reads
  User.maximum(:visits_count) # replica
  User.find(2)                # replica

  # until a write
  # then switch to master
  User.create!                # master
  User.last                   # master
end
```

## Contenders

We looked at a number of libraries, including [Octopus](https://github.com/tchandy/octopus), [Octoshark](https://github.com/dalibor/octoshark), and [Replica Pools](https://github.com/kickstarter/replica_pools).

The winner was [Makara](https://github.com/taskrabbit/makara) - it handles failover well and has a simple configuration.

## Getting Started

First, install Makara.

```ruby
gem 'makara', github: 'taskrabbit/makara'
```

There are 3 important `ENV` variables in our setup.

- `DATABASE_URL` - master database
- `REPLICA_DATABASE_URL` - replica database (can use the master database in development)
- `MAKARA` - feature flag for a smooth rollout

Here are sample values:

```sh
DATABASE_URL=postgres://nerd:secret@localhost:5432/db_development
REPLICA_DATABASE_URL=postgres://nerd:secret@localhost:5432/db_development
MAKARA=true
```

Next, update `config/database.yml`.

```yml
development: &default
  <% if ENV["MAKARA"] %>
  url: postgresql-makara:///
  makara:
    sticky: true
    connections:
      - role: master
        name: master
        <% uri = URI.parse(ENV["DATABASE_URL"]) %>
        host: <%= uri.host %>
        port: <%= uri.port %>
        database: <%= uri.path.tr("/", "") %>
        username: <%= uri.user %>
        password: <%= uri.password %>
      - name: replica
        <% uri = URI.parse(ENV["REPLICA_DATABASE_URL"]) %>
        host: <%= uri.host %>
        port: <%= uri.port %>
        database: <%= uri.path.tr("/", "") %>
        username: <%= uri.user %>
        password: <%= uri.password %>
  <% else %>
  adapter: postgresql
  url: <%= ENV["DATABASE_URL"] %>
  <% end %>

production:
  <<: *default
```

We don’t use the middleware, so we remove it by adding to `config/application.rb`:

```ruby
config.middleware.delete Makara::Middleware
```

Also, we want to read from master by default so have to patch Makara. Create an initializer `config/initializers/makara.rb` with:

```ruby
Makara::Cache.store = :noop

class Makara::Proxy
  protected
  def _appropriate_pool_with_master_default(*args)
    return @master_pool unless Thread.current[:distribute_reads]
    _appropriate_pool_without_master_default(*args)
  end
  alias_method_chain :_appropriate_pool, :master_default
end

module DistributeReads
  def distribute_reads
    previous_value = Thread.current[:distribute_reads]
    begin
      Thread.current[:distribute_reads] = true
      Makara::Context.set_current(Makara::Context.generate)
      yield
    ensure
      Thread.current[:distribute_reads] = previous_value
    end
  end
end

Object.send :include, DistributeReads
```

To distribute reads, use:

```ruby
total_users = distribute_reads { User.count }
```

You can also put multiple lines in a block.

```ruby
distribute_reads do
  User.max(:visits_count)
  Order.sum(:revenue_cents)
  Visit.average(:duration)
end
```

## Test Drive

In the Rails console, run:

```ruby
User.first                       # master
distribute_reads { User.last }   # replica
```

You’re set.

## New Relic

There’s a [big performance issue](https://github.com/newrelic/rpm/commit/82d2777a4222deb746467783eb0226ad60d307e7) when the latest versions of New Relic and Makara unite.  Use the `dev` branch to get around this.

```ruby
gem 'newreplic_rpm', github: 'newrelic/rpm', branch: 'dev'
```

:heart: Happy scaling
