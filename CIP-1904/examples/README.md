The following are examples of how the on-chain metadata would look like in an explorer.
This means in JSON format with byte-streams hex-encoded.

Please see the [CDDL](../schema.cddl) for the __correct__ schema with byte streams.

The [complex directory](./complex) contains examples of key rotation, public key URLs and > 64 byte length values.
As mentioned in the spec, some of these complex scenarios can "stack" or "mix" - for example, rotating a key that is greater than 64 bytes in length, or simply a transaction where both a private key and signature are > 64 bytes in length.
This means the examples are not exhaustive but the current set, coupled with the CDDL, should make it clear.
