# Verify Requests from Slack in Rails

Slack [signs its requests](https://api.slack.com/docs/verifying-requests-from-slack) so you can verify they’re authentic.

Here’s a method you can use in your Rails controllers for it.

```ruby
def request_verified?
  timestamp = request.headers["X-Slack-Request-Timestamp"]
  signature = request.headers["X-Slack-Signature"]
  signing_secret = ENV.fetch("SLACK_SIGNING_SECRET")

  if Time.at(timestamp.to_i) < 5.minutes.ago
    return false # expired
  end

  basestring = "v0:#{timestamp}:#{request.body.read}"
  my_signature = "v0=#{OpenSSL::HMAC.hexdigest("SHA256", signing_secret, basestring)}"

  ActiveSupport::SecurityUtils.secure_compare(my_signature, signature)
end
```

:lock:
