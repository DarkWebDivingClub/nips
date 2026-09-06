NIP-XX
======

Nostr Node Control (NNC)
------------------------

`draft` `optional`

## Rationale

[NIP-47](47.md) (Nostr Wallet Connect) defines a protocol for clients to access a remote Lightning wallet — paying invoices, creating invoices, checking balances, and listing transactions. These are **wallet operations**: spending and receiving sats.

However, Lightning node owners and administrators need a separate set of operations to **manage their node**: opening and closing channels, connecting to peers, adjusting fee policies, inspecting the network graph, and monitoring routing activity. These operations do not belong in a wallet protocol because they change the node's topology and configuration rather than moving funds.

NNC (Nostr Node Control) fills this gap. It is a companion protocol to NWC, using the same architectural patterns (connection URI, encrypted request/response over relays, replaceable info event) but with a distinct set of methods for node administration.

## Terms

* **client**: Nostr app on any platform that wants to manage a Lightning node.
* **user**: The person using the **client**, typically the node owner or an authorized administrator.
* **node service**: Nostr app running alongside the Lightning node (or on the node itself) that translates NNC requests into node API calls. This is analogous to the NIP-47 **wallet service**.
* **controller**: The keypair a **client** uses to reach a **node service**. Access is granted to a controller, not to an app: a grant names `controller_pubkey`, and every request is authorized by the key that signed it. One user may hold several — a phone, a dashboard, a script — with different permissions.
* **owner**: The node operator who controls access grants. An owner's key is the only one whose grants a **node service** accepts. An owner need not be a controller, and a controller is not an owner.
* **method**: An operation a **node service** implements, named in a request's `method` field — `open_channel`, `get_channel_fees`, and so on. Methods are what this document defines.
* **command**: A request that calls a method. Every command is either **synchronous** or **asynchronous**, according to the method it calls: a synchronous command's response carries the result, an asynchronous command's response is an acknowledgement and the result follows as a notification.

## Theory of Operation

The connection flow mirrors NIP-47:

1. The **node service** publishes an NNC info event (kind `13198`) to its relay(s). The **owner** configures the **node service** with their pubkey and publishes access grants (kind `30198`) for authorized controllers.

2. The **user** imports the connection URI into their **client** (QR code, deeplink, or paste). The **client** fetches the info event to discover which NNC methods the **node service** supports.

3. When the **user** wants to perform a node operation (e.g. open a channel), the **client** creates an NNC request event (kind `23198`), encrypts the payload with [NIP-44](44.md), and publishes it to the relay(s) from the connection URI.

4. The **node service** decrypts the request, executes the operation against the Lightning node, and publishes an encrypted NNC response event (kind `23199`).

5. Most methods answer in that response. Two — `open_channel` and `close_channel` — cannot, because they wait on an on-chain confirmation: their response is an acknowledgement, and the outcome follows as an encrypted NNC notification event (kind `23200`). A **client** MAY set `"notify": false` to suppress it.

6. Separately from any command's outcome, a **controller** may publish a subscription (kind `30199`) naming notification types it wants — including events no controller initiated, such as a peer force-closing a channel.

