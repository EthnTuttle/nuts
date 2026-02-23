# NUT-XX: Iroh QUIC P2P Transport

`optional`

`depends on: NUT-06, NUT-17, NUT-18`

---

This NUT defines how Cashu mints and wallets use [Iroh](https://iroh.computer) as a transport layer for all NUT protocol operations. Iroh provides NAT-traversing QUIC connections between peers identified by Ed25519 public keys (NodeId), replacing HTTP and WebSocket with a single QUIC-based channel. Mints become reachable by NodeId alone — no domain, IP address, or port forwarding required.

## Motivation

| Property | HTTP/WebSocket | Iroh QUIC |
|---|---|---|
| NAT traversal | Requires public IP or tunnel | Built-in hole-punching with relay fallback |
| Encryption | TLS (separate cert management) | Inherent to QUIC + Ed25519 identity |
| Real-time push | WebSocket upgrade required | Native on persistent QUIC stream |
| Identity | Domain name / IP | Ed25519 NodeId (self-sovereign) |
| Reachability | DNS + open port | NodeId only |

## Specification

### ALPN

Iroh connections for Cashu MUST use the ALPN protocol identifier:

```
cashu/mint/0
```

Mints MUST reject connections that do not present this ALPN token.

### Mint Identity

- On first run, the mint generates an Ed25519 keypair and persists it to stable storage.
- The public key of this keypair is the mint's **NodeId**, encoded in z32 (lowercase base32 without padding).
- This keypair is **independent** of the mint's Cashu signing keys and MUST NOT be reused for blind signature operations.
- The NodeId MUST remain stable across restarts. Changing the NodeId is equivalent to the mint becoming a new, unknown peer.

### Discovery (NUT-06 extension)

Mints supporting Iroh transport SHOULD include their NodeId in the `urls` array of the NUT-06 `GetInfoResponse` using the `iroh://` URL scheme:

```
iroh://<z32-encoded-NodeId>
```

Example `GetInfoResponse` with Iroh advertised:

```json
{
  "name": "Bob's Cashu mint",
  "pubkey": "0283bf290884eed3a7ca2663fc0260de2e2064d6b355ea13f98dec004b7a7ead99",
  "version": "cdk/0.1.0",
  "urls": [
    "https://mint.example.com",
    "iroh://b7q3kfma3mfxlcn7qe2yrkbyqo2nqlg3c7ux7bvt6p5lrgpucpq"
  ],
  "nuts": {
    "17": {
      "supported": [
        {
          "method": "bolt11",
          "unit": "sat",
          "commands": ["bolt11_mint_quote", "bolt11_melt_quote", "proof_state"]
        }
      ]
    },
    "XX": {
      "supported": true
    }
  }
}
```

Mints signal Iroh transport support via the `nuts` field using key `"XX"`:

```json
"nuts": {
  "XX": {
    "supported": true
  }
}
```

### Wire Protocol

#### Framing

Every message on an Iroh QUIC stream is framed as follows:

```
[4 bytes: length (big-endian u32)] [length bytes: UTF-8 JSON]
```

The length field encodes the byte length of the following JSON payload. Both sides MUST close the stream with an error if a message exceeds an implementation-defined maximum (RECOMMENDED: 4 MiB).

#### JSON-RPC 2.0

All messages use JSON-RPC 2.0. Each QUIC **bidirectional stream** carries exactly one request/response pair, except subscription streams (see [NUT-17 section](#nut-17-subscriptions-over-iroh)).

**Request** (wallet → mint):

```json
{
  "jsonrpc": "2.0",
  "method": "<method-name>",
  "params": {},
  "id": 1
}
```

**Response** (mint → wallet):

```json
{
  "jsonrpc": "2.0",
  "result": {},
  "id": 1
}
```

**Error response** (mint → wallet):

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32600,
    "message": "Invalid Request"
  },
  "id": 1
}
```

The `id` field MUST be an integer. The wallet MUST increment `id` for each new request on a connection.

After sending a response the mint MUST close its half of the stream (send FIN). After receiving the response the wallet MUST close its half of the stream. Wallets MAY pipeline multiple requests by opening multiple streams concurrently.

### Method Reference

The table below maps each NUT operation to its Iroh JSON-RPC method name and the equivalent HTTP endpoint.

| Method | HTTP Equivalent | Params |
|---|---|---|
| `cashu_getInfo` | `GET /v1/info` | _(none)_ |
| `cashu_getKeys` | `GET /v1/keys` | _(none)_ |
| `cashu_getKeysets` | `GET /v1/keysets` | _(none)_ |
| `cashu_getKeyById` | `GET /v1/keys/{id}` | `{ "id": "<keyset_id>" }` |
| `cashu_postSwap` | `POST /v1/swap` | NUT-03 `SwapRequest` |
| `cashu_postMintQuote` | `POST /v1/mint/quote/bolt11` | NUT-04 `MintQuoteRequest` + `{ "payment_method": "bolt11" }` |
| `cashu_getMintQuote` | `GET /v1/mint/quote/bolt11/{id}` | `{ "id": "<quote_id>", "payment_method": "bolt11" }` |
| `cashu_postMint` | `POST /v1/mint/bolt11` | NUT-04 `MintRequest` + `{ "payment_method": "bolt11" }` |
| `cashu_postMeltQuote` | `POST /v1/melt/quote/bolt11` | NUT-05 `MeltQuoteRequest` + `{ "payment_method": "bolt11" }` |
| `cashu_getMeltQuote` | `GET /v1/melt/quote/bolt11/{id}` | `{ "id": "<quote_id>", "payment_method": "bolt11" }` |
| `cashu_postMelt` | `POST /v1/melt/bolt11` | NUT-05 `MeltRequest` + `{ "payment_method": "bolt11" }` |
| `cashu_checkState` | `POST /v1/checkstate` | NUT-07 `CheckStateRequest` |
| `cashu_restore` | `POST /v1/restore` | NUT-09 `RestoreRequest` |
| `cashu_subscribe` | `GET /v1/ws` (NUT-17) | See [NUT-17 section](#nut-17-subscriptions-over-iroh) |

### Example: Swap

Wallet opens a new bidirectional stream and sends (framed as above):

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "cashu_postSwap",
  "params": {
    "inputs": ["<Proof>", "..."],
    "outputs": ["<BlindedMessage>", "..."]
  }
}
```

Mint responds on the same stream:

```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "result": {
    "signatures": ["<BlindSignature>", "..."]
  }
}
```

Both sides then close their half of the stream.

### NUT-17 Subscriptions over Iroh

For real-time subscriptions ([NUT-17][17]), the wallet opens a dedicated **long-lived** bidirectional QUIC stream and sends `cashu_subscribe` as the first message on that stream. The request and all subsequent messages on the stream follow the NUT-17 JSON-RPC 2.0 message format (`WsRequest`, `WsResponse`, `WsNotification`) unchanged — only the transport changes from WebSocket to QUIC stream.

The `id` counter for a subscription stream is independent of the `id` counters used on per-request streams.

The wallet MAY open multiple concurrent subscription streams on the same Iroh connection. Each stream is independent.

#### Subscribe

Wallet → mint (first message on the subscription stream):

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "subscribe",
  "params": {
    "kind": "proof_state",
    "filters": [
      "02e208f9a78cd523444aadf854a4e91281d20f67a923d345239c37f14e137c7c3d"
    ],
    "subId": "Ua_IYvRHoCoF_wsZFlJ1m4gBDB--O0_6_n0zHg2T"
  }
}
```

Mint confirms:

```json
{
  "jsonrpc": "2.0",
  "result": {
    "status": "OK",
    "subId": "Ua_IYvRHoCoF_wsZFlJ1m4gBDB--O0_6_n0zHg2T"
  },
  "id": 0
}
```

Mint immediately pushes the current state and subsequent updates as `WsNotification` messages on the same stream:

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "params": {
    "subId": "Ua_IYvRHoCoF_wsZFlJ1m4gBDB--O0_6_n0zHg2T",
    "payload": {
      "Y": "02e208f9a78cd523444aadf854a4e91281d20f67a923d345239c37f14e137c7c3d",
      "state": "UNSPENT",
      "witness": null
    }
  }
}
```

