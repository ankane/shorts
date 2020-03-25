# Jupyter + Rails

Jupyter notebooks are a great alternative to the Rails console for doing exploratory data analysis and building predictive models. Here’s how to get setup:

First, install [Jupyter](https://jupyter.org). With Homebrew, use:

```sh
brew install jupyterlab
```

Add to your Gemfile

```ruby
gem 'iruby', group: :development
```

Run

```sh
bundle install
bundle exec iruby register --force
```

Start Jupyter

```sh
jupyter notebook
```

Create a notebook and add to the top

```ruby
require "./config/environment"
```

> If not at Rails root, use `Dir.chdir("path/to/root") { require "./config/environment" }`

And science away

```ruby
User.last
```

If you use Git, add to `.gitignore`

```txt
.ipynb_checkpoints
```

If you use Nyaplot, use the `master` branch to fix [an issue](https://github.com/domitry/nyaplot/issues/52) with empty charts.

```ruby
gem 'nyaplot', github: 'domitry/nyaplot'
```
