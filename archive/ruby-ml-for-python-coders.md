# Ruby ML for Python Coders

<p style="text-align: center; margin-bottom: 0;">
  <img src="/images/python-ruby.png" alt="Python and Ruby" style="max-height: 200px;" />
</p>

Curious to try machine learning in Ruby? Here’s a short cheatsheet for Python coders.

Data structure basics

- [Numo: NumPy for Ruby](/numo)
- [Daru: Pandas for Ruby](/daru)

Libraries

Category | Python | Ruby
--- | --- | ---
Multi-dimensional arrays | [NumPy](https://github.com/numpy/numpy) | [Numo](https://github.com/ruby-numo/numo-narray)
Data frames | [Pandas](https://github.com/pandas-dev/pandas) | [Daru](https://github.com/SciRuby/daru)
Visualization | [Matplotlib](https://github.com/matplotlib/matplotlib) | [Nyaplot](https://github.com/domitry/nyaplot)
Predictive modeling | [Scikit-learn](https://github.com/scikit-learn/scikit-learn) | [Rumale](https://github.com/yoshoku/rumale)
Gradient boosting | [XGBoost](https://github.com/dmlc/xgboost), [LightGBM](https://github.com/Microsoft/LightGBM) | [XGBoost](https://github.com/ankane/xgb), [LightGBM](https://github.com/ankane/lightgbm)
Deep learning | [PyTorch](https://github.com/pytorch/pytorch), [TensorFlow](https://github.com/tensorflow/tensorflow) | [Torch-rb](https://github.com/ankane/torch-rb), [TensorFlow](https://github.com/ankane/tensorflow) (TensorFlow :construction:)
Recommendations | [Surprise](https://github.com/NicolasHug/Surprise), [Implicit](https://github.com/benfred/implicit/) | [Disco](https://github.com/ankane/disco)
Approximate nearest neighbors | [NGT](https://github.com/yahoojapan/NGT), [Annoy](https://github.com/spotify/annoy) | [NGT](https://github.com/ankane/ngt), [Hanny](https://github.com/yoshoku/hanny)
Factorization machines | [xLearn](https://github.com/aksnzhy/xlearn) | [xLearn](https://github.com/ankane/xlearn)
Natural language processing | [spaCy](https://github.com/explosion/spaCy), [NTLK](https://github.com/nltk/nltk) | [Many gems](https://github.com/arbox/nlp-with-ruby) (nothing comprehensive :cry:)
Text classification | [fastText](https://github.com/facebookresearch/fastText) | [fastText](https://github.com/ankane/fasttext)
Forecasting | [Prophet](https://github.com/facebook/prophet) | :cry:
Optimization | [CVXPY](https://github.com/cvxgrp/cvxpy), [PuLP](https://github.com/coin-or/pulp), [SCS](https://github.com/cvxgrp/scs), [OSQP](https://github.com/oxfordcontrol/osqp) | [CBC](https://github.com/gverger/ruby-cbc), [SCS](https://github.com/ankane/scs), [OSQP](https://github.com/ankane/osqp)
Reinforcement learning | [Vowpal Wabbit](https://github.com/VowpalWabbit/vowpal_wabbit) | [Vowpal Wabbit](https://github.com/ankane/vowpalwabbit)
Scoring engine | [ONNX Runtime](https://github.com/Microsoft/onnxruntime) | [ONNX Runtime](https://github.com/ankane/onnxruntime), [Menoh](https://github.com/pfnet-research/menoh-ruby)

This list is by no means comprehensive. Some Ruby libraries are ones I created, as mentioned [here](/new-ml-gems).

If you’re planning to add Ruby support to your ML library:

Category | Python | Ruby
--- | --- | ---
FFI (native) | [ctypes](https://docs.python.org/3/library/ctypes.html) | [Fiddle](https://ruby-doc.org/stdlib-2.7.0/libdoc/fiddle/rdoc/Fiddle.html)
FFI (library) | [cffi](https://cffi.readthedocs.io/en/latest/) | [FFI](https://github.com/ffi/ffi)
C++ extensions | [pybind11](https://github.com/pybind/pybind11) | [Rice](https://github.com/jasonroelofs/rice)
Compile to C | [Cython](https://github.com/cython/cython) | [Rubex](https://github.com/SciRuby/rubex)

Give Ruby a shot for your next maching learning project!
