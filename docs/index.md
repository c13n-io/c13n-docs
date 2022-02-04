# c13n documentation

## Gentle Introduction

### Goals

This project started as a proof of concept aiming at communication over [Lightning](https://lightning.network/).

To that end, c13n exploits both the decentralized nature of Lightning and its transactional security as a communication layer (as will be explained below). The fact that Lightning is a payment medium as well is just a sweet bonus.

Despite initially being geared exclusively towards communication, the direction has started to shift more towards adding full support for payment requests and payments in order to take advantage of more features such as [MPP](https://github.com/lightningnetwork/lightning-rfc/blob/master/04-onion-routing.md#basic-multi-part-payments) (see also [here](https://github.com/lightningnetwork/lightning-rfc/blob/master/09-features.md#bolt-9-assigned-feature-flags)) and [offers](https://github.com/rustyrussell/lightning-rfc/blob/guilt/offers/12-offer-encoding.md#bolt-12-flexible-protocol-for-lightning-payments) (if and when they become an official part of the BOLT specification).

### Lightning Introduction

After use of Bitcoin became more widespead, transaction throughput on its blockchain started to become insufficient.

Lightning is a network of *nodes* that enables transient transactions between participants in a trustless and decentralized manner by utilizing the Bitcoin blockchain for anchoring *on-chain transactions* and a gossip protocol over peer-to-peer connections[^peers] for discovery and routing, offering a solution to that problem.

Each node has a *wallet*, which controls funds and allows them to make transactions on the network. Nodes can use those funds to *open* a *channel* with another node with an on-chain transaction, manipulate the allocation of funds on it and *close* it with another on-chain transaction. Between opening and closing, the balance of a node on a channel -- which is essentially a redeemable IOU -- is manipulated by means of *off-chain transactions*. This and the fact that nodes can _route_ payments is what enables higher transaction throughput.
The *balance* of a node (on a channel) are the funds available to that node (on that channel), and one can imagine a channel as -- in a sense -- a pair of balances, one on each endpoint.

[*HTLC*s](https://en.bitcoin.it/wiki/Hash_Time_Locked_Contracts) are the [building blocks](https://docs.lightning.engineering/the-lightning-network/multihop-payments/hash-time-lock-contract-htlc) that, along with rationality -- or greediness, depending on one's definition--, enable the exchange of funds (a fancy way of saying atomic balance manipulation across channels) along payment routes. A *payment* is essentially an abstraction over a set of HTLCs, which attempt to *settle* it.
In order for a node to pay another they need a few logistical details, such as a way to reach the recipient (the recipient's public key if they are discoverable, or a way to route the payment if they are not) and a way for the recipient to identify the reason for payment.
The above requirements are fulfiled by an *invoice* for the payee, and by a [*payment request*](https://docs.lightning.engineering/the-lightning-network/lightning-overview/payment-lifecycle) -- the publicly-shared part of an invoice -- for the payer.

Apart from the method outlined above (of a payment fulfiling an invoice), there is also a way to send a payment without a corresponding invoice present on the receiving node. These payments, which are essentially unsolicited, are called *spontaneous* or *keysend payments*.

You can also head to [Lightning Labs Documentation](https://docs.lightning.engineering/the-lightning-network/lightning-overview) for more documentation related to Lightning Network.

## Architectural Overview

![](https://i.imgur.com/6AAG0qh.png)

### Stack Components

- **Bitcoin node**
    Used for Lightning Network to manage on-chain transactions (such as channel opening and closing transactions, and direct on-chain transactions)
- **Lightning node** (LND currently supported)
    Connects to a bitcoin node in order to manage channels and perform off-chain (onion routed) lightning-fast payments.
- **c13n node**
    Connects to a Lightning node, sending payments and listening for received ones in order to provide data transfer over Lightning Network.
- **Application**
    An application built on top of c13n, which takes advantage of data transfer functionalities.

### Daemon Communication

c13n sits on top of an `lnd` using its `rpc` interface, and offering  functionality over an `rpc` interface of its own to consumers.

It only needs access to the daemon's interface without utilizing any external or centralized services, with even data transferred inside a payment's onion payload.

Access to a node requires the wallet being unlocked, as well as the `rpc` interface connection credentials, and the offered interface is only protected by basic auth at the moment.

If you are looking for an application over c13n as a reference, make sure to check [Arc](), a chatting application that exclusively utilizes Lightning Network payments.

This project aims to respect the decentralized, peer-to-peer, secure and private character of the utilized medium, the Lightning Network.

## Compatibility

c13n is currently implemented on top of [lnd](https://github.com/lightningnetwork/lnd) since it provides easy access to custom records over its API, although extending support to other daemons is planned (most notably [c-lightning](https://github.com/ElementsProject/lightning)).

### Node Requirements

`lnd` provides both access to the custom records of HTLCs as well as support for signing with the node's private key.

At the moment, the requirements for setting up c13n is an operational `lnd` with enabled [Lightning](https://api.lightning.community/#service-lightning) and [Router](https://api.lightning.community/#service-router) `gRPC` services as well as `--accept-keysend` present in its configuration, which allows receipt of incoming spontaneous payments.

NOTE: c13n does not yet implement support for unlocking the node's wallet, so it should be unlocked prior to connection with `lnd`.

### A few words on TLVs

A *TLV* is an encoding used by Lightning for data exchange between nodes, which is able to represent structured (and nested) data.
Support for transmitting custom, application-level data has been allowed in the [specifications](https://github.com/lightningnetwork/lightning-rfc), by allowing them to be transmitted in [custom TLV record ranges](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#type-length-value-format) while respecting some basic requirements such as [*it's okay to be odd*](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#requirements).

c13n currently utilizes 3 (three) custom-range HTLC TLVs for payload transmission.
The `payload` containing the payload, the `sender` containing the sender's public key, and the `signature` containing the signature over the payload with the sending node's private key.
If the sender wishes to identify themselves[^gender], the `signature` and `sender` fields are transmitted along with the `payload`.

Since there is no standardization of TLVs on the custom range, an application should be aware of the specific types of interest to it and their encoding. As such, compatibility and retrieval of payloads on the receiving node requires either adherence to some common standards for transmission and data encoding or the presence of c13n over the receiving node as well.

The custom types used for data transmission in c13n are odd and are simply ignored by receiving nodes with [BOLT](https://github.com/lightningnetwork/lightning-rfc/blob/master/00-introduction.md#bolt-0-introduction-and-index)-compliant daemons.

### Payload TLV

c13n transmits all data on-medium, inside the fixed-size [onion packet](https://github.com/lightningnetwork/lightning-rfc/blob/master/04-onion-routing.md#packet-structure), which must also contain routing information necessary for the payment flow.
This means that there is a balance between the payload size and the route length.

The data sent by c13n are placed in the `payload` TLV. It actually is a serialized JSON containing the discussion participants list and the provided raw message string.

### Signing

The transmitted payloads are currently optionally signed with the node's private key so that the identity of the sender can be verified.
When enabled, the anonymity afforded to the payer by the medium is -- in a way -- defeated, but is the current way of identifying a sender on the network, which is important when data are involved.

> This project is still in its infancy, currently marked as `alpha`, meaning operation and compatibility are subject to change.

## Top-level c13n API functionality

As mentioned above, c13n creates higher level functionalities & concepts, and exposes them through its RPC API.

A brief overview of those functionalities is presented below:

### Messages & Payments

This is the core feature of c13n, you can easily send messages and/or payments to other network nodes. Each message/payment automatically triggers the calculation of an optimal path to the destination. "Estimating" a message, in terms of path selection and fees, is also possible through the API.

As a receiver, you can also open a listening stream from c13n for incoming payments/messages.

Each message contains the following main components.

- The message string
- An amount of msat

Populating the message string is optional, meaning that you can send a simple payment by specifying an empty payload.

You can also choose to send a message anonymously, meaning that the receiving party will not be able to verify the source of the message/payment.

Due to the LN payload size limitations, there is a maximum length for each message you send. This size limitation is due to a few parameters, which include:

- Payment route length
- Payload encoding
- Number of recipients (discussion participants)

Since the exact size of a message depends on the route length, it cannot be calculated until the attempt is made to send the it, but the limit is around ~1KB for an immediate neighbor node.

Last but not least, as of yet, messages are sent to **discussions**, which must be created before attempting to send one. This means that when you send a message you don't specify the recipient but instead reference the ID of an existing discussion.
Read the following section for more info about discussions.

### Discussions

You can create discussions with a set of participants (Lightning addresses, that is) and apply specific options to them. You can involve more than 2 people in a discussion, meaning that (in a rudimentary way) group chatting is supported.
Discussions offer a more organised solution for storing messages & payments, as these are related with the recipient address(es). There is a consistent history for each discussion, and each discussion is uniquely identified by the participant set.

A discussion contains the following information:

- The participant set (from which the user of c13n is excluded, since he is implicitly a participant)  
- The discussion options (such as fee limit per message)
- The ID of the last known message
- The ID of the last **read** message (meant for chatting applications: you can optionally update this field for cross-device discussion status)

It is up to you to utilize one, multiple, or all of a discussion's attributes.

> Note for group discussions: When you involve multiple addresses to a discussion, you basically create a group discussion. Our current implementation includes all the participants with each message. Appending many members to the discussion's participants list has the side effect of reducing the available message space.
> For example, for a discussion with ~15-20 participants and a route length of 1 hop, there is very little to no space left for writing messages.

### LN & Node information

You can use c13n to retrieve information about the Lightning Network and specific information about your own node. In detail, you can:

- Search for LN nodes based on their alias or address
- Retrieve a list of all network nodes visible from your point-of-view
- Connect to a node (as a peer)
- Get information about your node
- Get information about your wallet (only balance, no channel status supported yet)
- Get the version of c13n currently running (for compatibility / maintenance)

### Contacts (directory of addresses)

This is something optional you can use to store nodes found in the network. Making some address your "contact" means storing it in an exclusive list of nodes, and optionally applying a different "display name".

As a feature, it was introduced as a simple way of "renaming" nodes locally on your storage, but it can serve many other purposes.

### Openning Channels

You can open channels by using c13n (with advanced funding options being unsupported currently).
