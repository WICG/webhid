# WebHID in Extension Service Workers Explainer

A proposal to expose WebHID in Extension Service Workers.

<!-- TOC -->
<!-- /TOC -->

## Introduction
WebHID is a proposed specification that allows web pages to access connected Human Interface Devices (HID) with the user’s permission. At the moment, in browser extensions, it is common to access WebHID from the extension background page. As the mechanism of using service workers is more prevalent with the advantages like open web compatibility and performance, we want to provide an alternative to the extension developers for accessing WebHID in extension service workers.

## Potential specification changes

### requestDevice() method
Requesting device permissions requires that the browser display a permissions dialog to the user. Since the dialog cannot be shown from a service worker context, the steps for `requestDevice()` will be updated to throw [NotSupportedError](https://webidl.spec.whatwg.org/#notsupportederror) when the [relevant global object](https://html.spec.whatwg.org/multipage/webappapis.html#concept-relevant-global) is not a [Window](https://html.spec.whatwg.org/multipage/window-object.html#window) object.

### HID interface
The HID global accessible component `navigator.hid` will be exposed in Service Worker Context.

```webidl
[Exposed=(Window,ServiceWorker), SecureContext]
interface HID : EventTarget {
    attribute EventHandler onconnect;
    attribute EventHandler ondisconnect;
    Promise<sequence<HIDDevice>> getDevices();
    Promise<sequence<HIDDevice>> requestDevice(
        HIDDeviceRequestOptions options);
}
```

In the proposed implementation, `navigator.hid` is exposed to Service Workers **only in the browser extension**. This should be realized in user agents’ implementations.

## Security
Exposing WebHID in Extension Service Workers does not come with any significant security considerations. Both the WebHID and Service Worker features already contain security measures such as preventing cross-origin access and are also integrated into the Feature Policy, so developers can control where and how these features are exposed in their web applications. The Feature Policy is enforced across the pages and service workers from the same origin. To receive permission to access a HID device, the `requestDevice()` method prompts the user to select a device to connect to. Since `requestDevice()` will not be allowed to be accessed by service workers, the permission request should be done on the page. 