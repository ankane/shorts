# R, Postgres, and Database URLs

To use a `DATABASE_URL` with R, do:

```R
require(RPostgreSQL, quiet=TRUE)
require(httr, quiet=TRUE)

establishConnection <- function(url=Sys.getenv("DATABASE_URL"))
{
  cred <- parse_url(url)
  user <- toString(cred$username)
  password <- toString(cred$password)
  return(dbConnect(PostgreSQL(), host=cred$hostname, port=cred$port, user=user, password=password, dbname=cred$path))
}

con <- establishConnection()
dbGetQuery(con, "SELECT true AS success")
```

:cake:
