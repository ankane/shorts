# Development Rails

Best practices for developing with Rails

## Environment

Use [dotenv-rails](https://github.com/bkeepers/dotenv) for environment variables. Do not check this in. Create a `.env.example` without secrets.

Use [foreman](https://github.com/ddollar/foreman) to manage multiple processes.

## Debugging

Set up debugging tools behind environment variables so you can enable and disable without changing code.

Use [ruby-prof](https://github.com/ruby-prof/ruby-prof) to profile code.

Use [bullet](https://github.com/flyerhzm/bullet) to find n + 1 queries.

## Console

Enable console history and autocomplete in your `~/.irbrc`.

```ruby
require "irb/completion"
require "irb/ext/save-history"
IRB.conf[:SAVE_HISTORY] = 10000

# and for awesome print
require "awesome_print"
AwesomePrint.irb!
```

Use [awesome_print](https://github.com/michaeldv/awesome_print) or [pry-rails](https://github.com/rweng/pry-rails) for a friendlier console.

If you experience [double logging in the Rails console](https://github.com/rails/rails/issues/11415), create `config/initializers/logger.rb` with:

```ruby
if Rails.env.development?
  ActiveSupport::Logger.class_eval do
    def self.broadcast(logger)
      Module.new do
      end
    end
  end
end
```

## Logging

[Instrument caches](https://github.com/ankane/shorts/blob/master/Instrumenting-Caches.md) like ActiveRecord queries.

## Aliases

Use aliases for common commands. Here are a few of my favorites:

```sh
# rails
alias rc="bin/rails console"
alias dbm="bin/rake db:migrate"
alias fsw="foreman start -c web=1"

# git
alias gl="git pull -r"
alias gc="git commit"
alias gco="git checkout"
alias gcm="git checkout master"
```

## Also...

Check out [best practices](https://github.com/ankane/production_rails) for running Rails in production.
