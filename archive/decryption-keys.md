# Why and How to Keep Your Decryption Keys Off Web Servers

<p style="text-align: center;"><img src="/images/key-defense.jpg" alt="Keys" /></p>

Suppose a worst-case scenario happens: an attacker finds a remote code execution vulnerability and creates a [reverse shell](https://hackernoon.com/reverse-shell-cf154dfee6bd) on one of your web servers. They then find the database credentials, connect to your database, and steal the data.

For unencrypted data and data encrypted at the storage level, it’s game over. The attacker has it all. If data is encrypted at the application level with symmetric encryption but the encryption key is accessible from the server, it’s exactly the same. The attacker has all they need to decrypt the data offline.

This is the case whether you store the encryption key in configuration management, an environment variable, or dynamically load it from an outside source. If your app can access the key, it’s vulnerable to compromise.

The best way to defend against this attack is make sure the compromised server isn’t able to decrypt data. Web servers are typically most exposed to attacks. If your web servers accept sensitive data but don’t need to show it in its entirety back to users, they should be able to encrypt the data and write it to the database, but not decrypt it. The data can be decrypted and processed by background workers that don’t allow inbound traffic.

You likely can’t do this for all of your data, but you should do it for all of the data you can. Sometimes it’s possible to just show partial information back to users. This is universal for saved credit cards.

<p style="text-align: center;"><img src="/images/credit-cards.png" alt="Credit cards" /></p>

In these cases, you can store the partial data in a separate field which web servers can decrypt, while not allowing them to decrypt the full data.

## Practical Example

Suppose we have a service that sends text messages to customers. Customers enter their phone number through the website or mobile app.

We can set up web servers so they can only encrypt phone numbers. Text messages can be sent through background jobs which run on a different set of servers - ones that can decrypt and don’t allow inbound traffic. If internal employees need to view full phone numbers, they can use a separate set of web servers that are only accessible through the company VPN.

&nbsp; | Encrypt | Decrypt | &nbsp;
--- | --- | --- | ---
Customer web servers | ✓ |
Background workers | ✓ | ✓ | No inbound traffic
Internal web servers | ✓ | ✓ | Requires VPN

If customers need to see their saved phone numbers, you can show them the last 4 digits, which are stored in a separate field.

## Approaches

Two approaches you can take to accomplish this are:

1. Hybrid cryptography
2. Cryptography as a service

## Hybrid Cryptography

Public key cryptography, or asymmetric cryptography, uses different keys to perform encryption and decryption. Servers that need to encrypt have the encryption key and servers that need to decrypt  have the decryption key.

However, public key cryptography is much less efficient than symmetric cryptography, so most implementations combine the two. They use public key cryptography to exchange a symmetric key, and symmetric cryptography to encrypt the data. This is called hybrid cryptography, and it’s how TLS and GPG work.

X25519 is a modern key exchange algorithm that’s [widely deployed](https://ianix.com/pub/curve25519-deployment.html) and [currently recommended](https://paragonie.com/blog/2019/03/definitive-2019-guide-cryptographic-key-sizes-and-algorithm-recommendations#after-fold).

[Libsodium](https://libsodium.gitbook.io/doc/), which uses X25519, is a great option for hybrid cryptography in applications. It has [libraries](https://libsodium.gitbook.io/doc/bindings_for_other_languages) for most languages.

## Cryptography as a Service

Another approach is to use a service to perform encryption and decryption. This service can allow some sets of servers to encrypt and others to decrypt. You could write your own (micro)service, but there are a number of existing solutions, often called key management services (KMS).

- [Vault](https://www.vaultproject.io/)
- [AWS KMS](https://aws.amazon.com/kms/)
- [Google Cloud KMS](https://cloud.google.com/kms/)

These services don’t store the encrypted data - they just encrypt and decrypt on-demand. You can either encrypt data directly with the KMS or use envelope encryption.

### Direct Encryption

With direct encryption, you don’t need to set up encryption in your app. Whenever you need to encrypt or decrypt data, simply send the data to the KMS.

However, this has a few downsides. It exposes the unencrypted data to the KMS, which is disastrous if the KMS alone is breached. It’s also less efficient for large files and hosted services have a fairly low limit on the size of data you can encrypt.

### Envelope Encryption

Another approach is envelope encryption, which addresses the issues above but requires encryption in your app.

To encrypt, generate a random encryption key, known as a data encryption key (DEK), and use it to encrypt the data. Then encrypt the DEK with the KMS and store the encrypted version.

To decrypt, decrypt the DEK with the KMS and then use it to decrypt the data. This way, the KMS only ever sees the DEK.

### Auditing

Another benefit of cryptography as a service is auditing. You can see exactly when data or DEKs are decrypted, and there’s no way to get around the auditing without compromising the KMS. This makes it easy to tell which information was accessed during a breach.

## Conclusion

We don’t encrypt data for a sunny day. You’ve now seen two approaches to limit damage in the event of a web server breach.

If you use Ruby on Rails, I’ve written a companion piece on [hybrid cryptography](/hybrid-cryptography-rails) with code for how to do this.
