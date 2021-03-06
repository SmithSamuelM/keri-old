# KID0004 - Key Configuration (Signing threshold & key set) - Commentary

## Navigation

[Back to table of contents](readme.md)
|Link|Commentary|Section
|---|---|---|
|[0000](kid0000.md)|[X](kid0000Comment.md)|Glossary, overview, how to use|
|[0001](kid0001.md)|[X](kid0001Comment.md)|Prefixes, Derivation and derivation reference tables|
|[0002](kid0002.md)|[X](kid0002Comment.md)|Data model (field & event concepts and semantics)|
|[0003](kid0003.md)|[X](kid0003Comment.md)|Serialization|
|[0004](kid0004.md)|X|Key Configuration (Signing threshold & key set)|
|[0005](kid0005.md)|[X](kid0005Comment.md)|Next Key Commitment (Pre-Rotation)|
|[0006](kid0006.md)|[X](kid0006Comment.md)|Seals|
|[0007](kid0007.md)|[X](kid0007Comment.md)|Delegation (pending PR by Sam)|
|[0008](kid0008.md)|[X](kid0008Comment.md)|Key-Event State Machine|
|[0009](kid0009.md)|[X](kid0009Comment.md)|Indirect Mode & Witnesses|
|0010||Recovery/consensus Algorithm (KAACE)|
|0010||Database & Storage Considerations|
|0097|n/a|**Non-Normative** Implementation Guidance|
|0098|n/a|Use Cases|
|0099|n/a|Test Vectors and Normative Statement Index|


### Commentary on Key Configuration

(from:2.1)
The simplest form of a self-certifying identifier includes either the public key or a unique fingerprint of the public key as a prefix in the identifier. Because the public key is included in the identifier, the identifier is strongly and securely bound to the (public, private) key pair. This cryptographic secure binding makes the identifier self-certifying.

To restate, a cryptographic strength of 128 bits is strong enough that no adversary may break it by brute force attack with any possible amount of (current prequantum) computational resources. 

Cryptographic strength random numbers may be generated in various ways. A common software based approach is a cryptographically secure pseudo random number generator (CSPRNG) [45]. More sophisticated approaches use special purpose hardware such as trusted platform modules [144]. In any case, inexpensive methods to collect sufficient entropy to generate a cryptographic strength random number are readily available. What this means is that any entity may capture sufficient entropy to generate a cryptographically strong random seed. But once an entity captures that entropy, in a seed, no other entity may practically reproduce that seed; the likelihood is infinitesimally small. This makes the original capturing entity the sole holder, possessor, or controller over that random seed. It is thereby the sole sovereign, or the sole authority over that seed. The holder of that seed obtained control of that seed by virtue of capturing entropy without deference or permission or dependance on any other entity or authority. This is true decentralized control.

(from: 2.2)
Given current computing power, each cryptographic operation in the process of generating the self-certifying identifier should maintain no less than 128 bits of cryptographic strength. Each operation is a type of cryptographic one-way function [110]. A one way function is relatively easy to compute going in one direction but extremely hard to reverse (invert) [109]. A one-way function with 128 bits of security would take on the order of 2128 operations to invert. 

ECC scalar multiplication is a type of one-way function.
The cryptographic strength of a scalar multiplication compliant with the Ed25519 scheme using a 256 bit long scalar is no less than 128 bits [26; 87]. 
This means that the size of the underlying field should be roughly twice the security parameter.

Given a 128 bit random seed, the next step in generating a self-certifying identifier is to create an asymmetric (public, private) key pair. If the required length of the private key is 256 bits (as is the case for Ed25519) then we first need to stretch the random seed from 128 bits (16 bytes) to 256 bits (32 bytes). This stretched seed then becomes as an input to the Ed25519 key generation algorithm [48]. Key stretching is another type of one-way function [90]. A good algorithm for key stretching is Argon2 [15; 112]. Once stretched, this 256 bit seed becomes the private key. Using the LibSodium code library we can create the associated public key. Under the hood the Ed25519 code hashes the private key/seed before performing the scalar multiplication on the results to compute the public key. But this hashed value is never exposed to the user. The resultant 32 byte public key may then be encoded and used to generate the self-certifying identifier. 