#### Unsubscribe

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "unsubscribe",
  "params": {
    "subId": "Ua_IYvRHoCoF_wsZFlJ1m4gBDB--O0_6_n0zHg2T"
  }
}
```

After all subscriptions on a stream are closed, the wallet SHOULD close the stream.

### NUT-18 Transport Extension

This NUT adds a new transport type `"iroh"` to NUT-18 Payment Requests.

```json
{
  "t": "iroh",
  "a": "<z32-encoded-NodeId>",
  "g": [["relay", "<relay-url>"]]
}
```

Fields:

- `t`: `"iroh"` — transport type identifier.
- `a`: z32-encoded Iroh NodeId of the receiver.
- `g` (optional): tags. The `relay` tag provides a URL hint for NAT traversal bootstrap; multiple `relay` tags MAY be present.

To deliver a payment over Iroh transport, the sender:

1. Connects to the NodeId specified in `a`.
2. Opens a bidirectional stream and sends a `cashu_postPayment` request containing the `PaymentRequestPayload` (NUT-18).
3. Awaits the mint/receiver's JSON-RPC response confirming acceptance.

Wallets that do not support the `"iroh"` transport type MUST skip it and attempt the next transport in the list per the NUT-18 fallback behavior.

Example NUT-18 payment request with Iroh transport:

```json
{
  "i": "b7a90176",
  "a": 10,
  "u": "sat",
  "m": ["https://mint.example.com"],
  "t": [
    {
      "t": "iroh",
      "a": "b7q3kfma3mfxlcn7qe2yrkbyqo2nqlg3c7ux7bvt6p5lrgpucpq",
      "g": [["relay", "https://relay.iroh.network"]]
    },
    {
      "t": "post",
      "a": "https://mint.example.com/v1/payment"
    }
  ]
}
```

### Error Codes

Standard JSON-RPC 2.0 error codes apply. Cashu-specific errors reuse the existing error code space defined in the Cashu error codes specification.

| Code | Meaning |
|---|---|
| `-32700` | Parse error — invalid JSON |
| `-32600` | Invalid Request — malformed JSON-RPC object |
| `-32601` | Method not found |
| `-32602` | Invalid params |
| `-32603` | Internal error |
| `1xx` | Cashu-domain errors (mapped from existing HTTP error responses) |

Mint implementations SHOULD map existing Cashu HTTP error response bodies to JSON-RPC error objects, preserving the Cashu error `code` and `detail` fields inside the `error.data` key:

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": 11001,
    "message": "Token already spent.",
    "data": {
      "detail": "Token already spent.",
      "code": 11001
    }
  },
  "id": 4
}
```

