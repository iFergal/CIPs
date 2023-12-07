---
CIP: ?
Title: Verifiable Supply Chain Records
Category: Metadata
Status: Proposed
Authors:
    - Fergal O'Connor <fergal.oconnor@cardanofoundation.org>
    - Darlisa Consoni <darlisa.consoni@cardanofoundation.org>
Implementors: ["Cardano Foundation"]
Discussions:
    - https://github.com/cardano-foundation/cips/pulls/<todo>
Created: 2023-12-07
License: CC-BY-4.0
---

## Abstract
We propose a mechanism to create a verifiable data standard for supply chain records.
This specification describes a way to present batched verification information (signatures, hashes) of off-chain supply chain data in a robust, yet flexible manner.
The metadata label of 1904 is based on the year of birth of Joseph M. Juran who was an American engineer and management consultant who is widely regarded as one of the leading figures in the development of quality management.

## Motivation
One of the main challenges that the supply chain space faces is the lack of transparency and accountability due to the complex flow of products and information shared among all players involved in the supply chain ecosystem.
This usually can lead to errors, delays, and even fraud or counterfeiting.

Having a standard for supply chain and Certificate of Conformity data on the blockchain can help to address this issue by providing a common language for all parties to communicate, share and most importantly verify information.
This makes it easier to track products and transactions throughout the supply chain process by reducing errors and increasing efficiency, transparency, and accountability.
By recording transactions on a decentralised ledger, it becomes much harder to manipulate or falsify data.
This can help to prevent fraud, counterfeiting, and other types of criminal activity. 

In order to add industry-compliant metadata format, we need to define a common schema that can be used by all parties involved in the supply chain ecosystem which can be easily identified by explorers and other blockchain applications with the 1904 metadata label.
This benefits supply chain stakeholders, businesses, and consumers.

## Specification
This CIP intends to cover a broad range of use-cases in the supply chain ecosystem.
The specification will define a process of how off-chain supply chain data, that is signed and hashed can be coupled with an on-chain transaction containing as metadata the hash, signatures and any other relevant information such as public keys.
Our goal is also to be efficient which is why a single transaction may contain a batch of many signatures.

For a given verifiable data record within a batch, this specification does not make strict enforcements on the schema of the off-chain data as this is use-case dependent.
It only makes enforcements on the format and ordering of the batch of verifiable data records, and the corresponding on-chain metdata.

A use-case may be registered in this specification as a subtype (`st`) and the subtype itself should dictate the off-chain data schema. It should also determine how off-chain data and public keys are retrieved and validated.

### General structure and verification information
As previously mentioned, every transaction is a batch containing several verifiable data records.
All off-chain data for a batch is in JSON format.
The specific layout of a batch is determined by the `type` described later.

