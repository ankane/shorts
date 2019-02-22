# Backsolving in Ruby

QR decomposition is a [stable way to solve linear regression](https://statsmaths.github.io/stat612/lectures/lec13/lecture13.pdf).

```ruby
require "matrix"

x = Matrix.columns([[1, 1, 1, 1, 1], [1, 2, 3, 4, 5], [4, 2, 5, 6, 1]])
y = Matrix.column_vector([145, 225, 355, 465, 515])
```

You can use the [extendmatrix](https://github.com/clbustos/extendmatrix) gem to do decomposition in pure Ruby. Givens rotations [are faster](https://en.wikipedia.org/wiki/QR_decomposition#Using_Givens_rotations), but the implementation appears to have a bug.

```ruby
require "extendmatrix"

r = x.houseR
q = x.houseQ
```

Next, split R into a p-by-p matrix and Q into a n-by-p matrix (see [page 32](https://statsmaths.github.io/stat612/lectures/lec13/lecture13.pdf) for reasoning).

```ruby
r1 = r.minor(0...x.column_count, 0...x.column_count)
q1 = q.minor(0...x.row_count, 0...x.column_count)
```

Finally, backsolve. Code ported from [Stack Overflow](https://codereview.stackexchange.com/questions/110039/back-substitution-method-for-solving-linear-system).

```ruby
def backsolve(a, b)
  x = Matrix.zero(b.row_count, b.column_count)

  # use an array since matrices
  # are immutable in Ruby
  x = x.to_a

  c = 1.0 / a[-1, -1]
  x[-1] = b.row(-1).map { |v| c * v }

  (b.row_count - 2).downto(0) do |i|
    c = 1.0 / a[i, i]
    s = (a.minor(i..i, (i+1)..-1) * Matrix.rows(x[(i + 1)..-1])).to_a[0]
    x[i] = b.row(i).zip(s).map { |v, w| v - w }.map { |v| c * v }
  end

  Matrix.rows(x)
end
```

Run it to get the coefficients

```ruby
backsolve(r1, q1.transpose * y)
```

Regression solved!