## Security Considerations

- All Iroh connections are mutually authenticated and encrypted at the QUIC/TLS layer using the Ed25519 NodeId keypairs of both peers. No additional TLS certificate management is required.
- Wallets MUST verify that the NodeId of the connected mint matches the value advertised in `MintInfo.urls` before trusting any response. A mismatch MUST cause the wallet to abort the connection.
- The Iroh NodeId keypair MUST be kept separate from the Cashu mint signing keys. Reuse of the same keypair for both purposes is explicitly forbidden.
- Relay servers used during NAT traversal can observe connection metadata (which NodeIds connect to each other) but MUST NOT be able to observe stream content, which remains end-to-end encrypted.
- Wallets SHOULD prefer direct QUIC connections over relay-proxied connections where possible, falling back to relays only when hole-punching fails.
- Implementations MUST enforce a maximum message size to prevent memory exhaustion from malformed length-prefixed frames.

## Reference Implementation

EthnTuttle/cdk — see issues [#1](https://github.com/EthnTuttle/cdk/issues/1) through [#7](https://github.com/EthnTuttle/cdk/issues/7).

[00]: 00.md
[03]: 03.md
[04]: 04.md
[05]: 05.md
[06]: 06.md
[07]: 07.md
[09]: 09.md
[17]: 17.md
[18]: 18.md
