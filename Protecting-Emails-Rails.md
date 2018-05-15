# Protecting Email Addresses in Rails

The GDPR goes into effect next Friday. Whether or not you serve European residents, it’s a great reminder that we have the responsibility to build systems in a way that protects user privacy.

Email addresses are a common form of personal data, and they’re often stored unencrypted. If an attacker gains access to the database or backups, emails will be compromised.

This post will walk you through a practical approach to protecting emails. It works with [Devise](https://github.com/plataformatec/devise), the most popular authentication framework for Rails, and is general enough to work with others.

## Strategy

We’ll use two concepts to make this happen: encryption and blind indexing. Encryption gives us a way to securely store the data, and blind indexing provides a way to look it up.

Blind indexing works by computing a hash of the data. You’re probably familiar with hash functions like MD5 and SHA1. Rather than one of these, we use a hash function that takes a secret key and uses [key stretching](https://en.wikipedia.org/wiki/Key_stretching) to slow down brute force attempts. You can read more about [blind indexing here](https://www.sitepoint.com/how-to-search-on-securely-encrypted-database-fields/).

We’ll use the [attr_encrypted gem](https://github.com/attr-encrypted/attr_encrypted) for encryption and the [blind_index gem](https://github.com/ankane/blind_index) for blind indexing.

## Instructions

Let’s assume you have a `User` model with an email field.

Add to your Gemfile:

```ruby
gem 'attr_encrypted'
gem 'blind_index'
```

And run:

```sh
bundle install
```

Next, let’s replace the email field with an encrypted version. Create a migration:

```sh
rails g migration add_encrypted_email_to_users
```

And add:

```ruby
class AddEncryptedEmailToUsers < ActiveRecord::Migration[5.2]
  def change
    # encrypted data
    add_column :users, :encrypted_email, :string
    add_column :users, :encrypted_email_iv, :string

    # blind index
    add_column :users, :encrypted_email_bidx, :string
    add_index :users, :encrypted_email_bidx, unique: true

    # drop original here unless we have existing users
    remove_column :users, :email
  end
end
```

We use one column to store the encrypted data, one to store [the IV](http://www.cryptofails.com/post/70059609995/crypto-noobs-1-initialization-vectors), and another to store the blind index.

Then migrate:

```sh
rails db:migrate
```

Next, generate keys. *Do not commit them to source control.* We like to use environment variables to store them ([dotenv](https://github.com/bkeepers/dotenv) is great for this). Generate one key for encryption and one key for hashing. For development, you can use these:

```
EMAIL_ENCRYPTION_KEY=00000000000000000000000000000000
EMAIL_BLIND_INDEX_KEY=99999999999999999999999999999999
```

Add to your user model:

```ruby
class User < ApplicationRecord
  attr_encrypted :email, key: ENV["EMAIL_ENCRYPTION_KEY"]
  blind_index :email, key: ENV["EMAIL_BLIND_INDEX_KEY"]
end
```

Create a new user and confirm it works.

## Existing Users

If you have existing users, we need to backfill the data before dropping the email column. We temporarily use a virtual attribute - `protected_email` - so we can backfill without downtime.

```ruby
class User < ApplicationRecord
  attr_encrypted :protected_email, key: ENV["EMAIL_ENCRYPTION_KEY"], attribute: "encrypted_email"
  blind_index :protected_email, key: ENV["EMAIL_BLIND_INDEX_KEY"], attribute: "email", bidx_attribute: "encrypted_email_bidx"

  before_validation :protect_email, if: ->(r) { r.email_changed? }

  def protect_email
    self.protected_email = email
  end
end
```

Backfill the data in the Rails console:

```ruby
User.where(encrypted_email: nil).find_each do |user|
  user.protect_email
  user.save!
end
```

Then update the model to the desired state:

```ruby
class User < ApplicationRecord
  attr_encrypted :email, key: ENV["EMAIL_ENCRYPTION_KEY"]
  blind_index :email, key: ENV["EMAIL_BLIND_INDEX_KEY"]
end
```

Finally, drop the email column.

## Logging

We also need to make sure email addresses aren’t logged. Add to `config/initializers/filter_parameter_logging.rb`:

```ruby
Rails.application.config.filter_parameters += [:email]
```

Use [Logstop](https://github.com/ankane/logstop) to filter anything that looks like an email address as an extra line of defense. Add to your Gemfile:

```ruby
gem 'logstop'
```

And create `config/initializers/logstop.rb` with:

```ruby
Rails.logger.formatter = Logstop::Formatter.new(Rails.logger.formatter)
```

## Summary

We now have a way to encrypt data and query for exact matches. You can apply this same approach to other fields as well. For more security, consider a [key management service](https://github.com/ankane/kms_encrypted) to manage your keys.
