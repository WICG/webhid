# WebHID
## Responses to the W3C Security and Privacy Questionnaire

## Questions and Answers

### 1. [Does this specification deal with personally-identifiable information?](https://www.w3.org/TR/security-privacy-questionnaire/#pii)

**Yes, indirectly.** Information about which peripherals are connected to
the host may be used to fingerprint the user. Some types of HID devices
may also expose personally-identifiable information. This is mitigated by
only exposing a device to script once the user has explicitly granted
access for that device (for instance, by selecting it from a chooser).

### 2. [Does this specification deal with high-value data?](https://www.w3.org/TR/security-privacy-questionnaire/#credentials)

**Yes, indirectly.** Keyboards and FIDO devices are often implemented as
HID peripherals and may be used to enter credential information. To
mitigiate the risk of exposing credential information to script, FIDO
devices are excluded from the device chooser. Keyboards and pointer
devices (mice, touchscreens, etc) may be included, but reports carrying
keyboard and pointer information are not delivered to script and
information about these reports is blocked.

### 3. [Does this specification introduce new state for an origin that persists across browsing sessions?](https://www.w3.org/TR/security-privacy-questionnaire/#persistent-origin-specific-state) 

**No.** The WebHID specification does not introduce new state, however a
user agent may choose to persist device permissions across browsing
sessions.

### 4. [Does this specification expose persistent, cross-origin state to the web?](https://www.w3.org/TR/security-privacy-questionnaire/#persistent-identifiers) 

**No**

### 5. [Does this specification expose any other data to an origin that it doesn’t currently have access to?](https://www.w3.org/TR/security-privacy-questionnaire/#other-data)

**Yes**, it exposes a list of devices for which the origin has already
been granted access and allows data to be sent to and received from
these devices. If no devices have been granted access for the origin
then no additional data is exposed.

Device access is not exclusive, so if multiple origins have been
granted access for the same device then it can potentially be used to
communicate across origins.

### 6. [Does this specification enable new script execution/loading mechanisms?](https://www.w3.org/TR/security-privacy-questionnaire/#string-to-script)

**No**

### 7. [Does this specification allow an origin access to a user’s location?](https://www.w3.org/TR/security-privacy-questionnaire/#location)

**No**

### 8. [Does this specification allow an origin access to sensors on a user’s device?](https://www.w3.org/TR/security-privacy-questionnaire/#sensors)

**Yes.** Many types of sensors are implemented as HID peripherals.

### 9. [Does this specification allow an origin access to aspects of a user’s local computing environment?](https://www.w3.org/TR/security-privacy-questionnaire/#local-device)

**Yes.** WebHID exposes a list of connected HID peripherals for which
the current origin has been granted access. It also allows script to
send and receive data from the peripheral which may expose additional
information about the local computing environment.

### 10. [Does this specification allow an origin access to other devices?](https://www.w3.org/TR/security-privacy-questionnaire/#remote-device) 

**Yes.** WebHID allows an origin to access connected HID peripherals.

### 11. [Does this specification allow an origin some measure of control over a user agent’s native UI?](https://www.w3.org/TR/security-privacy-questionnaire/#native-ui) 

**No**

### 12. [Does this specification expose temporary identifiers to the web?](https://www.w3.org/TR/security-privacy-questionnaire/#temporary-id)

**No**

### 13. [Does this specification distinguish between behavior in first-party and third-party contexts?](https://www.w3.org/TR/security-privacy-questionnaire/#first-third-party)

**No**

### 14. [How should this specification work in the context of a user agent’s "incognito" mode?](https://www.w3.org/TR/security-privacy-questionnaire/#incognito)

Device permissions granted in incognito mode must not persist beyond the incognito session.

### 15. [Does this specification persist data to a user’s local device?](https://www.w3.org/TR/security-privacy-questionnaire/#storage) 

**No**, although a user agent may choose to persist site permissions to remember
devices that have already been granted access.

### 16. [Does this specification have a "Security Considerations" and "Privacy Considerations" section?](https://www.w3.org/TR/security-privacy-questionnaire/#considerations)

**Yes**

### 17. [Does this specification allow downgrading default security characteristics?](https://www.w3.org/TR/security-privacy-questionnaire/#relaxed-sop) 

**No**

## Mitigations

Access to the WebHID API is restricted to secure contexts. By default, no device
information is exposed to script. A user must explicitly grant access for an origin
to access a device by selecting the device from a chooser list. Information about
other devices in the list is not exposed to script.
