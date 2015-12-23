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

Use [awesome_print](https://github.com/michaeldv/awesome_print) or [pry-rails](https://github.com/rweng/pry-rails) for a friendlier console.

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
