NRP-1
=====

Binary event encoding
---------------------

`draft` `optional` `author:dr-orlovsky`

At the present moment nostr (NIP-1) uses text serialization of the event data,
represented in a form of a JSON object. While being simple-to-implement and
web-compatible, it has some significant drawbacks:

* Speed: serialization from JSON consumes most of the app time, according to
  both [Damus app] and [Nostr creator].
* Complex handling of binary data, which have to be encoded into Base64 or some
  other binary-to-ascii format, increasing the problem described above and
  leading to more traffic consumption.
* No explicit limits for the size of the data (i.e. it is not clear what is the
  maximal size of the message, or the maximal number of tags), leading to DoS
  vulnerability and interoperability. While each relay is free to choose the
  limits to prevent DoS attacks, the limits are not yet there, opening an attack
  surface which may down the network for some time. Also, absence of the
  standard will lead that some of the relays will stop propagating some events
  (as a part of DoS prevention), while others will support all of them, making
  it hard to get the predictable and consistent user experience.
* Protocol extensibility: with the current version of the protocol it is
  prohibited to add new fields to the JSON format.

These all problems can be solved with introducing binary encoding to the
messages with strict limits to the maximal size of the elements and optional TLV
extension stream.

Specification
-------------

The proposed binary encoding must follow this spec (in pseudocode):
```haskell
Event ::
    -- fields matching NIP-1:
    Id, Pubkey, Timestamp, Kind, Tags, Content,
    -- field matching NIP-3:
    OtsHash,
    -- Type-length-value encoded stream:
    TLVs,
    -- NIP-1 signature
    legacy Signature
    -- new signature over binary data (except legacy sig)
    binary Signature

Id :: [Byte ^ 32]     -- indicates fixed-size byte string of exactly 32 bytes
Pubkey :: [Byte ^ 32]
Timestamp :: U32 -- indicates 32-bit unsigned integer
Kind :: U64 -- indicates 64-bit unsigned integer
OtsHash :: [Byte ^ 32]

Tags :: [Tag ^ 0..0xFFF] -- indicates array of tags with up to 4096 elements
Tag :: TagName, TagData
TagName :: [AsciiPrintable ^ 1..0xFF] -- indicates string from 1 to 255 chars
                                      -- containing only printable ASCII chars
                                      -- (bytes 0x20-0x7e)
TagData :: [Entry ^ 0..0xFF]
Entry :: [Byte ^ 1..0xFF]     -- inticates non-empty byte string up to 255 bytes

Content :: [Byte ^ 0..0xFFFFFF] -- inticates byte string up to 2^24 bytes
Signature :: [Byte ^ 64] -- indicates Shnorr signature

-- Type-length-value encoded stream
TLVs :: [TLVEntry ^ 0..0xFF]
TLV :: type U32, value [Byte ^ 0..0xFFFF]
```

The pseudocode represents in-memory and network serialization layout of the
binary data, organized in form of fields listed after `::`. All integers are
serialized in little-endian format (to match the bitcoin protocol); hashes,
public keys and signatures match bitcoin consensus rules for serialization.
Variable-size collections (i.e. excluding id, pubkey and signatures) are
prefixed with either a single byte, a 16-bit or 24-bit word (in little-endian
encoding) matching the bit dimensionality for the maximum number of elements and
specifying the number of elements in the collection. The used notation follows
[strict type] system notation for a memory binary data layout, which can't be
deterministically specified if a protocol buffers or similar notation is used.

### Signatures

The binary events are provided with two signatures: a legacy one, created over
JSON-serialized message without TLV stream (see [Backward Compatibility](#backward-compatibility)
section below); and a new signature, created over binary-serialized event data,
including TLV stream but excluding the legacy signature.

### Maximal length

The maximal total length of the binary-serialized tags must not exceed 16MB
(0xFFFFFF bytes); thus the total length of the event can't be larger than
50 267 340 bytes (~50 MB).

### TLVs

Type-length-value encoded fields define type as a 32-bit tag. It's highest bit
(bit mask 0x80000000) signals whether the client must know the tag in order to
process the event: if the mask is not set and the client doesn't know the tag
it must ignore the event. This is like an "it's ok to be odd" rule in Lightning
network messages [BOLT-1]; however instead of tag parity, signalled by the least
significant bit, we use a bit mask, signalled by the most significant bit, which
simplifies human readability.


Backward compatibility
----------------------

Clients and relays not supporting the current proposal must ignore the events.

Clients and relays supporting the proposal must provide a NIP-1 &
NIP-3-compatible JSON encodings of the binary data for the legacy (pre-NIP-88)
clients and relay, where:
* `content` is given in the Base64 encoding;
* `ots` is given in the Base64 encoding;
* `signature` over binary data is skipped;
* `tlv` stream is skipped;
* `tag` entries are provided in lowercase hexadecimal (base64) encoding.

The proposal is not compatible with the current nostr protocol and requires a
binary protocol, which will be described in a dedicated NIP.


[Damus app]: https://nostrexplorer.com/e/9d89ef468ec690f2177635b8c0fd589aca840de1a749a08a44de5b148654bb49
[Nostr creator]: https://github.com/nostr-protocol/nips/blob/44ea6d84583b64f6d7d994bfb480a4d786ba1699/93.md
[strict type]: https://github.com/strict-types/strict-types
[BOLT-1]: https://github.com/lightning/bolts/blob/master/01-messaging.md
