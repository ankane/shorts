# Rails on Heroku

[The official guide](https://devcenter.heroku.com/articles/getting-started-with-rails4) is a great place to start, but there’s more you can do to make life easier.

:tangerine: Based on lessons learned in the early days of [Instacart](https://www.instacart.com/)

## Deploys

For zero downtime deploys, enable [preboot](https://devcenter.heroku.com/articles/preboot). This will cause deploys to take a few minutes longer to go live, but it’s better than impacting your users.

```sh
heroku features:enable -a appname preboot
```

First, make sure your app boots. Create `lib/tasks/preload.rake` with:

```ruby
task preload: :environment do
  Rails.application.eager_load!
  ::Rails::Engine.subclasses.map(&:instance).each { |engine| engine.eager_load! }
  ActiveRecord::Base.descendants
end
```

Add a [custom buildpack](https://github.com/gunpowderlabs/buildpack-ruby-rake-deploy-tasks) to run the preload script and (optionally) migrations.

```ruby
heroku buildpacks:add https://github.com/heroku/heroku-buildpack-ruby
heroku buildpacks:add https://github.com/gunpowderlabs/buildpack-ruby-rake-deploy-tasks
heroku config:set DEPLOY_TASKS="preload db:migrate"
```

Create a deployment script in `bin/deploy`. Here’s an example:

```sh
#!/usr/bin/env bash

function notify() {
  # add your chat service
  echo $1
}

notify "Deploying"

git checkout master -q && git pull origin master -q && \
git push origin master -q && git push heroku master

if [ $? -eq 0 ]; then
  notify "Deploy complete"
else
  notify "Deploy failed"
fi
```

Be sure to `chmod +x bin/deploy`. Replace the `echo` command with a call to your chat service ([Hipchat instructions](https://github.com/hipchat/hipchat-cli)).

Deploy with:

```sh
bin/deploy
```

## Migrations

Follow best practices for [zero downtime migrations](https://github.com/ankane/shorts/blob/master/Zero-Downtime-Migrations.md).

Also, turn off prepared statements. Otherwise, you’ll see errors whenever you add a column.

```yml
production:
  prepared_statement: false
```

Don’t worry! Your app will still be fast (and you’ll probably do this anyways at scale since PgBouncer requires it).

## Rollbacks

Create a rollback script in `bin/rollback`.

```sh
#!/usr/bin/env bash

function notify() {
  # add your chat service
  echo $1
}

notify "Rolling back"

heroku rollback

if [ $? -eq 0 ]; then
  notify "Rollback complete"
else
  notify "Rollback failed"
fi
```

Don’t forget to `chmod +x bin/rollback`. Rollback with:

```sh
bin/rollback
```

## Logs

Add [Papertrail](https://papertrailapp.com/) to search your logs.

```sh
heroku addons:create papertrail
```

Set it up to [archive logs to S3](http://help.papertrailapp.com/kb/how-it-works/permanent-log-archives/).

## Performance

Add a performance monitoring service like New Relic.

```sh
heroku addons:create newrelic
```

And follow the [installation instructions](https://devcenter.heroku.com/articles/newrelic#ruby-installation-and-configuration).

To speed up requests, use [heroku_rails_deflate](https://github.com/mattolson/heroku_rails_deflate).

## Productivity

Use [aliases](https://www.digitalocean.com/community/tutorials/an-introduction-to-useful-bash-aliases-and-functions) for less typing.

```sh
alias hc="heroku run console"
```

## Staging

Create a separate app for staging.

```sh
heroku create staging-appname -r staging
```

To push to staging, run:

```sh
git push staging branch:master
```

You may also want to password protect your staging environment with basic authentication.

```ruby
class ApplicationController < ActionController::Base
  http_basic_authenticate_with name: "happy", password: "carrots" if Rails.env.staging?
end
```

Happy coding!
