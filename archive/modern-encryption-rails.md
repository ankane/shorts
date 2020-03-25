# Modern Encryption for Rails

<p style="text-align: center;"><img src="/images/modern-encryption-rails.png" alt="Lockbox" /></p>

Encrypting sensitive data at the application-level is crucial for data security. Since writing [Securing Sensitive Data in Rails](https://ankane.org/sensitive-data-rails), I haven’t been able to shake the feeling that encryption in Rails could be easier and cleaner.

To address this, I created a library called [Lockbox](https://github.com/ankane/lockbox). Here are some of the principles behind it.

## Easy to Use, Hard to Misuse

Many cryptography mistakes happen during implementation. Lockbox provides good defaults and is designed to be hard to misuse. You don’t need to deal with initialization vectors and it only supports secure algorithms.

## Popular Integrations

Sensitive data can appear in many places, like database fields, file uploads, and strings. You shouldn’t need different libraries for each of these.

Lockbox can encrypt your data in all of these forms. It has built-in integrations with Active Record, Active Storage, and CarrierWave.

## Zero Downtime Migrations

At some point, you may want to encrypt existing data. This should be easy to do, and most importantly, not require any downtime. Lockbox provides a single method you can use for this once your model is configured:

```ruby
Lockbox.migrate(User)
```

No need to write one-off backfill scripts.

## Maximum Compatibility

Encrypting attributes shouldn’t break existing code or libraries. To make this possible, methods like `attribute_changed?` and `attribute_was` should behave similarly regardless of whether or not an attribute is encrypted. Lockbox includes these methods in its test suite for maximum compatibility.

This allows features like Devise’s ability to send email change notifications to work when the email attribute is encrypted, which is an important measure to prevent account hijacking.

```ruby
Devise.setup do |config|
  config.send_email_changed_notification = true
end
```

You can even query encrypted attributes thanks to the [blind_index](https://github.com/ankane/blind_index) gem.

## Modern Algorithms

Lockbox uses AES-GCM for [authenticated encryption](https://tonyarcieri.com/all-the-crypto-code-youve-ever-written-is-probably-broken). It also supports XSalsa20 (thanks to Libsodium), which is recommended by [some cryptographers](https://latacora.micro.blog/2018/04/03/cryptographic-right-answers.html).

## Less Keys To Manage

It’s a good practice to use a different encryption key for each field to make it more difficult for attackers and to reduce the likelihood of a [nonce collision](https://www.cryptologie.net/article/402/is-symmetric-security-solved/). However, this can be burdensome for developers.

Instead, we can use a single master key and derive separate keys for each field from it. This approach is taken from [CipherSweet](https://ciphersweet.paragonie.com/internals/key-hierarchy), an encryption library for PHP and Node.js. Now developers can safely add encrypted fields without having to worry about generating and storing additional secrets.

You can still specify keys for certain fields if you prefer, but it’s no longer required. Lockbox also works with [KMS Encrypted](https://github.com/ankane/kms_encrypted) if you want to use a key management service to manage your keys.

## Built-In Key Rotation

It’s good security hygiene to rotate your encryption keys from time-to-time. Lockbox makes this easy by allowing you to specify previous versions of keys and algorithm:

```ruby
class User < ApplicationRecord
  encrypts :email, previous_versions: [{key: previous_key}]
end
```

New data is encrypted with the new key and algorithm, while older data can still be decrypted.

## Cleaner Schema

[attr_encrypted](https://github.com/attr-encrypted/attr_encrypted), the de facto encryption library for database fields, uses two fields for each encrypted attribute: one for the ciphertext and another for the initialization vector.

```ruby
encrypted_email
encrypted_email_iv
```

However, it’s possible to store both in a single field for a cleaner schema.

```ruby
email_ciphertext
```

## Hybrid Cryptography

Hybrid cryptography allows certain servers to encrypt data without the ability to decrypt it. This can do a better job [protecting data](https://ankane.org/decryption-keys) than symmetric cryptography when you can use it. Lockbox makes it just as easy to use hybrid cryptography.

```ruby
class User < ApplicationRecord
  encrypts :email, algorithm: "hybrid", encryption_key: encryption_key, decryption_key: decryption_key
end
```

## Updates

Since this post was originally published:

- Lockbox also supports [types](https://ankane.org/lockbox-types)
- Here’s how to [encrypt user email addresses](https://ankane.org/securing-user-emails-lockbox)
- Lockbox supports [Mongoid](https://ankane.org/modern-encryption-mongoid)

## Summary

You’ve now seen what Lockbox brings to encryption for Rails. To summarize, it:

- Is hard to misuse
- Works with database fields, files, and strings
- Makes it easy to migrate existing data without downtime
- Maximizes compatibility with existing code and libraries
- Uses modern algorithms
- Requires you to only manage a single encryption key
- Makes key rotation easy
- Stores encrypted data in a single field
- Supports hybrid cryptography

Try out [Lockbox](https://github.com/ankane/lockbox) today.

*Already use a library for encryption? No worries, it’s [easy to migrate](https://github.com/ankane/lockbox#migrating-from-another-library).*
