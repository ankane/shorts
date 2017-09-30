# Instrumenting Caches

Love seeing how long ActiveRecord queries take and want the same for your cache stores?

:four_leaf_clover: You’re in luck.

Create `config/initializers/cache_instrumenters.rb` and add the code below.

## Memcached

```ruby
module MemcachedNotifications
  def request(op, *args)
    payload = {
      op: op,
      args: args
    }
    ActiveSupport::Notifications.instrument("request.memcached", payload) do
      super
    end
  end
end

class MemcachedInstrumenter < ActiveSupport::LogSubscriber
  def request(event)
    return unless logger.debug?

    name = "%s (%.2fms)" % ["Memcached #{event.payload[:op].to_s.titleize}", event.duration]
    debug "  #{color(name, BLUE, true)} #{event.payload[:args].join(" ")}"
  end
end

unless Rails.env.production?
  Dalli::Server.prepend(MemcachedNotifications)
  MemcachedInstrumenter.attach_to(:memcached)
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

RedisInstrumenter.attach_to(:redis) unless Rails.env.production?
```

## Thanks

- [This Gist from Jose Angel Cortinas](https://gist.github.com/jacortinas/1058808)
