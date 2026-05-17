NIP-XY
======

Nostr Signer Connect (NSC)
--------------------------

`draft` `optional`

## Rationale

[NIP-47](47.md) (Nostr Wallet Connect) defines a protocol for clients to access a remote Lightning wallet — paying invoices, creating invoices, checking balances. [NIP-XX](XX.md) (Nostr Node Control) defines a protocol for administrators to manage a Lightning node — opening channels, setting fees, monitoring routing.

Both protocols assume the node holds its own signing keys. However, security-conscious deployments separate the **signing function** from the **node** — the node runs in a DMZ while the signer runs in a hardware enclave, a separate machine, or a remote TEE. These deployments need a protocol for the node to reach its signer over Nostr.

NSC (Nostr Signer Connect) fills this gap. It allows an **NSC-capable node** to connect to an **NSC-capable signer** using the VLS (Validating Lightning Signer) protocol, transported as encrypted Nostr events over a relay.

## Terms

* **node**: Lightning node software (LDK, CLN, etc.) that needs signing services. The NSC **client**.
* **signer**: VLS-compatible signer that holds keys and produces signatures. The NSC **service**.
* **operator**: The person who provisions the node-to-signer connection.
* **relay**: Internal Nostr relay mediating communication between node and signer.

## Theory of Operation

The connection flow mirrors NIP-47:

1. The **operator** provisions the **signer** and obtains a connection URI (`nostr+signerconnect://`). The **signer** publishes an NSC info event (kind `13200`) to the relay.

2. The **operator** delivers the connection URI to the **node** (config file, provisioning API, or paste). The **node** derives its keypair from the `secret` in the URI and connects to the relay.

3. When the **node** needs a signature (e.g., signing a commitment transaction), it creates an NSC request event (kind `23201`), encrypts the payload with [NIP-44](44.md), and publishes it to the relay.

4. The **signer** decrypts the request, executes the VLS operation, and publishes an encrypted NSC response event (kind `23202`).

5. The **signer** MAY send unsolicited encrypted NSC notification events (kind `23203`) to the **node** (e.g., heartbeats, policy updates).

## Events

There are four event kinds:

- `NSC info event`: **13200** (replaceable)
- `NSC request`: **23201** (ephemeral)
- `NSC response`: **23202** (ephemeral)
- `NSC notification event`: **23203** (ephemeral)

### Info Event (kind 13200)

The info event is a replaceable event published by the **signer** on the relay to indicate which NSC methods it supports.

The content SHOULD be a plaintext string with the supported methods space-separated:

```
vls_create_channel vls_funding_created vls_funding_signed vls_sign_funding vls_channel_ready vls_commitment_signed vls_revoke_commitment vls_validate_revocation vls_closing_signed
```

The event SHOULD contain an `encryption` tag:

```jsonc
{
    "kind": 13200,
    "tags": [
        ["encryption", "nip44_v2"]
    ],
    "content": "vls_create_channel vls_funding_created vls_funding_signed vls_sign_funding vls_channel_ready vls_commitment_signed vls_revoke_commitment vls_validate_revocation vls_closing_signed"
    // ...
}
```

### Request Event (kind 23201)

The request event SHOULD contain one `p` tag with the public key of the **signer**.

Optionally, a request can have an `expiration` tag with a unix timestamp in seconds. If the request is received after this timestamp, it MUST be ignored. Recommended expiration: `created_at + 30` seconds.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "method": "vls_commitment_signed", // method name, string
    "params": {                        // params, object
        // method-specific parameters
    }
}
```

### Response Event (kind 23202)

The response event SHOULD contain one `p` tag with the public key of the **node** and an `e` tag with the id of the request event it is responding to.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "result_type": "vls_commitment_signed", // indicates the structure of the result field
    "error": {                              // object, non-null in case of error
        "code": "POLICY_VIOLATION",         // string error code, see below
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

### Notification Event (kind 23203)

The notification event SHOULD contain one `p` tag with the public key of the **node**.

The content is encrypted with [NIP-44](44.md) and is a JSON object:

```jsonc
{
    "notification_type": "heartbeat", // indicates the structure of the notification field
    "notification": {
        // notification-specific data
    }
}
```

### Error Codes

- `POLICY_VIOLATION`: The signer rejected the request due to policy rules.
- `INVALID_STATE`: Channel not found or in unexpected state.
- `INVALID_PARAMS`: Malformed or missing parameters.
- `SIGNATURE_INVALID`: Counterparty signature verification failed.
- `RATE_LIMITED`: The node is sending commands too fast.
- `NOT_IMPLEMENTED`: The method is not known or not implemented.
- `INTERNAL`: An internal signer error.
- `OTHER`: Other error.

## NSC Connection URI

The **node** discovers the **signer** by importing a connection URI (config file, paste, or provisioning API).

The connection URI uses the protocol `nostr+signerconnect://` and base path the hex-encoded `pubkey` of the **signer**, which SHOULD be unique per node connection.

