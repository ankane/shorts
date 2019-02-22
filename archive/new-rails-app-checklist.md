# New Rails App Checklist

How I personally start new apps

## Create Project

Get the latest version of Rails

```sh
gem install rails
```

Create a new app

```sh
rails new <name> -d postgresql --skip-turbolinks
```

Don’t fret too much over the name - you can easily update it later

## Version Control

Add Git

```sh
git add .
git commit -m "Hello app"
```

## App Config

Make a few updates to `config/application.rb`

Disable unwanted generators

```ruby
config.generators do |g|
  g.assets false
  g.helper false
  g.test_framework nil
end
```

Set time zone

```ruby
config.time_zone = "Pacific Time (US & Canada)"
```

## Services

Create a directory for services

```sh
mkdir app/services
```

## Templates

Add [Haml](https://github.com/indirect/haml-rails)

```ruby
gem 'haml-rails'
```

and run

```sh
rake haml:erb2haml
```

## Console

Add [Awesome Print](https://github.com/awesome-print/awesome_print)

```ruby
gem 'awesome_print', require: false
```

and have it run when the console starts in `config/application.rb`

```ruby
console do
  require "awesome_print"
  AwesomePrint.irb!
end
```

## Environment

Add [dotenv](https://github.com/bkeepers/dotenv)

```ruby
gem 'dotenv-rails', groups: [:development, :test]
```

Create an env file and exclude it from version control

```ruby
touch .env
echo ".env" >> .gitignore
```

## Lastly

Run

```sh
rails db:create
rails s
```

and create something awesome
