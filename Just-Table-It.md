# Just Table It

When it comes to data, you can mistakenly optimize by trying to choose the “right” technology for the job. Often, the best choice is right in front of you: your database. Relational database scale pretty well, despite what you’ve been told in recent years. Don’t introduce a new data store into your stack if you don’t need to, and don’t store interesting data in logs unless you can easily query them. Generally:

**If you need to query data, throw it in a table.**

At [Instacart](https://www.instacart.com), we’ve stored:

- customer analytics, like visits and page views (yes, customer analytics!!)
- emails sent
- errors our customers see
- slow requests
- location updates from shoppers
- audits for models

Even today, we store most of these in PostgreSQL. Your time is better spent adding value to customers than trying to anticipate how to handle this data at scale. You can figure it out when you get there.

:mount_fuji:

---

We’ve open sourced much of the technology we use to do the above.

- [Ahoy](https://github.com/ankane/ahoy) for analytics
- [Ahoy Email](https://github.com/ankane/ahoy_email) for emails
- [Notable](https://github.com/ankane/notable) for errors and slow requests

And

- [Blazer](https://github.com/ankane/blazer) to analyze the data
