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
gem 'makara'
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
        url: <%= ENV["DATABASE_URL"] %>
      - name: replica
        url: <%= ENV["REPLICA_DATABASE_URL"] %>
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

module DefaultToMasterPool
  def _appropriate_pool(*args)
    return @master_pool unless Thread.current[:distribute_reads]
    super
  end
end

Makara::Proxy.send :prepend, DefaultToMasterPool

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

:heart: Happy scaling
