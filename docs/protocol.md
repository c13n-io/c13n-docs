# c13n Payload Protocols (experimental)

c13n allows you to send arbitrary data from your node to another one in the network (destination node should use c13n, or be compatible with c13n utilized TLVs). You are free to use this space to the best of your interest.

For the specific case of messaging over c13n, we promote the establishment of a common messaging protocol. Given the fact that users of the Lightning Network control the components of their stack, the applications they choose for the purpose of messaging might differ. It is the best for the ecosystem to allow those applications to be interoperable, meaning that users of `ChattingApp-X` shall be able to normally chat with users of `ChattingApp-Y`.

## c13n Messaging Protocol

The concept of a payload protocol attempts to extend the concept of sending raw message strings as payload to sending JSON objects containing various information. This will enable sending messages that trigger special functionalities, as descriptive information regarding the type and content of the composite message will be contained in the message itself.

Since this protocol takes place inside the payload space of LN, it is critical to keep any metadata information as short as possible, due to the fact that this space is limited.

Also, since each exchange of information costs a very small (but noticable) amount of sats, this protocol attempts to cover all the common features of modern messaging applications without requiring a lot of back-and-forth Lightning Network payment-traffic.

```js
{
    # Name of protocol
    n: "c13n-mp"
    # Version of protocol
    v: "0.0.1c",
    # Type of payload
    t: "message" | "payreq",
    # Content related to payload type
    # message: the message text
    # payreq: description of payment request
    c: "",
    # (optional -- only for payreq) The invoice of the payment request
    payreq: 0,
    # (optional -- only for message) File / Media attachments
    att: [{
        # Type of attachment
        t: "image | file",
        # Location / URL
        u: "",
        # Metadata
        tags: "lsat",
        # Visibility flag for chat
        show: "true | false"
    }],
}
```

This protocol is based on the [c13n Payload Protocol Template](#c13n-payload-protocol-template), a set of common rules for protocol messages to follow. This is designed with the intention of future-proofing, as these baselines attempt to create protocols that are self-descriptive and handle each other gracefully.

## c13n Payload Protocol Template

Since protocol nesting is not a feasible scenario due to the limited payload size, a new concept where protocols share a same template can be considered.

Lightning applications that operate over Lightning micropayments will have to interact with a variety of other applications and services over Lightning through the same medium. In order to reduce the technical weight of every application over c13n, it is better to work under the convention that arbitrary messages originating from the network will follow a specific format which will describe the message itself. This makes it easier for the application to decode the message and decide whether it supports a protocol/feature or not.

Protocols do not need to define arbritrary data formats, but can instead respect the following template for better consistency, compatibility and application-level interoperability.

As nesting proves inefficient and not extendable as a concept, since we can not increase the available payload size, all protocols are brought down to the same level. Hence, it is wiser to make them self descriptive in order for clients to handle them gracefully.

```js
{
    # Name of protocol
    n: "string",
    # Version of protocol
    v: "x.y.z",
    # Protocol message type
    t: "type1" | "type2" | "type3" | ...
    # Main content of specific message type
    c: "",
    # Protocol-specific metadata fields (can be optional)
    field1: type1,
    field2: type2,
    field3: type3,
    ...
    
}
```

The required fields for protocol messages following the c13n payload protocol template are
- The name, `n`: Name of the protocol this message is part of
- The version, `v`: The version of the aforementioned protocol
- Message types, `t`: The type of the message. This is protocol-specific, as each protocol will utilize a different set of message types depending on its scope.
- Content, `c`: The content related to the specific message.
- Optional extra fields, `field1` ... `field99`: Any other field that protocol messages require for extra descriptiveness.



### `c13n-cp` follows the c13n payload protocol template

We can see that `c13n-cp` is a protocol that follows the c13n payload protocol template. In detail:

- It includes the name `n: "c13n-cp"` and version `v: "0.0.1c"` of the protocol.
- It defines a list of message types (`t`), which in this case are the message types that take place in a chatting application, these types are `message` (for standard messages) and  `payreq` (for messages that issue payment requests).
- There is a field `c` for the main content of the specific message type. For example, in the case of message type `message`, this field contains the actual message the user typed with his/her keyboard. In the case of message type `payreq`, it could contain a short description related to the payment request, i.e. "You owe me 50K sats for the concert tickets".
- We can see the rest of protocol-specific top level fields:
    - `payreq`: (used by `payreq` message type) The Lightning Network payment request.
    - `att`: (used by `message` message types) List of attachments (URL based) of this message, along with their metadata.
        * Type of attachment, `t`
        * URL of attachment, `u`
        * Optional tags related to accessing the file, `tags`
        * Flag that indicates if it should be rendered by default with the message, `show`