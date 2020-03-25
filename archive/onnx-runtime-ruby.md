# Score Almost Any Machine Learning Model in Ruby

Ruby isn’t a common choice for machine learning, but companies running Ruby can get tremendous value from it.

I’m happy to announce it’s now possible to build advanced models in TensorFlow, Scikit-learn, PyTorch, and a number of other tools, and score them in Ruby with minimal friction.

To do this previously, you’d need to either:

1. Shell out
2. Create a microservice
3. Use a bridge like [PyCall](https://github.com/mrkn/pycall.rb) or [RSRuby](https://github.com/alexgutteridge/rsruby)

All of these approaches require managing another language in production. Luckily, there’s now a better way.

ONNX (pronounced “On-ix”) is a serialization format for storing models created by Facebook and Microsoft. It was designed for neural networks but now supports traditional ML models as well. Based on its current adoption, I won’t be surprised if it replaces PMML as the de facto interchange format for ML models.

To run ONNX models, Microsoft created [ONNX Runtime](https://github.com/microsoft/onnxruntime), a “cross-platform, high performance scoring engine for ML models.” ONNX Runtime has a C API, which Ruby is happy to use. Thanks to FFI, it even works on JRuby!

<p style="text-align: center;"><img src="/images/onnx-runtime-ruby.png" alt="ONNX Runtime Ruby" /></p>

ONNX Runtime is designed to be fast, and Microsoft saw significant increases in [performance](https://cloudblogs.microsoft.com/opensource/2019/05/22/onnx-runtime-machine-learning-inferencing-0-4-release/) for a number of models after deploying it.

This is another step forward for machine learning in Ruby. Earlier this month, XGBoost and LightGBM also [came to Ruby](https://ankane.org/xgboost-lightgbm-come-to-ruby).

Check out the [ONNX Runtime](https://github.com/ankane/onnxruntime) gem today!
