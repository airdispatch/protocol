The Airdispatch Protocol
============

This repository will contain the specifications for the Airdispatch protocol.

### Introduction

Airdispatch is a new application-level protocol that seeks to make it easier to provide federated communications platforms. Currently, new services spring up everyday that provide immense functionality to the user with the caveat of being centralized. Airdispatch aims to make more and more products into interoperable services.

The protocol is able to abstract away the problems of addressing, security, and delivery of messages so that developers can focus on the bigger issues. This spec will run through the expected usage of each of the components and how they should interact.

### Table of Contents

  1. [Summary of Terms](https://github.com/airdispatch/ad-spec#summary-of-terms)
  2. [Message Structure](https://github.com/airdispatch/ad-spec#message-structure)
  3. [Message Signatures](https://github.com/airdispatch/ad-spec#message-signatures)
  4. [Messsage Types](https://github.com/airdispatch/ad-spec/blob/master/README.md#message-types)
    1. Tracker Messages
    2. Server Messages
    3. Utility Messages
  5. Server Protocol
    1. Responds To
    2. Returns
  6. Tracker Protocol
    1. Responds To
    2. Returns
  7. Mail Data Format
    1. Supported Data Types
    2. Encryption
  8. Race Conditions
  9. Returning Errors

### Summary of Terms

  - **Network Object**: Client or server that connects to the Airdispatch network
  - **Tracker**: Any object in the network graph that responds to tracker messages (Query and Registration) and keeps a database that maps airdispatch addresses to network locations (any resolvable address, IP or URL)
  - **Server**: (Also referred to as the 'mailserver') The location in the network that stores outgoing messages and receives incoming alerts for a specific user. A user may only have one mailserver associated with their address at a time.
  - **Client**: Software that interacts with the user to coordinate the sending and receiving of mail. This software must interact with the server (which actually sends and receives the mail).
  - **Field Omission**: In terms of this protocol, a field may be considered 'omitted' if it is not present, initialized to `nil`, an empty array (for the `repeated` datatype), or the [protocol buffers empty value](https://developers.google.com/protocol-buffers/docs/proto#optional) for that data type.
  - **Optional Field**: An optional field is one that may be omitted in a message. It is designated in this document by the word `OPTIONAL` in the field descritpion.

### Message Structure

All Airdispatch messages are prefixed with a six-byte code that alerts the 
receiving network object to the version of the Airdispatch protocol.

The first two bytes are the Airdispatch version identifier, and currently there is only one allowed value:
  - `AD`: Sent over the wire as `41 44`

The following four bytes specify the length of the payload (in bytes) in big-endian format.

Example:

    Sending a 617kB (631808 bytes) Airdispatch Message...
    
    Message Header:
    41 44 00 09 A4 00
    
    Message Body:
    ..
    ..
    ..

The body of the message is always encoded with [protocol buffers](https://code.google.com/p/protobuf/), so we do not have to worry about the format by which we transfer primatives over the wire.

### Message Signatures

All Airdispatch messages are signed with an [ECDSA](http://en.wikipedia.org/wiki/Elliptic_Curve_DSA) keypair. This ensures that addresses are not spoofed.

We are currently using the P256 curve as specified by [FIPS](http://csrc.nist.gov/publications/fips/fips186-3/fips_186-3.pdf). This curve is also known as the secp256r1 curve as specified by [SECG](http://www.secg.org/collateral/sec2_final.pdf).

Airdispatch converts the key (specified using 32-byte integers X and Y) to wire-transferrable bytes by concatenating the following values:

  - 0x03 (this is used to identify the type of encryption key used, currently, the only supported value is 0x03 which specifies an ECDSA key on the P256 curve)
  - 32-byte X Value in Big Endian Format
  - 32-byte Y Value in Big Endian Format

Each message transferred on the Airdisaptch protocol is embedded in the [SignedMessage data structure](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L102). It is as defined below:

| Field        | Object Type  | Description                          | Protocol Buffers Field Number  |
|--------------|--------------|--------------------------------------|--------------------------------|
| payload      | Byte Array   | The actual message being transferred | 1 |
| signing_key  | Byte Array   | The ECDSA public key converted to bytes as shown above | 2 |
| signature    | `Signature`  | An additional data type that stores the signature | 3 |
| message_type | string       | Any defined 3-character `Message Type` as specified in the table [here](https://github.com/airdispatch/ad-spec/blob/master/README.md#message-types). | 4 |
| signature_function | string | **OPTIONAL**: currently unused, will be used in the future to specify using an algorithm other than ECDSA on the P256 curve | 5 |

The `Signature` data structure ([implemented here](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L112)):

| Field | Object Type | Description | Protocol Bufferes Field Number |
|-------|-------------|-------------|--------------------------------|
| r     | Byte Array  | The r parameter from the ECDSA signature function (big-endian) | 1 |
| s     | Byte Array  | The s parameter from the ECDSA signature function (big-endian) | 2 |


### Message Types

The current protocol is divided into several different types of messages as outlined below and defined in relevant sections.

| Message Type  | Implementation Name | Description                            | Defined As      |
|---------------|---------------------|----------------------------------------|-----------------|
| REG           | [AddressRegistration](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L17) | Used to register an address and location with a tracker. | Tracker |
| QUE           | [AddressRequest](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L25)      | Used to query a tracker for an address's location. | Tracker |
| RES           | [AddressResponse](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L35)     | Returned by a tracker specifying the location for the requested address. | Tracker |
| ALE           | [Alert](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L56)               | Used to alert a receiving server to new mail. | Server |
| RET           | [RetrieveData](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L65)        | Used to download data from a mailserver.      | Server |
| SEN           | [SendMailRequest](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L47)     | Used to request that a server send a message on a client's behalf. | Server |
| ARR           | [ArrayedData](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L77)         | Used to signifiy that multiple Airdispatch messages follow. | Utility |
| MAI           | [Mail](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L82)                | The message type that holds the actual message. | Utility |
| ---           | [MailData](https://github.com/airdispatch/airdispatch-protocol/blob/master/airdispatch/Message.proto#L88)            | The key-value store that contains the message data and metadata. | Utility |

#### Tracker Messages

The tracker is the portion of protocol that manages the translation of addresses and locations. This section will detail the structure of the messages that the tracker is associated with.

###### REG Message (AddressRegistration)

The AddressRegistration message is used to store the location of the mail server for a specific address with a tracker.

| Field | Object Type | Description | Protocol Bufferes Field Number |
|-------|-------------|-------------|--------------------------------|
| address     | string  | The address being registered (this must match the key used to verify the signature of the message or the message will be discarded). | 1 |
| public_key     | Byte Array  | The public key used to encrypt messages sent from this address (read more in section 7.2 on Encryption). NOTE: This is **not** the key used to sign messages. | 2 |
| location    | string | Any DNS-resolvable (i.e. URL or IP address with a port number) string that designates the location of the mailserver that messages sent to this address should be forwarded to. | 3 |
| username | string | OPTIONAL: This field is used to register a username with the tracker, so that addresses may be queried without knowing the long-form. | 4 |

###### QUE Message (AddressRequest)

The AddressRequest message is used to look-up the location of an address.

| Field | Object Type | Description | Protocol Buffers Field Number |
|-------|-------------|-------------|-------------------------------|
| address | string | OPTIONAL: The address that is being queried for. | 1 |
| username | string | OPTIONAL: The username that is being queried for. | 2|
| need_key | bool | OPTIONAL: This field should be set to false when a server originating the query message does not wish to receive the encryption key in the response. | 3 |

**IMPORTANT NOTE:** The AddressRequest message must contain the address field *or* the username field, but not both. One or the other must be omitted.

###### RES Message (AddressResponse)

The AddressResponse message is used to return the location of an address based on an AddressRequest message.

| Field | Object Type | Description | Protocol Buffers Field Number |
|-------|-------------|-------------|-------------------------------|
| server_location | string | The exact value of the `location` field specified for the address queried for when it originally sent an `AddressRegistration` message. | 1 |
| address | string | The addresss that was specified in the `AddressRequest` message to be queried for. | 2 |
| public_key | Byte Array | OPTIONAL: The exact value of the `public_key` field originally registered for the address. Only returned when the `need_key` field of the `AddressRequest` message is not false. | 3 |

#### Server Messages

#### Utility Messages
