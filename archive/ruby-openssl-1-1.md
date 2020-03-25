# Ruby with OpenSSL 1.1

Some Ruby features like `scrypt` and `hkdf` require OpenSSL 1.1. Here’s how to make it work on Mac:

Install rbenv and OpenSSL 1.1

```sh
brew install rbenv ruby-build openssl@1.1
```

Install Ruby

```sh
RUBY_CONFIGURE_OPTS="--with-openssl-dir=/usr/local/opt/openssl@1.1" \
rbenv install 2.6.3
```

Open an interactive shell to confirm it worked

```sh
rbenv shell 2.6.3
irb
```

And run

```sh
require "openssl"
OpenSSL::OPENSSL_VERSION
OpenSSL::KDF.methods - Object.methods
```
