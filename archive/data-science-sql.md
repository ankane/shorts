# Data Science SQL

[Root mean squared error](https://www.kaggle.com/wiki/RootMeanSquaredError)

```sql
SELECT SQRT(AVG(POWER(y - y_pred, 2))) AS rmse FROM ...
```

[Mean absolute error](https://www.kaggle.com/wiki/MeanAbsoluteError)

```sql
SELECT AVG(ABS(y - y_pred)) AS mae FROM ...
```

Mean error

```sql
SELECT AVG(y_pred - y) AS me FROM ...
```

Median - [get it here](https://github.com/ankane/median.sql)

```sql
SELECT MEDIAN(y) FROM ...
