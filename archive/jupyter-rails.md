# Jupyter + Rails

Jupyter notebooks are a great alternative to the Rails console for building predictive models. Here’s how to get setup:

First, install [Jupyter Notebook](https://jupyter.org) and [ZeroMQ](http://zeromq.org). With Homebrew, use:

```sh
brew install jupyter zeromq
```

Add to your Gemfile

```ruby
group :development do
  gem 'iruby'
  gem 'ffi-rzmq'
end
```

Run

```sh
iruby register --force
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