The connection URI contains the following query string parameters:

- `relay` Required. URL of the relay where the **signer** is connected and will be listening for events. May be more than one.
- `secret` Required. 32-byte randomly generated hex encoded string. The **node** MUST use this to sign events and encrypt payloads when communicating with the **signer**. The **signer** MUST use the corresponding public key of this secret to communicate with the **node**.

### Example connection string

```sh
nostr+signerconnect://b889ff5b1513b641e2a139f661a661364979c5beee91842f8f0ef42ab558e9d4?relay=ws%3A%2F%2Frelay.internal%3A7777&secret=71a8c14c1407c113601079c4302dab36460f0ccd0ad506f1f2dc73b5100e4f3c
```

## Methods

### Channel Open

#### `vls_create_channel`

Description: Creates a new channel and returns the signer's basepoints and initial per-commitment point.

Request:
```jsonc
{
    "method": "vls_create_channel",
    "params": {
        "peer_id": "02abc...",          // counterparty node pubkey
        "channel_value_sats": 1000000,  // total channel capacity in sats
        "push_value_sats": 0,           // initial push to counterparty in sats
        "is_outbound": true             // whether we are the funder
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_create_channel",
    "result": {
        "channel_id": 42,               // assigned channel identifier
        "basepoints": {
            "revocation": "02abc...",
            "payment": "02def...",
            "htlc": "02ghi...",
            "delayed_payment": "02jkl..."
        },
        "funding_pubkey": "02mno...",
        "per_commitment_point_0": "02pqr..."
    }
}
```

#### `vls_funding_created`

Description: Sets up the channel and signs the counterparty's initial commitment transaction. Called by the funder after `open_channel`/`accept_channel` exchange.

Request:
```jsonc
{
    "method": "vls_funding_created",
    "params": {
        "channel_id": 42,
        "is_outbound": true,
        "channel_value_sats": 1000000,
        "push_value_sats": 0,
        "funding_txid": "abc123...",
        "funding_txout": 0,
        "to_self_delay": 144,
        "remote_basepoints": {
            "revocation": "03abc...",
            "payment": "03def...",
            "htlc": "03ghi...",
            "delayed_payment": "03jkl..."
        },
        "remote_funding_pubkey": "03mno...",
        "remote_to_self_delay": 144,
        "channel_type": "0100...",
        "remote_per_commitment_point": "03pqr...",
        "commitment_number": 0,
        "feerate": 5000,
        "to_local_value_sats": 0,
        "to_remote_value_sats": 1000000,
        "htlcs": []
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_funding_created",
    "result": {
        "signature": "3045..."          // signature on counterparty's commitment_0
    }
}
```

#### `vls_funding_signed`

Description: Called by the non-funder. Sets up the channel, validates own initial commitment, and signs the counterparty's initial commitment.

Request:
```jsonc
{
    "method": "vls_funding_signed",
    "params": {
        "channel_id": 42,
        "is_outbound": false,
        "channel_value_sats": 1000000,
        "push_value_sats": 0,
        "funding_txid": "abc123...",
        "funding_txout": 0,
        "to_self_delay": 144,
        "remote_basepoints": {
            "revocation": "02abc...",
            "payment": "02def...",
            "htlc": "02ghi...",
            "delayed_payment": "02jkl..."
        },
        "remote_funding_pubkey": "02mno...",
        "remote_to_self_delay": 144,
        "channel_type": "0100...",
        "remote_per_commitment_point": "02pqr...",
        "commitment_number": 0,
        "feerate": 5000,
        "to_local_value_sats": 0,
        "to_remote_value_sats": 1000000,
        "htlcs": [],
        "counterparty_signature": "3045...",
        "counterparty_htlc_signatures": []
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_funding_signed",
    "result": {
        "signature": "3045...",                // signature on counterparty's commitment_0
        "next_per_commitment_point": "03abc..."  // point for commitment_1
    }
}
```