Cryptographic operations produce large binary numbers. Binary number format is not very useful for identifiers in most applications. Identifiers are better represented as strings of characters. Thus in order to use a public key to create an identifier, we first need to convert it to a string of characters. A highly interoperable and relatively compact encoding is RFC-4648 Base-64
(URL safe) [85]. Base-64 encodes every 3 bytes of a binary number into 4 ASCII characters. When the N byte binary number is not an exact multiple of 3 bytes there will be either 1 or 2 bytes of pad characters added to the end of the Base-64 encoding. A 32 byte public key would encode to 44 Base-64 characters with one trailing pad character inclusive. For example consider the following 32 byte private key represented as a 64 character hex string:
0caac9c64711f66e6ed71b37dc5e69c5124fe93ee12446e1a47ad4b650dd861d
This is encoded as the following 44 character Base-64 string (includes one trailing pad character):
DKrJxkcR9m5u1xs33F5pxRJP6T7hJEbhpHrUtlDdhh0=

(from:2.2.1 Derivation Codes)
*Note: merge with Section 14 - Derivation Codes*

To properly extract and use the public key embedded in a self-certifying identifier we need to know the cryptographic signing scheme used by the key pair. In this case we need to know that the key-pair follows Ed25519 and is used for signing. This information would also allow us to infer the length of the public key. This is often referred to as the cypher-suite and operation. In general this provides the process used to derive the self-certifying identifier. This derivation information must either be assumed or included in the identifier. One way to include this very compactly in the identifier is to replace the pad character with a special character that encodes the derivation process. Call this the derivation code. Because this derivation information is needed to correctly parse the encoded public key and the convention is to parse from left-to-right, we prepend the derivation code to the public key and delete the pad character. The result is still 44 characters in length. For example suppose that the 44 character Base-64 with trailing pad character for the public key is as follows:
F5pxRJP6THrUtlDdhh07hJEDKrJxkcR9m5u1xs33bhp=

If B is the value of the derivation code then the resultant self-contained string is as follows:
BF5pxRJP6THrUtlDdhh07hJEDKrJxkcR9m5u1xs33bhp

Now the derivation code is part of the identifier. This binds both the public key and its derivation process to the identifier. Because the string is valid Base-64 it may be converted to binary before being parsed with binary operations or it may be parsed before conversion. If parsed before conversion, the derivation code character must be extracted from the front of the string. The first character tells how to parse the remaining characters including the length. This allows compact but parseable concatenation of cryptographic material. One pad character must be re-appended before converting the remaining characters that comprise the encoded public key back to binary. Once converted the binary version of the public key may be used in cryptographic operations. Proposed sets of derivation codes for KERI are provided in Section 14.2. Each is either 1, 2 or 4 characters long to replace 1, 2 or 0 pad characters respectively. This ensures that each selfcontained identifier string is compliant with Base-64 specification which must be a multiple of 4 base-64 characters. This allows conversion before parsing which may be more efficient. This approach to encoding the derivation process is similar to that followed by the multicodec standard but is more compact because it efficiently exploits pad bytes and only needs to support cryptographic material in KERI events [103]. Furthermore a KERI design goal is to support cryptographic agility for all the cryptographic material in its events. A compact derivation code makes supporting this degree of granular cryptographic agility much more efficient. The derivation codes are not only used for self-certifying identifier prefixes but also for keys, signatures, and digests. For example, the Base-64 representation of the private key from the end of Section 2.2 above may be encoded with a derivation code as follows:
ADKrJxkcR9m5u1xs33F5pxRJP6T7hJEbhpHrUtlDdhh0
This enables use of derivation codes for efficient representation of cryptographic material throughout the key management infrastructure used to support KERI.

(from:2.2.2) 
Inception Statement

Practical use of a self-certifying identifier may require some initial configuration data. We call this the inception data and it is formally represented in a signed inception statement. The inception data must include the public key, the identifier derivation from that public key, and may include other configuration data. The identifier derivation may be simply represented by the derivation code. A statement that includes the inception data with attached signature made with the private key comprises a cryptographic commitment to the derivation and configuration of the identifier that may be cryptographically verified by any entity that receives it. It is completely self-contained. No additional infrastructure is needed or more importantly must be trusted in order to verify the derivation and initial configuration (inception) of the identifier. The initial trust basis for the identifier is simply the signed inception statement. A diagram of a basic inception statement is shown below: 
(Figure 2.2)

(from: )
A *hidden signing threshold and public key set* is a signing threshold specifying followed by a set of qualified public keys where the set???s derivation includes an additional one-way hashing step that produces a digest of the the serialized threshold specifier and set of qualified public keys. In which case the actual public keys are not provided explicitly, but merely the digest, hence hidden. This may be useful in making a compact verifiable cryptographic commitment to a set of public keys that may be disclosed in the future.
