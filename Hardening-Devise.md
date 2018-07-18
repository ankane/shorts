# Hardening Devise

A few basic steps to make your [Devise](https://github.com/plataformatec/devise) setup more secure :lock:

Replace the `puts` calls with actual code :)

#### Send notifications for important events

Like a user changing his or her email or password. For email changes, notify the old email addresses to prevent quiet account takeovers (changing the email then resetting the password).

In `config/initializers/devise.rb`, add:

```ruby
config.send_email_changed_notification = true
config.send_password_change_notification = true
```

#### Rate limit login attempts

Use Devise’s `Lockable` module and a library like [Rack::Attack](https://github.com/kickstarter/rack-attack)

Create `config/initializers/rack_attack.rb` with:

```ruby
Rack::Attack.throttle("logins/ip", limit: 20, period: 1.hour) do |req|
  req.ip if req.post? && req.path.start_with?("/users/sign_in")
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
    puts "Login success: #{user.email} from #{req.remote_ip}"
  end
end

Warden::Manager.before_failure do |env, opts|
  if opts[:message]
    req = ActionDispatch::Request.new(env)
    puts "Login failure: #{req.params[:user][:email]} from #{req.remote_ip} for #{opts[:message]}"
  end
end
```

Remember, [defense in depth](https://en.wikipedia.org/wiki/Defense_in_depth_%28computing%29)!

For more, check out [Secure Rails](https://github.com/ankane/secure_rails).
