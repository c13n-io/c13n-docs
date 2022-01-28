# c13n Payload Protocols (experimental)

## Introduction

c13n allows you to send arbitrary data from your node to another one in the network (destination node should use c13n, or be compatible with [c13n utilized TLVs](../#a-few-words-on-tlvs)). You are free to use this space to the best of your interest.

For cases like messaging & communicating payment requests over c13n, we promote the establishment of common payload protocols. Given the fact that users of the Lightning Network control the components of their stack, the applications they choose might differ. It is the best for the ecosystem to allow those applications to be interoperable, meaning that users of `LightningApp-X` shall be able to normally interact with users of `LightningApp-Y`.

The concept of a payload protocol attempts to extend the concept of sending raw message strings to sending (serialized) JSON objects containing various information. This will enable sending messages that trigger special functionalities, as descriptive information regarding the type and content of the composite message will be contained in the message itself.

Since payload protocols take place inside the available payload space of Lightning Network, it is important to keep any metadata information as short as possible, due to the fact that this space is limited. Also, since each exchange of information costs a very small (but noticable) amount of sats, such protocols attempt to cover their desired features without requiring a lot of back-and-forth Lightning Network payment-traffic.



## Protocols

These protocols are based on the [c13n Payload Protocol Template](#c13n-payload-protocol-template), a set of common rules for protocol messages to follow. This is designed with the intention of future-proofing, as these baselines attempt to create protocols that are self-descriptive and handle each other gracefully.

> [Arc](https://github.com/c13n-io/arc) is an application which supports both `c13n-mp` and `c13n-pp`.

- ### c13n Messaging Protocol

```js
{
    # Name of protocol
    n: "c13n-mp"
    # Version of protocol
    v: "0.0.1c",
    # Type of payload
    t: "message",
    # Content related to payload type
    # message: the message text
    c: "",
    # (Optional) File / Media attachments
    att: [{
        # Type of attachment
        t: "image" | "file",
        # Location / URL
        u: "",
        # Metadata
        tags: "lsat" | "",
        # Visibility flag for chat
        show: "true" | "false"
    }]
}
```

- ### c13n Payreq Protocol

```js
{
    # Name of protocol
    n: "c13n-pp"
    # Version of protocol
    v: "0.0.1a",
    # Type of payload
    t: "payreq",
    # Content related to payload type
    # payreq: the payment request
    c: "",
    # (Optional) Description of payment request
    d: ""
}
```
## c13n Payload Protocol Template

Since protocol nesting is not an efficient approach due to the limited payload size, a new concept where 

- protocols are brought to the same level and 
- protocol messages share the same template

can be considered.

For example, a chatting application that wants to integrate payments into chatting should not carry payment related information inside `c13n-mp`, but instead send `c13n-pp` messages to the other party.

Lightning applications that operate over Lightning micropayments will have to interact with a variety of other applications and services over Lightning through the same medium. In order to reduce the technical weight of every application over c13n, it is better to work under the convention that arbitrary messages originating from the network will follow a specific format which will describe the message itself. This makes it easier for the application to decode the message and decide whether it supports a protocol/feature or not.

Protocols do not need to define arbritrary data formats, but can instead respect the following template for better consistency, compatibility and application-level interoperability.

As nesting proves inefficient in this context (at least since the payload size is fixed for now), all protocols are brought down to the same level. Hence, it is wiser to make them self descriptive in order for clients to handle them gracefully.

```js
{
    # Name of protocol (required field)
    n: "string",
    # Version of protocol (required field)
    v: "x.y.z",
    # Protocol message type (required field)
    t: "type1" | "type2" | "type3" | ...
    # Main content of specific message type (required field)
    c: "",
    # Protocol-specific metadata fields
    field1: type, # (required or optional)
    field2: type, # (required or optional)
    field3: type, # (required or optional)
    ...
    
}
```

The required fields for protocol messages following the c13n payload protocol template are
- The name, `n`: Name of the protocol this message is part of
- The version, `v`: The version of the aforementioned protocol
- Message types, `t`: The type of the message. This is protocol-specific, as each protocol will utilize a different set of message types depending on its scope.
- Content, `c`: The content related to the specific message.
- Optional extra fields, `field1` ... `field99`: Any other field that protocol messages require for extra functionality / descriptiveness.



### `c13n-mp` follows the c13n payload protocol template

We can see that `c13n-mp` is a protocol that follows the c13n payload protocol template. In detail:

- It includes the name `n: "c13n-mp"` and version `v: "0.0.1c"` of the protocol.
- It defines a list of message types (`t`), which in this case is just one type ,`message`. On a future version of `c13n-mp` more message types could be defined, like `reply` or `reaction`.
- There is a field `c` for the main content of the specific message type. For example, in the case of message type `message`, this field contains the actual message the user typed inside the Lightning Application.
- We can see the rest of protocol-specific top level fields:
    - `att`: (used by `message` types) List of attachments (URL based) of this message, along with their metadata.
        * Type of attachment, `t`
        * URL of attachment, `u`
        * Optional tags related to accessing the file, `tags`
        * Flag that indicates if it should be rendered by default with the message, `show`