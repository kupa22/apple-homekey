# Apple HomeKey

Warning: This documentation is intended only for studying purposes. Most of the information provided here is based on reverse engineering, guessing and testing (trial-and-error). So the documentation may be inaccurate or incomplete.

## Overview
HomeKey is a proprietary technology which allows to unlock a compatible smart lock by tapping it with a mobile device (phone or watch). Although the NFC lock is setup like a regular lock via HomeKit, the data exchange while tapping is based on NFC. For this purpose a digital key is automatically generated and stored in the Wallet app.

<img src="./resources/screenshot-setup.png" height="200" alt="![HomeKey Setup]"> <img src="./resources/screenshot-wallet.png" height="200" alt="![HomeKey Wallet]">

So HomeKeys use two technologies:
1. HomeKit
   * Required for lock setup, manual locking/unlocking and credential exchange
2. NFC
   * Data exchange between the lock (reader) and the phone

## How it works
When a NFC capable smart lock is added via HomeKit, the phone generates two things:

* A secp256r1 key pair, which is the **Reader Key**.
* Another secp256r1 key pair, which is the **Device Credential**.

The private part of the **Reader Key** is sent to the smart lock and stored there. Its purpose is to authenticate the lock againts the phone during a NFC transaction. Only one Reader Key exists per home.

The public part of the **Device Credential** is also sent to the smart lock and stored there. The Device Credential is unique per device (phone or watch) and used to identify and authenticate the device at the lock during a NFC transaction. Each device invited to the home generates its own Device Credential which is then sent to the lock. A lock can normally store multiple Device Credentials.

<img src="./resources/homekey-keys.png" width="800" alt="![HomeKey Key Overview]">

## HomeKit specification
A HomeKey compatible lock has the same services and characteristics as a normal lock, but exposes additional services and characteristics. Only the latter are described here.

---

### Services
| Name                   | UUID                                     |
| ---------------------- | ---------------------------------------- |
| NFC Access             | 00000266-0000-1000-8000-0026BB765291     |

---

### Characteristics (overview)
**Service "NFC Access"**
| Name                               | UUID                                     | Type                   | Purpose                                                    |
| ---------------------------------- | ---------------------------------------- | ---------------------- | ---------------------------------------------------------- |
| Configuration State                | 00000263-0000-1000-8000-0026BB765291     | UINT16 - read only     | *unknown*                                                  |
| NFC Access Control Point           | 00000264-0000-1000-8000-0026BB765291     | TLV8 - write-response  | Key and credential provisioning                            |
| NFC Access Supported Configuration | 00000265-0000-1000-8000-0026BB765291     | TLV8 - read only       | Number of credentials that can be stored in the lock       |

**Service "Accessory Information"**
| Name                               | UUID                                     | Type                   | Purpose                                                    |
| ---------------------------------- | ---------------------------------------- | ---------------------- | ---------------------------------------------------------- |
| Hardware Finish                    | 0000026C-0000-1000-8000-0026BB765291     | TLV8 - read only       | Color of the Digital Key in the Wallet app                 |

---

### Characteristic "Hardware Finish"
This characteristic is optional and allows to specify the color of the virtual HomeKey in the Wallet app.

TLV8 encoding:
| Description                 | Type        | Length       | Value                                                                                                                                                                                                    |
| --------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Wallet Key Color            | 1           | 4            | One of the following values (hexadecimal):<br>`CED5DA00: Tan (default)`<br>`AAD6EC00: Gold`<br>`E3E3E300: Silver`<br>`00000000: Black`                                                                   |

Example response:
`0104CED5DA00`

---

### Characteristic "NFC Access Supported Configuration"
I'm not sure about the purpose of this characteristic - maybe it's used to tell the phone how many different user credentials can be stored on the lock.

TLV8 encoding:
| Description                                               | Type        | Length       | Value                                                                                                                                                                                                    |
| --------------------------------------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Number of Issuer Keys                                     | 1           | 1            | Number                                                                                                                                                                                                   |
| Number of inactive credentials                            | 2           | 1            | Number                                                                                                                                                                                                   |

Example response:
`010110020110`

Hint: The Issuer Key seems to be identical for all devices on the same iCloud account.

---

### Characteristic "NFC Access Control Point"
This characteristic is used to provision the Reader Key and the Device Credentials in the lock.

The phone always sends a write request containing a TLV8 object to the lock - and the lock then answers to that request with a TLV8 response.

TLV8 encoding:
| Description                                               | Type        | Length       | Value                                                                                                                                                                                                    | Presence                  |
| --------------------------------------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| Operation                                                 | 1           | 1            | `1: get`<br>`2: add`<br>`3: remove`                                                                                                                                                                      | Request only              |
| Device Credential Request                                 | 4           | N            | Sub-TLV8, see table "Device Credential Request"                                                                                                                                                          | Request only              |
| Device Credential Response                                | 5           | N            | Sub-TLV8, see table "Device Credential Response"                                                                                                                                                         | Response only             |
| Reader Key Request                                        | 6           | N            | Sub-TLV8, see table "Reader Key Request"                                                                                                                                                                 | Request only              |
| Reader Key Response                                       | 7           | N            | Sub-TLV8, see table "Reader Key Response"                                                                                                                                                                | Response only             |

