# navigator.sendBeacon and Rails

[navigator.sendBeacon](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon) is a neat new API. It allows you to send an asynchronous `POST` request without delaying the page unload.

To prevent `Can't verify CSRF token authenticity` with Rails, use the method below:

```javascript
var data = new FormData();
data.append("hello", "beacon");

// add CSRF
var param = document.querySelector("meta[name=csrf-param]").getAttribute("content");
var token = document.querySelector("meta[name=csrf-token]").getAttribute("content");
data.append(param, token);

navigator.sendBeacon("/beacon", data);
```

For a real-world use case, check out [Ahoy.js](https://github.com/ankane/ahoy.js).

:anchor:
