# Jupyter + Rails

Install [Jupyter Notebook](https://jupyter.org)

```sh
pip install jupyter
```

Install ZeroMQ

```sh
brew install zeromq
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

And science away

```ruby
User.last
```
