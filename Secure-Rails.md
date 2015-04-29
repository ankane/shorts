# Secure Rails

The [Rails Security Guide](http://guides.rubyonrails.org/security.html) is a great starting point.

## Secrets

Keep secret tokens out of your code. `ENV` variables are a good practice.

## SQL Injection

Even with ActiveRecord, SQL injection is still possible if misused.

```ruby
User.group(params[:column])
```

is vulnerable to injection. [Learn about other methods](http://rails-sqli.org).

## Headers

Use [SecureHeaders](https://github.com/twitter/secureheaders).

## Data Transport

Use SSL for everything. Add the following to `config/environments/production.rb`.

```ruby
config.force_ssl = true
```

## Data At Rest

Use a library like [attr_encrypted](https://github.com/attr-encrypted/attr_encrypted) for sensitive information.

## Authentication and Authorization

What’s the difference? [Stack Overflow](http://stackoverflow.com/questions/6556522/authentication-versus-authorization) says it best:

- Authentication = login + password (who you are)
- Authorization = permissions (what you are allowed to do)

### Authentication

Use [Devise](https://github.com/plataformatec/devise). Rate limit login attempts with [Rack Attack](https://github.com/kickstarter/rack-attack).

### Authorization

Rails has a number of gems for [authorization](https://www.ruby-toolbox.com/categories/rails_authorization). We like [Pundit](https://github.com/elabs/pundit).

## Tools

- [Brakeman](https://github.com/presidentbeef/brakeman) is a great static analysis tool - it scans your code for vulnerabilities
- [CodeClimate](https://codeclimate.com/) provides a similar hosted version
- [HackerOne](https://hackerone.com/) is another good resource

## Additional Reading

- [Rails’ Insecure Defaults](http://blog.codeclimate.com/blog/2013/03/27/rails-insecure-defaults/)
- [The Inadequate Guide to Rails Security](http://blog.honeybadger.io/ruby-security-tutorial-and-rails-security-guide/)
- [The Matasano Crypto Challenges](http://cryptopals.com/)
