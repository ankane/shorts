# Bulk Upsert in Ruby/Rails

The [upsert](https://github.com/seamusabshere/upsert) gem is great for individual upserts, but for performant bulk upserts, use the [activerecord-import](https://github.com/zdennis/activerecord-import) gem.

Add a unique index on the columns to upsert on (if it’s not your primary key)

```rb
class AddUpsertIndexOnForecasts < ActiveRecord::Migration[5.2]
  def change
    add_index :forecasts, [:date], unique: true
  end
end
```

Prep your records

```ruby
records = [
  {date: "2018-01-01", value: 10},
  {date: "2018-02-01", value: 15},
  {date: "2018-03-01", value: 23}
]
```

For PostgreSQL 9.5+ and SQLite 3.24+, do:

```ruby
Forecast.import(records,
  validate: false,
  on_duplicate_key_update: {
    conflict_target: [:date],
    columns: [:value]
  }
)
```

For MySQL, do:

```ruby
Forecast.import(records,
  validate: false,
  on_duplicate_key_update: [:value]
)
```

[Official docs](https://github.com/zdennis/activerecord-import/wiki/On-Duplicate-Key-Update)
