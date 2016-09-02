# navigator.sendBeacon and Rails

[navigator.sendBeacon](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/sendBeacon) is a neat new API. It allows you to send an asynchronous `POST` request without delaying the page unload.

To prevent the `Can't verify CSRF token authenticity` with Rails, use the method below:

```javascript
function csrfProtect(payload) {
  var param = $("meta[name=csrf-param]").attr("content");
  var token = $("meta[name=csrf-token]").attr("content");
  if (param && token) payload[param] = token;
  return new Blob([JSON.stringify(payload)], {type : "application/json; charset=utf-8"});
}
```

And do:

```javascript
var payload = {hello: "beacon"};
navigator.sendBeacon("/some/path", csrfProtect(payload));
```

For a real-world use case, check out [Ahoy.js](https://github.com/ankane/ahoy.js).

:anchor:
