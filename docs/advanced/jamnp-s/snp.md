---
id: spec
sidebar_label: JAMNP-S Spec
sidebar_position: 1
---

(source https://github.com/zdave-parity/jam-np/blob/main/simple.md from {new Date().toISOString().split('T')[0]})

<!-- The raw MD from above will be downloaded and appended -->
 # JAM Simple Networking Protocol (JAMNP-S)

All connections are based on the QUIC protocol.

## Encryption and handshake

TLS 1.3 is used for connection encryption and peer authentication.

During the TLS handshake, both the client (the peer that initiated the connection) and the server
(the peer that accepted the connection) must present X.509 certificates. A peer's certificate must:

- Use Ed25519 as the signature algorithm.
- Use the peer's Ed25519 key. If the peer is a validator, this key should have been published on
  chain.
- Have a single alternative name, which must be a 53-character DNS name consisting of "e" followed
  by the Ed25519 public key, base-32 encoded using the alphabet "abcdefghijklmnopqrstuvwxyz234567".

The certificates _should_ be self-signed, however this is not required and need not be verified.

The connection should be closed if the remote peer does not send a certificate, or the certificate
does not meet the above requirements. Note that in the case where a connection is being accepted,
the remote peer's Ed25519 key is not known in advance, but consistency between the certificate's
key and alternative name should still be checked.

Encryption and authentication occur as usual in TLS:

- Diffie-Hellman key exchange is used to produce an ephemeral encryption key.
- Both peers sign the handshake transcript hash.

### ALPN

The protocol name, version, and chain are identified using QUIC/TLS "ALPN" (Application Layer
Protocol Negotiation). The protocol identifier should be either `jamnp-s/V/H` or
`jamnp-s/V/H/builder`. Here `V` is the protocol version, `0`, and `H` is the first 8 nibbles of the
hash of the chain's genesis block header.

The `/builder` suffix should always be permitted by the side accepting the connection, but only
used by the side initiating the connection if it is connecting as a work-package builder. Note that
guarantors should accept work-package submission streams (CE 133) on all connections, regardless of
how they were opened. The purpose of identifying as a builder at connection time is merely to
request use of a slot reserved for builders, increasing the likelihood of a successful connection.

## Required connectivity

All validators in the previous, current, and next epochs should ensure they are connected to all
other such validators.

The validators' IP-layer endpoints are given as IPv6/port combinations, to be found in the first 18
bytes of validator metadata, with the first 16 bytes being the IPv6 address and the latter 2 being
a little endian representation of the port.

**Strategy**: For all such pairwise connections, we assign a Preferred Initiator for the connection
based on the Ed25519 public keys of the two validators ($k_e, k \in \mathbb{K}$). After a
reasonable timeout (e.g. 5 seconds), the other validator may choose to initiate the connection.
Given two Ed25519 public keys $(a, b)$, the Preferred Initiator $P(a, b) \in \{a, b\}$ is given as:

```math
P(a, b) \equiv \begin{cases}
    a &\text{when } \text{bits}(a)_{255} \oplus \text{bits}(b)_{255} \oplus (a < b)\\
    b &\text{otherwise}
\end{cases}
```

Validators should accept connections from other nodes too, with a reasonable number of slots (e.g.
20) reserved for work-package builders. Builders may reasonably be required to prove their
credentials through submission of a valid work-package in order to retain their connection.

### Grid structure

Validators are conceptually linked in a grid structure. Two validators are linked if:

- They are validators in the same epoch and either have the same row (`index / 31`) or the same
  column (`index % 31`).
- They are validators in different epochs but have the same index.

### Epoch transitions

At the beginning of a new epoch, validators should wait to apply connectivity changes until at
least one block in the previous or new epoch is finalized. The intent of this rule is to ensure
that the validator set currently required to advance finality remains connected. In normal
operation, where finality is running shortly behind block production, no waiting should be
required.

## Protocols and streams

All communication happens over bidirectional QUIC streams. Many such streams may be in existence at
once over a connection. Each stream has an associated _stream kind_, an identifier for the protocol
in force over the stream.

All stream kinds may be characterised into one of two patterns:

1. Unique Persistent (UP) streams: Only one stream may be open for each UP stream kind on a
   connection. UP streams will tend to be opened after connection and stay alive for the duration
   of the connection. UP streams are always opened by the initiator of the connection. It is
   possible, e.g. due to packet loss after a UP stream is closed and reopened, for the acceptor of
   the connection to observe multiple open streams with the same UP kind. In this case, the stream
   with the greatest ID (as defined in the QUIC specification) should be kept, and the other
   streams should be reset/stopped.
2. Common Ephemeral (CE) streams: Zero or many streams may be open for each CE stream kind on a
   connection. They will tend to be opened to make a specific query or submit a particular datum
   and closed immediately after completion. Unless otherwise specified, resetting/stopping of a CE
   stream by the stream initiator should be considered cancellation of the request, and
   resetting/stopping of a stream by the stream acceptor should be considered rejection of the
   request or failure to process the request.

After opening a stream, the stream initiator must send a single byte identifying the stream kind.
UP stream kinds are numbered starting from 0, CE stream kinds are numbered starting from 128. The
identified protocol becomes active immediately after this byte is sent.

All stream protocols are specified in terms of _messages_. A message is transmitted in two parts.
First, the size (in bytes) of the message content is transmitted, encoded as a little-endian 32-bit
unsigned integer. Second, the message content itself is transmitted.

In the protocol descriptions below:

- `X -> X` indicates the expected classes of the initiator and acceptor of the stream respectively.
  For example, `Builder -> Guarantor` means streams with this protocol are expected to be opened by
  builders on connections to guarantors.
- `-->` indicates a message sent by the initiator of the stream.
- `<--` indicates a message sent by the acceptor of the stream.
- `--> FIN` and `<-- FIN` indicate clean termination of the transmit half of the stream by the
  initiator and acceptor respectively (no message should be sent). In QUIC, the transmit half of a
  stream is cleanly terminated by setting the `FIN` bit in the last `STREAM` frame. Note that,
  unless otherwise specified, CE streams should only be considered successful if both halves of the
  stream are cleanly terminated.
- `++` indicates concatenation.
- Encoding is as per the serialization codec defined in the gray paper.
- `len++` preceding a sequence indicates that the sequence should be explicitly prefixed by its
  length. If a sequence is not preceded by `len++`, the length is either fixed or implied by
  context.

### UP 0: Block announcement

This should be opened between two nodes if either:

- Both nodes are validators, and are linked in the grid structure.
- At least one of the nodes is not a validator.

Here, "validator" means a validator in either the previous, current, or next epoch. As this is a UP
protocol, the initiator of the connection is responsible for opening the stream when necessary.

Both sides should begin by sending a handshake message containing all known leaves (descendants of
the latest finalized block with no known children).

An announcement should be sent on the stream whenever a new block is produced or received.
Announcement of a block may only be skipped if:

- A descendant of the block is announced instead.
- The block is not a descendant of the latest finalized block.

The hash and slot of the latest finalized block should be included in the handshake message and
also in every announcement message that is sent.

```
Header Hash = [u8; 32]
Slot = u32
Final = Header Hash ++ Slot
Leaf = Header Hash ++ Slot
Handshake = Final ++ len++[Leaf]
Header = As in GP
Announcement = Header ++ Final

Node -> Node

--> Handshake AND <-- Handshake (In parallel)
loop {
    --> Announcement OR <-- Announcement (Either side may send)
}
```

### CE 128: Block request

Request for a sequence of blocks.

There are two types of request:

- Ascending exclusive: The sequence of blocks in the response should start with a child of the
  given block, followed by a grandchild, and so on.
- Descending inclusive: The sequence of blocks in the response should start with the given block,
  followed by its parent, grandparent, and so on.

The number of blocks in the response should be limited to the given maximum.

Note that blocks directly contain only the hashes of included work-reports and preimages. If
unknown, the actual work-reports and preimages should be requested using protocols 136 and 143
respectively.

```
Header Hash = [u8; 32]
Direction = { 0 (Ascending exclusive), 1 (Descending inclusive) } (Single byte)
Maximum Blocks = u32
Block = As in GP

Node -> Node

--> Header Hash ++ Direction ++ Maximum Blocks
--> FIN
<-- [Block]
<-- FIN
```

### CE 129: State request

Request for a range of a block's posterior state.

A contiguous range of key/value pairs from the state trie should be returned, starting at the given
start key and ending at or before the given end key. The start and end keys are both "inclusive"
but need not exist in the state trie. The returned key/value pairs should be sorted by key.

Additionally, a list of "boundary" nodes should be returned, covering the paths from the root to
the given start key and to the last key/value pair included in the response. The list should
include only nodes on these paths, and should not include duplicate nodes. If two nodes in the
list have a parent-child relationship, the parent node must come first. Note that in the case where
the given start key is not present in the state trie, the "path to the start key" should terminate
either at a fork node with an all-zeroes hash in the branch that would be taken for the start key,
or at a leaf node with a different key.

The total encoded length of the response should not exceed the given maximum size in bytes, unless
the response contains only a single key/value pair. As such, the response may not cover the full
requested range.

Note that the keys in the response are only 31 bytes, as the final key byte is ignored by the
Merklization function.

```
Header Hash = [u8; 32]
Key = [u8; 31] (First 31 bytes of key only)
Maximum Size = u32
Boundary Node = As returned by B/L, defined in the State Merklization appendix of the GP
Value = len++[u8]

Node -> Node

--> Header Hash ++ Start Key ++ End Key ++ Maximum Size
--> FIN
<-- [Boundary Node]
<-- [Key ++ Value]
<-- FIN
```

### CE 131/132: Safrole ticket distribution

Sharing of a Safrole ticket for inclusion in a block.

Safrole tickets are distributed (and included on chain) in the epoch prior to the one in which they
are used. They are distributed in two steps. Each ticket is first sent from the generating
validator to a deterministically-selected "proxy" validator. This proxy validator then sends the
ticket to _all_ current validators.

Protocol 131 is used for the first step (generating validator to proxy validator), protocol 132 is
used for the second step (proxy validator to all current validators). Both protocols look the same
on the wire; the difference is only in which step they are used for.

The first step should be performed shortly after the connectivity changes for a new epoch are
applied. The index of the proxy validator for a ticket is determined by interpreting the last 4
bytes of the ticket's VRF output as a big-endian unsigned integer, modulo the number of validators.
The proxy validator is selected from the next epoch's validator list. If the generating validator
is chosen as the proxy validator, then the first step should effectively be skipped and the
generating validator should distribute the ticket to the current validators itself, as per the
following section.

Proxy validators should verify the proof of any ticket they receive, and verify that they are the
correct proxy for the ticket. If these checks succeed, they should forward the ticket to all
current validators. Forwarding should be delayed until 2 minutes into the epoch, to avoid exposing
the timing of the message from the generating validator. Forwarding should be evenly spaced out
from this point until half-way through the Safrole lottery period. Forwarding may be stopped if the
ticket is included in a finalized block.

If finality is running far enough behind that the state required to verify a received ticket is not
known with certainty, the stream should be reset/stopped. This applies to both protocol 131 and
132.

```
Epoch Index = u32 (Should identify the epoch that the ticket will be used in)
Attempt = { 0, 1 } (Single byte)
Bandersnatch RingVRF Proof = [u8; 784]
Ticket = Epoch Index ++ Attempt ++ Bandersnatch RingVRF Proof

Validator -> Validator

--> Ticket
--> FIN
<-- FIN
```

### CE 133: Work-package submission

Submission of a work-package from a builder to a guarantor assigned to the relevant core.

```
Core Index = u16
Work Package = As in GP
Extrinsic = [u8]

Builder -> Guarantor

--> Core Index ++ Work Package
--> [Extrinsic] (Message length should equal sum of extrinsic data lengths)
--> FIN
<-- FIN
```

### CE 134: Work-package sharing

Sharing of a work-package between guarantors on the same core assignment.

A work-package received via CE 133 should be shared with the other guarantors assigned to the core
using this protocol, but only after:

- It has been determined that it is possible to generate a work-report that could be included on
  chain. This will involve, for example, verifying the WP's authorization.
- All import segments have been retrieved. Note that this will involve mapping any WP hashes in the
  import list to segment roots.

The refine logic need not be executed before sharing a work-package; ideally, refinement should be
done while waiting for the other guarantors to respond.

Unlike CE 133, a full work-package bundle is sent, along with any necessary work-package hash to
segment root mappings. The bundle includes imported data segments and their justifications as well
as the work-package and extrinsic data. The bundle should precisely match the one that is
ultimately erasure coded and made available in the case where the work-report gets included on
chain.

The guarantor receiving the work-package bundle should perform basic verification first and then
execute the refine logic, returning the hash of the resulting work-report and a signature that can
be included in a guaranteed work-report. The basic verification should include checking the
validity of the authorization and checking the work-package hash to segment root mappings. If the
mappings cannot be verified, the guarantor may, at their discretion, either refuse to refine the
work-package or blindly trust the mappings.

```
Core Index = u16
Segment Root Mappings = len++[Work Package Hash ++ Segment Root]
Work Package Bundle = As in GP
Work Report Hash = [u8; 32]
Ed25519 Signature = [u8; 64]

Guarantor -> Guarantor

--> Core Index ++ Segment Root Mappings
--> Work Package Bundle
--> FIN
<-- Work Report Hash ++ Ed25519 Signature
<-- FIN
```

### CE 135: Work-report distribution

Distribution of a fully guaranteed work-report ready for inclusion in a block.

After sharing a work-package received via CE 133 and getting back a signature from at least one
other guarantor, a guaranteed work-report may be constructed and distributed.

Guaranteed work-reports should be distributed to all current validators, and during the last core
rotation of an epoch, additionally to all validators for the next epoch. Note that these validator
sets are likely to overlap.

Once in possession of two signatures for a work-report, the third guarantor should be given a
reasonable amount of time (e.g. two seconds) to produce an additional signature before the
guaranteed work-report is distrubuted.

```
Work Report = As in GP
Slot = u32
Validator Index = u16
Ed25519 Signature = [u8; 64]
Guaranteed Work Report = Work Report ++ Slot ++ len++[Validator Index ++ Ed25519 Signature]

Guarantor -> Validator

--> Guaranteed Work Report
--> FIN
<-- FIN
```

## CE 136: Work-report request

Request for the work-report with the given hash.

This should be used to request missing work-reports on receipt of a new block; blocks directly
contain only the hashes of included work-reports.

A node announcing a new block may be assumed to possess the referenced work-reports. Such nodes
should thus be queried first for missing reports.

```
Work Report Hash = [u8; 32]
Work Report = As in GP

Node -> Node

--> Work Report Hash
--> FIN
<-- Work Report
<-- FIN
```

### CE 137: Shard distribution

This protocol should be used by assurers to request their EC shards from the guarantors of a
work-report. The response should include a work-package bundle shard and a sequence of
exported/proof segment shards. The response should also include a "justification", proving the
correctness of the shards.

The justification should be the co-path of the path from the erasure root to the shards, given by:

```math
T(\mathbf{s}, i, \mathcal{H})
```

Where:

- $\mathbf{s}$ is the sequence of (work-package bundle shard hash, segment shard root) pairs
  satisfying $u = \mathcal{M}_B(\mathbf{s})$, where $u$ is the erasure root and $\mathcal{M}_B$ is
  as defined in the General Merklization appendix of the GP.
- $i$ is the shard index.
- $\mathcal{H}$ is the Blake 2b hash function.
- $T$ is as defined in the General Merklization appendix of the GP.

Note that the last entry in the justification may consist of a pair of hashes.

```
Erasure Root = [u8; 32]
Shard Index = u16
Bundle Shard = [u8]
Segment Shard = [u8; 12]
Hash = [u8; 32]
Justification = [Hash OR (Hash ++ Hash)]

Assurer -> Guarantor

--> Erasure Root ++ Shard Index
--> FIN
<-- Bundle Shard
<-- [Segment Shard] (Should include all exported and proof segment shards with the given index)
<-- Justification
<-- FIN
```

### CE 138: Audit shard request

Request for a work-package bundle shard (a.k.a. "audit" shard).

This protocol should be used by auditors to request work-package bundle shards from assurers in
order to reconstruct work-package bundles for auditing. In addition to the requested shard, the
response should include a justification, proving the correctness of the shard.

The justification should be the co-path of the path from the erasure root to the shard. The assurer
should construct this by appending the corresponding segment shard root to the justification
received via CE 137. The last-but-one entry in the justification may consist of a pair of hashes.

```
Erasure Root = [u8; 32]
Shard Index = u16
Bundle Shard = [u8]
Hash = [u8; 32]
Justification = [Hash OR (Hash ++ Hash)]

Auditor -> Assurer

--> Erasure Root ++ Shard Index
--> FIN
<-- Bundle Shard
<-- Justification
<-- FIN
```

### CE 139/140: Segment shard request

Request for one or more 12-byte segment shards.

This protocol should be used by guarantors to request import segment shards from assurers in order
to complete work-package bundles for guaranteeing.

This protocol has two variants: 139 and 140. In the first variant, the assurer does not provide any
justification for the returned segment shards. In the second variant, the assurer provides a
justification for each returned segment shard, allowing the guarantor to immediately assess the
correctness of the response.

The justification for a segment shard should be the co-path from the erasure root to the shard,
given by:

```math
\mathbf{j} \frown [b] \frown T(\mathbf{s}, i, \mathcal{H})
```

Where:

- $\mathbf{j}$ is the relevant justification provided to the assurer via CE 137.
- $b$ is the corresponding work-package bundle shard hash.
- $\mathbf{s}$ is the full sequence of segment shards with the given shard index.
- $i$ is the segment index.
- $\mathcal{H}$ is the Blake 2b hash function.
- $T$ is as defined in the General Merklization appendix of the GP.

Note that one entry in the justification may be a pair of hashes, and the last entry may be a
segment shard.

Guarantors should initially use protocol 139 to fetch segment shards. If a reconstructed import
segment is inconsistent with its reconstructed proof, the segment and proof should be reconstructed
again, using shards retrieved with protocol 140. When using this protocol, the guarantor can verify
the correctness of each response as it is received, requesting shards from a different assurer in
the case of an incorrect response. If the reconstructed segment and proof are still inconsistent,
then it can be concluded that the erasure root is invalid.

```
Erasure Root = [u8; 32]
Shard Index = u16
Segment Index = u16
Segment Shard = [u8; 12]
Hash = [u8; 32]
Justification = [Hash OR (Hash ++ Hash) OR Segment Shard]

Guarantor -> Assurer

--> [Erasure Root ++ Shard Index ++ len++[Segment Index]]
--> FIN
<-- [Segment Shard]
[Protocol 140 only] for each segment shard {
[Protocol 140 only]     <-- Justification
[Protocol 140 only] }
<-- FIN
```

### CE 141: Assurance distribution

Distribution of an availability assurance ready for inclusion in a block.

Assurers should distribute availability assurances approximately 2 seconds before each slot, to all
possible block authors. Note that the assurer set and the block author set should switch over to
the next validator set for the distribution 2 seconds before a new epoch -- the assurances
extrinsic and block seal are both checked using the posterior keysets.

```
Anchor Hash = [u8; 32]
Bitfield = [u8; 43] (One bit per core)
Ed25519 Signature = [u8; 64]
Assurance = Anchor Hash ++ Bitfield ++ Ed25519 Signature

Assurer -> Validator

--> Assurance
--> FIN
<-- FIN
```

### CE 142: Preimage announcement

Announcement of possession of a requested preimage. This should be used by non-validator nodes to
introduce preimages.

The recipient of the announcement is expected to follow up by requesting the preimage using
protocol 143, provided the preimage has been requested on chain and the recipient is not already in
possession of it.

Preimage announcements _should not_ be forwarded to other validators; validators should propagate
preimages only be including them in blocks they author.

```
Hash = [u8; 32]

Node -> Validator

--> Hash
--> FIN
<-- FIN
```

### CE 143: Preimage request

Request for a preimage of the given hash.

This should be used to request:

- Preimages announced via protocol 142.
- Missing preimages of hashes in the lookup extrinsics of new blocks.

Requests for a preimage should be made to nodes that have announced possession of either the
preimage itself or of a block containing the hash of the preimage in its lookup extrinsic.

Note that this protocol is essentially the same as protocol 136 (work-report request), but the hash
is expected to be checked against a different database (in the case of this protocol, the preimage
lookup database).

```
Hash = [u8; 32]
Preimage = [u8]

Node -> Node

--> Hash
--> FIN
<-- Preimage
<-- FIN
```

### CE 144: Audit announcement

Announcement of requirement to audit.

Auditors of a block (defined to be the posterior validator set) should, at the beginning of each
tranche, broadcast an announcement to all other such auditors specifying which work-reports they
intend to audit, unless they do not intend to audit any work-reports, in which case no announcement
should be sent.

An announcement contains a list of work-reports as well as evidence backing up the announcer's
decision to audit them. In combination with the block being audited and the prior state, the
evidence should be sufficient for recipients of the announcement to verify the audit requirement
claim. Note that although the announcements for any claimed no-shows must be provided, there is no
way to prove/verify that the judgments were not received; this must simply be accepted.

```
Header Hash = [u8; 32]
Tranche = u8
Core Index = u16
Work Report Hash = [u8; 32]
Reports = len++[Core Index ++ Work Report Hash]
Ed25519 Signature = [u8; 64]
Bandersnatch Signature = [u8; 96]
Validator Index = u16
No Shows = len++[Validator Index ++ Reports ++ Ed25519 Signature] (Previous tranche announcements)

Auditor -> Auditor

--> Header Hash ++ Tranche ++ Reports ++ Ed25519 Signature
[Tranche 0] --> Bandersnatch Signature (s_0 in GP)
[Tranche not 0] --> [Bandersnatch Signature (s_n(w) in GP) ++ No Shows] (One entry per WR in the first message)
--> FIN
<-- FIN
```

### CE 145: Judgment publication

Announcement of a judgment, ready for inclusion in a block and as a signal for potential further
auditing.

An announcement declaring intention to audit a particular work-report must be followed by a
judgment, declaring the work-report to either be valid or invalid, as soon as this has been
determined.

Any judgments produced should also be broadcast to the validator set(s) for the epoch(s) following
the block(s) in which the audited work-report was declared available. This is to ensure the
judgments are available for block authors to include in the disputes extrinsic. For positive
judgments, this broadcasting may optionally be deferred until a negative judgment for the
work-report is observed (which may never happen).

On receipt of a new negative judgment for a work-report that the node is (potentially) responsible
for auditing, the judgment should be forwarded to all other known auditors that are linked via the
grid structure. The intent of this is to increase the likelihood that negative judgments are seen
by all auditors.

```
Epoch Index = u32
Validator Index = u16
Validity = { 0 (Invalid), 1 (Valid) } (Single byte)
Work Report Hash = [u8; 32]
Ed25519 Signature = [u8; 64]

Auditor -> Validator

--> Epoch Index ++ Validator Index ++ Validity ++ Work Report Hash ++ Ed25519 Signature
--> FIN
<-- FIN
```