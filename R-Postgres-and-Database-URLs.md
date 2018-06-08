# R and Database URLs

**Note:** This approach is now built into the [dbx package](https://github.com/ankane/dbx)

---

To use a `DATABASE_URL` with R, do:

## Postgres

```R
library(RPostgreSQL)
library(httr)

establishConnection <- function(url=Sys.getenv("DATABASE_URL"))
{
  cred <- parse_url(url)
  if (!identical(cred$scheme, "postgres")) stop("Invalid database url")
  if (is.null(cred$username)) cred$username <- ""
  if (is.null(cred$password)) cred$password <- ""
  if (is.null(cred$port)) cred$port <- 5432
  dbConnect(PostgreSQL(), host=cred$hostname, port=cred$port,
    user=cred$username, password=cred$password, dbname=cred$path)
}

con <- establishConnection()
dbGetQuery(con, "SELECT true AS success")
```

## MySQL

```R
library(RMySQL)
library(httr)

establishConnection <- function(url=Sys.getenv("DATABASE_URL"))
{
  cred <- parse_url(url)
  if (!identical(cred$scheme, "mysql")) stop("Invalid database url")
  if (is.null(cred$username)) cred$username <- "root"
  if (is.null(cred$password)) cred$password <- ""
  if (is.null(cred$port)) cred$port <- 3306
  dbConnect(MySQL(), host=cred$hostname, port=cred$port,
    user=cred$username, password=cred$password, dbname=cred$path)
}

con <- establishConnection()
dbGetQuery(con, "SELECT true AS success")
```

:cake:
