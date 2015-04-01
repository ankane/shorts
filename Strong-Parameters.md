# attr_accessible to Strong Parameters

Running Rails 4 with `attr_accessible`? Upgrade in three **safe and easy** steps

## 1

First, log all instances of forbidden attributes. Add to `config/application.rb`:

```ruby
config.action_controller.permit_all_parameters = false
```

And create an initializer `config/initializers/forbidden_attributes.rb` with:

```ruby
class ActiveRecord::Base
  protected
  def sanitize_for_mass_assignment_with_forbidden_attributes(*args)
    attributes = args[0]
    if attributes.respond_to?(:permitted?) && !attributes.permitted?
      if Rails.env.development? || Rails.env.test? || ENV["RAISE_FORBIDDEN_ATTRIBUTES"]
        raise ActiveModel::ForbiddenAttributesError
      end
      Rails.logger.warn "Forbidden attributes: #{self.class.name}"
    end
    sanitize_for_mass_assignment_without_forbidden_attributes(*args)
  end
  alias_method_chain :sanitize_for_mass_assignment, :forbidden_attributes
end
```

## 2

Fix all instances.

```ruby
User.create(params[:user])
```

to

```ruby
User.create(params.require(:user).permit(:name))
```

## 3

Remove:

- all instances of `attr_accessible`
- `config/initializers/forbidden_attributes.rb`
- `protected_attributes` from your Gemfile
