# XGBoost and LightGBM Come to Ruby

<p style="text-align: center; margin-bottom: 0;">
  <img src="/images/ruby-xgboost.png" alt="Ruby and XGBoost" />
</p>

I’m happy to announce that XGBoost - and it’s cousin LightGBM from Microsoft - are now available for Ruby!

XGBoost and LightGBM are powerful machine learning libraries that use a technique called gradient boosting. Gradient boosting performs well on a [large range of datasets](https://machinelearningmastery.com/start-with-gradient-boosting/) and is common among winning solutions in ML competitions.

XGBoost and LightGBM are already available for popular ML languages like Python and R. The Ruby gems follow similar interfaces and use the same C APIs under the hood.

Make predictions with as little code as:

```ruby
model = Xgb::Regressor.new
model.fit(x_train, y_train)
model.predict(x_test)
```

While Ruby still lags behind other languages for machine learning, the ecosystem is getting better. [Rumale](https://github.com/yoshoku/rumale) is under active development and supports a large number of algorithms with an interface similar to Scikit-Learn. There’s also [Daru](https://github.com/SciRuby/daru) which is similar to Pandas. The addition of gradient boosting covers another key category.

Check out the [Xgb](https://github.com/ankane/xgb) and [LightGBM](https://github.com/ankane/lightgbm) gems today!
