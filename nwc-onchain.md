NWC-XX
======

On-chain Payments
-----------------

`draft` `optional`

## Summary

This specification defines one optional Nostr Wallet Connect method:

- `pay_onchain` sends an on-chain Bitcoin payment to an address.

## Motivation

Some wallets hold on-chain funds and no Lightning channels at all: a
faucet paying from a mining wallet, a treasury service, a custodian
sweeping to cold storage. They cannot implement any Lightning method, and
today NWC gives them nothing to implement.

[NWC-321](https://github.com/nostr-wallet-connect/nwc) covers BIP-321
payment URIs, but an implementation of its `pay` method MUST support
`lightning` or `lno` instructions, so an on-chain-only wallet cannot
conform to it.

The two are complementary rather than alternatives, and the difference is
capability discovery. `pay` is polymorphic: it accepts a URI that may carry
several instruction types, and a client cannot tell from the method name
which ones the wallet will actually pay. `pay_onchain` is self-describing —
the method name *is* the capability — so it needs nothing beyond the
method-level discovery NWC core already has.

It is also a much smaller obligation. `pay` requires URI parsing,
rejection of unknown `req-` parameters, network checking, instruction
selection, and the BIP-321 `pop` / `req-pop` proof-of-payment rules. A
wallet that only ever sends coins to an address should not have to
implement any of that, particularly one whose keys are worth stealing.

## Methods

### `pay_onchain`

Sends an on-chain Bitcoin payment.

Request:
```yaml
{
    "method": "pay_onchain",
    "params": {
        "address": "bc1q...", // Bitcoin address, required
        "amount_sats": 50000, // amount in sats, required
        "feerate": 5 // fee rate in sat/vB, optional; the wallet picks a default if omitted
    }
}
```

Amounts are in **sats**, not msats, because on-chain outputs have no
sub-satoshi precision. This differs from the rest of NWC deliberately: a
msat amount here would have values that cannot be expressed on-chain.

The wallet service MUST reject an address for a different Bitcoin network.

If `feerate` is absent the wallet service selects one. If present, the
wallet service SHOULD use it and MUST NOT exceed it.

Response:
```yaml
{
    "result_type": "pay_onchain",
    "result": {
        "txid": "abc123...", // transaction id
        "fee_sats": 250 // fee paid in sats, optional if unavailable
    }
}
```

The transaction is broadcast, not confirmed. A client that needs
confirmation observes the chain or subscribes to notifications; this
response says only that the wallet service accepted and broadcast it.

Errors:

- `PAYMENT_FAILED`: The payment failed.
- `INSUFFICIENT_BALANCE`: The wallet does not have enough funds.
- `UNSUPPORTED_NETWORK`: The address is for a different Bitcoin network.
- `BAD_REQUEST`: The address or another parameter is invalid.

The wallet service can also return applicable NWC core errors.

## Relationship to other specs

- NWC core defines the request and response envelope, the error codes, and
  method discovery through the info event.
- NWC-321 defines BIP-321 payment URIs. A wallet MAY implement both. A
  wallet that implements `pay_onchain` makes no claim about Lightning, and
  a wallet that implements NWC-321's `pay` makes no claim about paying a
  bare address.
