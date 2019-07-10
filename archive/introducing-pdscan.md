# Introducing pdscan: Scan Your Data Stores for Unencrypted Personal Data

It's important to understand where personal data is stored in your applications. Personal data that’s not encrypted at the application level is especially vulnerable in the event of a breach.

[pdscan](https://github.com/ankane/pdscan) is a command line tool to help you identify this data.

<p style="text-align: center;"><img src="/images/pdscan.gif" alt="pdscan" /></p>

It uses data sampling and column naming to find data and produces minimal database load. It scans for:

- Last names
- Email addresses
- IP addresses
- Street addresses (US)
- Phone numbers (US)
- Credit card numbers
- Social security numbers
- Dates of birth
- Location data

It also scans for other unencrypted sensitive data, like OAuth tokens, which could be used to access personal data. It currently supports Postgres, MySQL, MariaDB, and SQLite, but it shouldn’t be too difficult to add other data stores like MongoDB and Elasticsearch. It’s written in Go, so it’s fast and has no runtime dependencies.

Give [pdscan](https://github.com/ankane/pdscan) a try today.
