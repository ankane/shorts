# Rails, Meet Data Science

<p style="text-align: center;"><img src="/images/rails-meet-ds.png" alt="Rails, Meet Data Science" /></p>

Organizations today have more data than ever. Predictive modeling is a powerful way to use this data to solve problems and create better experiences for customers. For instance, do a better job keeping items in stock by predicting demand or lower costs by predicting fraud. If you use Ruby on Rails, it can be tough to know how to incorporate this into your app.

We’ll go over four patterns you can use for prediction with Rails. We used all four successfully during my time at [Instacart](https://www.instacart.com). They can work when you have no data scientists (when I started) as well as when you have a strong data science team.

## Patterns

With predictive modeling, you first train a model and then use it to predict. The patterns can be grouped by the language used for each task:

Pattern | Train | Predict
--- | --- | ---
1 | 3rd Party | 3rd Party
2 | Ruby | Ruby
3 | Another Language | Ruby
4 | Another Language | Another Language

Two popular languages for data science are Python and R.

You can decide which pattern to use for each model you build. We’ll walk through the approaches and discuss the pros and cons of each.

## Pattern 1: Use a 3rd Party

Before building a model in-house, it’s good to see what already exists. There are a number of external services you can use for specific problems. Here are a few:

- Fraud - [Sift Science](https://siftscience.com/)
- Recommendations - [Tamber](https://tamber.com/)
- Anomaly Detection & Forecasting - [Trend](https://trendapi.org/)
- NLP - [Amazon Comprehend](https://aws.amazon.com/comprehend/) and [Google Cloud Natural Language](https://cloud.google.com/natural-language/)
- Vision - [AWS Rekognition](https://aws.amazon.com/rekognition/) and [Google Cloud Vision](https://cloud.google.com/vision/)

Pros

- Get domain knowledge from the company
- Fast to implement and easy to maintain

Cons

- Not easy to iterate if it doesn’t fit your needs
- Vendor lock-in

## Pattern 2: Train and Predict in Ruby

Ruby has a number of libraries for building simple models. Simple models can perform very well since a large part of model building is [feature engineering](https://en.wikipedia.org/wiki/Feature_engineering). This is a great option if there are no data scientists in your company or on your team. A developer can own the model end-to-end, which is great for speed and iteration.

Here are a few libraries for building models in Ruby:

- [Eps](https://github.com/ankane/eps) - Linear Regression
- [Classifier Reborn](https://github.com/jekyll/classifier-reborn) - Bayesian Classification and LSI
- [rb-libsvm](https://github.com/febeling/rb-libsvm) - Support Vector Machines
- And [many more](https://github.com/arbox/machine-learning-with-ruby)

Once a model is trained, you’ll need to store it. You can use methods provided by the library, or marshal if none exist. You can store the models as files or in the database.

Be sure to commit the code used to train models so you can update them with newer data in the future. The Rails console is a decent place to create them, or use a [Jupyter notebook](https://jupyter.org/) running [IRuby](https://github.com/SciRuby/iruby) for better visualizations (see [setup instructions for Rails](https://ankane.org/jupyter-rails)).

Pros

- Simple models can perform well
- No need to introduce a new language

Cons

- Limited tools for building models
- Limited model selection
- Many people who have experience building models don’t know Ruby

## Pattern 3: Train in Another Language, Predict in Ruby

Ruby is getting better for data science thanks to [SciRuby](https://github.com/SciRuby/sciruby). However, languages like R and Python currently have much better tools. Also, many people who have experience building models don’t know Ruby.

Luckily, you can build models in another language and predict in Ruby. This way, you can use more advanced tools for visualization, validation, and tuning without adding complexity to your production stack. If you don’t have data scientists, you can use this pattern to contract with one.

Here are models that can currently predict in Ruby:

- [Eps](https://github.com/ankane/eps) - Linear Regression
- [Scoruby](https://github.com/asafschers/scoruby) - Random Forest, GBM, Decision Tree, Naive Bayes
- [XGBoost](https://github.com/PairOnAir/xgboost-ruby) - XGBoost

For this to work, models need to be stored in a shared format that both languages understand. PMML and PFA are two interchange formats. PFA is newer but has less adoption than PMML. Andrey Melentyev has a [great post](https://www.andrey-melentyev.com/model-interoperability.html) on the topic.

Once again, it’s important that models are reproducible. This allows you to update them with newer data in the future. Be sure to follow software engineering best practices like:

- Use source control (create a new repo or add to your existing repo)
- Use a package manager for a reproducible environment
- Keep credentials out of source control (use `.env` or `.Renviron`)

Here are some tools you can use:

Function | Python | R
--- | --- | --- | ---
Package management | [Pipenv](https://pipenv.readthedocs.io/en/latest/) | [Jetpack](https://github.com/ankane/jetpack)
Database access | [SQLAlchemy](https://www.sqlalchemy.org/) | [dbx](https://github.com/ankane/dbx)
PMML export | [scikit2pmml](https://github.com/vaclavcadek/scikit2pmml) | [pmml](https://cran.r-project.org/package=pmml)

One place to be careful is implementing the features in Ruby. It must be consistent with how they were implemented in training. To ensure this is correct, verify it programmatically. Create a CSV file with ids and predictions from the original model and confirm the Ruby predictions match. Here’s some [example code](https://github.com/ankane/eps#verifying).

Pros

- Better tools for model building
- No need to operate a new language in production

Cons

- Need to introduce a new language in development
- Limited model selection
- Need to create features in two languages

## Pattern 4: Train and Predict in Another Language

The last option we’ll cover is doing both training and prediction outside Ruby. This is great if you have a team of data scientists who specialize in another language. This pattern allows data scientists to own models end-to-end.

It also gives you access to models that are not available in Ruby. For instance, there are forecasting libraries like [Prophet](https://facebook.github.io/prophet/) and deep learning libraries like [TensorFlow](https://www.tensorflow.org/).

The implementation depends on how predictions are generated. Two common ways are batch and real-time.

---

### Batch Predictions

Batch predictions are generated asynchronously and are typically run on a regular interval. This can be every minute or once a week. An example is a daily job that updates demand forecasts for the following weeks. Predictions can be stored and later used by the Rails app as needed.

Don’t be afraid to read and write directly to the database. While microservice design patterns caution against using the database as an API, we didn’t have much issue with it. When updating records, it’s also a good idea to write audits to see how predictions change over time.

Jobs can be scheduled with cron, or ideally a distributed scheduler like [Mani](https://github.com/sherinkurian/mani) for high availability. If you need to let the Rails app know a job has completed, you can do this through your messaging system. HTTP works great if you don’t have one.

---

### Real-Time Predictions

Real-time predictions are generated synchronously and are triggered by calls from the Rails app. An example is recommending items to a user at checkout based off what’s in their cart.

HTTP is a common choice for retrieving predictions, but you can use a messaging system or even pipes. Great tools for HTTP are [Django](https://www.djangoproject.com/) and [Flask](http://flask.pocoo.org/) for Python and [Plumber](https://www.rplumber.io/) for R.

---

As with the other patterns, follow best engineering practices. In addition to ones previously mentioned:

- Use a framework, or at the very least a consistent project structure
- Keep code [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself)

Don’t be afraid to use Rails to manage the database schema. It’s easy enough for data scientists to learn to create and run migrations. Otherwise, you need to support another system for schema changes.

To store models, you most likely won’t use an interchange format, since libraries can’t load them. Instead, use serialization specific to the language, like pickle in Python and serialize in R.

If deciding between Python and R, Python has more general purpose libraries, so it’s easier to run in production.

Pros

- Larger selection of models available
- Data scientists can own models end-to-end

Cons

- Need to run multiple languages in production

## Conclusion

You’ve now seen four great patterns for bringing predictive models to Rails. Each has different trade-offs, so we recommend taking the simplest approach that works for you. No matter which you choose, make sure your models are reproducible.

Happy modeling!