These three patterns are defined in [Commands and Notifications](#commands-and-notifications).

## Commands and Notifications

A **command** calls a **method**. Every command is one of **two** kinds,
fixed by the method it calls and stated on every method below.

### Synchronous command

Request → response, and the response carries the result. Most methods are
synchronous: the **node service** can answer from what it already knows, or
complete the work before replying.

```
client ──request(23198)──▶ node service
client ◀─response(23199)── node service   result
```

### Asynchronous command

Request → response → notification. The work cannot finish inside a
request, so the response is an **acknowledgement** — the node service has
accepted the work — and the outcome follows later as a notification to the
controller that issued the command.

```
client ──request(23198)────▶ node service
client ◀─response(23199)──── node service   accepted
                            ...time passes, the operation completes...
client ◀─notification(23200)─ node service   outcome
```

**Only `open_channel` and `close_channel` are asynchronous methods.** Both
wait on an on-chain confirmation, which is why they cannot answer
synchronously.

Two rules keep the acknowledgement honest:

- **A success response means *accepted*, never *done*.** It does not say
  the channel exists.
- **Anything knowable immediately MUST be an error, not a deferred
  failure** — an unknown peer, insufficient funds, a malformed request. A
  controller can then tell *refused* from *in progress* from the response
  alone, without waiting for a notification that may never come.

Set `"notify": false` in the request to suppress the notification for that
operation. The operation still happens; only the message about it is
suppressed.

### Where notifications come from

A subscription is **not** a third kind of command. Notifications have
exactly two sources:

| Source | Sent to | Stops |
|---|---|---|
| an asynchronous command completing | the controller that issued it | when the operation completes — one notification |
| a **subscription** | every subscribed controller | on unsubscribe, or when the controller's grant is revoked |

The same notification type can arrive by either.

### Subscriptions

A subscription is **not a command**. It is durable state a controller
publishes for itself: a kind `30199` event naming the notification types it
wants, whatever caused them. See
[Subscriptions (kind 30199)](#subscriptions-kind-30199) for the event.

```
controller ──subscription(30199, ["channel_closed"])──▶ relay
node service ◀────────────────────────────────────────── reads it
client ◀─notification(23200)─────────────── node service   any matching event
client ◀─notification(23200)─────────────── node service   ...
controller ──subscription(30199, [])──────────────────▶ relay   cleared
```

There is no subscribe, resubscribe or unsubscribe method, because there is
nothing to distinguish between them: a subscription has one piece of state,
the event is addressable, and publishing a new one **replaces** it.

| From | The controller publishes | To |
|---|---|---|
| not subscribed | `["channel_closed"]` | subscribed to `channel_closed` |
| subscribed to `A` | `["B"]` | subscribed to `B` — **replaced, not merged** |
| subscribed | `[]` | not subscribed |
| — | *(the controller's grant is revoked or narrowed)* | not subscribed |

Making it an event rather than a command is what makes it survive. A
subscription established by a command lives in the node service's memory,
so a restart drops every subscription silently — and a controller watching
for a force-close cannot distinguish a dropped subscription from a quiet
node, which is the worst way for monitoring to fail. An addressable event
is re-read on startup exactly as grants are.

Replacement rather than accumulation is deliberate: a controller can state
what it wants without knowing what it previously asked for, so a
reconnecting client is one publication away from a known state rather than
having to reconcile. It also means the node service holds one bounded set
per controller and nothing that grows.

**Subscriptions are not a convenience for the asynchronous case.** Some
events follow from no command at all — a peer force-closing a channel, a
peer broadcasting a revoked state, a peer opening a channel to this node —
and a subscription is the only way to learn of them. A controller that
exists to monitor a node needs this source, not the first one.

## Events

There are six event kinds. Four carry traffic and are not stored; two hold
durable state and are addressable, so the latest replaces the previous.

| Kind | Event | Published by | Storage |
|---|---|---|---|
| **13198** | NNC info event | node service | replaceable |
| **23198** | NNC request | client | ephemeral |
| **23199** | NNC response | node service | ephemeral |
| **23200** | NNC notification | node service | ephemeral |
| **30198** | [access grant](#access-grants-kind-30198) | owner | addressable |
| **30199** | [subscription](#subscriptions-kind-30199) | controller | addressable |

The two addressable kinds are the protocol's only durable state, and they
are symmetric: an owner publishes what a controller **may** do, and a
controller publishes what it **wants** to be told about.

### Info Event (kind 13198)

The info event is a replaceable event published by the **node service** on the relay to indicate which NNC capabilities it supports.

The content SHOULD be a plaintext string with the supported methods space-separated:

```
list_channels open_channel close_channel list_peers connect_peer disconnect_peer get_channel_fees set_channel_fees get_forwarding_history get_pending_htlcs query_routes list_network_nodes get_network_stats get_network_node get_network_channel sign_message
```

### Request Event (kind 23198)

The request event SHOULD contain one `p` tag with the public key of the **node service**.

Optionally, a request can have an `expiration` tag with a unix timestamp in seconds. If the request is received after this timestamp, it should be ignored.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "method": "list_channels", // method name, string
    "params": {               // params, object
        // method-specific parameters
    }
}
```

### Response Event (kind 23199)

The response event SHOULD contain one `p` tag with the public key of the **user** and an `e` tag with the id of the request event it is responding to.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "result_type": "list_channels", // indicates the structure of the result field
    "error": {                      // object, non-null in case of error
        "code": "UNAUTHORIZED",     // string error code, see below
        "message": "human readable error message"
    },
    "result": {   // result, object. null in case of error.
        // method-specific result data
    }
}
```

The `result_type` field MUST contain the name of the method that this event is responding to.
The `error` field MUST contain a `message` field with a human readable error message and a `code` field with the error code if the command was not successful.
If the command was successful, the `error` field must be null.

### Notification Event (kind 23200)

A notification event reaches a controller by one or both of the two routes in [Commands and Notifications](#commands-and-notifications), and **it MUST name what caused it**.

- It MUST contain one `p` tag with the public key of the receiving **controller**.
- **As the deferred result of an asynchronous command**, it MUST contain an `e` tag with the id of the original request event. This is what lets a client tie an outcome to the command that caused it, rather than guessing from content — two `open_channel` commands to the same peer are otherwise indistinguishable. Suppressed by `"notify": false`.
- **As the delivery of a subscription**, it MUST contain an `a` tag whose value is `30199:<controller_pubkey>:<node_pubkey>`, naming the [subscription](#subscriptions-kind-30199) of the controller the notification is addressed to. A peer force-closing a channel follows from no command at all, but it does not follow from nothing: it follows from the subscription that asked to hear about it.
- **When one notification is both** — the controller issued the command *and* subscribed to the type — it MUST contain **both** tags. See [Delivery](#delivery).
- A notification containing **neither** tag is malformed. A client MUST ignore it, and SHOULD report the node service as non-conforming rather than discarding it silently.

The `a` tag MUST name the subscription of the controller in the same event's `p` tag, since a notification is addressed to one controller and encrypted to it. A client SHOULD ignore an `a` tag naming any other subscription.

> **Why the subscription route is a MUST and not merely "no `e` tag".**
> Both obligations are stated in their own right, so that an implementation
> has a sentence to satisfy on each route and a conformance claim has
> something to cite. The alternative — requiring the `e` tag and describing
> the subscription case as its absence — makes a **defect indistinguishable
> from a legitimate event**: a node service that fails to set the mandatory
> `e` tag emits something byte-identical to a subscription delivery, and a
> client cannot tell that anything is wrong. It waits for an outcome that
> has already arrived and been discarded, and reports only a timeout.
> Naming the cause positively on both routes turns that silence into a
> detectable error.

Because the `a` tag is derived per recipient, a node service fanning one event out to several controllers sends a separate notification to each, and each names that controller's own subscription:

```jsonc
// to a controller that issued the command and is also subscribed
"tags": [["p", "<controller>"], ["e", "<request_id>"], ["a", "30199:<controller>:<node>"]]

// to every other subscriber
"tags": [["p", "<other>"], ["a", "30199:<other>:<node>"]]
```

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "notification_type": "channel_opened", // indicates the structure of the notification field
    "notification": {
        // notification-specific data
    }
}
```

### Units

Follows the NIP-47 rule: every field whose value is denominated in sats MUST use the `_sats` suffix in its name; every amount-bearing field without a `_sats` suffix is in msats. On-chain physical values (funding tx amount, on-chain fee rates) take `_sats`; Lightning values (channel balances, capacity, HTLCs, forwarding fees, network-graph capacity) stay plain.

### Error Codes

- `RATE_LIMITED`: The client is sending commands too fast. It should retry in a few seconds.
- `NOT_IMPLEMENTED`: The method is not known or is intentionally not implemented.
- `RESTRICTED`: This public key is not allowed to do this operation.
- `UNAUTHORIZED`: This public key has no node connected.
- `QUOTA_EXCEEDED`: The controller has exceeded its spending quota.
- `NOT_FOUND`: The requested resource (channel, peer, node, etc.) was not found.
- `CHANNEL_FAILED`: A channel operation could not be completed — insufficient funds, the peer refused, or similar.
- `CONNECTION_FAILED`: A peer could not be reached.
- `INTERNAL`: An internal error.
- `OTHER`: Other error.

## NNC Connection URI

The **client** discovers the **node service** by scanning a QR code, handling a deeplink, or pasting in a URI.

The connection URI uses the protocol `nostr+nodecontrol://` and base path the hex-encoded `pubkey` of the **node service** with the following query string parameters:

- `relay` Required. URL of the relay where the **node service** is connected and will be listening for events. May be more than one.

The **client** uses its own key pair to sign events and encrypt payloads. The **owner** must publish an access grant (kind `30198`) for the **client**'s pubkey before it can issue requests.

### Example connection string

```sh
nostr+nodecontrol://b889ff5b1513b641e2a139f661a661364979c5beee91842f8f0ef42ab558e9d4?relay=wss%3A%2F%2Frelay.damus.io
```

## Methods

### Channel Management

#### `list_channels`

**Synchronous.** A command calling this method gets its result in the response.

Description: Lists the node's channels with their current status and balances.

Request:
```jsonc
{
    "method": "list_channels",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "list_channels",
    "result": {
        "channels": [
            {
                "id": "abc123",                // channel ID (implementation-specific)
                "short_channel_id": "800000x1x0", // short channel ID, optional
                "peer_pubkey": "02abc...",      // remote peer's pubkey
                "state": "active",             // "active", "inactive", "pending_open", "pending_close", "force_closing"
                "is_private": false,           // whether the channel is private
                "local_balance": 500000,       // local balance in msats
                "remote_balance": 500000,      // remote balance in msats
                "capacity": 1000000,           // total channel capacity in msats
                "funding_txid": "abc123...",    // funding transaction ID
                "funding_output_index": 0      // funding transaction output index
            }
        ]
    }
}
```

#### `open_channel`

**Asynchronous.** A command calling this method is acknowledged; the outcome arrives later as a `channel_opened` notification. See [Commands and Notifications](#commands-and-notifications).

Description: Opens a channel to a peer. If not already connected, the **node service** SHOULD connect to the peer first.

Request:
```jsonc
{
    "method": "open_channel",
    "params": {
        "pubkey": "02abc...",             // peer's pubkey, required
        "amount_sats": 1000000,          // channel capacity in sats, required
        "push_amount": 0,                // amount to push to peer in msats, optional, default 0
        "private": false,                // whether the channel should be private, optional, default false
        "host": "10.0.0.1:9735",         // peer's host:port, optional (for auto-connect)
        "close_address": "bc1q...",      // cooperative close address, optional
        "notify": true                   // send notification when the channel is confirmed, optional, default true
    }
}
```

Response:
```jsonc
{
    "result_type": "open_channel",
    "result": {}
}
```

The response confirms the channel open was accepted. The `funding_txid` is delivered asynchronously via the `channel_opened` notification (kind 23200) once the funding transaction is confirmed.

Errors:
- `CHANNEL_FAILED`: The channel could not be opened. This may be due to insufficient funds, peer refusing, or similar.

#### `close_channel`

**Asynchronous.** A command calling this method is acknowledged; the outcome arrives later as a `channel_closed` notification. See [Commands and Notifications](#commands-and-notifications).

Description: Closes a channel. Defaults to a cooperative close; set `force` to true for a force close.

Request:
```jsonc
{
    "method": "close_channel",
    "params": {
        "id": "abc123",                  // channel ID, required
        "force": false,                  // force close, optional, default false
        "close_address": "bc1q...",      // address to send funds to, optional
        "notify": true                   // send notification when the close is confirmed, optional, default true
    }
}
```

Response:
```jsonc
{
    "result_type": "close_channel",
    "result": {}
}
```

The response confirms the channel close was accepted. The `closing_txid` is delivered asynchronously via the `channel_closed` notification (kind 23200) once the closing transaction is confirmed.

Errors:
- `NOT_FOUND`: The channel was not found.
- `CHANNEL_FAILED`: The channel could not be closed.

### Peer Management

#### `list_peers`

**Synchronous.** A command calling this method gets its result in the response.

Description: Lists the node's connected peers.

Request:
```jsonc
{
    "method": "list_peers",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "list_peers",
    "result": {
        "peers": [
            {
                "pubkey": "02abc...",         // peer's pubkey
                "address": "10.0.0.1:9735",  // peer's address
                "connected": true,           // whether currently connected
                "alias": "ACINQ",            // peer's alias, optional
                "num_channels": 2            // number of channels with this peer
            }
        ]
    }
}
```

#### `connect_peer`

**Synchronous.** A command calling this method gets its result in the response.

Description: Connects to a peer.

Request:
```jsonc
{
    "method": "connect_peer",
    "params": {
        "pubkey": "02abc...",              // peer's pubkey, required
        "host": "10.0.0.1:9735"           // peer's host:port, required
    }
}
```

Response:
```jsonc
{
    "result_type": "connect_peer",
    "result": {}
}
```

Errors:
- `CONNECTION_FAILED`: Could not connect to the peer.

#### `disconnect_peer`

**Synchronous.** A command calling this method gets its result in the response.

Description: Disconnects from a peer.

Request:
```jsonc
{
    "method": "disconnect_peer",
    "params": {
        "pubkey": "02abc..."               // peer's pubkey, required
    }
}
```

Response:
```jsonc
{
    "result_type": "disconnect_peer",
    "result": {}
}
```

Errors:
- `NOT_FOUND`: The peer was not found.

### Fees & Routing

#### `get_channel_fees`

**Synchronous.** A command calling this method gets its result in the response.

Description: Gets the fee policies for the node's channels.

Request:
```jsonc
{
    "method": "get_channel_fees",
    "params": {
        "id": "abc123"                   // channel ID, optional (omit for all channels)
    }
}
```

Response:
```jsonc
{
    "result_type": "get_channel_fees",
    "result": {
        "fees": [
            {
                "id": "abc123",              // channel ID
                "short_channel_id": "800000x1x0", // short channel ID, optional
                "peer_pubkey": "02abc...",    // remote peer's pubkey
                "base_fee": 1000,            // base fee in msats
                "fee_rate": 1,               // proportional fee rate in millionths (ppm)
                "min_htlc": 1000,            // minimum HTLC size in msats, optional
                "max_htlc": 500000000        // maximum HTLC size in msats, optional
            }
        ]
    }
}
```

#### `set_channel_fees`

**Synchronous.** A command calling this method gets its result in the response.

Description: Updates the fee policy for a channel or all channels.

Request:
```jsonc
{
    "method": "set_channel_fees",
    "params": {
        "id": "abc123",                  // channel ID, optional (omit to apply to all channels)
        "base_fee": 1000,                // base fee in msats, optional
        "fee_rate": 1,                   // proportional fee rate in ppm, optional
        "min_htlc": 1000,                // minimum HTLC size in msats, optional
        "max_htlc": 500000000            // maximum HTLC size in msats, optional
    }
}
```

Response:
```jsonc
{
    "result_type": "set_channel_fees",
    "result": {}
}
```

Errors:
- `NOT_FOUND`: The channel was not found.

#### `get_forwarding_history`

**Synchronous.** A command calling this method gets its result in the response.

Description: Lists forwarded payments (routing events).

Request:
```jsonc
{
    "method": "get_forwarding_history",
    "params": {
        "from": 1693876973,              // starting timestamp in seconds (inclusive), optional
        "until": 1703225078,             // ending timestamp in seconds (inclusive), optional
        "limit": 50,                     // maximum number of forwards to return, optional
        "offset": 0                      // offset of the first forward to return, optional
    }
}
```

Response:
```jsonc
{
    "result_type": "get_forwarding_history",
    "result": {
        "forwards": [
            {
                "incoming_channel_id": "abc123",   // incoming channel
                "outgoing_channel_id": "def456",   // outgoing channel
                "incoming_amount": 10000,          // incoming amount in msats
                "outgoing_amount": 9990,           // outgoing amount in msats
                "fee_earned": 10,                  // fee earned in msats
                "settled_at": 1703225000           // timestamp in seconds
            }
        ]
    }
}
```

#### `get_pending_htlcs`

**Synchronous.** A command calling this method gets its result in the response.

Description: Lists in-flight HTLCs across channels.

Request:
```jsonc
{
    "method": "get_pending_htlcs",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "get_pending_htlcs",
    "result": {
        "htlcs": [
            {
                "channel_id": "abc123",        // channel this HTLC is on
                "direction": "incoming",       // "incoming" or "outgoing"
                "amount": 10000,               // HTLC amount in msats
                "hash_lock": "abcdef...",      // payment hash
                "expiry_height": 800100        // block height at which this HTLC expires
            }
        ]
    }
}
```

**Note:** Fee estimation methods (`estimate_onchain_fees` and `estimate_routing_fees`) are defined in [NIP-47](47.md) as wallet operations.

#### `query_routes`

**Synchronous.** A command calling this method gets its result in the response.

Description: Finds possible routes to a destination.

Request:
```jsonc
{
    "method": "query_routes",
    "params": {
        "destination": "02abc...",       // destination pubkey, required
        "amount": 100000,               // payment amount in msats, required
        "max_routes": 3                  // maximum number of routes to return, optional, default 1
    }
}
```

Response:
```jsonc
{
    "result_type": "query_routes",
    "result": {
        "routes": [
            {
                "total_fee": 150,            // total fee for this route in msats
                "total_time_lock": 40,       // total CLTV delta
                "hops": [
                    {
                        "pubkey": "02abc...",       // hop node pubkey
                        "short_channel_id": "800000x1x0", // channel used
                        "fee": 50,                  // fee for this hop in msats
                        "expiry": 20               // CLTV delta for this hop
                    }
                ]
            }
        ]
    }
}
```

Errors:
- `NOT_FOUND`: No route found to the destination.

### Network Graph

#### `list_network_nodes`

**Synchronous.** A command calling this method gets its result in the response.

Description: Lists nodes in the public network graph.

Request:
```jsonc
{
    "method": "list_network_nodes",
    "params": {
        "limit": 50,                     // maximum number of nodes to return, optional
        "offset": 0                      // offset of the first node to return, optional
    }
}
```

Response:
```jsonc
{
    "result_type": "list_network_nodes",
    "result": {
        "nodes": [
            {
                "pubkey": "02abc...",          // node pubkey
                "alias": "ACINQ",             // node alias, optional
                "color": "#3399ff",           // node color, optional
                "num_channels": 150,          // number of public channels
                "total_capacity": 50000000000, // total capacity across channels in msats
                "addresses": [                // network addresses, optional
                    "10.0.0.1:9735"
                ],
                "last_update": 1703225000     // last gossip update timestamp
            }
        ]
    }
}
```

#### `get_network_stats`

**Synchronous.** A command calling this method gets its result in the response.

Description: Gets aggregate statistics about the network graph.

Request:
```jsonc
{
    "method": "get_network_stats",
    "params": {}
}
```

Response:
```jsonc
{
    "result_type": "get_network_stats",
    "result": {
        "num_nodes": 15000,                   // total number of nodes
        "num_channels": 65000,                // total number of channels
        "total_capacity": 500000000000000,    // total network capacity in msats
        "avg_channel_size": 7692307692,       // average channel size in msats
        "max_channel_size": 1000000000000     // largest channel size in msats
    }
}
```

#### `get_network_node`

**Synchronous.** A command calling this method gets its result in the response.

Description: Gets information about a specific node in the network graph.

Request:
```jsonc
{
    "method": "get_network_node",
    "params": {
        "pubkey": "02abc..."               // node pubkey, required
    }
}
```

Response:
```jsonc
{
    "result_type": "get_network_node",
    "result": {
        "pubkey": "02abc...",              // node pubkey
        "alias": "ACINQ",                 // node alias, optional
        "color": "#3399ff",               // node color, optional
        "num_channels": 150,              // number of public channels
        "total_capacity": 50000000000,    // total capacity in msats
        "addresses": [                    // network addresses, optional
            "10.0.0.1:9735"
        ],
        "last_update": 1703225000,        // last gossip update timestamp
        "features": {}                    // feature bits, optional
    }
}
```

Errors:
- `NOT_FOUND`: The node was not found in the graph.

#### `get_network_channel`

**Synchronous.** A command calling this method gets its result in the response.

Description: Gets information about a specific channel in the network graph.

Request:
```jsonc
{
    "method": "get_network_channel",
    "params": {
        "short_channel_id": "800000x1x0"   // short channel ID, required
    }
}
```

Response:
```jsonc
{
    "result_type": "get_network_channel",
    "result": {
        "short_channel_id": "800000x1x0",     // short channel ID
        "capacity": 1000000000,                // channel capacity in msats
        "node1_pubkey": "02abc...",            // first node's pubkey
        "node2_pubkey": "03def...",            // second node's pubkey
        "node1_policy": {                      // first node's routing policy, optional
            "base_fee": 1000,
            "fee_rate": 1,
            "min_htlc": 1000,
            "max_htlc": 500000000,
            "time_lock_delta": 40,
            "disabled": false,
            "last_update": 1703225000
        },
        "node2_policy": {                      // second node's routing policy, optional
            "base_fee": 500,
            "fee_rate": 2,
            "min_htlc": 1000,
            "max_htlc": 500000000,
            "time_lock_delta": 40,
            "disabled": false,
            "last_update": 1703224000
        }
    }
}
```

Errors:
- `NOT_FOUND`: The channel was not found in the graph.

### Node Identity

#### `sign_message`

**Synchronous.** A command calling this method gets its result in the response.

Description: Signs a message with the node's **Lightning identity key** —
the `node_id` it announces to its peers. This is not the Nostr key the
node service signs its responses with, and the two are different lengths
and encodings: a `node_id` is a 33-byte compressed secp256k1 point,
serialised as 66 hex characters beginning `02` or `03`, while a Nostr
pubkey is a 32-byte x-only key with no prefix.

The distinction is the whole point of the method. Every response is
already signed with the service's Nostr key, so a message signed with
that key proves only that the service is itself. `sign_message` is how a
controller proves that the service really does front the Lightning node
it claims to — which is a node operator's concern, and why this is a
control method rather than a wallet one.

Request:
```jsonc
{
    "method": "sign_message",
    "params": {
        "message": "I am node 02abc..."   // message to sign, required
    }
}
```

Response:
```jsonc
{
    "result_type": "sign_message",
    "result": {
        "message": "I am node 02abc...",  // the message, echoed
        "signature": "d9tibmnic9t5...",   // zbase32, see below
        "pubkey": "02abc..."              // the signing node_id, hex
    }
}
```

Errors:
- `INTERNAL`: The node's signer is unavailable.

##### Signature scheme

Implementations MUST produce the signature as:

```
signature = zbase32(SigRec(sha256d("Lightning Signed Message:" || message)))
```

A 65-byte recoverable ECDSA signature over the double-SHA256 of the
message with the fixed ASCII prefix `Lightning Signed Message:`
prepended, encoded in [zbase32](https://philzimmermann.com/docs/human-oriented-base-32-encoding.txt).

This is not a new scheme. It is the one LND's `signmessage`, Core
Lightning's `signmessage` and LDK's `lightning::util::message_signing`
all implement, and their signatures verify against each other. Naming it
here is what makes a signature produced by one node service checkable by
a verifier that has never heard of NIP-XX — the alternative is a
`signature` field whose contents differ per implementation, which proves
nothing to anybody.

##### The prefix is not optional

`message` is chosen entirely by the caller, so without domain separation
this method would sign arbitrary caller-supplied bytes with the node's
identity key. The prefix is what confines a signature obtained here to
this use, and stops it being replayed as a signature over something else
that key is asked to sign. Implementations MUST NOT omit it, and MUST NOT
offer a variant that signs the raw message.

##### Verifying

The signature is pubkey-recoverable: a verifier recovers the signing key
from `signature` and `message`, and compares it against the `node_id` it
expected to hear from. `pubkey` is returned for convenience only.
Verifiers MUST NOT accept it in place of that comparison — a response
that simply names the key it wishes it were is not a proof.

## Notifications

### Notification Model

A notification is a kind `23200` event. It reaches a controller by one of
**two independent routes**, defined in
[Commands and Notifications](#commands-and-notifications): as the outcome of an
**asynchronous command** it issued, or through a **subscription**. The same
notification type can arrive by either.

#### Delivery

- A controller that both initiated an operation and subscribed to its type
  receives **one** notification for it, not two — and that one notification
  carries **both** an `e` tag and an `a` tag, because it has two causes and
  [MUST name them](#notification-event-kind-23200).

  Carrying both is what makes one delivery sufficient. A node service does
  not have to notice that a caller is also a subscriber and suppress
  something, and a client is not left to guess which route a single tag
  meant. Both of a client's mechanisms may act on it: the handle awaiting
  that command resolves, **and** the subscription stream receives it. Those
  are ordinarily two different consumers in the same application — a
  dashboard's button stops spinning, and its channel list updates — and
  neither should have to know the other exists.
- An event with no initiating call is delivered to subscribers only, with
  an `a` tag and no `e` tag.
- `"notify": false` suppresses the asynchronous-command route for that
  operation. It does **not** suppress a subscription: a controller that
  subscribed to a type has asked to see every event of that type. The two
  together are a coherent request — *do not send me a correlated result, I
  will see it on my subscription* — and produce one notification carrying
  an `a` tag and no `e` tag.
- **A subscription lives and dies with the grant that permits it.** A
  subscription event grants nothing by itself: it states what a controller
  *wants*, and the grant states what it *may have*. Delivery is the
  intersection. When a grant is revoked or narrowed, delivery stops at once
  and the subscription event — which the node service does not own and
  cannot delete — becomes inert. A subscription that outlived its grant
  would be a revocation that does not revoke.

Payment-related notifications (`payment_received`, `payment_sent`) belong
to NIP-47 (NWC).

### `channel_opened`

Description: Sent when a channel has been confirmed on-chain and is now active. Delivered to the controller whose `open_channel` request opened it, and to any controller subscribed to this type — including for a channel a peer opened to this node, which no controller requested.

Notification:
```jsonc
{
    "notification_type": "channel_opened",
    "notification": {
        "id": "abc123",                    // channel ID
        "short_channel_id": "800000x1x0",  // short channel ID, optional
        "peer_pubkey": "02abc...",          // remote peer's pubkey
        "capacity": 1000000,               // total channel capacity in msats
        "local_balance": 1000000,          // local balance in msats
        "remote_balance": 0,               // remote balance in msats
        "funding_txid": "abc123...",        // funding transaction ID
        "is_private": false                // whether the channel is private
    }
}
```

### `channel_closed`

Description: Sent when a channel close has been confirmed on-chain. Delivered to the controller whose `close_channel` request closed it, and to any controller subscribed to this type — including for a close no controller requested, which `close_type` distinguishes: a peer's force-close is `force_remote`, and a revoked-state broadcast is `breach`.

Notification:
```jsonc
{
    "notification_type": "channel_closed",
    "notification": {
        "id": "abc123",                    // channel ID
        "short_channel_id": "800000x1x0",  // short channel ID, optional
        "peer_pubkey": "02abc...",          // remote peer's pubkey
        "capacity": 1000000,               // total channel capacity in msats
        "closing_txid": "def456...",        // closing transaction ID
        "close_type": "cooperative"        // "cooperative", "force_local", "force_remote", "breach"
    }
}
```

## Encryption

All NNC request, response, and notification payloads MUST be encrypted using [NIP-44](44.md). The encryption uses the **client**'s private key and the **node service**'s public key.

### Access grants are not encrypted

**Kind `30198` access grant content is plaintext**, and this is stated so
that it is a decision rather than an omission.

The consequence is real and should be understood before publishing one: a
node's authorization policy is public. Anyone reading the relay learns
which keys may administer the node, which methods each may call, and what
each may spend — which is also a map of which controller key is worth
stealing and what it is worth. Some of that leaks regardless, since tags
are never encrypted and the `d` tag is `node_pubkey:controller_pubkey`, but
today the profile leaks too.

#### Why it is deferred

Not because it is unsolved. A construction that needs no new cryptography
is specified in
[Encrypting Content for Multiple Readers](#encrypting-content-for-multiple-readers)
below — it is simply not built yet, and shipping an unimplemented
encryption requirement would mean every existing consumer silently failing
to read a grant it must enforce.

A grant has **two** natural readers: the **node service**, which enforces
it, and the **controller**, which is entitled to know its own limits.
NIP-44 is strictly pairwise — one ECDH, one conversation key, one reader —
so it cannot address both, which is what that section exists to solve.

When it is adopted, a grant's readers are the **node service**, the
**controller** it names, and the **owner** that wrote it — the last so that
an owner can read back what it published.

## Encrypting Content for Multiple Readers

> **Status: specified, not yet used.** No event kind in this document
> requires this today. It is written here because access grants will want
> it, and written **generically** — about authors, readers and content,
> with nothing specific to node control — because it is intended to be
> lifted into a NIP of its own once it has an implementation and tests.
> Implement and test it separately from the rest of this specification.

### The problem

[NIP-44](44.md) derives one conversation key from one ECDH, so a payload
has exactly one author and one reader. Content that two or more parties
must read has no standard answer: [NIP-59](59.md) gift wrap addresses each
recipient with a **separate copy**, and the MLS-based [NIP-EE](EE.md) is
marked `unrecommended` and superseded.

Separate copies are not equivalent to one shared payload. Copies can
disagree — an author may encrypt different plaintext to different readers,
by accident or design — and each copy is separately replaceable, so an
update can land for one reader and not another.

### The construction

Encrypt the content **once** under a one-time key, then give that key to
each reader. Both steps use NIP-44 unchanged; no new cipher, padding
scheme or nonce handling is introduced.

**To publish**, an author with keypair `(a, A)` and readers `R₁…Rₙ`:

1. Generate a one-time keypair `(t, T)` with a CSPRNG. It MUST be fresh
   for this publication and used for nothing else.
2. Encrypt the content with NIP-44 from `a` to `T`. This is the event's
   `content`.
3. For each reader `Rᵢ`, encrypt the one-time secret `t` — serialised as
   **64 lowercase hexadecimal characters** — with NIP-44 from `a` to `Rᵢ`,
   and place the result at the **fourth** position of a `p` tag naming
   `Rᵢ`.
4. Discard `t`. Retaining it serves no purpose and extends the window in
   which it can be stolen.

**To read**, a reader with keypair `(r, R)`:

1. Find the `p` tag whose second element is `R`. If it has no fourth
   element, this reader is not addressed.
2. Decrypt that element with NIP-44 from `r` to the event's `pubkey`,
   recovering the hexadecimal `t`. A payload that is not 64 hexadecimal
   characters encoding a valid secret key is a failure, not a key.
3. Decrypt `content` with NIP-44 from `t` to the event's `pubkey`.

Step 3 works because NIP-44 conversation keys are symmetric — the
specification states `conv(a, B) == conv(b, A)` — so `conv(t, A)` is the
same key the author used as `conv(a, T)`. The one-time **public** key
therefore never has to appear in the event.

#### Why hex, and not raw bytes or `nsec`

The wrapped key is **hex**, and both alternatives are ruled out rather than
merely disfavoured.

**Not raw bytes.** Thirty-two raw key bytes are not valid UTF-8. Many
NIP-44 implementations expose only `encrypt(sk, pk, string) -> string` and
`decrypt(...) -> string`, with no bytes-level entry point, and cannot carry
such a payload at all. A construction that only some libraries can express
is not one that gets adopted.

**Not `nsec`.** [NIP-19](19.md) says of its bech32 forms that they are
*"not meant to be used anywhere in the core protocol"* and *"not meant to
be used inside the standard NIP-01 event formats"* — they exist for
display, copy-paste and QR codes.

**Hex is what Nostr already uses for keys inside events** — NIP-01
serialises a `p` tag's pubkey as 32 bytes of lowercase hex — so a wrapped
secret in the same encoding needs no explanation.

The cost is 32 additional plaintext bytes per reader. Under NIP-44's
padding a 32-byte plaintext pads to 32 and a 64-byte plaintext to 64, so a
wrapped key grows from roughly 132 to roughly 176 base64 characters: once
per reader, per publication.

### Tag format

A `p` tag is `["p", <pubkey>, <relay hint>, ...]`. [NIP-01](01.md) states
that all elements after the second have no conventional meaning, and that
**only a tag's first value is indexed** — so the wrapped key adds nothing
to any relay index, and `#p` filtering still finds the event for every
reader.

```jsonc
"tags": [
    ["p", "<reader 1 pubkey>", "", "<one-time secret, NIP-44 from the author to reader 1>"],
    ["p", "<reader 2 pubkey>", "", "<one-time secret, NIP-44 from the author to reader 2>"]
]
```

The wrapped key MUST be at the fourth position. The third is a recommended
relay URL by NIP-01 convention, and a key placed there would be read as
one; leave it empty if there is no hint to give.

### Requirements

- The one-time keypair MUST be generated with a CSPRNG and MUST be fresh
  for every publication. **Reusing it across updates of a replaceable
  event means a reader removed by a later version still holds a key that
  opens it**, and removal stops meaning anything.
- An author that needs to read its own content back MUST list itself as a
  reader. The one-time public key appears nowhere in the event, so an
  author that discarded `t` and did not wrap it to itself cannot decrypt
  what it published.

  **Implementations should check that their event-building library keeps
  that tag.** Some remove a `p` tag matching the event's own author by
  default — `nostr-sdk`'s `EventBuilder` does, and offers
  `allow_self_tagging()` to opt out. The removal is silent, so a conforming
  sealed payload becomes a non-conforming event with nothing to indicate
  it. This was found by running the construction against a relay, not by
  reading anything.
- A reader that cannot decrypt content it requires MUST NOT proceed as
  though the content were absent or permissive. Where the content carries
  authorization, undecryptable MUST be treated as denied.
- Implementations MUST NOT infer anything from a `p` tag with no fourth
  element beyond "not addressed to this reader": the same event may
  address others.

#### It must work with a remote signer

Not a nicety. Where this construction is used for access grants, the author
is the **owner** — whose key is the only one a node service accepts grants
from, and therefore the key most likely to be held on separate hardware or
behind [NIP-46](46.md). An implementation that requires the author's secret
key in the process doing the sealing excludes exactly the deployment that
most wants encrypted grants.

The construction supports it, and an implementation should be checked
against a signer rather than assumed to work with one. Only the wrapped
keys need the author's identity:

| Operation | Needs the author's key |
|---|---|
| encrypt the content | **no** — see below |
| wrap the one-time secret to each reader | yes |
| unwrap one's own copy | yes, the reader's |
| decrypt the content | no — the one-time secret is local |

The content is encrypted under `conv(a, T)`, which by NIP-44's own symmetry
equals `conv(t, A)` — computable from the one-time secret, which the author
holds locally by construction. So an identity that offers only
"encrypt to X" and "decrypt from X", as a remote signer does, is
sufficient.

Note also that such a signer is typically **string-only**, as
`NostrSigner::nip44_encrypt` is. That is the second reason the wrapped key
is hex rather than raw bytes, and it means anything crossing the identity
boundary must be text while local operations may use bytes.

### What it gives, and what it does not

- **Every reader provably receives the same plaintext**, because there is
  one ciphertext. This is the property separate copies cannot offer.
- **The wrapped keys are authenticated.** NIP-01 serializes
  `[0, pubkey, created_at, kind, tags, content]`, so tags are inside the
  signed hash: a wrapped key cannot be substituted without breaking the
  signature.
- **Adding a reader is adding a `p` tag**, not a format change.
- **Replaceable and addressable events keep their atomicity** — one event,
  one `created_at`, one replacement.
- **No forward secrecy.** A reader's long-term key being compromised
  exposes every one-time key ever wrapped to it, and so all content it
  could ever read. This is true of every NIP-44 payload, but it is the
  reason this is a mitigation rather than a solution.
- **The reader set is public.** Tags are never encrypted, so who may read
  is visible even though what they read is not.
- **Removal requires republishing.** A reader is removed by publishing a
  new version with a fresh one-time key and without that reader's tag.
  The old event, and the key it carried, remain wherever they were stored.

### Known question for review

NIP-44 says encrypted payloads "MUST be included in an event's payload,
hashed, and signed as defined in NIP-01". Read strictly, *payload* means
the `content` field, and this construction places NIP-44 payloads in tags.
The security intent is met — they are hashed and signed, as shown above —
but the wording does not anticipate it, and a NIP proposing this should
say so rather than leave it to be noticed.

### Testing

This is a self-contained mechanism and SHOULD be implemented and tested on
its own, independently of any event kind that uses it. A conforming
implementation should be tested for at least:

- round trip with one, two and three readers
- a party not listed as a reader cannot decrypt the content
- every reader recovers **identical** plaintext
- an author listed as a reader can decrypt its own published content
- an author *not* listed as a reader cannot
- a reader removed in a later publication cannot read the later event
- a `p` tag with no fourth element is treated as "not addressed", not as
  an error, when other readers are addressed
- a fourth element that this reader cannot decrypt is a failure, not a
  fallback to plaintext
- the one-time key differs between two publications of the same content
- a wrapped key that decrypts to something other than 64 hexadecimal
  characters is rejected rather than coerced

## Access Control

This section defines the access control model for both NWC (NIP-47) and NNC methods. A single **node service** MAY serve both protocols and SHOULD use this unified model.

### Access Grants (kind 30198)

The **owner** publishes addressable events of kind `30198` to grant access to controllers.

- The event `pubkey` is the **owner**'s pubkey.
- The event `content` is a JSON-encoded `UsageProfile`. It is **not encrypted** — see [Access grants are not encrypted](#access-grants-are-not-encrypted), which is a deliberate deferral and not an oversight.
- The event includes a `d` tag whose value is `node_pubkey:controller_pubkey`.
- The event `tags` MUST include a `p` tag with the **node service**'s pubkey so it receives the event.
- The event `tags` MAY include auxiliary metadata (e.g., label, scope, or policy identifiers).
- The event is signed by the **owner** and published to the relay(s) from the connection URI.
- Subsequent updates use the same kind, pubkey, and `d` tag; the newest `created_at` replaces earlier grants.

#### Why not kind 30078

Earlier drafts used kind `30078`, [NIP-78](78.md)'s arbitrary application
data. That was wrong for two reasons, and the second is not theoretical.

NIP-78 states its own scope: *"These kinds are not meant to be used as a
generic interchange format for data that should be public or exchanged
between different applications. Applications that need such
interoperability should use dedicated kinds instead."* A grant is written
by an owner's tool and read by an independent node service, which is
exactly the case it excludes.

More seriously, NIP-78 says relays *"SHOULD only serve these events to the
authenticated owner, i.e. the client must authenticate with the same pubkey
as the event author."* A grant's author is the **owner**; the reader that
must have it is the **node service**. On a relay that implements NIP-78 as
written, a node service can never fetch its own grants, and authorization
fails closed for reasons nothing in this protocol can diagnose. Using
`30078` meant depending on relays *not* implementing it correctly.

#### The `OTHERS` controller

A grant MAY use the literal `OTHERS` in place of a controller pubkey:

```
d = <node_pubkey>:OTHERS
```

This is a grant like any other; what differs is who it names. It applies
to **every controller not named by a grant of its own**, and a grant
naming a specific pubkey takes precedence over it. A controller matched
by neither is denied with `UNAUTHORIZED`, exactly as before.

Its limits are instantiated **per controller**: each unknown key gets its
own buckets of that size. A single shared bucket would let one caller
starve every other.

This is what makes a service open to callers the owner has never seen —
a public faucet, a demo node, a test service.

Two things about it are worth knowing before using it. Neither is a
prohibition; an owner who wants either can have it.

- **Per-controller limits under `OTHERS` bound nothing on their own.**
  Nostr keys are free, so a caller can hold as many allowances as it
  cares to generate, and `quota` under `OTHERS` is advisory against
  anyone who notices. The only limit that binds is an aggregate one
  across all `OTHERS`-derived access, and a node service offering
  `OTHERS` SHOULD enforce one. It belongs to the node rather than to a
  grant, so that raising it is a deliberate act by whoever runs the node
  rather than a side effect of publishing.
- **`OTHERS` may name `control` methods**, and that grants node
  administration to callers the owner has never seen — including
  whichever of them arrives first. It is permitted because the owner may
  have a reason; it is rarely what is meant.

Revocation is unchanged and composes with this. A specific controller is
denied by publishing an empty grant for it — an explicit entry beats
`OTHERS`, so an empty grant is a deny-list entry.

### Subscriptions (kind 30199)

A **controller** publishes addressable events of kind `30199` to say which
notification types it wants to receive.

- The event `pubkey` is the **controller**'s pubkey. A controller
  subscribes for itself and for nobody else; there is no third party to
  name, which is why the `d` tag is shorter than a grant's.
- The event includes a `d` tag whose value is `node_pubkey`.
- The event `tags` MUST include a `p` tag with the **node service**'s
  pubkey so it receives the event.
- The event `content` is a JSON array of notification type names, e.g.
  `["channel_opened", "channel_closed"]`. An empty array subscribes to
  nothing.
- Subsequent updates use the same kind, pubkey and `d` tag; the newest
  `created_at` replaces earlier subscriptions.

```jsonc
{
    "kind": 30199,
    "pubkey": "<controller_pubkey>",
    "tags": [
        ["d", "<node_pubkey>"],
        ["p", "<node_pubkey>"]
    ],
    "content": "[\"channel_closed\"]"
}
```

A node service MUST verify that the `d` tag names itself before applying a
subscription, exactly as it does for a grant. It MUST NOT accept a
subscription whose author differs from the controller it would subscribe —
the author *is* the subscriber, so there is nothing else it could mean.

**A subscription authorizes nothing.** It states a want. What a controller
may actually receive is in its grant, and delivery is the intersection of
the two. A controller with no grant, or whose grant permits no notification
types, receives nothing however it subscribes — deny by default, as
everywhere else. There is no response and therefore no error: an
unauthorized subscription is silent, not refused.

A node service SHOULD NOT hold subscription state for a controller with no
grant. Anyone may publish a kind `30199` event naming any node, so a
registry keyed by whoever published one is unbounded; keyed by controllers
that hold grants, it is bounded by the grants the owner wrote.

### UsageProfile JSON

The `UsageProfile` defines per-controller permissions and limits. All numeric values are unsigned integers.

```json
{
  "methods": {
    "get_info": {},
    "get_balance": {
      "rate": {
        "amount": 1000,
        "per_secs": 3600,
        "max_capacity": 1000
      }
    },
    "pay_invoice": {}
  },
  "control": {
    "connect_peer": {},
    "open_channel": {},
    "close_channel": {},
    "list_channels": {}
  },
  "notifications": {
    "channel_opened": {},
    "channel_closed": {}
  },
  "quota": {
    "amount": 100000,
    "per_secs": 86400,
    "max_capacity": 1000000
  }
}
```

Fields:

- `methods` (object, optional): Map of NWC method name to a `MethodAccessRule`.
    - Missing or empty `methods` means no NWC permissions are granted.
    - The special key `"OTHERS"` supplies the rule for every NWC method **not named** in the map. An explicit entry always takes precedence over `"OTHERS"`.
    - Its rate limit is instantiated **per method**: each unnamed method gets its own bucket of that size, not a shared one.
    - A method MUST be explicitly present, or `"OTHERS"` MUST be present, to be allowed.
- `control` (object, optional): Map of NNC method name to a `MethodAccessRule`.
    - Missing or empty `control` means no NNC permissions are granted.
    - The special key `"OTHERS"` supplies the rule for every NNC method **not named** in the map, with the same precedence and per-method instantiation as in `methods`.
    - A method MUST be explicitly present, or `"OTHERS"` MUST be present, to be allowed.
- `methods.<method>.rate` (object, optional): Per-method rate limit. If missing, no rate limit is applied. **One token is one call.**
    - `amount` (u64, optional): Tokens added per period. Default `0`.
    - `per_secs` (u64, optional): The period, in seconds. Default `1`. MUST be greater than `0` — see *Invalid values*.
    - `max_capacity` (u64, optional): Maximum token capacity. Default `u64::MAX`.
- `control.<method>.rate` (object, optional): Per-control-method rate limit. If missing, no rate limit is applied. **One token is one call.**
    - `amount` (u64, optional): Tokens added per period. Default `0`.
    - `per_secs` (u64, optional): The period, in seconds. Default `1`. MUST be greater than `0` — see *Invalid values*.
    - `max_capacity` (u64, optional): Maximum token capacity. Default `u64::MAX`.
- `notifications` (object, optional): Map of notification type name to an empty object, naming the types this controller may receive. Absent or empty permits **none**. `OTHERS` may be used as a key, with the same meaning as elsewhere: it covers every type not named.

    This is what a kind `30199` subscription is intersected with. A controller receives a notification type only if it appears here *and* in that controller's subscription — except for the deferred outcome of an asynchronous command it issued itself, which needs no entry: a controller permitted to call `open_channel` is permitted to be told how it went.

- `quota` (object, optional): Controller-wide spending quota. If missing, no quota is applied. **One token is one satoshi.**
    - `amount` (u64, optional): Satoshis added per period. Default `0`.
    - `per_secs` (u64, optional): The period, in seconds. Default `1`. MUST be greater than `0` — see *Invalid values*.
    - `max_capacity` (u64, optional): Maximum quota capacity, in satoshis. Default `u64::MAX`.

#### Invalid values

A grant arrives over a relay and may say anything. Every value a node
service reads is therefore input, not configuration, and the spec says
what to do rather than leaving each implementation to decide.

**`per_secs` must be greater than `0`.** Zero is a division by zero in
the refill, and there is no sensible value to substitute: treating it as
`1` invents a rate nobody asked for, and treating it as "no refill"
silently converts a rate limit into a one-off allowance.

The constraint is stated on the **value**, not left to the type. JSON
carries no unsigned integers, and an implementation is free to hold these
fields in a signed type — so `-1` may reach a parser that `u64` would
have refused. Every numeric field in a `UsageProfile` is a non-negative
integer, and a value that is negative, fractional, or too large for the
implementation's type is invalid on the same terms as `per_secs = 0`.

An implementation MUST reject such a profile as it would any
unparseable one — Authorization step 2 — and deny with `UNAUTHORIZED`. It
MUST NOT fall back to a default, and MUST NOT leave an older, more
generous grant in force: a grant that cannot be understood grants
nothing.

Values that are **valid**, and should not be rejected:

- `amount = 0` — a one-off allowance, defined below.
- `max_capacity = 0` — an allowance of nothing, which denies every
  request. This is a legitimate way to say "permitted, but not yet", and
  is distinct from omitting the method, which is `RESTRICTED`.

#### What the quota counts

> **Known defect, not yet fixed.** What follows describes the quota as
> currently specified, and it under-counts: the node spends `amount + fee`
> while only `amount` is checked and recorded, on **every** spending call.
> The check must instead be **absolute** — taken against the full cost,
> before anything moves — which the pipeline cannot express today, because
> the true cost exists only between step 4 and step 6. See
> [nostr-ln#1](https://github.com/DarkWebDivingClub/nostr-ln/issues/1).


`quota` is a bucket like any other, and **one token is one satoshi**. What
the specification must say, and until now did not, is *how many satoshis a
given request costs* — because a limit whose cost is undefined is not a
limit.

**A request costs the amount it sends. Fees are not counted.**

| Protocol | Methods that cost | The cost |
|---|---|---|
| NNC (this document) | `open_channel` | `amount_sats`, converted to satoshis |
| NWC ([NIP-47](47.md)) | the paying methods — `pay_invoice`, `pay_keysend`, `pay_onchain` and their kin | the amount sent |
| both | everything else | **zero** |

Fifteen of NNC's sixteen methods cost nothing. `close_channel` returns
funds rather than spending them, and the reading methods move nothing at
all.

##### Why fees are excluded

**Not because a fee is unknowable.** It is: `query_routes` in this document
returns `total_fee` before anything is sent, and implementations take fee
limits — LND's `--fee_limit`, LDK's `max_total_routing_fee_msat`. What
varies is the *exact* fee, which depends on the route finally taken and can
differ across retries; an upper bound is available throughout.

The reason given here is that **the request carries no bound to check
against** — which is true of the request and **irrelevant to the node**,
since the node picks a feerate before funding and computes a route before
paying. That is why this is recorded above as a defect rather than a
design: the check belongs where the node knows the total, not where the
request happens to carry a number.
NIP-47's `PayInvoiceRequest` is an invoice and an optional amount, with no
fee-limit field, so a node service has nothing in the request from which to
derive a fee-inclusive cost at pipeline step 4. Counting fees would
therefore mean guessing a reserve, or committing at step 7 a figure larger
than step 4 approved.

That is a gap in the paying methods rather than a fact about Lightning, and
it could be closed: a request carrying a fee limit would make
`amount + max_fee` a real upper bound, checkable before execution and never
exceeded. Until such a field exists, the quota counts what is actually
knowable from a request.

Denominating it in what a controller **sends** is also what a controller
can reason about: *"I may send one coin a week"* is a statement they can
act on, where *"I may cost the node one coin a week"* depends on routing
they do not choose.

##### A quota therefore cannot be exceeded

This is the property the rule exists to produce, and implementations should
treat losing it as a bug rather than an edge case:

- The cost is a function of the **request**, so it is known at step 4.
- Step 4 checks that cost without mutating.
- Step 7 commits **the same cost**, and only if execution succeeded.

There is no estimate, no reserve, and no settlement against a different
number — so there is no path by which a committed spend exceeds what was
checked. An implementation that computes its step 7 amount from the
*response* has reintroduced the problem, and will eventually need to record
a spend larger than the bucket holds.

A node that wishes to bound what a controller costs it in fees needs a
separate control; the per-controller quota is not it.

#### How a bucket behaves

A bucket holds tokens. It starts full, at `max_capacity`, when the grant
is applied.

Refill is **continuous**, not periodic: a bucket does not pay out in a
lump at each period boundary. At any instant it holds

```
min(balance + amount × elapsed_secs / per_secs, max_capacity)
```

where `balance` is what it held at `since`, and `elapsed` is the time
from `since` to now. `max_capacity` is therefore what separates a rate
limit from an allowance: set it equal to `amount` and a controller can
never bank more than one period's worth; set it higher and unused
allowance accumulates up to that ceiling.

**Checking a limit MUST NOT mutate the bucket**, and **consuming MUST
happen only after the request has succeeded** — pipeline steps 4 and 7.
A refused request, or one whose execution failed, leaves every counter
untouched, so a caller is never charged for something they did not
receive.

**When tokens are consumed, `since` advances to that instant, and it
advances at no other time.** Leaving it behind causes the interval before
each withdrawal to be credited again at every later check, so the bucket
refills faster than its rule specifies. Advancing it on a mere check
discards the truncated remainder of the division and the bucket leaks.

Implementations should note that `amount × elapsed` can exceed 64 bits
for a large allowance over a long interval, and compute the refill in a
wider type.

#### Worked example

```jsonc
{
  "methods": {
    "pay_invoice": { "rate": { "amount": 5, "per_secs": 604800, "max_capacity": 5 } },
    "OTHERS":      { }
  },
  "control": {
    "list_channels": { }
  },
  "quota": { "amount": 100000, "per_secs": 604800, "max_capacity": 100000 }
}
```

| Request | Outcome |
|---|---|
| `pay_invoice` | allowed, five a week — the explicit entry wins |
| `get_balance` | allowed, unlimited — falls to `OTHERS`, which carries no rate |
| `list_channels` (NNC) | allowed |
| `open_channel` (NNC) | denied, `RESTRICTED` — `control` has no `OTHERS` |

The two maps are independent. Being permitted to spend does not make a
controller an administrator, and an `OTHERS` in `methods` says nothing
about `control`.

Defaults:

- Missing numeric fields use their defaults (`amount = 0`, `per_secs = 1`, `max_capacity = u64::MAX`).
- `amount = 0` means the bucket never refills: the controller receives `max_capacity` tokens once and no more until a new grant is published. An implementation MUST NOT offer a retry time for it, because none will arrive.

**A limit that is absent and a limit of zero are opposites**, and they are
easily confused because they sit next to each other:

| `rate` | meaning |
|---|---|
| absent | **no limit at all** — the method may be called without restriction |
| `{ "amount": 0, "max_capacity": 1000 }` | **1000 tokens, ever** — the strictest bound expressible |
| `{ "amount": 5, "per_secs": 604800, "max_capacity": 5 }` | five per week, banking at most five |

Omitting a limit is permissive; a zero refill rate is the most
restrictive setting there is. An implementation that treats a missing
`rate` as "no allowance" is as wrong as one that treats `amount = 0` as
"unlimited", and both mistakes are silent.

A non-refilling bucket is a supported configuration rather than a
degenerate one. It expresses a one-off allowance — a fixed budget for a
task, a bounded trial, a grant that should be topped up deliberately
rather than by the passage of time. `per_secs` is unused in that case but
MUST still be valid, since an implementation may compute the refill
before noticing that `amount` is zero.
- Missing optional objects are treated as absent limits.

### Revocation

To revoke a controller's access:

1. Publish a grant with empty `methods` `{}` and empty `control` `{}`. This immediately removes all permissions.
2. Publish a kind `5` deletion event referencing the grant event id. This cleans up the grant from relays.

Step 1 ensures immediate revocation regardless of whether the relay processes the deletion. Step 2 is housekeeping — relay deletion propagation is not guaranteed, but the empty grant from step 1 already denies all access.

### Request Handling Pipeline

1. **Decode**: Decrypt the event (NIP-44) and parse the JSON request.
2. **Resolve Method**: Check that the method name is known. If not, return `NOT_IMPLEMENTED`.
3. **Authorize**: Check whether the controller pubkey is permitted to call the method. Check `methods` (NWC) or `control` (NNC) in the `UsageProfile`.
4. **Enforce Limits**: Apply rate and quota checks using the controller's access state. Limits are evaluated without mutating state.
5. **Validate**: Validate the request parameters for the given method. If validation fails, return an error response.
6. **Execute**: Dispatch the request to the node.
7. **Commit Usage**: If execution succeeds, apply the rate/quota usage to state. Usage is committed only after the request is accepted for execution, and the quota amount committed MUST be the one checked at step 4 — not a figure recomputed from the response, which would allow a committed spend to exceed what was authorized.

### Authorization Steps

1. Resolve the grant that applies to the caller: the latest kind `30198` grant for `d = node_pubkey:controller_pubkey` if one exists, otherwise the one for `d = node_pubkey:OTHERS`. If neither exists, deny with `UNAUTHORIZED`.
2. Parse the grant content as `UsageProfile`. If parsing fails, deny with `UNAUTHORIZED`.
3. Look up the requested method in the applicable map — `methods` for NWC requests, `control` for NNC requests. If the map is missing, empty, or the requested method is not present and `"OTHERS"` is not present, deny with `RESTRICTED`.
4. Check per-method rate limit (from the `rate` field on the method entry). If the rate is exceeded, deny with `RATE_LIMITED`.
5. Check `quota`. Compute the request's cost — see *What the quota counts*; it is zero for every method that does not send — and if the bucket cannot cover it, deny with `QUOTA_EXCEEDED`. The check MUST NOT mutate, and the same cost is committed at pipeline step 7.
6. Grant access.

## Relationship to NIP-47

NIP-47 (NWC) and NIP-XX (NNC) are complementary protocols with a clear boundary:

| | NIP-47 (NWC) | NIP-XX (NNC) |
|---|---|---|
| **Purpose** | Wallet operations | Node administration |
| **Audience** | Apps, merchants, services | Node owners, admins |
| **Operations** | Pay, receive, check balance | Channels, peers, fees, graph |
| **Info kind** | 13194 | 13198 |
| **Request kind** | 23194 | 23198 |
| **Response kind** | 23195 | 23199 |
| **Notification kind** | 23196 | 23200 |
| **URI scheme** | `nostr+walletconnect://` | `nostr+nodecontrol://` |

A node dashboard application would typically use **both** protocols: NWC for payment operations (pay invoice, create invoice, check balance) and NNC for node management (open channel, set fees, monitor routing).

The two protocols use separate connection URIs and separate key pairs, allowing independent authorization. A user can grant an application NWC access without granting NNC access, and vice versa.

## Example Flow

1. The node operator configures their **node service** and scans or copies the `nostr+nodecontrol://` URI into their **client** application.
2. The **client** fetches the NNC info event (kind `13198`) from the relay to learn which NNC methods are available.
3. The user wants to open a channel. The **client** creates an `open_channel` request with `"notify": true`, encrypts it with NIP-44, and publishes kind `23198` to the relay.
4. The **node service** decrypts the request, calls the Lightning node's channel open API, and publishes an encrypted kind `23199` response with the funding transaction ID.
5. Once the channel is confirmed on-chain, the **node service** sends a `channel_opened` notification (kind `23200`) back to the **client** that made the request.
