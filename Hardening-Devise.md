# Hardening Devise

A few tips to make your [Devise](https://github.com/plataformatec/devise) setup more secure :lock:

Replace the `puts` calls with actual code :)

#### Send notifications for important events

Like a user changing his or her email or password. For email changes, notify both the old and new email addresses to prevent quiet account takeovers (changing the email then resetting password).

```ruby
class User < ActiveRecord::Base
  after_update :notify_email_change, if: -> { email_changed? }
  after_update :notify_password_change, if: -> { encrypted_password_changed? }

  def notify_email_change
    puts "Sending email change to #{email_was}"
    puts "Sending email change to #{email}"
  end

  def notify_password_change
    puts "Sending password change to #{email}"
  end
end
```

#### Rate limit login attempts

Use Devise’s `Lockable` module and a library like [Rack::Attack](https://github.com/kickstarter/rack-attack)

Create `config/initializers/rack_attack.rb` with:

```ruby
Rack::Attack.throttle("logins/ip", limit: 20, period: 1.hour) do |req|
  req.ip if req.post? && req.path == "/users/sign_in"
end

ActiveSupport::Notifications.subscribe("rack.attack") do |name, start, finish, request_id, req|
  puts "Throttled #{req.env["rack.attack.match_discriminator"]}"
end
```

#### Record and monitor login attempts

Create `config/initializers/warden.rb` with:

```ruby
Warden::Manager.after_set_user except: :fetch do |user, auth, opts|
  unless auth.winning_strategy.is_a?(Devise::Strategies::Rememberable)
    req = ActionDispatch::Request.new(auth.env)
    puts "Login success: #{user.email} from #{req.ip}"
  end
end

Warden::Manager.before_failure do |env, opts|
  if opts[:message]
    req = ActionDispatch::Request.new(env)
    puts "Login failure: #{req.params[:user][:email] rescue "unknown"} from #{req.ip} for #{opts[:message]}"
  end
end
```
