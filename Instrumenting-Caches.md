# Instrumenting Caches

Love seeing how long ActiveRecord queries take and want the same for your cache stores?

:four_leaf_clover: You’re in luck.

Create `config/initializers/cache_instrumenters.rb` and add the code below.

## Rails Cache

Silence the default logging

```ruby
Rails.cache.silence!
```

And if different than Redis

```ruby
class CacheInstrumenter < ActiveSupport::LogSubscriber
  def cache_read(event)
    return unless logger.debug?
    name = "%s (%.2fms)" % ["Cache Read", event.duration]
    debug "  #{color(name, BLUE, true)} #{event.payload[:key]} #{event.payload.except(:key)}"
  end

  def cache_write(event)
    return unless logger.debug?
    name = "%s (%.2fms)" % ["Cache Write", event.duration]
    debug "  #{color(name, BLUE, true)} #{event.payload[:key]}"
  end
end

if Rails.env.development? || Rails.env.test?
  CacheInstrumenter.attach_to(:active_support)
end
```

## Redis

```ruby
class RedisInstrumenter < ActiveSupport::LogSubscriber
  def query(event)
    return unless logger.debug?

    name = "%s (%.2fms)" % ["Redis", event.duration]
    cmds = event.payload[:query]

    output = "  #{color(name, RED, true)}"

    cmds.each do |name, *args|
      if args.present?
        output << " [ #{name.to_s.upcase} #{args.join(" ")} ]"
      else
        output << " [ #{name.to_s.upcase} ]"
      end
    end

    debug output
  end
end

if Rails.env.development? || Rails.env.test?
  RedisInstrumenter.attach_to(:redis)
end
```

## Thanks

- [This Gist from Jose Angel Cortinas](https://gist.github.com/jacortinas/1058808)
