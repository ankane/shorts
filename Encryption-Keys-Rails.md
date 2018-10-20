# Strong Encryption Keys for Rails

Encryption is a common way to protect sensitive data.

[attr_encrypted](https://github.com/attr-encrypted/attr_encrypted) is a popular encryption library for Rails. The default algorithm is AES-256-GCM, which takes a 256-bit (or 32-byte) key.

So how can you generate a secure one? Your first thought may be something like this:

```ruby
SecureRandom.base64(32).first(32)
```

This generates a 32 character string that looks secure. Each character has 64 possible values (letters, numbers, / and +). However, a single byte can hold 256 possible values. This is only 25% of possible values per byte. This compounds across all 32 bytes. A random 32-byte key has 256<sup>32</sup> (or 2<sup>256</sup>) possible values. However, our key has only 64<sup>32</sup> (or 2<sup>192</sup>), which is equivalent to a 192-bit key. This reduces the number of possible keys by 99.999999999999999994%, which sounds pretty scary! Luckily, computers have not (yet) been able to brute force 128-bit keys.

So why do we use 256-bit keys to begin with? [Polynomial](https://security.stackexchange.com/questions/14068/why-most-people-use-256-bit-encryption-instead-of-128-bit) puts it well:

“Essentially it’s about security margin. The longer the key, the higher the effective security. If there is ever a break in AES that reduces the effective number of operations required to crack it, a bigger key gives you a better chance of staying secure.”

Also, quantum computers are expected to brute force in [square root time](https://blog.agilebits.com/2013/03/09/guess-why-were-moving-to-256-bit-aes-keys/). This means a 256-bit key could be brute forced in the same time as traditional computers can brute force a 128-bit key.

## A Better Way

A better way to generate a random 32-byte key is:

```ruby
SecureRandom.random_bytes(32)
```

However, we can’t store this directly in Rails credentials or as an environment variable. We need to encode it first. Hex is a popular encoding. Rails uses this for its master key in Rails 5.2.

```ruby
SecureRandom.random_bytes(32).unpack("H*").first
```

Ruby provides a helper to do this:

```ruby
SecureRandom.hex(32)
```

Then in your app, unpack it.

```ruby
[hex_key].pack("H*")
```

You now have a much stronger key. If you store the key as an environment variable, your model should look something like:

```ruby
class User < ApplicationRecord
  attr_encrypted :email, key: [ENV["EMAIL_ENCRYPTION_KEY"]].pack("H*")
end
```

Libraries should educate users on how to generate sufficiently random keys. [Libsodium](https://download.libsodium.org/doc/) prides itself on being encryption that’s hard to mess up. It’s Ruby binding - the [rbnacl](https://github.com/crypto-rb/rbnacl) gem - requires keys to be binary. This means passing a normal string won’t work. The check is as simple as:

```ruby
if key.encoding != Encoding::BINARY
  raise ArgumentError, "Insecure key - key must use binary encoding"
end
```

I’ve incorporated this approach into the [blind_index](https://github.com/ankane/blind_index) gem and [opened an issue](https://github.com/attr-encrypted/attr_encrypted/issues/311) with attr_encrypted to get the author’s thoughts.

While secure key generation provides better protection against brute force attacks, it won’t help at all if your key is compromised. Limit who has access to your encryption keys as well.

Happy encrypting!
