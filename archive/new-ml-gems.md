# 16 New ML Gems for Ruby

<p style="text-align: center; margin-bottom: 0;">
  <img src="/images/ml-gems-2.png" alt="New ML Gems" style="max-height: 300px;" />
</p>

In August, I set out to improve the machine learning ecosystem for Ruby. I wasn’t sure where it would go. Over the next 5 months, I ended up releasing 16 libraries and learned a lot along the way. I wanted to share some of that knowledge and introduce some of the libraries you can now use in Ruby.

## The Theme

There are many great machine libraries for Python, so a natural place to start was to see what it’d take to bring them to Ruby. It turned out to be a lot less work than expected based on a common theme.

ML libraries want to be fast. This means less time waiting and more time iterating. However, interpreted languages like Python and Ruby aren’t relatively fast. How do libraries overcome this?

The key is they do most of the work in a compiled language - typically C++ - and have wrappers for other languages like Python.

This was really great news. The same approach and code could be used for Ruby.

## The Patterns

Ruby has a number of ways to call C and C++ code.

Native extensions are one method. They’re written in C or C++ and use [Ruby’s C API](https://silverhammermba.github.io/emberb/c/). You may have noticed gems with native extensions taking longer to install, as they need to compile.

```c
void Init_stats()
{
    VALUE mStats = rb_define_module("Stats");
    rb_define_module_function(mStats, "mean", mean, 2);
}
```

A more general way for one language to call another is a foreign function interface, or FFI. It requires a C API (due to C++ name mangling), which many machine learning libraries had. An advantage of FFI is you can define the interface in the host language - in our case, Ruby.

Ruby supports FFI with Fiddle. It was added in Ruby 1.9, but appears to be [“the Ruby standard library’s best kept secret.”](https://www.honeybadger.io/blog/use-any-c-library-from-ruby-via-fiddle-the-ruby-standard-librarys-best-kept-secret/)

```ruby
module Stats
  extend Fiddle::Importer
  dlload "libstats.so"
  extern "double mean(int a, int b)"
end
```

There’s also the [FFI](https://github.com/ffi/ffi) gem, which provides higher-level functionality and overcomes some limitations of Fiddle (like the ability to pass structs by value).

```ruby
module Stats
  extend FFI::Library
  ffi_lib "stats"
  attach_function :mean, [:int, :int], :double
end
```

For libraries without a C API, [Rice](https://github.com/jasonroelofs/rice) provides a really nice way to bind C++ code (similar to Python’s pybind11).

```cpp
void Init_stats()
{
    Module mStats = define_module("Stats");
    mStats.define_singleton_method("mean", &mean);
}
```

Another approach is SWIG (Simplified Wrapper and Interface Generator). You create an interface file and then run SWIG to generate the bindings. Gusto has a [good tutorial](https://engineering.gusto.com/simple-ruby-c-extensions-with-swig/) on this.

```swig
%module stats

double mean(int, int);
```

There’s also [Rubex](https://github.com/SciRuby/rubex), which lets you write Ruby-like code that compiles to C (similar to Python’s Cython). It also provides the ability to interface with C libraries.

```ruby
lib "<stats.h>"
  double mean(int, int)
end
```

None of the approaches above are specific to machine learning, so you can use them with any C or C++ library.

## The Libraries

Libraries were chosen based on popularity and performance. Many have a similar interface to their Python counterpart to make it easy to follow existing tutorials. Libraries are broken down into categories below with brief descriptions.

### Gradient Boosting

[XGBoost](https://github.com/ankane/xgb) and [LightGBM](https://github.com/ankane/lightgbm) are gradient boosting libraries. Gradient boosting is a powerful technique for building predictive models that fits many small decision trees that together make robust predictions, even with outliers and missing values. Gradient boosting performs well on tabular data.

### Deep Learning

[Torch-rb](https://github.com/ankane/torch-rb) and [TensorFlow](https://github.com/ankane/tensorflow) are deep learning libraries. Torch-rb is built on LibTorch, the library that powers PyTorch. Deep learning has been very successful in areas like image recognition and natural language processing.

### Recommendations

[Disco](https://github.com/ankane/disco) is a recommendation library. It looks at ratings or actions from users to predict other items they might like, known as collaborative filtering. Matrix factorization is a common way to accomplish this.

[LIBMF](https://github.com/ankane/libmf) is a high-performance matrix factorization library.

Collaborative filtering can also find similar users and items. If you have a large number of users or items, an approximate nearest neighbor algorithm can speed up the search. Spotify [does this](https://github.com/spotify/annoy#background) for music recommendations.

[NGT](https://github.com/ankane/ngt) is an approximate nearest neighbor library that performs extremely well on benchmarks (in Python/C++).

<p style="text-align: center; margin-bottom: 0;">
  <img src="/images/ann-benchmarks.png" alt="ANN Benchmarks" />
</p>

<p class="image-description">
  Image from <a href="https://github.com/erikbern/ann-benchmarks" target="_blank">ANN Benchmarks</a>, MIT license</a>
</p>

Another promising technique for recommendations is factorization machines. The traditional approach to collaborative filtering builds a model exclusively from past ratings or actions. However, you may have additional *side information* about users or items. Factorization machines can incorporate this data. They can also perform classification and regression.

[xLearn](https://github.com/ankane/xlearn) is a high-performance library for factorization machines.

### Optimization

Optimization finds the best solution to a problem out of many possible solutions. Scheduling and vehicle routing are two common tasks. Optimization problems have an objective function to minimize (or maximize) and a set of constraints.

Linear programming is an approach you can use when the objective function and constraints are linear. Here’s a really good [introductory series](https://www.youtube.com/watch?v=0TD9EQcheZM) if you want to learn more.

[SCS](https://github.com/ankane/scs) is a library that can solve [many types](https://www.cvxpy.org/tutorial/advanced/index.html#choosing-a-solver) of optimization problems.

[OSQP](https://github.com/ankane/osqp) is another that’s specifically designed for quadratic problems.

### Text Classification

[fastText](https://github.com/ankane/fasttext) is a text classification and word representation library. It can label documents with one or more categories, which is useful for content tagging, spam filtering, and language detection. It can also compute word vectors, which can be compared to find similar words and analogies.

### Interoperability

It’s nice when languages play nicely together.

[ONNX Runtime](https://github.com/ankane/onnxruntime) is a scoring engine for ML models. You can build a model in one language, save it in the ONNX format, and run it in another. Here’s [an example](/tensorflow-ruby).

[Npy](https://github.com/ankane/npy) is a library for saving and loading NumPy `npy` and `npz` files. It uses [Numo](/numo) for multi-dimensional arrays.

### Others

[Vowpal Wabbit](https://github.com/ankane/vowpalwabbit) specializes in online learning. It’s great for reinforcement learning as well as supervised learning where you want to train a model incrementally instead of all at once. This is nice when you have a lot of data.

[ThunderSVM](https://github.com/ankane/thundersvm) is an SVM library that runs in parallel on either CPUs or GPUs.

[GSLR](https://github.com/ankane/gslr) is a linear regression library powered by GSL that supports both ordinary least squares and ridge regression. It can be used alone or to improve the performance of [Eps](https://github.com/ankane/eps).

## Shout-out

I wanted to also give a shout-out to another library that entered the scene in 2019.

[Rumale](https://github.com/yoshoku/rumale) is a machine learning library that supports many, many algorithms, similar to Python’s Scikit-learn. Thanks [@yoshoku](https://github.com/yoshoku) for the amazing work!

## Final Word

There are now many state-of-the-art machine learning libraries available for Ruby. If you’re a Ruby engineering who’s interested in machine learning, now’s a good time to try it. Also, if you come across a C or C++ library you want to use in Ruby, you’ve seen a few ways to do it. Let’s make Ruby a great language for machine learning.
