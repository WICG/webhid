# WebHID Explainer

This document is an explainer for the [WebHID API](https://wicg.github.io/webhid/), a proposed specification for allowing a web page to communicate with HID devices.

<!-- TOC -->
<!-- /TOC -->

## Basic terminology

A **HID (Human Interface Device)** is a type of device that takes input from or provides output to humans. It also refers to the HID protocol, a standard for bi-directional communication between a host and a device that is designed to simplify the installation procedure. The HID protocol was originally developed for USB devices but has since been implemented over many other protocols, including Bluetooth.

A **HID report** is a binary data packet communicated between the host and the device. Input reports are sent from the device to the host, output reports are sent from the host to the device, and feature reports may be sent in either direction. The format of HID reports is device-specific.

A **HID report descriptor** can be requested by the host during device enumeration. The report descriptor describes the binary format of reports supported by a device. The structure of the descriptor is hierarchical and can group reports together as distinct collections within the top-level collection. The format of the descriptor is defined by the HID specification.

A **HID usage** is a numeric value referring to a standardized input or output. The list of HID usages is available as part of the USB specification. Usage values allow a device to describe the intended use of the device itself as well as the purpose of each field in its reports. Usages are organized into usage pages, which provide an indication of the high-level category of the device or report field.

## Use cases

The web platform already supports input from HID. Keyboards, pointing devices (mice, touchscreens, etc.), and gamepads are all typically implemented using the HID protocol. However, this support relies on the operating system's HID drivers that transform the HID input into native input APIs. Devices that are not well supported by the HID driver are often inaccessible to web pages. Similarly, the outputs on most devices are also inaccessible.

The inability to access uncommon or unusual HID devices is particularly painful when it comes to gamepad support. Gamepads designed for PC often use HID for gamepad inputs (buttons, joysticks, triggers) and outputs (LEDs, rumble). However, gamepads inputs and outputs are not well standardized and web browsers often require custom logic for specific devices. This is unsustainable and results in poor support for the long tail of older and uncommon devices. It also causes the browser to depend on quirks present in the behavior of specific devices. A HID standard for the web platform would allow the device-specific logic to be moved into javascript for devices that lack support in the browser.

The HID protocol is popular in large part because of the ease of installation, which encourages device manufacturers to use HID for purposes other than communication with humans. For instance, HID reports may be used to pair devices over a vendor-proprietary wireless link or update device firmware. These features usually require a small native app on the host's side to initiate the process, but with API support this app could be implemented as a web page.

## Example

The example below requests a HID device by specifying its vendor and product IDs, sends an initialization packet, and listens for input reports and connection events.

```js
let deviceFilter = { vendorId: 0x1234, productId: 0xabcd };
let requestParams = { filters: [deviceFilter] };
let outputReportId = 0x01;
let outputReport = new Uint8Array([42]);

function handleConnectedDevice(e) {
  console.log("Device connected: " + e.device.productName);
}

function handleDisconnectedDevice(e) {
  console.log("Device disconnected: " + e.device.productName);
}

function handleInputReport(e) {
  console.log(e.device.productName + ": got input report " + e.reportId);
  console.log(new Uint8Array(e.data.buffer));
}

navigator.hid.addEventListener("connect", handleConnectedDevice);
navigator.hid.addEventListener("disconnect", handleDisconnectedDevice);

navigator.hid.requestDevice(requestParams).then((devices) => {
  if (devices.length == 0) return;
  devices[0].open().then(() => {
    console.log("Opened device: " + device.productName);
    device.addEventListener("inputreport", handleInputReport);
    device.sendReport(outputReportId, outputReport).then(() => {
      console.log("Sent output report " + outputReportId);
    });
  });
});
```

## Proposed IDL

The globally accessible component of the WebHID API is attached to the navigator interface as `navigator.hid`.

```webidl
partial interface Navigator {
    readonly attribute HID hid;
};
```

The `navigator.hid` member supports registration of event listeners for connect and disconnect events. The page can request an array of all devices currently accessible by the page with `getDevices`. For security reasons, no HID devices are available by default. The page may call `requestDevice` to display a chooser dialog where the user can select from connected HID devices. Once the selection is completed, the `Promise` returned by `requestDevice` resolves to a `sequence<HIDDevice>`.

```webidl
interface HID : EventTarget {
    attribute EventHandler onconnect;
    attribute EventHandler ondisconnect;
    Promise<sequence<HIDDevice>> getDevices();
    Promise<sequence<HIDDevice>> requestDevice(HIDDeviceRequestOptions options);
};

interface HIDConnectionEvent : Event {
    readonly attribute HIDDevice device;
}

interface HIDInputReportEvent : Event {
    readonly attribute HIDDevice device;
    readonly attribute octet reportId;
    readonly attribute DataView data;
};

dictionary HIDDeviceRequestOptions {
    required sequence<HIDDeviceFilter> filters;
    sequence<HIDDeviceFilter> exclusionFilters;
};

dictionary HIDDeviceFilter {
    unsigned long vendorId;
    unsigned short productId;
    unsigned short usagePage;
    unsigned short usage;
};
```

The returned `HIDDevice` objects contain vendor and product ID values for device identification. The devices are returned in the closed state and must be opened before data can be sent or received. The `collections` attribute is initialized with a hierarchical description of the device's report formats.

Once opened, the `HIDDevice` object can be used to send output reports with `sendReport` or listen for input reports by registering an `oninputreport` event listener. Feature reports can be sent and received with `sendFeatureReport` and `receiveFeatureReport`. All methods return a `Promise` that resolves once the operation is complete.

```webidl
interface HIDDevice {
    attribute EventHandler oninputreport;
    readonly attribute boolean opened;
    readonly attribute unsigned short vendorId;
    readonly attribute unsigned short productId;
    readonly attribute DOMString productName;
    readonly attribute FrozenArray<HIDCollectionInfo> collections;
    Promise<void> open();
    Promise<void> close();
    Promise<void> forget();
    Promise<void> sendReport([EnforceRange] octet reportId, BufferSource data);
    Promise<void> sendFeatureReport([EnforceRange] octet reportId, BufferSource data);
    Promise<DataView> receiveFeatureReport([EnforceRange] octet reportId);
};
```

The information contained in the report descriptor is not needed to communicate with the device, but may be useful for applications that rely on the report's characteristics to recognize supported devices. (For instance, an app may not care if a device has other inputs as long as it has at least one button.) It also serves as a guide for parsing data from input reports.

```webidl
interface HIDCollectionInfo {
    readonly attribute unsigned short usagePage;
    readonly attribute unsigned short usage;
    readonly attribute octet type;
    readonly attribute FrozenArray<HIDCollectionInfo> children;
    readonly attribute FrozenArray<HIDReportInfo> inputReports;
    readonly attribute FrozenArray<HIDReportInfo> outputReports;
    readonly attribute FrozenArray<HIDReportInfo> featureReports;
};

interface HIDReportInfo {
    readonly attribute octet reportId;
    readonly attribute FrozenArray<HIDReportItem> items;
};

enum HIDUnitSystem {
    // No unit system in use.
    "none",
    // Centimeter, gram, seconds, kelvin, ampere, candela.
    "si-linear",
    // Radians, gram, seconds, kelvin, ampere, candela.
    "si-rotation",
    // Inch, slug, seconds, Fahrenheit, ampere, candela.
    "english-linear",
    // Degrees, slug, seconds, Fahrenheit, ampere, candela.
    "english-rotation",
    "vendor-defined",
    "reserved",
};

interface HIDReportItem {
    readonly attribute boolean isAbsolute;
    readonly attribute boolean isArray;
    readonly attribute boolean isBufferedBytes;
    readonly attribute boolean isConstant;
    readonly attribute boolean isLinear;
    readonly attribute boolean isRange;
    readonly attribute boolean isVolatile;
    readonly attribute boolean hasNull;
    readonly attribute boolean hasPreferredState;
    readonly attribute boolean wrap;
    readonly attribute FrozenArray<unsigned long> usages;
    readonly attribute unsigned long usageMinimum;
    readonly attribute unsigned long usageMaximum;
    readonly attribute unsigned short reportSize;
    readonly attribute unsigned short reportCount;
    readonly attribute byte unitExponent;
    readonly attribute HIDUnitSystem unitSystem;
    readonly attribute byte unitFactorLengthExponent;
    readonly attribute byte unitFactorMassExponent;
    readonly attribute byte unitFactorTimeExponent;
    readonly attribute byte unitFactorTemperatureExponent;
    readonly attribute byte unitFactorCurrentExponent;
    readonly attribute byte unitFactorLuminousIntensityExponent;
    readonly attribute long logicalMinimum;
    readonly attribute long logicalMaximum;
    readonly attribute long physicalMinimum;
    readonly attribute long physicalMaximum;
    readonly attribute FrozenArray<DOMString> strings;
};
```