Sub-TLV8 encoding of "Device Credential Request":
| Description                                               | Type        | Length       | Value                                                                                                                                                                                                    | Presence                                         |
| --------------------------------------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Key type                                                  | 1           | 1            | `1: curve25519 (not seen yet)`<br>`2: secp256r1`                                                                                                                                                         | Operation "add"                                  |
| Device Credential Public Key                              | 2           | 64           | secp256r1 Public Key (X and Y coordinates concatenated)                                                                                                                                                  | Operation "add"                                  |
| Issuer Key Identifier                                     | 3           | 8            | 8 Byte Issuer Identifier (seems to be identical for all devices on the same iCloud account)                                                                                                              | Operation "add"                                  |
| Key state                                                 | 4           | 1            | `0: inactive (not seen yet)`<br>`1: active`                                                                                                                                                              | Operation "add"                                  |
| Key Identifier                                            | 5           | ?            | ?                                                                                                                                                                                                        | Probably operation "remove" (not seen yet)       |

Sub-TLV8 encoding of "Device Credential Response":
| Description                                               | Type        | Length       | Value                                                                                                                                                                                                    | Presence                                         |
| --------------------------------------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Key identifier                                            | 1           | ?            | ?                                                                                                                                                                                                        | Probably operation "get" (not seen yet)          |
| Issuer Key Identifier                                     | 2           | 8            | 8 Byte Issuer Identifier of the provisioned Device Credential                                                                                                                                            | Operation "add"                                  |
| Status                                                    | 3           | 1            | `0: success`<br>`1: out of resources`<br>`2: duplicate`<br>`3: does not exist`<br>`4: not supported`                                                                                                     | Operations "add" and "remove"                    |

Sub-TLV8 encoding of "Reader Key Request":
| Description                                               | Type        | Length       | Value                                                                                                                                                                                                    | Presence                                         |
| --------------------------------------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Key type                                                  | 1           | 1            | `1: curve25519 (not seen yet)`<br>`2: secp256r1`                                                                                                                                                         | Operation "add"                                  |
| Reader Private Key                                        | 2           | 32           | secp256r1 Private Key                                                                                                                                                                                    | Operation "add"                                  |
| *Unknown*                                                 | 3           | 8            | 8 Byte Identifier (seems to be unique per reader, no known usage)                                                                                                                                        | Operation "add"                                  |
| Key Identifier                                            | 4           | 8            | Identifier for Reader Private Key (see description below how to calculate)                                                                                                                               | Operation "remove"                               |

Sub-TLV8 encoding of "Reader Key Response":
| Description                                               | Type        | Length       | Value                                                                                                                                                                                                    | Presence                                         |
| --------------------------------------------------------- | ----------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------ |
| Key Identifier                                            | 1           | 8            | Identifier for Reader Private Key (see description below how to calculate)                                                                                                                               | Operation "get"                                  |
| Status                                                    | 2           | 1            | `0: success`<br>`1: out of resources`<br>`2: duplicate`<br>`3: does not exist`<br>`4: not supported`                                                                                                     | Operations "add" and "remove"                    |

Calculation of the Identifier for Reader Private Key ("Reader Identifier"):
```
SHA256( concatenate( 6B65792D6964656E746966696572, readerPrivateKey ) ) --> first 8 Bytes is the Reader Identifier
```
"6B65792D6964656E746966696572" is hexadecimal binary encoded ("key-identifier" in ASCII)

Example request "get configured reader key" (no reader key configured):
```
Request:
  01 01 01   --> Operation "get"
  06 00      --> Sub-TLV8 "Reader Key Request" (empty in this case)

Response:
  <empty>
```

Example request "configure reader key":
```
Request:
  01 01 02                                        --> Operation "add"
  06 2f                                           --> Sub-TLV8 "Reader Key Request"
    01 01 02                                      --> Key type "secp256r1"
    02 20 84b660e12d9ec8a331bc65f071cb3679(...)   --> Reader Private Key (secp256r1)
    03 08 a1b2c3d4e5f6a7b8                        --> 8 Byte Identifier (usage not known)

Response:
  07 03        --> Sub-TLV8 "Reader Key Response"
    02 01 00   --> Status "success"
```

Example request "get configured reader key" (reader key configured):
```
Request:
  01 01 01   --> Operation "get"
  06 00      --> Sub-TLV8 "Reader Key Request" (empty in this case)

Response:
  07 0a                      --> Sub-TLV8 "Reader Key Response"
    01 08 94625fd2c32cf3c2   --> Reader Identifier (see description above how to calculate)
```

Example request "provision device credential":
```
Request:
  01 01 02                                        --> Operation "add"
  04 52                                           --> Sub-TLV8 "Device Credential Request"
    01 01 02                                      --> Key type "secp256r1"
    02 40 970eead94f9564a85dbafa06abb04293(...)   --> Device Credential Public Key (secp256r1)
    03 08 12ab34cd56ef0815                        --> Issuer Key Identifier
    04 01 01                                      --> Key state "active"

Response:
  05 0d                      --> Sub-TLV8 "Device Credential Response"
    02 08 12ab34cd56ef0815   --> Issuer Key Identifier
    03 01 00                 --> Status "success"
```

## NFC specification

in progress...

## Useful links and documents
* https://patents.google.com/patent/EP3748900A1/
* https://github.com/homebridge/HAP-NodeJS
* https://github.com/KhaosT/HAP-NodeJS/commit/80cdb1535f5bee874cc06657ef283ee91f258815
* https://github.com/kormax/apple-home-key
* https://developer.apple.com/bug-reporting/profiles-and-logs/
* https://developer.apple.com/documentation/
* HomeKit Accessory Protocol Specification Non-Commercial Version R2