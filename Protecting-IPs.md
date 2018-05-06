# Protecting IPs in Ruby

With the [GDPR](https://en.wikipedia.org/wiki/General_Data_Protection_Regulation) just around the corner, here are two useful ways to protect your users’ IP addresses. Both support IPv4 and IPv6.

## Masking

This is the approach [Google Analytics uses for IP anonymization](https://support.google.com/analytics/answer/2763052):

- For IPv4, the last octet is set to 0
- For IPv6, the last 80 bits are set to zeros

```ruby
require "ipaddr"

def mask_ip(ip)
  addr = IPAddr.new(ip)
  if addr.ipv4?
    # set last octet to 0
    addr.mask(24).to_s
  else
    # set last 80 bits to zeros
    addr.mask(48).to_s
  end
end
```

Examples

```ruby
mask_ip("8.8.4.4")
# => "8.8.4.0"

mask_ip("2001:4860:4860:0:0:0:0:8844")
# => "2001:4860:4860::"
```

## Hashing

This transforms IP addresses with a keyed hash function (PBKDF2-HMAC-SHA256). If an unkeyed function is used (like SHA1), it’s trivial to build a rainbow table.

```ruby
require "ipaddr"
require "openssl"

def hash_ip(ip, key:)
  addr = IPAddr.new(ip)
  key_len = addr.ipv4? ? 4 : 16
  family = addr.ipv4? ? Socket::AF_INET : Socket::AF_INET6

  keyed_hash = OpenSSL::PKCS5.pbkdf2_hmac(addr.to_s, key, 20000, key_len, "sha256")
  IPAddr.new(keyed_hash.bytes.inject {|a, b| (a << 8) + b }, family).to_s
end
```

Examples

```ruby
hash_ip("8.8.4.4", key: "secret")
# => "114.124.40.57"

hash_ip("2001:4860:4860:0:0:0:0:8844", key: "secret")
# => "49a2:718:9704:cf11:2068:4c15:587c:1e15"
```
