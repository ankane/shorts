# Client-Side Encryption with AWS and Ruby

AWS makes it easy to enable server-side encryption on many of its services, but it also provides ways to do client-side encryption well. Here are a few ways in Ruby.

## S3

Gem: [aws-sdk-s3](https://github.com/aws/aws-sdk-ruby)

```ruby
client = Aws::S3::Encryption::Client.new(
  kms_key_id: "alias/my-key"
)

client.put_object(
  body: File.read("test.txt"),
  bucket: "my-bucket",
  key: "test.txt"
)

resp = client.get_object(
  bucket: "my-bucket",
  key: "test.txt"
)

puts resp.body.read
```

Use `encryption_key` instead of `kms_key_id` to manage keys yourself.

## S3 + CarrierWave

Gem: [carrierwave-aws](https://github.com/sorentwo/carrierwave-aws)

```ruby
CarrierWave.configure do |config|
  config.storage    = :aws
  config.aws_bucket = "my-bucket"
  config.aws_acl    = "private"

  config.aws_credentials = {
    client: Aws::S3::Encryption::Client.new(kms_key_id: "alias/my-key")
  }
end
```

Use `encryption_key` instead of `kms_key_id` to manage keys yourself.

This can also be set on individual uploaders.

```ruby
class AvatarUploader < CarrierWave::Uploader::Base
  def aws_credentials
    {
      client: Aws::S3::Encryption::Client.new(kms_key_id: "alias/my-key")
    }
  end
end
```

## Active Record Attributes

Gem: [kms_encrypted](https://github.com/ankane/kms_encrypted)

```ruby
class User < ApplicationRecord
  has_kms_key key_id: "alias/my-key"

  attr_encrypted :email, key: :kms_key
end
```

## Active Storage

Check out [this post](https://ankane.org/activestorage-s3-encryption).
