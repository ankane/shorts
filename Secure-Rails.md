# Secure Rails

Everyone writing code must be responsible for security. :lock:

Start with the [Rails Security Guide](http://guides.rubyonrails.org/security.html) to see how Rails protects you.

## Best Practices

- Keep secret tokens out of your code - `ENV` variables are a good practice

- Even with ActiveRecord, SQL injection is still possible if misused

  ```ruby
  User.group(params[:column])
  ```

  is vulnerable to injection. [Learn about other methods](http://rails-sqli.org)

- Use [SecureHeaders](https://github.com/twitter/secureheaders)

- Protect all data in transit with HTTPS - add the following to `config/environments/production.rb`

  ```ruby
  config.force_ssl = true
  ```

- Protect sensitive data at rest with a library like [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted)

- Set `autocomplete="off"` for sensitive form fields, like credit card number

- Use [Devise](https://github.com/plataformatec/devise) for authentication

- Rate limit login attempts with [Rack Attack](https://github.com/kickstarter/rack-attack)

- Notify users of password changes and attempts to change email addresses

- Rails has a number of gems for [authorization](https://www.ruby-toolbox.com/categories/rails_authorization) - we like [Pundit](https://github.com/elabs/pundit)

## Tools

- [Brakeman](https://github.com/presidentbeef/brakeman) is a great static analysis tool - it scans your code for vulnerabilities
- [CodeClimate](https://codeclimate.com/) provides a similar hosted version
- [HackerOne](https://hackerone.com/) is another good resource

## Additional Reading

- [Rails’ Insecure Defaults](http://blog.codeclimate.com/blog/2013/03/27/rails-insecure-defaults/)
- [The Inadequate Guide to Rails Security](http://blog.honeybadger.io/ruby-security-tutorial-and-rails-security-guide/)
- [The Matasano Crypto Challenges](http://cryptopals.com/)
