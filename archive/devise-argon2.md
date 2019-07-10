# Argon2 with Devise

bcrypt has been a great choice for safely storing passwords. However, as time has passed, a better alternative has emerged: [Argon2](https://password-hashing.net/). OWASP [now recommends](https://github.com/OWASP/CheatSheetSeries/blob/master/cheatsheets/Password_Storage_Cheat_Sheet.md) Argon2 for new applications. With a little bit of code, you can use Argon2 with Devise.

Devise supports [custom encryptors](https://github.com/plataformatec/devise-encryptable). However, it requires a separate column to store a salt, which isn’t needed as Argon2 stores the salt in the password hash (like bcrypt).

Instead, add [argon2](https://github.com/technion/ruby-argon2) to your Gemfile:

```ruby
gem 'argon2', '>= 2'
```

And create `config/initializers/devise_argon2.rb` with:

```ruby
module Argon2Encryptor
  def digest(klass, password)
    if klass.pepper.present?
      password = "#{password}#{klass.pepper}"
    end
    ::Argon2::Password.create(password)
  end

  def compare(klass, hashed_password, password)
    return false if hashed_password.blank?

    if hashed_password.start_with?("$argon2")
      if klass.pepper.present?
        password = "#{password}#{klass.pepper}"
      end
      ::Argon2::Password.verify_password(password, hashed_password)
    else
      super
    end
  end
end

Devise::Encryptor.singleton_class.prepend(Argon2Encryptor)
```

All new passwords will be hashed with Argon2. For existing passwords, rotate to Argon2 when a user signs in. Add to your model:

```ruby
class User < ApplicationRecord
  def valid_password?(password)
    valid = super
    if valid && !encrypted_password.start_with?("$argon2")
      self.password = password
      save(validate: false)
    end
    valid
  end
end
```

You can also [rehash all passwords at once](https://www.michalspacek.com/upgrading-existing-password-hashes), but it’s a bit more complicated.

Congrats, your password storage is even stronger!