#### `vls_sign_funding`

Description: Validates own initial commitment and signs the funding transaction (funder only).

Request:
```jsonc
{
    "method": "vls_sign_funding",
    "params": {
        "channel_id": 42,
        "counterparty_signature": "3045...",
        "counterparty_htlc_signatures": [],
        "utxos": [...],                   // UTXOs being spent
        "psbt": "cHNidP8..."             // unsigned funding PSBT
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_sign_funding",
    "result": {
        "next_per_commitment_point": "03abc...",
        "signed_psbt": "cHNidP8..."       // signed funding PSBT
    }
}
```

#### `vls_channel_ready`

Description: Confirms funding is buried and locks the outpoint. Called after the funding transaction confirms.

Request:
```jsonc
{
    "method": "vls_channel_ready",
    "params": {
        "channel_id": 42,
        "funding_txid": "abc123...",
        "funding_txout": 0
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_channel_ready",
    "result": {
        "is_buried": true,
        "per_commitment_point_1": "03abc..."
    }
}
```

### Commitment Updates

#### `vls_commitment_signed`

Description: Signs the counterparty's new commitment transaction. Called when the node needs to send `commitment_signed` to the peer.

Request:
```jsonc
{
    "method": "vls_commitment_signed",
    "params": {
        "channel_id": 42,
        "remote_per_commitment_point": "03abc...",
        "commitment_number": 1,
        "feerate": 5000,
        "to_local_value_sats": 0,
        "to_remote_value_sats": 800000,
        "htlcs": [
            {
                "side": 0,
                "amount_sats": 200000,
                "payment_hash": "deadbeef...",
                "cltv_expiry": 100
            }
        ]
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_commitment_signed",
    "result": {
        "signature": "3045...",
        "htlc_signatures": ["3044..."]
    }
}
```

#### `vls_revoke_commitment`

Description: Validates the node's new commitment (using the counterparty's signature) and revokes the old commitment. Called when the node receives `commitment_signed` from the peer and needs to send `revoke_and_ack`.

Request:
```jsonc
{
    "method": "vls_revoke_commitment",
    "params": {
        "channel_id": 42,
        "commitment_number": 1,
        "feerate": 5000,
        "to_local_value_sats": 0,
        "to_remote_value_sats": 800000,
        "htlcs": [
            {
                "side": 1,
                "amount_sats": 200000,
                "payment_hash": "deadbeef...",
                "cltv_expiry": 100
            }
        ],
        "counterparty_signature": "3045...",
        "counterparty_htlc_signatures": ["3044..."]
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_revoke_commitment",
    "result": {
        "old_commitment_secret": "abc123...",      // 32-byte hex revocation secret
        "next_per_commitment_point": "03abc..."    // for revoke_and_ack
    }
}
```

#### `vls_validate_revocation`

Description: Validates the counterparty's revocation secret. Called when the node receives `revoke_and_ack` from the peer.

Request:
```jsonc
{
    "method": "vls_validate_revocation",
    "params": {
        "channel_id": 42,
        "commitment_number": 0,
        "commitment_secret": "abc123..."   // 32-byte hex counterparty's revocation secret
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_validate_revocation",
    "result": {}
}
```

### Cooperative Close

#### `vls_closing_signed`

Description: Signs the cooperative close transaction. Called when the node needs to send `closing_signed` to the peer.

Request:
```jsonc
{
    "method": "vls_closing_signed",
    "params": {
        "channel_id": 42,
        "to_local_value_sats": 800000,
        "to_remote_value_sats": 200000,
        "local_script": "0014abc...",         // local shutdown scriptpubkey (hex)
        "remote_script": "0014def...",        // remote shutdown scriptpubkey (hex)
        "local_wallet_path_hint": [84, 0, 0, 0, 5]  // BIP32 derivation path
    }
}
```

Response:
```jsonc
{
    "result_type": "vls_closing_signed",
    "result": {
        "signature": "3045..."
    }
}
```

## Notifications

The **signer** MAY send unsolicited notifications to the **node**:

### `heartbeat`

