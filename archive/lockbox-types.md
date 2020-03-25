# Lockbox: Now with Types

A new version of [Lockbox](https://ankane.org/modern-encryption-rails) was just released with support for types, making it easier to encrypt non-string fields.

```ruby
class User < ApplicationRecord
  encrypts :born_on, type: :date
  encrypts :salary, type: :integer
end
```

Previously, you’d need to perform typecasting yourself, making it harder to work with encrypted fields. All of these types are supported:

- date
- datetime
- boolean
- integer
- float
- binary
- json
- hash

Types are automatically detected for serialized fields for maximum compatibility with existing code and libraries.

```ruby
class User < ApplicationRecord
  serialize :properties, JSON
  encrypts :properties # detects JSON type
end
```

It even works with custom serializers.

## Padding

This release also adds support for padding. Padding can help conceal the exact length of messages. As the [Libsodium docs](https://libsodium.gitbook.io/doc/padding) explain:

> Most modern cryptographic constructions disclose message lengths. The ciphertext for a given message will always have the same length, or add a constant number of bytes to it. For most applications, this is not an issue. But in some specific situations, [...] hiding the length may be desirable.

Suppose a person’s health is categorized as either:

- excellent
- average
- poor

Even if this value is encrypted, it’s easy to know the status of a person since each category has a different length, which carries over to the ciphertext. Padding addresses this by adding data to the end of each message before encryption. You can enable padding for a field with:

```ruby
class Person < ApplicationRecord
  encrypts :health_status, padding: true
end
```

This expands all messages to a multiple of 16 bytes. You can configure the multiple as needed based on your data.

Get started with types and padding by grabbing the [latest version](https://github.com/ankane/lockbox) of Lockbox today!
