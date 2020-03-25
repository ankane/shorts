# Securing User Emails in Rails with Lockbox

<p style="text-align: center;"><img src="/images/securing-user-emails-lockbox.png" alt="Model Code" /></p>

---

*This is an update to [Securing User Emails in Rails](https://ankane.org/securing-user-emails-in-rails) with a number of improvements:*

- *Works with Devise’s email changed notifications*
- *Works with Devise’s reconfirmable option*
- *Stores encrypted data in a single field*
- *You only need to manage a single key*

---

Email addresses are a common form of personal data, and they’re often stored unencrypted. If an attacker gains access to the database or backups, emails will be compromised.

This post will walk you through a practical approach to protecting emails. It works with [Devise](https://github.com/plataformatec/devise), the most popular authentication framework for Rails, and is general enough to work with others.

## Strategy

We’ll use two concepts to make this happen: encryption and blind indexing. Encryption gives us a way to securely store the data, and blind indexing provides a way to look it up.

Blind indexing works by computing a hash of the data. You’re probably familiar with hash functions like MD5 and SHA1. Rather than one of these, we use a hash function that takes a secret key and uses [key stretching](https://en.wikipedia.org/wiki/Key_stretching) to slow down brute force attempts. You can read more about [blind indexing here](https://www.sitepoint.com/how-to-search-on-securely-encrypted-database-fields/).

We’ll use the [Lockbox](https://github.com/ankane/lockbox) gem for encryption and the [Blind Index](https://github.com/ankane/blind_index) gem for blind indexing.

## Instructions

Let’s assume you have a `User` model with an email field.

Add to your Gemfile:

```ruby
gem 'lockbox'
gem 'blind_index'
```

And run:

```sh
bundle install
```

Generate a key

```ruby
Lockbox.generate_key
```

Store the key with your other secrets. This is typically Rails credentials or an environment variable ([dotenv](https://github.com/bkeepers/dotenv) is great for this). Be sure to use different keys in development and production.

Set the following environment variables with your key (you can use this one in development)

```sh
LOCKBOX_MASTER_KEY=0000000000000000000000000000000000000000000000000000000000000000
```

or create `config/initializers/lockbox.rb` with something like

```ruby
Lockbox.master_key = Rails.application.credentials.lockbox_master_key
```

Next, let’s replace the email field with an encrypted version. Create a migration:

```sh
rails generate migration add_email_ciphertext_to_users
```

And add:

```ruby
class AddEmailCiphertextToUsers < ActiveRecord::Migration[5.2]
  def change
    # encrypted data
    add_column :users, :email_ciphertext, :string

    # blind index
    add_column :users, :email_bidx, :string
    add_index :users, :email_bidx, unique: true

    # drop original here unless we have existing users
    remove_column :users, :email
  end
end
```

Then migrate:

```sh
rails db:migrate
```

Add to your user model:

```ruby
class User < ApplicationRecord
  encrypts :email
  blind_index :email
end
```

Create a new user and confirm it works.

## Existing Users

If you have existing users, we need to backfill the data before dropping the email column.

```ruby
class User < ApplicationRecord
  encrypts :email, migrating: true
  blind_index :email, migrating: true
end
```

Backfill the data in the Rails console:

```ruby
Lockbox.migrate(User)
```

Then update the model to the desired state:

```ruby
class User < ApplicationRecord
  encrypts :email
  blind_index :email

  # remove this line after dropping email column
  self.ignored_columns = ["email"]
end
```

Finally, drop the email column.

## Reconfirmable

If you use the confirmable module with `reconfirmable`, you should also encrypt the `unconfirmed_email` field.

```ruby
class AddUnconfirmedEmailToUsers < ActiveRecord::Migration[5.2]
  def change
    add_column :users, :unconfirmed_email_ciphertext, :text
  end
end
```

And add `unconfirmed_email` to the list of encrypted fields and a new method:

```ruby
class User < ApplicationRecord
  encrypts :email, :unconfirmed_email
end
```

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
Logstop.guard(Rails.logger)
```

## Summary

We now have a way to encrypt emails and query for exact matches. You can apply this same approach to other fields as well. For more security, consider a [key management service](https://github.com/ankane/kms_encrypted) to manage your keys.
