# Gem Patterns

I’ve created [a few](https://ankane.org/opensource?language=Ruby) Ruby gems over the years, and there are a number of patterns I’ve found myself repeating that I wanted to share. I didn’t invent them, but have long forgotten where I first saw them. They are:

- [Rails Migrations](#rails-migrations)
- [Rails Dependencies](#rails-dependencies)
- [Testing Against Multiple Dependency Versions](#testing-against-multiple-dependency-versions)
- [Testing Against Rails](#testing-against-rails)
- [Coding Your Gemspec](#coding-your-gemspec)

Let’s dig into each of them. In the examples, the gem is called `hello`.

## Rails Migrations

Create a template in `lib/generators/hello/templates/migration.rb.tt`:

```ruby
class <%= migration_class_name %> < ActiveRecord::Migration<%= migration_version %>
  def change
    # your migration
  end
end
```

The `.tt` extension denotes Thor template. [Thor](https://github.com/erikhuda/thor) is what Rails uses under the hood.

Add `lib/generators/hello/install_generator.rb`

```ruby
require "rails/generators/active_record"

module Hello
  module Generators
    class InstallGenerator < Rails::Generators::Base
      include ActiveRecord::Generators::Migration
      source_root File.join(__dir__, "templates")

      def copy_migration
        migration_template "migration.rb", "db/migrate/install_hello.rb", migration_version: migration_version
      end

      def migration_version
        "[#{ActiveRecord::VERSION::MAJOR}.#{ActiveRecord::VERSION::MINOR}]"
      end
    end
  end
end
```

This lets you run:

```sh
rails generate hello:install
```

Change the generator path and class name to match your gem. They must match exactly what Rails expects to work.

[Example](https://github.com/ankane/archer/blob/master/lib/generators/archer/install_generator.rb)

## Rails Dependencies

If your gem depends on Rails, add `railties` and any other Rails libraries it needs.

```ruby
spec.add_dependency "railties", ">= 5"
spec.add_dependency "activerecord", ">= 5"
```

I typically require a [supported version](https://rubyonrails.org/security/) of Rails.

In code, don’t require Rails gems directly, as this can cause them to load early and introduce issues.

```ruby
require "active_record" # bad!!

ActiveRecord::Base.include(Hello::Model)
```

Instead, do:

```ruby
require "active_support"

ActiveSupport.on_load(:active_record) do
  include Hello::Model
end
```

[Example](https://github.com/ankane/hightop/blob/master/lib/hightop.rb)

## Testing Against Multiple Dependency Versions

If your gem has dependencies, you may want to test against multiple versions of a dependency. For instance, you may want to test against multiple versions of Active Record.

To do this, create a `test/gemfiles` directory (or `spec/gemfiles` if you use RSpec).

Create `test/gemfiles/activerecord50.gemfile` with:

```ruby
source "https://rubygems.org"

gemspec path: "../../"

gem "activerecord", "~> 5.0.0"
```

Install with:

```sh
BUNDLE_GEMFILE=test/gemfiles/activerecord50.gemfile bundle install
```

And run with:

```sh
BUNDLE_GEMFILE=test/gemfiles/activerecord50.gemfile bundle exec rake
```

[Example](https://github.com/ankane/groupdate/tree/master/test/gemfiles)

On Travis CI, you can add to `.travis.yml`:

```yml
gemfile:
  - Gemfile
  - test/gemfiles/activerecord50.gemfile
```

You can also use a library like [Appraisal](https://github.com/thoughtbot/appraisal) to help generate and run these files.

## Testing Against Rails

To test against Rails, use a library like [Combustion](https://github.com/pat/combustion). It’s designed to be used with RSpec, but I haven’t had any issues with Minitest. Combustion generates some files that aren’t needed, so I just delete them.

```ruby
Combustion.initialize! :all
```

[Example](https://github.com/ankane/field_test/tree/master/test)

## Coding Your Gemspec

There are a variety of ways to code your gemspec. Here’s the one I like to use:

```ruby
require_relative "lib/hello/version"

Gem::Specification.new do |spec|
  spec.name          = "hello"
  spec.version       = Hello::VERSION
  spec.summary       = "Hello world"
  spec.homepage      = "https://github.com/you/hello"
  spec.license       = "MIT"

  spec.author        = "Your Name"
  spec.email         = "you@example.com"

  spec.files         = Dir["*.{md,txt}", "{lib}/**/*"]
  spec.require_path  = "lib"

  spec.required_ruby_version = ">= 2.4"

  spec.add_dependency "activesupport", ">= 5"

  spec.add_development_dependency "bundler"
  spec.add_development_dependency "rake"
end
```

Change `files` if your gem has `app`, `config`, or `vendor` directories. I typically use the [last supported version](https://www.ruby-lang.org/en/downloads/branches/) for the minimum Ruby version.

If your gem has an executable file, add:

```ruby
spec.bindir        = "exe"
spec.executables   = ["hello"]
```

[Don’t check in](https://yehudakatz.com/2010/12/16/clarifying-the-roles-of-the-gemspec-and-gemfile/) `Gemfile.lock`.

Some gems have moved development dependencies entirely out of the gemspec and into the Gemfile, which is another option.

## Summary

You’ve now seen five patterns that can be useful for Ruby gems. Now go build something awesome!
