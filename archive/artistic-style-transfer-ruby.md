# Artistic Style Transfer in Ruby

The [ONNX Model Zoo](https://github.com/onnx/models) has a number of interesting pretrained deep learning models. Thanks to the ONNX Runtime, we can run them in Ruby.

Today, we’ll look at artistic style transfer. Here’s [the model](https://github.com/onnx/models/tree/master/vision/style_transfer/fast_neural_style) we’ll use.

First, download the [pretrained model](https://github.com/onnx/models/raw/master/vision/style_transfer/fast_neural_style/models/opset9/rain_princess.onnx) and this awesome shot of a lynx.

<p style="text-align: center; margin-bottom: 0;"><img src="/images/lynx.jpg" alt="Lynx" /></p>

<p class="image-description">
  Photo from the <a href="https://www.flickr.com/photos/usfws_alaska/32576677167/" target="_blank">U.S. Fish and Wildlife Service</a>
</p>

Install the ONNX Runtime, MiniMagick, and Numo::NArray gems. MiniMagick allows us to manipulate images, and Numo::NArray makes it easier to work with multi-dimensional arrays.

```ruby
gem "onnxruntime"
gem "mini_magick"
gem "numo-narray"
```

Next, load the image. We resize it to be the model dimensions.

```ruby
img = MiniMagick::Image.open("lynx.jpg")
img.resize "224x224^", "-gravity", "center", "-extent", "224x224"
pixels = img.get_pixels
```

And load the model

```ruby
model = OnnxRuntime::Model.new("rain_princess.onnx")
```

Perform the preprocessing steps from the model docs

```ruby
pixels = Numo::NArray.cast(img.get_pixels)
pixels = pixels.transpose(2, 0, 1)
pixels = pixels.expand_dims(0)
```

Run the model

```ruby
result = model.predict(input1: pixels)
```

Perform the postprocessing steps

```ruby
out_pixels = Numo::NArray.cast(result["output1"].first)
out_pixels = out_pixels.clip(0, 255).cast_to(Numo::UInt8)
out_pixels = out_pixels.transpose(1, 2, 0)
```

And save the image

```ruby
img = MiniMagick::Image.import_pixels(out_pixels.to_binary, img.width, img.height, 8, "rgb", "jpg")
img.write("output.jpg")
```

<p style="text-align: center;"><img src="/images/lynx-rain-princess.jpg" alt="Lynx with Rain Princess style" /></p>

Four other styles are also available.

<p style="text-align: center; margin-bottom: 0;" class="img-grid">
  <img src="/images/lynx-udnie.jpg" alt="Lynx with Udnie style" />
  <img src="/images/lynx-candy.jpg" alt="Lynx with Candy style" />
  <img src="/images/lynx-mosaic.jpg" alt="Lynx with Mosaic style" />
  <img src="/images/lynx-pointilism.jpg" alt="Lynx with Pointilism style" />
</p>

Here’s the [complete code](https://gist.github.com/ankane/33ffa59ea0f5add37edee04fa7aacd9e). Now go out and try it with your own images!
