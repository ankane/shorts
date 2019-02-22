# Active Storage S3 Client-Side Encryption

Use [client-side encryption](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingClientSideEncryption.html) to encrypt your data before sending it to S3.

You can provide an encryption key to use directly or a [KMS](https://aws.amazon.com/kms/) key for envelope encryption. With envelope encryption, a data encryption key is retrieved from KMS and used to encrypt the file. An encrypted version of the key is stored in the object metadata. When downloading the file, the encrypted key is sent to KMS to be decrypted and then used to decrypt the file. The AWS SDK handles all this automatically.

An advantage of this approach is S3 never sees the unencrypted file (unlike with server-side encryption).

First, we’ll create a service that extends the built-in S3 service. Create `lib/active_storage/services/encrypted_s3_service.rb` with:

```ruby
require "active_storage/service/s3_service"

module ActiveStorage
  class Service::EncryptedS3Service < Service::S3Service
    attr_reader :encryption_client

    def initialize(bucket:, upload: {}, **options)
      super_options = options.except(:kms_key_id, :encryption_key)
      super(bucket: bucket, upload: upload, **super_options)
      @encryption_client = Aws::S3::Encryption::Client.new(options)
    end

    def upload(key, io, checksum: nil, **)
      instrument :upload, key: key, checksum: checksum do
        begin
          encryption_client.put_object(
            upload_options.merge(
              body: io,
              content_md5: checksum,
              bucket: bucket.name,
              key: key
            )
          )
        rescue Aws::S3::Errors::BadDigest
          raise ActiveStorage::IntegrityError
        end
      end
    end

    def download(key, &block)
      if block_given?
        raise NotImplementedError, "#get_object with :range not supported yet"
      else
        instrument :download, key: key do
          encryption_client.get_object(
            bucket: bucket.name,
            key: key
          ).body.string.force_encoding(Encoding::BINARY)
        end
      end
    end

    def download_chunk(key, range)
      raise NotImplementedError, "#get_object with :range not supported yet"
    end
  end
end
```

Note that downloading chunks isn’t supported with client-side encryption.

Then update `config/storage.yml` to use it:

```yml
amazon:
  service: EncryptedS3
  bucket: my-bucket
  kms_key_id: alias/my-key
```

Use `encryption_key` instead of `kms_key_id` to manage keys yourself.

And that’s it!

For client-side encryption without Active Storage, check out [this post](https://ankane.org/aws-client-side-encryption).
