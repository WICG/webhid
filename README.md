# WebHID API

[HID API](http://wicg.github.io/webhid/) for the web platform. 

### Explainer

Details about the API including example usage code snippets and its motivation, privacy, and security considerations are described in the [Explainer](./EXPLAINER.md).

### Code of conduct

We are committed to providing a friendly, safe and welcoming environment for all. Please read and respect the [W3C Code of Ethics and Professional Conduct](https://www.w3.org/Consortium/cepc/).

### Implementation status

This API is implemented in [Blink](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/hid/). It is available in browsers based on Chromium 89 and later, such as Google Chrome and Microsoft Edge. Individual Chromium-based browsers may choose to enable or disable this API. Chromium-based Android browsers do not support this API because Android itself does not provide a direct API for accessing HID devices ([Chromium issue 964441](https://crbug.com/964441)). For the same reason this API is not available in Android WebView ([Chromium issue 1164125](https://crbug.com/1164125)).