# Zero Downtime Migrations with Postgres

- Do not drop or rename tables or columns that are used

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

    def up
      # add column without default
      add_column :users, :orders_count, :integer

      # populate
      User.find_each do |user|
        user.update_column :orders_count, user.orders.count
      end

      # add default
      change_column :users, :orders_count, :integer, default: 0, null: false
    end

    def down
      remove_column :users, :orders_count
    end
  end
  ```
