# Remote Procedure Call Module (RPC)

Richard T. Carback III

## Introduction

Remote Procedure Call (RPC) messages are sent from a `client` user to
a `server` user over the xx network. The `server` then responds to the
`client` over the xx network with the results of the remote procedure
function call or command. For the purposes of this section we will
use `client` and `server` to mean the RPC client and RPC server.

In the RPC scenario, the `server` is offering a public application
programming interface (API) over the xx network that any public
`client` can access, so messages sent to it are not authenticated and
can be sent by any party. Messages in the RPC module are also
self-contained and stateless.

The RPC module is also suitable for REST APIs. It may not be suitable
for APIs where the contents of the request or response would span
across many xx network packets, but our specifications do include
multi-packet support.

The RPC module has the following properties and trade-offs:
* Instead of a [cMix Reception ID](message_pickup.md), RPCs use
  identities chosen by the `server` and an ephemeral ID generated
  on the fly by the `client` receiving the response.
* While RPC requests are not authenticated by design as RPC `server`s
  are public, responses are resistant to key-compromise impersonation
  (KCI).
* Forward secrecy for senders (`client`s), as they generate and use a
  new key for each request.
* If the `server`s key is compromised, then requests can be decrypted
  although responses remain anonymous and protected by the ephemeral key.
* Support for post-quantum (PQ) security. Our initial implementationw
  will use the [Noise Protocol Framework](https://noiseprotocol.org/),
  which will require an upgrade to support PQ.


Like [DMs](dm.md), RPCs perform one-way non-interactive handshakes
using the known public key of the RPC `server`. RPCs include ephemeral
recipient information derived from the ephemeral public key encrypted
in the packet. There is no static key used by the `client`.

## Sending and Receiving

We use the [Noise Protocol Framework](https://noiseprotocol.org/) to
implement the non-interactive one-way handshake. Specifically, we use
the `NK` pattern, because the recipient `server` key is known in
advance:

```
NK:
1.  <- s
    ...
2.  -> e, es
3.  <- e, ee
```

Where:

1. `<- s` denotes the static key for the responder (`server`) is
   known.
2. On the first message, the `client` generates an ephemeral key `e`
   derives a shared key with the static key `s` from the
   `server`. This is used to encrypt the RPC request.
3. On the response message, the `server` generates a new ephemeral
   key, then derives a second shared key to encrypt the RPC response.

RPCs use the following Noise Protocol Name:

```
Noise_NK_25519_ChaChaPoly_BLAKE2s
```

This noise protocol uses ECDH asymmetric encryption with
XChaCha20Poly1305 symmetric encryption and Blake2s hashes inside of the
protocol.  Additionally, the prologue is set to the current RPC protocol
version (`0x0 0x0` at the time of this writing). Full details on Noise
Protocol and the syntax used above can be found in the
[noise specification document](https://noiseprotocol.org/noise.html).

For PQ support, the `25519` algorithm will need to be modified using a
method similar to that outlined in the [E2E spec](end_to_end.md). This
will increase the key size between 512 and 2048 bits depending on the
key sizes chosen. These sizes can be supported and still work well with
many RPC APIs.

## Reception IDs

The `server` can select it's own reception ID. This can be derived
from its public key, manually set by the operator, or through some
other method. A pseudorandomly generated ID shall be the
default. Reception of each message will be determined by encrypting
for the server's static key, so using the same reception ID is not a
concern.

The `client` selects a random id determined by the key they
generate. This is a one-time use key and reception ID. For simplicity,
the client shall be live the entire time this temporary reception ID
is in use, so no storage will be necessary to e.g., receive a response
after the client is closed.


## RPC Messages

Every RPC message has the following contents of note:

```
message RPCMessage{
    byte Version = 1;
    ...
    bytes PubKey = 3;
    ...
    bytes Ciphertext = 5;
    ...
    bytes Nonce = 7;
}

```

These fields are used as follows:
* `Version` of the rpc message, set to 0x0.
* `PubKey` the public key sent as part of the NK Noise protocol.
* `Ciphertext` of the request or response.
* `Nonce` set by the sender to avoid repeats. Repeated
  nonces should be ignored.

Requests and responses look the same from a cMix packet perspective.

## RPC Encryption

The format of the plaintext inside the RPC ciphertext for the request or the
response uses a message partitioning format:

```
plaintext = varuint(len(message)) | varuint(offset) | message
```

Where:
* `varuint` is a variable length integer encoding of an unsigned integer.
* `len` is the length function.
* `offset` number of bytes into the message that this part represents.
* `message` the contents of the request or response.

### Request

The sender client creates and encrypts the request plaintext as follows:

```
requestPublicKey, requestPrivateKey = generateKeyPair(rng)
key = H(DH(requestPrivateKey, RPCServerStaticPublicKey))
ciphertext = NoiseNK.Encrypt(key, plaintext)
```

### Response

The server creates and encrypts the plaintext response as follows:

```
responsePublicKey, responsePrivateKey = generateKeyPair(rng)
key = H(DH(responsePrivateKey, requestPublicKey))
ciphertext = NoiseNK.Encrypt(key, plaintext)
```

## Security Considerations

The cMix protocol is providing the metadata protection and Noise is
providing all of the core confidentiality properties. A nonce and timestamp
prevent excessive resending of the same request and replays.

Requests are encryption to a known recipient, which provides forward
secrecy for sender compromise only. Without the nonce this message can
be replayed, since there's no ephemeral contribution from the
recipient.

For requests, the payload is also encrypted based only on DHs
involving the recipient's static key pair. If the server's static
private key is compromised, even at a later date, this payload can be
decrypted. You can mitigate by rotating the key, creating keys for
specific users, or another method.

Responses are encryption to an ephemeral recipient, so the payload has
forward secrecy as the encryption involves an ephemeral-ephemeral DH
(`ee`). However, the sender cannot authenticate the recipient, which
means the RPC server must treat all requests as untrusted. This could
be used to exhaust resources, but can be mitigated with anonymized api
keys, paid return postage, or another mechanism.

The identity of the requestor remains protected in all cases because
it is not transmitted. It is important that the RPC protocol itself
not add information which could be used to break this protection. The
server also enjoys significant protection but is using a "public"
address in practice, so while it's physical location is protected its
cMix address is protected via recipient ID collisions.