Description: Periodic liveness signal indicating the signer is operational and at a known chain state.

```jsonc
{
    "notification_type": "heartbeat",
    "notification": {
        "chain_tip_height": 800000,
        "chain_tip_hash": "000000000000..."
    }
}
```

### `policy_update`

Description: A new UsageProfile (kind `30078`) has been ingested by the signer.

```jsonc
{
    "notification_type": "policy_update",
    "notification": {
        "profile_event_id": "abc123...",    // event id of the kind:30078 event
        "created_at": 1700000000
    }
}
```

### `signer_error`

Description: The signer encountered an error outside of a request context.

```jsonc
{
    "notification_type": "signer_error",
    "notification": {
        "code": "ORPHAN_BLOCK",
        "message": "Received orphan block during chain sync"
    }
}
```

## Encryption

All NSC request, response, and notification payloads MUST be encrypted using [NIP-44](44.md). The encryption uses the **node**'s `secret` (from the connection URI) and the **signer**'s public key.

NIP-04 is NOT supported. NSC implementations MUST use NIP-44 exclusively.

## Access Control

NSC uses relay-level access control ([NIP-AB](AB.md)) rather than application-level access grants. Only pubkeys authorized by the relay's allowlist can publish events. This is appropriate because:

- NSC operates on an internal relay (never public-facing)
- The connection is one-to-one (one node, one signer)
- The operator controls both the node and signer provisioning

For deployments that also run NWC and NNC on the same relay, the relay's NIP-AB allowlist includes the node's NSC pubkey, the signer's NSC pubkey, and any NWC/NNC service pubkeys.

## Relationship to NIP-47 and NIP-XX

NIP-47 (NWC), NIP-XX (NNC), and NIP-XY (NSC) are three complementary protocols:

| | NIP-47 (NWC) | NIP-XX (NNC) | NIP-XY (NSC) |
|---|---|---|---|
| **Purpose** | Wallet operations | Node administration | Signer operations |
| **Connects** | App → Wallet/Node | Admin → Node | Node → Signer |
| **Operations** | Pay, receive, balance | Channels, peers, fees | Sign, revoke, validate |
| **Info kind** | 13194 | 13198 | 13200 |
| **Request kind** | 23194 | 23198 | 23201 |
| **Response kind** | 23195 | 23199 | 23202 |
| **Notification kind** | 23197 | 23200 | 23203 |
| **URI scheme** | `nostr+walletconnect://` | `nostr+nodecontrol://` | `nostr+signerconnect://` |
| **Access control** | kind 30078 grants | kind 30078 grants | Relay-level (NIP-AB) |

A full deployment uses all three:

```
  Owner/App                 Node                     Signer
     │                       │                         │
     │── NWC (pay, recv) ──→ │                         │
     │── NNC (channels) ───→ │                         │
     │                       │── NSC (sign, revoke) ──→│
```

## Example Flow

1. The **operator** provisions the **signer** in a secure enclave. The signer generates a keypair and produces a `nostr+signerconnect://` URI.
2. The **operator** configures the **node** with the connection URI. The node derives its keypair from the `secret` parameter.
3. Both the node and signer connect to the internal relay via persistent WebSocket.
4. The node receives an `update_add_htlc` from a peer and needs to sign the counterparty's new commitment. It creates a `vls_commitment_signed` request, encrypts it with NIP-44, and publishes kind `23201` to the relay.
5. The signer receives the event, decrypts it, validates policy rules, signs the commitment transaction, and publishes an encrypted kind `23202` response.
6. The node receives the response, extracts the signature, and sends `commitment_signed` to the peer.

## Implementation Notes

### Latency

Lightning signing operations are latency-sensitive. Peer timeouts are typically 30-60 seconds, but good performance requires sub-second signer responses. NSC is designed for internal relays with < 10ms round-trip times (local network or vsock).

### Concurrency

The node may have multiple channels and multiple signing operations in flight simultaneously. Each request event has a unique `id`, and each response references it via `e` tag. The node maintains a map of pending request IDs to callbacks. The signer processes requests for different channels concurrently but maintains sequential ordering within a single channel.

### Ephemeral Events

NSC request and response events are ephemeral — they do not need to be stored by the relay. The relay acts as a real-time message router. This minimizes disk I/O and latency on the critical signing path.
