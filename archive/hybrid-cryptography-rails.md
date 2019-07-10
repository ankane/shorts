# Hybrid Cryptography on Rails

<p style="text-align: center;"><img src="/images/key-defense.jpg" alt="Keys" /></p>

[Hybrid cryptography](https://en.wikipedia.org/wiki/Hybrid_cryptosystem) allows certain servers to encrypt data without the ability to decrypt it. This can greatly limit damage in the event of a breach.

Suppose we have a service that sends text messages to customers. Customers enter their phone number through the website or mobile app.

With hybrid cryptography, we can set up web servers to only encrypt phone numbers. Text messages can be sent through background jobs which run on a different set of servers - ones that can decrypt and don’t allow inbound traffic. If internal employees need to view phone numbers, they can use a separate set of web servers that are only accessible through the company VPN.

&nbsp; | Encrypt | Decrypt | &nbsp;
--- | --- | --- | ---
Customer web servers | ✓ |
Background workers | ✓ | ✓ | No inbound traffic
Internal web servers | ✓ | ✓ | Requires VPN

## Setup

Install [Libsodium](https://github.com/crypto-rb/rbnacl/wiki/Installing-libsodium) and add [Lockbox](https://github.com/ankane/lockbox) and [RbNaCl](https://github.com/crypto-rb/rbnacl) to your Gemfile:

```ruby
gem 'lockbox'
gem 'rbnacl'
```

Generate keys in the Rails console with:

```ruby
Lockbox.generate_key_pair
```

Store the keys with your other secrets. This is typically Rails credentials or an environment variable ([dotenv](https://github.com/bkeepers/dotenv) is great for this). Be sure to use different keys in development and production.

```sh
PHONE_ENCRYPTION_KEY=...
PHONE_DECRYPTION_KEY=...
```

Only set the decryption key on servers that should be able to decrypt.

## Database Fields

We’ll store phone numbers in an encrypted database field. Add to your Gemfile:

```ruby
gem 'attr_encrypted'
```

Create a migration to add a new column for the encrypted data. We don’t need a separate IV column, as this will be included in the encrypted data.

```ruby
class AddEncryptedPhoneToUsers < ActiveRecord::Migration[5.2]
  def change
    add_column :users, :encrypted_phone, :string
  end
end
```

In the model, use a custom encryptor provided by Lockbox.

```ruby
class User < ApplicationRecord
  attr_encrypted :phone, encryptor: Lockbox::Encryptor, algorithm: "hybrid", encryption_key: ENV["PHONE_ENCRYPTION_KEY"], decryption_key: ENV["PHONE_DECRYPTION_KEY"]

  attr_accessor :encrypted_phone_iv # prevent attr_encrypted error
end
```

Set a user’s phone number to ensure it works.

## Files

Suppose we also need to accept sensitive documents. We can take a similar approach with file uploads.

For CarrierWave, use:

```ruby
class DocumentUploader < CarrierWave::Uploader::Base
  encrypt algorithm: "hybrid", encryption_key: ENV["PHONE_ENCRYPTION_KEY"], decryption_key: ENV["PHONE_DECRYPTION_KEY"]
end
```

For Active Storage, use:

```ruby
class User < ApplicationRecord
  attached_encrypted :document, algorithm: "hybrid", encryption_key: ENV["PHONE_ENCRYPTION_KEY"], decryption_key: ENV["PHONE_DECRYPTION_KEY"]
end
```

You can also encrypt an IO stream directly.

```ruby
box = Lockbox.new(algorithm: "hybrid", encryption_key: ENV["PHONE_ENCRYPTION_KEY"], decryption_key: ENV["PHONE_DECRYPTION_KEY"])
box.encrypt(params[:file])
```

## Conclusion

You’ve now seen an approach for keeping your data safe in the event a server is compromised. For more on data protection, check out [Securing Sensitive Data in Rails](https://ankane.org/sensitive-data-rails).
