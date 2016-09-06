# Google OAuth with Devise

Here’s a quick guide to setting up Google OAuth as your app’s exclusive authentication method

Add to your Gemfile

```ruby
gem 'devise'
gem 'omniauth-google-oauth2'
gem 'dotenv-rails', groups: [:development, :test]
```

And run

```sh
rails generate devise:install
```

Create a `User` model

```sh
rails g model User
```

In the migration, add

```ruby
create_table :users do |t|
  t.string :email
  t.string :provider
  t.string :uid
  t.string :remember_token
  t.datetime :remember_created_at
  t.timestamps null: false
end

add_index :users, :email, unique: true
```

In your `User` model, add

```ruby
devise :rememberable, :omniauthable, omniauth_providers: [:google_oauth2]
```

Create a controller

```
rails g controller OmniauthCallbacks
```

with

```ruby
class OmniauthCallbacksController < Devise::OmniauthCallbacksController
  # replace with your authenticate method
  skip_before_action :authenticate_user!

  def google_oauth2
    auth = request.env["omniauth.auth"]
    user = User.where(provider: auth["provider"], uid: auth["uid"])
            .first_or_initialize(email: auth["info"]["email"])
    user.save!

    user.remember_me = true
    sign_in(:user, user)

    redirect_to after_sign_in_path_for(user)
  end
end
```

In your routes, add

```ruby
devise_for :users, controllers: {omniauth_callbacks: "omniauth_callbacks"}
```

In `config/devise.rb`, add

```ruby
config.omniauth :google_oauth2, ENV["GOOGLE_CLIENT_ID"], ENV["GOOGLE_CLIENT_SECRET"], access_type: "online"
```

Follow the [Google API Setup](https://github.com/zquestz/omniauth-google-oauth2#google-api-setup) instructions and add your credentials in `.env`

```
GOOGLE_CLIENT_ID=0000000
GOOGLE_CLIENT_SECRET=0000000
```
