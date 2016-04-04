# The Origin of SQL Queries

Do you know what part of your application is generating that time-consuming database query?  There’s a much simpler way than `grep`.

**Add comments to your queries!!**

Turn:

```sql
SELECT * FROM "pandas" WHERE "name" ILIKE 'sad%'
```

into

```sql
SELECT * FROM "pandas" WHERE "name" ILIKE 'happy%'
/*application:Instacart,controller:pandas,action:eating*/
```

Whether you use [PgHero](https://github.com/ankane/pghero), [pg_stat_statements](http://www.postgresql.org/docs/9.4/static/pgstatstatements.html) on its own, or `log_min_duration_statement` to log slow queries, comments can help!

## Ruby on Rails

[Marginalia](https://github.com/basecamp/marginalia) is great.  We prefer to customize slightly in `config/initializers/marginalia.rb`.

```ruby
module Marginalia
  module Comment
    # add namespace to controller
    def self.controller
      if marginalia_controller.respond_to? :controller_path
        marginalia_controller.controller_path
      end
    end
  end
end
```

## Python

```python
coming("soon")
```

## R

With [RPostgreSQL](http://cran.r-project.org/web/packages/RPostgreSQL/index.html), create `queryComments.R` and source appropriately.

```r
dbGetQuery <- function(con, statement)
{
  path <- sub(".*=", "", commandArgs()[4])
  script <- normalizePath(paste0(dirname(path), "/", path))
  statement <- paste0("/*application:Instacart,script:", script, "*/ ", statement)
  RPostgreSQL::dbGetQuery(con, statement)
}
```

We add comments to the beginning since pg_stat_statements truncates long queries before PostgreSQL 9.4.
