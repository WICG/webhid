# WebHID Explainer

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

The example below requests a HID device by specifying its vendor and product IDs, sends an initialization packet, and listens for an input report containing a button input.

    const deviceFilter = { vendorId: 0x1234, productId: 0xabcd };
    const requestParams  = { filters: [deviceFilter] };
    const initReport = new Uint8Array(1);
    initReport[0] = 42;

    function handleInputReport(e) {
        // Fetch the value of the first field in the report.
        const fieldValue = e.data.getUint8(0);
        console.log('Button value is ' + fieldValue);
    }

    navigator.hid.requestDevice(requestParams).then((devices) => {
        const device = devices[0];
        device.open().then(() => {
            console.log('Opened HID device');
            device.addEventListener('inputreport', handleInputReport);
            device.sendReport(0x01, initReport).then(() => {
                console.log('Sent initialization packet');
            });
        });
    });

## Proposed IDL

The globally accessible component of the WebHID API is attached to the navigator interface as `navigator.hid`.

    partial interface Navigator {
        readonly attribute HID hid;
    };

The `navigator.hid` member allows the page to register event listeners for connection events. The page can request an array of all devices currently accessible by the page with `getDevices`. For security reasons, no HID devices are available by default. The page may call `requestDevice` to display a chooser dialog where the user can select from connected HID devices. Once the selection is completed, the `Promise` returned by `requestDevice` resolves to an array of `HIDDevice` objects.

    interface HID : EventTarget {
        attribute EventHandler onconnect;
        attribute EventHandler ondisconnect;
        Promise<sequence<HIDDevice>> getDevices();
        Promise<sequence<HIDDevice>> requestDevice(HIDDeviceRequestOptions options);
    };

    interface HIDInputReportEvent : Event {
        readonly attribute HIDDevice device;
        readonly attribute octet reportId;
        readonly attribute DataView data;
    };

    dictionary HIDDeviceRequestOptions {
        sequence<HIDDeviceFilter> filters;
    };

    dictionary HIDDeviceFilter {
        unsigned long vendorId;
        unsigned short productId;
        unsigned short usagePage;
        unsigned short usage;
    };

The returned `HIDDevice` objects contain vendor and product ID values for device identification. The devices are returned in the closed state and must be opened before data can be sent or received. The collection attribute is initialized with a hierarchical description of the device's report formats.

Once opened, the `HIDDevice` object can be used to send output reports with `sendReport` or listen for input reports by registering an `oninputreport` event listener. Feature reports can be sent and received with `setFeatureReport` and `receiveFeatureReport`. All methods return a `Promise` that resolves once the operation is complete.

    interface HIDDevice {
        attribute EventHandler oninputreport;
        readonly attribute boolean opened;
        readonly attribute unsigned short vendorId;
        readonly attribute unsigned short productId;
        readonly attribute DOMString productName;
        readonly attribute FrozenArray<HIDCollectionInfo> collections;
        Promise<void> open();
        Promise<void> close();
        Promise<void> sendReport(octet reportId, BufferSource data);
        Promise<void> setOutputReport(octet reportId, BufferSource data);
        Promise<void> setFeatureReport(octet reportId, BufferSource data);
        Promise<DataView> getFeatureReport(octet reportId);
    };

The information contained in the report descriptor is not needed to communicate with the device, but may be useful for applications that rely on the report's characteristics to recognize supported devices. (For instance, an app may not care if a device has other inputs as long as it has at least one button.) It also serves as a guide for parsing data from input reports. Convenience methods `getField` and `setField` allow simpler access to field values within the report buffer.

The IDL below is based off of the information returned by the Windows HID API and is likely to change. On Windows it is not possible to retrieve the full report descriptor from a HID device; instead, applications can query device capabilities from a "preparsed data" buffer to receive equivalent information for a particular collection. To ensure the WebHID API can expose collection information on Windows, `HIDCollectionInfo` will aim to expose a subset of the information available through the Windows HID API.

    interface HIDCollectionInfo {
        readonly attribute unsigned short usagePage;
        readonly attribute unsigned short usage;
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
        "none", "si-linear", "si-rotation", "english-linear",
        "english-rotation", "vendor-defined", "reserved"
    };

    interface HIDReportItem {
        readonly attribute boolean isAbsolute;
        readonly attribute boolean isArray;
        readonly attribute boolean isRange;
        readonly attribute boolean hasNull;
        readonly attribute FrozenArray<unsigned long> usages;
        readonly attribute unsigned long usageMinimum;
        readonly attribute unsigned long usageMaximum;
        readonly attribute unsigned short reportSize;
        readonly attribute unsigned short reportCount;
        readonly attribute unsigned long unitExponent;
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

## Security and privacy considerations

Exposing access to HID devices also potentially exposes personally-identifiable information. Knowledge about which peripherals are connected to the host may be used to fingerprint the user. Some types of HID devices may also expose personally-identifiable information in device details (like the serial number string or product name) or through reports sent by the device to the host. To mitigate this risk, a device will only be exposed to script once the user has explicitly granted access for that device by selecting it from a chooser.

HID devices may also be used to transmit high-value data. Keyboards and Universal 2nd Factor (U2F) devices are often implemented using the HID protocol and may be used to enter credential information. To mitigate the risk of exposing credentials to script, devices that use the FIDO U2F usage page (0xf1d0) are excluded from the device chooser.

WebHID does not provide exclusive access to a device. If two origins have been granted access to the same device, this access can potentially be used to send and receive data between origins by manipulating the state stored on the device.

WebHID may expose additional aspects of a user's local computing environment. Once an origin has been granted access to a device, it can inspect the device details and determine whether the device has been disconnected. In some cases, a peripheral may have capabilities which allow it to expose additional information about the local computing environment like nearby wireless networks.

