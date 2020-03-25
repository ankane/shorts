# Securing Sensitive Data in Rails

It feels like data breaches are showing up every week in the news. If you haven’t taken a second look at how you’re storing sensitive data, now is probably a good time. Users trust you with the privacy and security of their information.

This guide will walk through what data is sensitive, best practices for storing it, and pitfalls to avoid.

## What’s Sensitive?

The National Institute of Standards and Technology [defines](https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-122.pdf) personally identifiable information (PII) as:

1. Any information that can be used to distinguish or trace an individual’s identity
2. Any other information that is linked or linkable to an individual

And the GDPR [defines](https://gdpr-info.eu/art-4-gdpr/) personal data as “any information relating to an identified or identifiable natural person.”

Here are some examples of data that can identify a person:

- Full Name
- Email address
- Street address
- Phone number
- Credit card number
- Social Security number, passport number, or driver’s license number
- IP address

And here are some examples that aren’t necessarily sensitive on their own, but can be when linked to a person:

- Date of birth
- Location data
- Background check
- Medical record

The line between these isn’t always clear. For instance, location data can be used to identify a person in cases.

This data also has different levels of sensitivity depending on the level of harm it can cause. Harm can be financial (like identity theft), social (like embarrassment), or even physical. You’ll need to make this determination for your situation, but here’s a rough starting point.

- **Low (relative to others):** IP address
- **Medium:** Full name, email address
- **High:** Street address, phone number, date of birth, location data
- **Very High:** Credit card number, Social Security number, passport number, driver’s license number, background check, medical record

If you don’t have strong safeguards for all of this data today, start with the highest sensitivity and work down. You can use [pdscan](https://github.com/ankane/pdscan) to scan your data stores for some of this information.

## Transmitting Data

Users typically submit information through a browser or native app. Make sure data is encrypted in transit. The most popular browser, Google Chrome, now warns users in the address bar when they visit a website that doesn’t use HTTPS.

<p style="text-align: center;"><img src="/images/sensitive-data-insecure.png" alt="Google Chrome Insecure Screenshot" /></p>

You can get free SSL certificates from [Let’s Encrypt](https://letsencrypt.org) and web servers can terminate SSL with little overhead. You can require your app to use it in `config/environments/production.rb` with:

```ruby
config.force_ssl = true
```

This will redirect all non-HTTPS requests to HTTPS.

If all of your subdomains support HTTPS, add your domain to the [HSTS Preload List](https://hstspreload.org/). This list is included in browsers and ensures HTTPS is always used for your domain, preventing an attacker from performing a middleperson attack on the HTTP to HTTPS redirect. You need to set the following options for your domain to be eligible for the list.

```ruby
config.ssl_options = {hsts: {subdomains: true, preload: true, expires: 1.year}}
```

There are [many versions](https://sslversions.com/) of SSL you can support and many ciphers. You can check for insecure versions, weak ciphers, and vulnerabilities with [testssl.sh](https://github.com/drwetter/testssl.sh).

```sh
./testssl.sh example.org
```

[Cipherli.st](https://cipherli.st/) provides configuration examples for strong versions and ciphers for popular web servers, and infrastructure providers like AWS allow you to specify a [security policy](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html#describe-ssl-policies) for this.

Also, when collecting data through forms, disable autocomplete for fields with highly sensitive data.

```html
<input autocomplete="off">
```

This will prevent the browser from storing it.

## Storing Data

Once data is submitted, we need a place to store it. Three common places are:

2. 3rd Parties
3. Databases
4. Files

We’ll look at best practices for each of these.

### 3rd Parties

There are services that specialize in storing sensitive data. Use one of these services if you can. A few categories are:

- Payments - [Stripe](https://stripe.com/), [Braintree](https://www.braintreepayments.com/)
- Documents - [HelloSign](https://www.hellosign.com/), [DocuSign](https://www.docusign.com/)
- Background Checks - [Checkr](https://checkr.com/)

In many cases, users can submit data to these services without it passing through your servers.

Limit who can log in and access the information, set up 2-factor authentication, and don’t use shared credentials. Also, be sure to protect any API keys that can retrieve sensitive data.

### Databases

For data you need to store yourself, the database can be a good place. It’s important for the data to be encrypted at rest. This applies to relational databases as well as document databases like Elasticsearch. There are multiple ways to accomplish this with different security implications.

#### Storage Level

The simplest is full-disk encryption. This encrypts all of the data at rest. The database doesn’t even need to be aware that the data is encrypted. Hosted providers like Heroku, Amazon RDS, and Google Cloud SQL all make this easy.

- Heroku: All production plans have encryption at rest.
- Amazon RDS: Enable encryption on creation. For existing databases, create an encrypted read replica and promote it. For PostgreSQL, the only way is to create an encrypted snapshot and restore from it, which can incur significant downtime.
- Google Cloud SQL: All instances have encryption at rest.

This approach protects data from physical theft. However, it doesn’t protect against an attacker who successfully connects to the database.

#### Application Level

For this, you need to encrypt data before it’s sent to the database. This is known as application-level encryption. Use storage-level encryption for all data and application-level encryption for sensitive data.

*2019 Update:* Check out [Lockbox](https://ankane.org/modern-encryption-rails) for application-level encryption.

There’s a popular gem called [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted) that integrates nicely with Active Record.

```ruby
class User
  attr_encrypted :phone, key: key
end
```

To use it, you add two columns to your model - one to store the encrypted data and another to store the initialization vector (IV). The IV is a random - but not secret - value that’s passed to the encryption algorithm to make the encrypted data different in cases where rows have the same value. This prevents someone from gaining information about the data.

```ruby
encrypt("secret", iv: "abc") != encrypt("secret", iv: "123")
```

The attr_encrypted gem uses symmetric encryption, meaning the same key is used to encrypt and decrypt (also referred to as secret key encryption).

#### Encrypt vs Decrypt

Limiting decryption access is essential to good security. Parts of your system should be able to encrypt without being able to decrypt. Web servers are most exposed to attacks. They should be able to encrypt data they collect and write it to the database, but not decrypt data from the database unless it needs to be shown back to the user. The data can be decrypted and processed by background workers that don’t allow inbound traffic if needed.

One way to address this to use asymmetric encryption (also referred to as public key encryption). This uses a public key to encrypt the data and a private key to decrypt it. The [strongbox](https://github.com/spikex/strongbox) gem is one option for this.

```ruby
class User < ApplicationRecord
  encrypt_with_public_key :email, key_pair: ...
end
```

It may not sound intuitive, but there are also ways to accomplish this with symmetric encryption, which we’ll see shortly.

#### Audit Trail

A big drawback to both the approaches we’ve discussed so there’s no reliable audit trail, which is critical for security. You need to reliably know what data was accessed in the event of a breach. If an attacker gains access to the key, they can decrypt all the data offline without a trace. You could add an audit trail in Ruby, but this is trivially bypassed as an attacker can just download the database, steal the key, and decrypt offline. We need something else.

#### Cryptography as a Service

One way to solve this is to use a service to encrypt and decrypt for us. This service can manage the key, audits, and permissions. These are known as key management services (KMS). A few options are [Vault](https://www.vaultproject.io/), [AWS KMS](https://aws.amazon.com/kms/), and [Google Cloud KMS](https://cloud.google.com/kms/). With Amazon, keys are stored in a tamper-resistant [hardware security module](https://en.wikipedia.org/wiki/Hardware_security_module) for added security. These services do not store the encrypted data - they just encrypt and decrypt on-demand.

You can either:

1. Send data directly to the KMS
2. Use envelope encryption

For the first approach, the KMS sees the unencrypted data. This is the approach used by [Vault Rails](https://github.com/hashicorp/vault-rails).

```ruby
class User < ApplicationRecord
  include Vault::EncryptedModel
  vault_attribute :email
end
```

Another approach is envelope encryption, which prevents the KMS from seeing the unencrypted data. This provides extra protection in the event the KMS is compromised. To encrypt, a random encryption key - known as data encryption key - is generated and used to encrypt the data. The data key is then passed to the KMS to be encrypted and the encryption version is stored in the database. A different data key is generated for each record. To decrypt, the data key is first decrypted with the KMS. Then it’s used to decrypt the data. This is a pretty standard pattern, and Google has a good list of [best practices](https://cloud.google.com/kms/docs/envelope-encryption) for it.

[KMS Encrypted](https://github.com/ankane/kms_encrypted) is great for this.

```ruby
class User < ApplicationRecord
  has_kms_key
  attr_encrypted :email, key: :kms_key
end
```

Use a separate master key for each table and each highly sensitive field.

#### Context

Whether sending the data directly or using envelope encryption, it’s important to log what data was accessed. This is where encryption context comes in. You can pass context along with the data when encrypting. You must pass this same context when decrypting, so make sure it’s a value that doesn’t change (it’s actually used as part of the encryption/decryption process to make sure it’s not tampered with). The context shows up in auditing so you can understand the data being encrypted and decrypted.

Recent versions of KMS Encrypted handle this automatically. Google Cloud KMS doesn’t log context, so it’s not recommended right now.

#### Algorithms

Use an encryption algorithm that provides [authenticated encryption](https://tonyarcieri.com/all-the-crypto-code-youve-ever-written-is-probably-broken). Authentication ensures that the encrypted data wasn’t tampered with before decryption. AES-GCM is a good choice for symmetric encryption. It’s natively supported in Ruby and the default for `ActiveSupport::MessageEncryptor` and attr_encrypted. There are also newer algorithms like XChaCha20-Poly1305 in [RbNaCl](https://github.com/crypto-rb/rbnacl). AES-CBC does not provide authentication on its own.

#### Key Rotation

You should design your system so it’s easy to rotate keys. Key rotation is important for when:

1. A weakness is discovered in the encryption scheme (in which case you should also rotate the encryption scheme)
2. Someone with access to keys leaves the company

With key management services, you can typically make an API call to rotate the master key (AWS doesn’t offer this, so you need to create a new key, but has the option for yearly rotation). If you use envelope encryption, you should also rotate the data encryption keys. With KMS Encrypted, you can do:

```ruby
User.find_each do |user|
  user.rotate_kms_key!
end
```

#### Searching Data

When data is encrypted, it’s much more difficult to search. There are a few techniques you can use for exact equality.

One approach is called [blind indexing](https://www.sitepoint.com/how-to-search-on-securely-encrypted-database-fields/), which uses a keyed hash function to compute a hash of the data. The same values will have the same hash. The hash functions have cost parameters you can tune to make them take more resources, which will slow down an attacker. You can use the [blind_index](https://github.com/ankane/blind_index) gem for this.

```ruby
class User < ApplicationRecord
  blind_index :email
end
```

With blind indexing, all the rows need to be hashed with the same key. Ideally, there would be a service for this as well to not expose the key to the application, but as of now, neither Vault, AWS KMS, nor Google Cloud KMS provide this.

Since hashing isn’t encryption, it’s not directly reversible if the key is compromised. However, an attacker can build a [rainbow table](https://en.wikipedia.org/wiki/Rainbow_table) based on the key. This is trivial for data with a small number of possible values, like phone numbers, so protect the key as if it were an encryption key.

Another approach is to use a deterministic encryption scheme, like AES-SIV. In this approach, the encrypted data will be the same for matches. The [miscreant](https://github.com/miscreant/miscreant) gem supports this. Do not use a static IV with algorithms like AES-GCM, as this will [compromise](https://csrc.nist.gov/csrc/media/projects/block-cipher-techniques/documents/bcm/joux_comments.pdf) the encryption key.

For range queries, you can use order-preserving encryption or an order-preserving hash function. However, it can [leak a significant amount of information](http://cryptowiki.net/index.php?title=Order-preserving_encryption#Security) if an attacker is able to encrypt data, so I don’t recommend it.

#### Passwords

Don’t use encryption for user passwords, as passwords should not be reversible. Use [Devise](https://github.com/plataformatec/devise) or Rails’ built-in `has_secure_password` instead. Both use bcrypt to hash the password.

#### Anonymization

One way to protect data without encryption is anonymization. For IP addresses, you can use [IP Anonymizer](https://github.com/ankane/ip_anonymizer). Two methods are supported: masking and hashing.

Masking is the approach Google Analytics uses for [IP anonymization](https://support.google.com/analytics/answer/2763052). For IPv4, set the last octet to 0, and for IPv6, set the last 80 bits to zeros.

```ruby
IpAnonymizer.mask_ip("8.8.4.4")
# => "8.8.4.0"
```

An advantage of this approach is geocoding will still work, only with slightly less accuracy. A potential disadvantage is different IPs will have the same mask (8.8.4.4 and 8.8.4.5 both become 8.8.4.0).

The other approach is hashing, where you use a keyed hash function and secret to obfuscate the IP.

```ruby
IpAnonymizer.hash_ip("8.8.4.4", key: key)
# => "6.128.151.207"
```

An advantage of this approach is different IPs will have different hashes (with the exception of collisions).

#### Database Functions

One option we haven’t discussed yet is using built-in database functions for encryption, like the ones in [Postgres](https://www.postgresql.org/docs/current/pgcrypto.html) and [MySQL](https://dev.mysql.com/doc/refman/8.0/en/encryption-functions.html). However, for a brief period, the key and unencrypted data are both present on the database server where someone with complete access can intercept them. Use application-level encryption instead.

#### Backups

Be sure to encrypt backups as well. If you have an encrypted database with a hosted provider, this is typically handled automatically. Also, limit who has access to backups.

### Files

Files are another place where sensitive data can be stored. Like with the database, use storage-level encryption for all data and application-level encryption for sensitive data.

#### Storage Level

Storage services refer to this as server-side encryption. Again, this helps in the event of physical theft. Also, it’s a nice option for direct uploads.

Both Amazon S3 and Google Cloud Storage support this. You can use their default keys, your own keys, or tie into their KMS. With AWS, you can use [s3tk](https://github.com/ankane/s3tk) check the status of your buckets, set up encryption, and encrypt existing files if needed.

Turn on logging to see when files are accessed.

#### Application Level

Application-level encryption means the storage service never sees the unencrypted file. This can protect against misconfigurations where data is exposed. If someone gains access to a bucket, they won’t be able to access the data without a key. Storage services refer to this as client-side encryption.

You can use [Lockbox](https://github.com/ankane/lockbox) to encrypt files.

```ruby
box = Lockbox.new(key: key)
box.encrypt(File.binread("license.jpg"))
```

It also supports Active Storage:

```ruby
class User < ApplicationRecord
  has_one_attached :license
  attached_encrypted :license, key: key
end
```

And CarrierWave:

```ruby
class LicenseUploader < CarrierWave::Uploader::Base
  encrypt key: key
end
```

You can use the [kms_encrypted](https://github.com/ankane/kms_encrypted) gem with it as well.

The AWS SDK also has built-in support for encryption and can be used with [Active Storage](https://ankane.org/activestorage-s3-encryption) and [CarrierWave](https://ankane.org/aws-client-side-encryption).

To serve files that are encrypted, create a controller action that requires authentication, logs access, and performs the decryption.

```ruby
def license
  send_data @user.license.download, type: @user.license.content_type
end
```

## Not Leaking Data

One of the major concerns with sensitive data is leakage. A few common places data gets leaked are:

1. Logs
2. Audits
3. 3rd Parties
4. Cache Stores
5. Emails

### Logs

Logs see a lot of information. Logs are often stored for a significant period to time, giving them a chance to be compromised.

Rails logs each request along with its parameters. Add sensitive fields to `config/initializers/filter_parameter_logging.rb`. This will replace the sensitive data with `[FILTERED]` in your logs.

```ruby
Rails.application.config.filter_parameters += [:email, :phone]
```

Add another layer of defense with [Logstop](https://github.com/ankane/logstop), which filters based on patterns.

```ruby
Logstop.guard(Rails.logger)
```

To scrub existing logs, check out [scrubadub](https://github.com/datascopeanalytics/scrubadub). When looking for leaks, be sure to check load balancer logs, application logs, and database logs.

### Audits

If you use an auditing library like [Audited](https://github.com/collectiveidea/audited), make sure it doesn’t capture sensitive data. You can exclude certain fields from being audited with:

```ruby
class User < ApplicationRecord
  audited except: [:phone]
end
```

### 3rd Parties

Analytics, instrumentation, and error reporting services are all places data can leak. Thanks to the [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation), many of these services now have options to prevent this.

Use ids instead of email addresses or names to identify users.

```js
// bad!!!
mixpanel.identify('hi@example.org');

// better
mixpanel.identify('123');
```

Also, anonymize IP addresses. With Google Analytics, use:

```js
ga('set', 'anonymizeIp', true);
```

For backend services, scrub sensitive parameters in addition to IP addresses. Here’s how to do it with [Rollbar](https://docs.rollbar.com/docs/ruby#section-managing-sensitive-data):

```ruby
Rollbar.configure do |config|
  config.anonymize_user_ip = true
  config.scrub_fields |= [:email, :phone]
end
```

### Cache Stores

Make sure you don’t accidentally cache sensitive data.

### Emails

Don’t include sensitive data in emails. You don’t want it sitting in inboxes. Instead, include a link to your product. This is true for emails to admins as well.

## Managing Access to Data

### Admin Access

Use different roles to limit access to people inside the company. You can use a gem like [Rolify](https://github.com/RolifyCommunity/rolify) to manage roles and [Pundit](https://github.com/varvet/pundit) to limit access.

Log access to information to an immutable data store. This can be a table in your database with update and delete privileges revoked to start. Log the time, admin, IP address, fields accessed, and the user whose data it was.

Rate limit access as well. After too many unique views within a time period, lock access and send an alert. This helps limit damage if an admin account is stolen or an admin goes rogue.

#### BI Tools

Business intelligence tools like [Blazer](https://github.com/ankane/blazer) may also have access to sensitive data. Democratizing data is extremely powerful, but does not need to come at the expense of user privacy. Limit table and column access with database privileges. You can use [Hypershield](https://github.com/ankane/hypershield) to create shielded views, which hide sensitive columns but still allow for `SELECT *` queries.

#### Downloads

If you absolutely need to make sensitive data downloadable for admins, make sure it’s encrypted. The most user-friendly way to do this is to use a format that supports password protection. Use a team password manager like [1Password](https://1password.com/) to share the password with the people who need it.

For spreadsheets, the [Office Open XML format](https://en.wikipedia.org/wiki/Office_Open_XML) provides a standard for encryption and password protection. Excel, Numbers, and LibreOffice Calc all support it. You can use [Secure Spreadsheet](https://github.com/ankane/secure-spreadsheet) to convert a CSV into a password-protected, AES-256 encrypted XLSX.

For other types of files, you can create a password-protected ZIP file. ZipCrypto is the standard ZIP encryption format, but it’s [very weak](https://pdfs.semanticscholar.org/1e2f/7685e8505d82dc38f8e713cdfd9e20656b42.pdf), so don’t use it. AES-256 is much better, but can’t be opened without additional software.

To prevent user confusion, we’ll use the [7z format](https://en.wikipedia.org/wiki/7z) instead. You can use [Seven Zip Ruby](https://github.com/masamitsu-murase/seven_zip_ruby) to do:

```ruby
File.open("archive.7z", "wb") do |file|
  SevenZipRuby::Writer.open(file, password: "secret") do |szw|
    szw.header_encryption = true
    szw.add_file("file1.txt")
    szw.add_file("file2.txt")
  end
end
```

Users can use [7-Zip](https://www.7-zip.org/) or [The Unarchiver](https://theunarchiver.com/) to open them.

### Developer Access

Be extremely selective of which systems and people have access to the decryption keys. Only make decryption keys accessible to as few servers as possible, ideally workers that don’t allow inbound traffic. With AWS, use [machine credentials](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html) to grant access to KMS and prevent developers from accessing those machines.

On Heroku, this is difficult without a [Shield](https://www.heroku.com/shield) plan. Any developer can access config or spin up a one-off process to perform decryption. With KMS, you’ll have an audit trail, but this doesn’t prevent the data from being compromised. One option is to create another app with the same code, just extra config, that a limited number of people can access.

### 3rd Party Access

If you share data with 3rd parties, it’s still your responsibility to make sure the data is secure. For each 3rd party, vet their security practices by having them fill out a vendor security questionnaire. You can find examples online, like [this one](https://www.vendorsecurityalliance.org/questionnaire2018.html) by the Vendor Security Alliance. If you adhere to the GDPR, the company must also be added as a data processor.

When sharing data, use envelope encryption and encrypt the data key with asymmetric encryption so the vendor can decrypt it with their private key. The OpenPGP standard is great for this. It’s often used for email but is designed for any files. You can use [GPG](https://gnupg.org/) and [IOStreams](https://github.com/rocketjob/iostreams) for this:

```ruby
IOStreams::Pgp::Writer.open("data.txt.gpg", recipient: "hi@example.org") do |output|
  output << "data"
end
```

Alternatively, you can use public key encryption from [RbNaCl](https://github.com/crypto-rb/rbnacl).

```ruby
box = RbNaCl::SimpleBox.from_keypair(public_key, private_key)
box.encrypt("data")
```

## Monitoring Data

### Alerts

Set up alerts for anomalous activity. This includes admin data access and application decryptions.

Use a service like [PhishLabs](https://www.phishlabs.com/dark-web-monitoring/) to monitor the dark web for stolen data or exploits against your infrastructure.

### Intrusion Detection

There are a number of tools to detect intrusions. They fall into two main categories: host-based and network-based.

Host-based intrusion detection systems monitor logs and file integrity and check for rootkits. [OSSEC](https://www.ossec.net/) is a popular one. It can be run in local mode for an individual host or an agent/server mode for multiple hosts. Others include:

- [Wuzah](https://wazuh.com/)
- [Tripwire](https://github.com/Tripwire/tripwire-open-source)
- [rkhunter](https://help.ubuntu.com/community/RKhunter)

Network-based intrusion detection systems monitor network traffic for suspicious activity. There are open source ones like:

- [Suricata](https://suricata-ids.org/)
- [Snort](https://www.snort.org/)
- [Bro](https://www.bro.org/)

Here’s a [nice comparison](https://www.alienvault.com/blogs/security-essentials/open-source-intrusion-detection-tools-a-quick-overview) of the open source options.

There are also services from infrastructure providers. [Amazon GuardDuty](https://aws.amazon.com/guardduty/) looks at CloudTrail events, VPC flow logs, and DNS logs to detect threats. [Amazon Macie](https://aws.amazon.com/macie/) is designed to give you visibility into data access and movement. Google currently has [Cloud Security Command Center](https://cloud.google.com/security-command-center/) in alpha.

## Hardening Systems

### Code

Subscribe to [ruby-security-ann](https://groups.google.com/forum/#!forum/ruby-security-ann) to get security announcements for Ruby, Rails, Rubygems, Bundler, and other Ruby ecosystem projects.

Use [bundler-audit](https://github.com/rubysec/bundler-audit) to check for vulnerable versions of gems. Make this part of your build process.

```sh
bundle audit check --update
```

Use [Brakeman](https://github.com/presidentbeef/brakeman) to scan your code for vulnerabilities. There are hosted services like [Hakiri](https://hakiri.io/) and [CodeClimate](https://codeclimate.com/) for this as well.

Use a service like [HackerOne](https://hackerone.com/) or [Bugcrowd](https://www.bugcrowd.com/) to create a bug bounty program and enlist hackers to surface vulnerabilities.

### Internal Traffic

Consider encrypting all traffic on your internal network. This protects against sniffing in case of a breach. As discussed earlier, sensitive data should be encrypted by the app before passing it to a database or job queue, but this is a nice way to protect all data. Many hosted service providers have easy ways to set this up.

PostgreSQL has an `sslmode` option to control connection security. Use `verify-full` with a root certificate, as [no other options](https://ankane.org/postgres-sslmode-explained) protect against both eavesdropping and middleperson attacks.

```yml
production:
  url: <%= ENV["DATABASE_URL"] %>
  sslmode: verify-full
  sslrootcert: root.crt
```

If you use PgBouncer, [set it up](https://ankane.org/securing-pgbouncer-amazon-rds) there as well.

MySQL has [similar options](https://dev.mysql.com/doc/refman/8.0/en/encrypted-connection-options.html#option_general_ssl-mode).

```yml
production:
  url: <%= ENV["DATABASE_URL"] %>
  sslmode: VERIFY_IDENTITY
  sslca: root.crt
```

Redis doesn’t natively support SSL and recommends an SSH tunnel like [spiped](https://github.com/Tarsnap/spiped) or [stunnel](https://www.stunnel.org/index.html). Luckily, the Ruby Redis library has support for this functionality so you don’t need to run anything special on your application servers. [Redis Labs](https://docs.redislabs.com/latest/rc/securing-redis-cloud-connections/), [AWS ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/in-transit-encryption.html) and [Heroku Redis](https://devcenter.heroku.com/articles/securing-heroku-redis) all support this.

```ruby
Redis.new(ssl: true)
```

You can use [httplog](https://github.com/trusche/httplog) to see if your app is making any non-HTTPS calls.

```ruby
HttpLog.configure do |config|
  config.url_whitelist_pattern = /\Ahttp\:/
end
```

With AWS and Google Cloud, you can use VPC flow logs to check for unencrypted traffic.

## Conclusion

With <strike>great power</strike> **user data** comes great responsibility. Encrypt sensitive data at the application level and the rest of your data at the storage level. Use the cryptography as a service model to manage keys, audits, and permissions. Take care not to leak information to logs, audits, 3rd parties, cache stores, or emails. Limit who has access to sensitive data and set up proper monitoring to detect issues.

You’ve now seen some practical approaches for securing sensitive data. We can do better as an industry. Be part of that change.
