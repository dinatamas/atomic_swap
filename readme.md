Atomic Swaps
============

A Proof-of-Concept implementation of cross-chain atomic swaps.

Theoretical Background
----------------------

### Elliptic Curve Cryptography

The `elliptic_curve` directory contains the implementation of elliptic curve cryptography functions
which will be considered primitives when working with signatures. First there is `field.py` which
constitutes the framework for performing modular arithmetic over a prime field. Then in `curve.py`
there is an implementation of Weierstrass elliptic curve groups over prime fields, where the group
elements are points that satisfy a specified Weierstrass equation (and an extra 'infinity' point).
With these mathematical constructs in place, `secp256k1.py` instantiates the actual elliptic curve
that will be used and provides additional parameter values as specified for use within Bitcoin.
This module also provides functions to generate keys over the Secp256k1 curve.

[1](https://en.wikipedia.org/wiki/Modular_arithmetic)
[2](https://en.wikipedia.org/wiki/Finite_field_arithmetic)
[3](https://en.wikipedia.org/wiki/Modular_multiplicative_inverse)
[4](https://en.wikipedia.org/wiki/Elliptic_curve_point_multiplication)
[5](https://jeremykun.com/2014/02/16/elliptic-curves-as-algebraic-structures/)
[6](https://crypto.stackexchange.com/q/70507)
[7](https://fission.codes/blog/everything-you-wanted-to-know-about-elliptic-curve-cryptography/)
[8](https://cryptobook.nakov.com/asymmetric-key-ciphers/elliptic-curve-cryptography-ecc)
[9](https://github.com/bitcoin-core/secp256k1)
[10](https://onyb.gitbook.io/secp256k1-python/)
[11](https://www.secg.org/sec2-v2.pdf)
[12](https://en.bitcoin.it/wiki/Secp256k1)

### Schnorr Signatures

Suppose Alice has a message m and wants to send it to Bob. To prove to Bob the authenticity of the
message she will create a digital signature s that accompanies the message. If Bob knows Alice's
public key then he can verify that in fact the signature for message m was produced by someone who
knows Alice's private key, i.e. Alice herself. This is the most common 1-of-1 case, but there are
also m-of-n signatures where out of a total of n authorized signers at least m of them must co-sign
the message to produce a valid verifiable aggregated signature. For atomic swaps we are interested
in 2-of-2 signatures, i.e. both parties must sign a message to make it acceptable.

The Schnorr signature scheme is a popular choice for implementing such aggregated signatures due to
its nice mathematical properties. If we have Alice's signature s1 and Bob's signature s2 then we
can produce the 'common' signature by simply adding them together: s = s1 + s2. This property
allows us to implement multi-signature schemes very easily.

[1](https://en.wikipedia.org/wiki/Schnorr_signature)
[2](https://asecuritysite.com/encryption/schnorr_test3)
[3](https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki)
[4](https://github.com/bitcoin/bips/blob/master/bip-0340/reference.py)
[5](https://tlu.tarilabs.com/cryptography/introduction-schnorr-signatures)

### MuSig Protocol

MuSig is an improved version of the Schnorr signature scheme which is less susceptible to certain
types of attacks. This is the algorithm implemented in `musig.py`. To perform a MuSig signature,
a MuSigSession object must be instantiated. This is basically a container for relevant information
for one signing participant. It has 'stored' attributes which must be provided from external
sources (e.g. generated keys, other parties' public keys, etc.) and 'calculated' attributes which
are derived automatically.

The MuSig protocol consists of the following steps:

1. The parties agree on a message to be signed.
2. Each party generates a key pair and broadcasts their public key.
   * From these the common public key can be derived.
3. Each party generates a nonce key pair and broadcasts the hash of their public nonce.
4. Upon receiving every public nonce hash, the actual nonce public keys are broadcast.
   * The parties must verify that the received nonce public keys match the previous hashes.
   * From this the common nonce public key can be derived.
5. Each party generates a signature for the message and broadcasts their signature.
   * The common signature is the sum of all signatures.

This protocol is resilient against malicious signers up to the last step. If a bad actor does not
transmit their own keys or the hash they produced does not match the nonce public key they provided
then the well-meaning signers can simply abort the signature attempt and no harm was done. In the
last step however if a single signer withholds their signature while receiving everyone else's then
they will be the only one who know the signature. This must be taken into consideration when using
the protocol.

[1](https://blog.blockstream.com/en-musig-key-aggregation-schnorr-signatures/)
[2](https://allquantor.at/blockchainbib/pdf/maxwell2018simple.pdf)
[3](https://tlu.tarilabs.com/cryptography/The_MuSig_Schnorr_Signature_Scheme)
[4](https://asecuritysite.com/encryption/schnorr_test)

### Adaptor Signatures

The MuSig protocol is useful for co-signers to prove their agreement on a single message.
Complications arise when multiple self-interested parties want to co-sign a set of messages where
the parties may want to cheat - if given a chance - and only produce common signatures for only a
subset of all messages. In this case the non-atomicity of step (5) of the MuSig protocol poses a
problem. For simplicity's sake let's only consider the 2-party 2-message scenario which is at the
heart of the atomic swap problem. Alice wants Bob to sign her message m1 and Bob wants Alice to
sign his message m2. The following demonstrates how adaptor signatures solve this problem.

**Notation:**

* x: private key
* X: public key
* m: message
* r: private nonce
* R: public nonce
* t: hash of R (used in MuSig)
* o: private offset
* O: public offset
* s: signature
* S: offset signature

Indexing:

* Substring 1 / 2: Refers to the messages (m1 / m2).
* Substring A / B / C: Refers to the parties (Alice / Bob / common).

**Communication:**

    Alice                             Bob

      |                                |
      |         (1) m1, m2, XA         |
      |----------------------------->>>|
      |                                |
      |             (2) XB             |
      |<<<-----------------------------|
      |                                |
      |          (3) tA1, tA2          |
      |----------------------------->>>|
      |          (3) tB1, tB2          |
      |<<<-----------------------------|
      |                                |
      |          (4) RA1, RA2          |
      |----------------------------->>>|
      |          (4) RB1, RB2          |
      |<<<-----------------------------|
      |                                |
      |        (5) O, SA1, SA2         |
      |----------------------------->>>|
      |                                |
      |            (6) sB1             |
      |<<<-----------------------------|
      |                                |
      |            (7) sC1             |
      |----------------------------->>>|
      |                                |

**Details:**

1. Initial signing request: Alice asks Bob if he wants to participate in the agreement - i.e. if
   he wants to co-sign messages m1 and m2 with her.
   * Alice generates her 'permanent' private and public keys (xA, XA). These are similar to GPG or
     SSH keys in that they can be reused between signing sessions. Therefore if Alice already has
     such a key pair then there is no need for her to generate a new one.
2. Upon receiving the signing request, Bob can decide if he accepts the terms (the messages m1 and
   m2) and if he does, he too will need to send his 'permanent' public key over to Alice. If he
   decides to decline the offer he may inform her about his decision, but even if he stays silent
   Alice should give up after a while (timeout).
   * In general, whenever a party is closing the session or not answering for some time, the other
     party too can drop the connection. The protocol is designed to be resilient against such
     behavior.
   * When the parties have exchanged public keys they both can calculate the common public key XC.
     The final common signatures will be verified against this.
3. To proceed with the MuSig scheme both parties generate and exchange 'temporary' nonce values.
   These are also pairs of private-public keys but they must not be reused between signatures!
   * Every signature must have their own nonce values, therefore Alice and Bob both have 2-2
     nonces. For example Bob's private nonce for message 1 is called rB1.
   * In step 3 only the public nonce hashes (labelled with t) are exchanged.
4. Once a party has received the correct number of hashes they can start broadcasting their public
   nonces. Whenever they receive a public nonce they must verify it against the list of received
   hashes. If there is a mismatch the session must be aborted because someone is either not
   implementing the protocol correctly or is deliberatly acting maliciously.
   * Once all public nonces are shared the parties can calculate the common public nonce RC.
5. This is where adaptor signatures come into play and where the MuSig non-atomicity is fixed.
   * Alice generates a new pair of keys called offsets (o and O). She sends Bob the public offset
     as well as her offset signatures SA1 = sA1 + o and SA2 = sA2 + o. Without the private offset
     Bob will not be able to recover Alice's original signatures, but with the public offset he can
     verify that Alice is indeed sending valid data and not just random noise to cheat him.
6. If Bob assured that Alice's offset signatures are valid he sends over his original signature to
   Alice's message m1. He can safely do this because once Alice uses the common signature for
   message 1 he will be able to recover Alice's original signature and the private offset, which is
   enough for him to close his part of the transaction.
7. Once Alice receives Bob's signature for her message she has received everything she needed. She
   can simply calculate sC1 = sA1 + sB1 and use it for her own purposes. The most important
   prerequisite for using this protocol is that Alice's use of the aggregated signature must result
   in her publishing it to a public medium from which Bob can read it. If this is not the case then
   Bob should not have partaken in the signature process. If this condition is true then Bob will
   be able to finish his side of the transaction (i.e. produce the common signature for m2) without
   the cooperation of Alice - although she may be kind and send her own signature to Bob to make
   his job a tiny bit easier.
   * Upon Bob reading the common signature for message 1 from the public ledger he can recover the
     private offset Alice used with the following formula: o = -(sC1 - SA1 - sB1). Once he knowns
     the offset he can simply calculate Alice's signature from the offset signature SA2 she sent
     him previously: sA2 = SA2 - o. From here he produces the common signature for message 2
     (sC2 = sA2 + sB2) and proceeds to use it to claim his own part of the agreement.

[1](https://medium.com/crypto-garage/da0663c2adc4)

### Atomic Swaps

TODO.

[1](https://medium.com/crypto-garage/3f41c8fb221b)

### Miscellaneous

[1](https://en.bitcoin.it/wiki/Hash_Time_Locked_Contracts)
[2](https://adiabat.github.io/dlc.pdf)

Tasks
-----

* Implement the Tonelli-Shanks algorithm to calculate curve points from x coordinates.
* Figure out the best way to handle 'infinity' popping up in ECC operations.
* Write `elliptic_curve` unit tests.