A batch may contain an arbitrary number of signatures from an arbitrary number of parties (if applicable).
As we are working with off-chain JSON, every signature must be a [JSON Web Signature](https://datatracker.ietf.org/doc/html/rfc7515) (JWS - RFC-7515) and the JWS header must be included in the transaction metadata.

A single hash is taken of the entire JSON batch and must be represented as a [Content Identifier](https://github.com/multiformats/cid) (CID).
Usage of CIDs allow us to represent the hash in a self-describing way that is not ambiguous.
The subtype `st` may limit the types of CIDs used in order to not require 3rd party verification applications to support too many hash algorithms.

The CID itself, which is embedded in the metadata of the transaction is used as an identifier for the batch and hence we recommend using this as a unique identifier to resolve the off-chain data.

### Transaction types
Currently, there are 3 different types of verifiable supply chain records supported:
- Supply Chain Management data,
- Certificate of Conformity issue,
- Certificate of Conformity revoke

The [CDDL schema](./schema.cddl) covers all transaction types.

#### Supply Chain Management data
Supply Chain Management (SCM) data is associated with a good or product as a result of production, manufacturing and transport processes throughout its lifecycle.
This data may be shared between all supply chain stakeholders as well as delivered to end customers.
A producer is responsible to collect the SCM data associated with its product/good and insert it into the supply chain ecosystem in a defined manner specific to the use-case.

We strongly recommend that the off-chain SCM schema follows an industry standard relevant to the particular use-case.
For example, the Georgian Wine use-case from the Cardano Foundation with the National Wine Agency of Georgia uses an extended version of the [OIV Standard](https://www.oiv.int/standards/international-standard-for-the-labelling-of-wines) from the [International Organisation of Vine and Wine](https://www.oiv.int).
Using such standards brings credibility to the transactions and helps foster enterprise adoption.

##### Off-chain format
Each verifiable record comes from a specific producer by `producerID`.
An SCM batch can contain records from multiple producers, hence the off-chain data is a JSON map of `producerID` to an array of JSON objects.
The structure of the JSON objects within the array is dependant on the subtype `st`.

The `producerID` is a unique identifier for a producer in the supply chain ecosystem of the given subtype.
We recommend short IDs if possible to save on space and costs.

The following example contains 3 verifiable records from producer `a2gh` and producer `p33l`, where `{...}` represents an arbitrary JSON object for a given verifiable record:
```
{
  a2gh: [{...}, {...}, {...}],
  p33l: [{...}, {...}]
}
```

##### On-chain format
The following field labels are used within the on-chain metadata:

|**Field**|**Description**|**Type**|**Set Value**|
|---|---|---|---|
|`t` for type|The type defines the structure of the data.|string|scm|
|`st` for subtype|This **optional** field can be used to specify/describe a use-case. Please see the Subtypes section for more information.|string||
|`v` for version|The version of the specification.|string|1|
|`cid`|The Content Identifier of the off-chain data as defined by [Multiformats](https://github.com/multiformats/cid).|string||
|`d` for verification data|Map of `producerID`s to their respective verfication information.|map||
|`pk` for public key|The public key belonging to `producerID`. The field accepts the direct byte stream of the public key (embedded), or alternatively accepts a URL to the public key. The public key URL when resolved must serve the raw bytes using a HTTP Content-Type of `application/octet-stream`. Public keys must be verified by some predefined process relevant to the subtype. In case a particular producer performs a key rotation within a single transaction, `pk` can accept an array of URLs or byte streams. In this scenario, `h` must also be converted into an array of headers, and `s` must be converted into an array of arrays, where each top-level array represents all signatures for a given public key.|byte-stream or array of byte-streams||
|`pkv` for public key version|If public keys are embedded as a byte stream, this optional field indicates the version number of the public key. This allows us to support key rotation. If omitted, the version of an embedded public key is assumed to be the first with index `0`. If a key rotation occurs within a single transaction (the `pk` field is an array), `pkv` must be an array of matching, in-order versions to avoid ambiguity.|uint or array of uints||
|`h` for JWS header|This is the JSON Web Signature header for the signatures created with the key pair. This is in raw byte stream format. In case the header is > 64 bytes, the byte stream must be broken up into an array of byte streams. The header must indicate the type of key pair used - for example, `{ alg: "EdDSA" }` for Ed25519 key pairs.|byte-stream or array of byte-streams||
|`s` for signatures|An array of signatures where each item is a detached JSON Web Signature of the corresponding item in the off-chain array associated with the `producerID`. Each individual signature is in a raw byte stream format. In case the signature is > 64 bytes, the byte stream must be broken up into an array of byte streams. In case a signature is > 64 bytes, the byte stream must be broken up into an array of byte streams. In theory, some signatures may be <= 64 bytes, and others may be > 64 bytes so there may be a mix of formats within the array.|array of signatures, where each signature is a byte-stream or array of byte-streams (nested)||

**Note:** Some of the optional array rules described above might be stacked. In the simple case, the `s` field representing signatures is an array of byte streams if there is a single public key, and all signatures are at most 64 bytes long. However, if a key rotation occurs, and some signatures are > 64 bytes, `s` becomes a 3–level deep array. Note that not all signatures need to be split, `<pk1Sig1Bytes>` is less than 64 bytes.
```
[
  [
    [<pk0Sig0BytesPart0>, <pk0Sig0BytesPart1>],
    [<pk0Sig1BytesPart0>, <pk0Sig1BytesPart1>]
  ],
  [
    [<pk1Sig0BytesPart0>, <pk1Sig0BytesPart1>],
    <pk1Sig1Bytes>
  ]
]
```

##### On-chain example
This example is in JSON for ease of readability, as if it were displayed in an explorer.
This also means the signatures, public keys and JWS headers are all in hex encoding, but in reality the CBOR metadata uses byte-streams.
Please refer to the [CDDL](./schema.cddl) for the correct schema.

```
{
  "t": "scm",
  "v": "1",
  "st": "georgianWine",
  "d": {
    "a2gh": {
      "s": [
        "0c2b48602f8aefde43fbab49e7225d50f02d091f508d9dc84e538b4d656940d57301141070612016a50a14f26976cb20378b40a965f63d125dd0bc18fae1f00c",
        "7b226b6964223a2237326363346636342d623238312d343038622d383430342d393966396264646538393139222c22616c67223a224564445341227d"
      ],
      "h": "7b22616c67223a224564445341227d",
      "pk": "0e459119216db26ba7fec7a66462ed4502ef5bf5e25f447ce134e0acccf2be6f",
      "pkv": 3
    },
    "p33l": {
      "s": [
        "727b7dbbdacf861297cc25fe4f58ca62fe6a3bbde4577562abb348182e49ca7f634ab76bff00224a5e91ab47b3b24e6547e8edf4d248fb9211c0f1e534cf6f0a",
        "27e03340999899aa32712639ca895656c39d50524316d8b7a8abd2600fba270b583c31c4a2052789c7b11b4c1e11ffd0d5d8a301e53590d682e8f5bf54349b0c"
      ],
      "h": "7b22616c67223a224564445341227d",
      "pk": "095f9a1a595dde755d82786864ad03dfa5a4fbd68832566364e2b65e13cc9e44"
    }
  },
  "cid": "zCT5htke9ZJ963Tsk5hDXizLzVQTha63ojw4X6SyWa3EBXtmWTPQ"
}
```

This is the basic example of 2 producers with 2 verifiable data records each.
The public key version is explicitly set for the first producer `a2gh` as some key rotations have occured prior to this transaction.

Something that is important to note is that the on-chain and off-chain ordering of items must match.
The order of signatures in each producer's verification information must match the corresponding order of supply chain data objects in the off-chain data given the same `producerID`.

Please see the more [complex examples](./examples/complex) to see how to form metadata for key rotations, byte streams >64 bytes, etc.

#### Certificate of Conformity issue
A Certificate of Conformity is a document issued by an authoritative entity in the country responsible to verify the quality and authenticity of the goods inserted into the supply chain.
The Certificate of Conformity is internationally required for goods to be exported to other countries. 
This authoritative entity is responsible for ensuring that products or services meet the international quality, specifications, and regulations standards throughout the supply chain process required in the origin country.
This entity may use a range of techniques and tools for testing, inspection, and auditing processes, as well as quality management systems and certifications.

##### Off-chain format
Each verifiable data record corresponds to an issued Certificate of Conformity.
In this scenario, there is assumed to be a single authorative entity. For this reason, a batch of certificates is simply a JSON array.

Here, each `{...}` should be a JSON representation of an issued Certificate of Conformity:
```
[{...}, {...}, {...}]
```

##### On-chain format
A subset of the same field labels as `scm` are used in this type of transaction. The only exception is that the `type` must be equal to `conformityCert`.



##### On-chain example
Once again, this example is in JSON for ease of readability, as if it were displayed in an explorer.
The signatures, public keys and JWS headers are all in hex encoding, but in reality the CBOR metadata uses byte-streams.
Please refer to the [CDDL](./schema.cddl) for the correct schema.

```
{
  "t": "conformityCert",
  "v": "1",
  "st": "georgianWine",
  "s": [
    "0c2b48602f8aefde43fbab49e7225d50f02d091f508d9dc84e538b4d656940d57301141070612016a50a14f26976cb20378b40a965f63d125dd0bc18fae1f00c",
    "7b226b6964223a2237326363346636342d623238312d343038622d383430342d393966396264646538393139222c22616c67223a224564445341227d"
  ],
  "h": "7b22616c67223a224564445341227d",
  "pk": "0e459119216db26ba7fec7a66462ed4502ef5bf5e25f447ce134e0acccf2be6f",
  "pkv": 3,
  "cid": "zCT5htke9ZJ963Tsk5hDXizLzVQTha63ojw4X6SyWa3EBXtmWTPQ"
}
```

This example contains 2 Certificates of Conformity that were both signed using the key pair after multiple key rotations.

Note that the order of signatures in `s` must correspond and match the order of JSON certificates in the off-chain array.

The more [complex examples](./examples/complex) are for type `scm`, but the principles apply to this type in the same manner.

#### Certificate of Conformity revoke
On occasions where the data in a Certificate of Conformity is incorrect, or the certificate is no longer needed (selling of the goods were cancelled), the authoritative entity should be able to revoke the certificate on-chain.

##### Off-chain format
The off-chain of a certificate revocation must contain a JSON object with enough identifying information to mark a previoulsy issued certificate as revoked.
This will be use-case specific as the identifying information must be publicly available.

For example, in the `georgianWine` use-case, the unique identifier of a certificate is not publicly shared but the public attributes `certificate_number` and `certificate_type` together act as a secondary unique identifier.

The format is again an array to allow for batch revocations:
```
[{...}, {...}, {...}]
```

##### On-chain format
The on-chain format of certificate revocation is identical to the on-chain format for issuing, except the `type` must be `conformityCertRevoke`.
Please see an example [here](./examples/conformityCertRevoke.json).

### Registered subtypes
This metadata label may be used for a variety of use-cases, and for this reason, the `st` field may be used to specify a specific use-case.
This allows 3rd party applications that are only interested in a specific use-case to filter by a particular subtype, but still allow us to maintain a standard across the supply chain space.

Registered subtypes must also provide a clear method to retrieve the off-chain data and verify any public keys are authentic. Embedded public keys must have a clear way to cross-reference for authenticity, and public key URLs must only come from trusted domains for the given subtype.

**Note:** The subtype field is optional! You may choose to omit the subtype in favour of a smaller transaction fee provided there is some other way of retrieving authentic transactions. For example, this might be 1904 labelled transactions that originate from a specific address.

Registered subtypes and their descriptions can be found in [subtypes.json](./subtypes.json).

## Rationale
The CIP provides a clear way to present verification data for supply chain data records.
To tackle the possible scale of such a large ecosystem, the specification pays close attention to mechanisms such as batching and key rotation.
The use of subtypes allows us to capture a broad variety of use-cases and not make opinionated decisions on off-chain data formats.

## Path to Active

### Acceptance Criteria
This has already been implemented by the Cardano Foundation for the Georgian wine use-case.
Acceptance as a CIP should be determined if

### Implementation Plan
The Cardano Foundation have developed a proof of origin application to integrate with the National Wine Agency of Georgia (`georgianWine` subtype).
Example preprod `scm` [tx](https://preprod.beta.explorer.cardano.org/en/transaction/ad770e46fdfbc3cd656aa1d1e7ac97cdc5349f0b2535c269d153c8b7bee39c43/metadata) and `conformityCert` [tx](https://preprod.beta.explorer.cardano.org/en/transaction/fdcacdc933cbf69342b4afff04f5bbdd77c915d8d6e6c4dea8136424439b0df2/metadata).

## Copyright
This CIP is licensed under [CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/legalcode).
