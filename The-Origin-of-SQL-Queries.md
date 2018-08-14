# The Origin of SQL Queries

Do you know what part of your application is generating that time-consuming database query? There’s a much simpler way than `grep`.

**Add comments to your queries!!**

Turn:

```sql
SELECT * FROM pandas WHERE mood = 'sad'
```

into

```sql
SELECT * FROM pandas WHERE mood = 'happy'
/*application:Nature,job:EatBambooJob*/
```

Whether you use [PgHero](https://github.com/ankane/pghero), [pg_stat_statements](https://www.postgresql.org/docs/current/static/pgstatstatements.html) on its own, or `log_min_duration_statement` to log slow queries, comments can help!

## Ruby on Rails

[Marginalia](https://github.com/basecamp/marginalia) is great. We prefer to customize slightly in `config/initializers/marginalia.rb`.

```ruby
module Marginalia
  module Comment
    # add namespace to controller
    def self.controller
      if marginalia_controller.respond_to?(:controller_path)
        marginalia_controller.controller_path
      end
    end
  end
end

# add job
Marginalia::Comment.components << :job
```

## Python

With SQLAlchemy and Flask:

```python
from flask import current_app, request
from sqlalchemy.engine import Engine
from sqlalchemy import event

@event.listens_for(Engine, "before_cursor_execute", retval=True)
def annotate_queries(conn, cursor, statement, parameters, context, executemany):
    comment = ""
    try:
        comment = " /*application:{},endpoint:{}*/".format(current_app.name,
                                                             request.endpoint)
    except RuntimeError:  # running in the CLI
        try:
            comment = " /*application:{}*/".format(current_app.name)
        except RuntimeError:  # running in a REPL
            pass
    return statement + comment, parameters
```

## R

With [dbx](https://github.com/ankane/dbx), use:

```r
options(dbx_comment=TRUE)
```

## Other Languages and Frameworks

Please [submit a PR](https://github.com/ankane/shorts/pulls)!
