# Emotion Recognition in Ruby

Welcome to another installment of deep learning in Ruby. Today, we’ll look at [FER+](https://github.com/Microsoft/FERPlus), a deep convolutional neural network for emotion recognition developed at Microsoft. The project is open source, and there’s a pretrained model in the [ONNX Model Zoo](https://github.com/onnx/models/tree/master/vision/body_analysis/emotion_ferplus) that we can get running quickly in Ruby.

First, download [the model](https://onnxzoo.blob.core.windows.net/models/opset_8/emotion_ferplus/emotion_ferplus.tar.gz) and this photo of a park ranger.

<p style="text-align: center; margin-bottom: 0;"><img src="/images/ranger.jpg" alt="Park Ranger" /></p>

<p class="image-description">
  Photo from <a href="https://www.flickr.com/photos/yellowstonenps/45800621201/" target="_blank">Yellowstone National Park</a>
</p>

We’ll use MiniMagick to prepare the image and the ONNX Runtime gem to run the model.

```ruby
gem "mini_magick"
gem "onnxruntime"
```

For the image, we need to zoom in on her face, resize it to 64x64, and convert it to grayscale. Typically, we’d use a face detection model to find the bounding box and use that information to crop the image, but for simplicity, we’ll do just do it manually.

```ruby
img = MiniMagick::Image.open("ranger.jpg")
img.crop "100x100+60+20", "-gravity", "center" # manual crop
img.resize "64x64^", "-gravity", "center", "-extent", "64x64"
img.colorspace "Gray"
img.write "resized.jpg"
```

Here’s a blown up version:

<p style="text-align: center;"><img src="/images/ranger-resized.jpg" alt="Park Ranger" /></p>

Finally, create a 64x64 matrix of the grayscale intensities.

```ruby
# all pixels are the same for grayscale, so just get one of them
pixels = img.get_pixels.flat_map { |r| r.map(&:first) }
input = OnnxRuntime::Utils.reshape(pixels, [1, 1, 64, 64])
```

Now that the input is prepared, we can load and run the model.

```ruby
model = OnnxRuntime::Model.new("model.onnx")
output = model.predict("Input3" => input)
```

We use [softmax](https://victorzhou.com/blog/softmax/) to convert the model output into probabilities.

```ruby
def softmax(x)
  exp = x.map { |v| Math.exp(v - x.max) }
  exp.map { |v| v / exp.sum }
end

probabilities = softmax(output["Plus692_Output_0"].first)
```

Then map the labels and sort by highest probability.

```ruby
emotion_labels = [
  "neutral", "happiness", "surprise", "sadness",
  "anger", "disgust", "fear", "contempt"
]
pp emotion_labels.zip(probabilities).sort_by { |_, v| -v }.to_h
```

And the results are in:

```
{
  "happiness" => 0.9999839207138284,
  "surprise"  => 1.0569785479062501e-05,
  "neutral"   => 4.826811128840592e-06,
  "anger"     => 4.63037778140089e-07,
  "sadness"   => 9.574742925740587e-08,
  "contempt"  => 7.941520916580971e-08,
  "fear"      => 2.8803367665891773e-08,
  "disgust"   => 1.568577943664937e-08
}
```

There’s a 99.9% probability she looks happy in the photo. Not bad!

Here’s the [complete code](https://gist.github.com/ankane/3bb4ddbf84edd7f05a24cd3697ccd9a7). Now go out and try it with your own images!
