# Zero Downtime Migrations with Postgres

- Add and remove indexes concurrently

  ```ruby
  class AddEmailIndexToUsers < ActiveRecord::Migration
    disable_ddl_transaction! # required

    def change
      add_index :users, :email, algorithm: :concurrently
    end
  end
  ```

- Populate columns before adding a default value

  ```ruby
  class AddOrdersCountToUsers < ActiveRecord::Migration
    disable_ddl_transaction! # very, very important!

    def change
      # add column without default
      add_column :users, :orders_count, :integer

      # populate
      User.find_each do |user|
        user.update_column :orders_count, user.orders.count
      end

      # add default
      change_column :users, :orders_count, :integer, default: 0, null: false
    end
  end
  ```

- Do not rename tables or columns that are used
