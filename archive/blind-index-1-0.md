# Blind Index 1.0

<p style="text-align: center;"><img src="/images/blind-index-1-0.png" alt="Blind Index 1.0" /></p>

Blind indexing is an approach to [securely search encrypted data](https://paragonie.com/blog/2017/05/building-searchable-encrypted-databases-with-php-and-sql) with minimal information leakage.

I’m happy to announce that [Blind Index 1.0](https://github.com/ankane/blind_index) was just released! Here are the key improvements.

## Stronger Algorithm

This release adds support for Argon2id and makes it the default algorithm.

Argon2 is a memory-hard function. You specify the amount of memory required to compute a hash, and if an attacker tries to compute the hash with less memory, it takes significantly more time to compute. This allows it to better resist attacks on specialized hardware like ASICs.

Argon2 is significantly better than PBKDF2 (the previous default), so we recommend upgrading for better security.

## Less Keys to Manage

It’s a good practice to use a separate key for each blind index. However, generating, storing, and deploying new keys can be burdensome. Thanks to [this key separation method](https://ciphersweet.paragonie.com/internals/key-hierarchy) by CipherSweet, this is no longer needed. Instead, you can use a single master key and the library will derive separate keys for each blind index automatically. You no longer have to worry about managing additional secrets.

## Better Naming

In earlier versions, blind index columns took the format `encrypted_#{name}_bidx`. This was done to match the encrypted columns of the attr_encrypted gem. However, this column is a hash rather than encrypted data, so the `encrypted_` prefix doesn’t really make sense. It was removed in this release.

## Support for Lockbox

This release also adds support for [Lockbox](https://ankane.org/modern-encryption-rails), a modern encryption library for Rails.

## Summary

Blind Index 1.0 brings a number of improvements, and there’s a [smooth path to upgrading](https://github.com/ankane/blind_index#upgrading) with zero downtime.

If you’re not encrypting data today because it makes it impossible to query, check out [Blind Index](https://github.com/ankane/blind_index).
