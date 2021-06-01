```
LNPBP: 0002
Layer: Transactions (1)
Vertical: Bitcoin protocol
Title: Deterministic embedding of cryptographic commitments into transaction 
       output scriptPubkey
Authors: Dr Maxim Orlovsky <orlovsky@protonmail.ch>,
         Giacomo Zucco,
         Martino Salvetti,
         Federico Tenga
Comments-URI: https://github.com/LNP-BP/lnpbps/issues/4
Status: Proposal
Type: Standards Track
Created: 2019-10-27
License: CC0-1.0
```

## TOC

- [Abstract](#abstract)
- [Background](#background)
- [Motivation](#motivation)
- [Design](#design)
- [Specification](#specification)
  * [Commitment procedure](#commitment-procedure)
  * [Reveal procedure](#reveal-procedure)
  * [Verification procedure](#verification-procedure)
  * [Deterministic public key extraction from Bitcoin script](#deterministic-public-key-extraction-from-Bitcoin-script)
- [Compatibility](#compatibility)
- [Rationale](#rationale)
- [Reference implementation](#reference-implementation)
- [Acknowledgements](#acknowledgements)
- [References](#references)
- [License](#license)
- [Appendix A. Test vectors](#test-vectors)
  * [Correct test vectors](#correct-test-vectors)
  * [Invalid test vectors](#invalid-test-vectors)
  * [Protocol failures](#protocol-failures)


## Abstract

This standard defines an algorithm for deterministic embedding and verification
of cryptographic commitments within the `scriptPubKey` part of Bitcoin transaction outputs.
It depends on the elliptic-curve public-key-tweaking procedure which is defined in the 
[LNPBP-1](https://github.com/LNP-BP/LNPBPs/blob/master/lnpbp-0001.md) standard.
Naturally, only `scriptPubKey` types existing at the time of writing are covered here, 
namely P2(W)PK(H), P2(W)SH, P2TR and OP_RETURN style types.


## Background

Embedding of cryptographic commitments into bitcoin transactions is a widely used
practice. Occurrences include timestamping [1], single-use seals [2],
pay-to-contract settlement schemes [3], sidechains [4], blockchain anchoring
[5], Taproot, Graftroot proposals [6, 7, 8], Scriptless scripts [9] and many
others.


## Motivation

A multitude of Bitcoin based, cryptographic commitment (CC) use cases benefit from 
the homomorphic properties of elliptic curve public keys. They allow for secure,
privacy-preserving and blockspace-saving cryptographic commitments (CCs), by tweaking 
a public key [12].
However, many applications require a commitment to be present within non-P2(W)PK(H) 
outputs. Whereas P2(W)PK(H) outputs are convenient carriers for CCs — because they are 
unambiguously just a single public key — non-P2(W)PK(H) outputs can contain multiple 
public keys. 

Hence, the purpose of this protocol is to guarantee — for co-operating parties only — 
elimination of any ambiguity about which public key carries the commitment — while 
hiding this fact from any non-involved party. 
Moreover — due to the open, public, permissionless nature of Bitcoin blockchain 
transactions — this protocol hides from non-involved parties at all, if a CC is present 
or not, in a given Bitcoin transaction output.

Hence, a clear and deterministic definition is presented, of how the respective public 
key — which single-handedly carries the commitment — can be found among the multitude 
of potential public keys within a Bitcoin `scriptPubKey` script.

A similar need arises, when embedding CCs into Lightning Network (LN) payment channel 
state updates. In the current LN version, commitment transaction outputs — which 
must be tweaked for CCs — can either be P2WSH [10] or P2WPKH [11] outputs. Upcoming 
LN versions will add additional P2WSH outputs [17, 18]. 
Hence, there is the need for a standardized, safe (deterministic) way of embedding 
CCs inside Lightning Network commitment transaction outputs (PW2SH).

Additionally, P2(W)SH and other non-standard outputs (like explicit bare script outputs, 
including OP_RETURN type) are not trivial to use for cryptographic commitments, because 
CCs require deterministic definition of the actual commitment carrier. Generally, these 
outputs just present a hash of the actual script, hiding the information of the used 
public key(s) or their hashes. Moreover, it raises the question of how a public key can
be defined/detected within Bitcoin Script, because it can be represented in various 
different ways (explicitly or by a hash). It's not trivial to understand if given hash
in a Bitcoin script represents a public key or some other type of preimage data. 
All of this raises demand for a standard way of embedding CCs in non-P2(W)PK(H) outputs, 
by using best practices. 
This proposal tries to address the issues by proposing a common standard for the use of
public-key-tweaking based CCs [12] within all possible Bitcoin transaction output types.


## Design

The protocol requires that exactly one public key of all keys present or
referenced in `scriptPubkey` and `redeemScript` must contain the commitment
(made with LNPBP-1 procedure) to a given message. This commitment is
deterministically singular, i.e. it can be proven that there is no other
alternative message that the given transaction output commits to under this
protocol. The singularity is achieved by committing to the sum of all original
(i.e. before the message commitment procedure) public keys applies the spending
of a given transaction output. Thus, the given protocol modifies LNPBP-1
standard [12] for all possible options of `scriptPubkey` transaction output
types.

The commitment consists of the updated `scriptPubkey` value, which may be embed
into the transaction output, and an **extra-transaction proof** (ETP), required
for the verification of the commitment. The structure and information in the
proof depends on the used `scriptPubkey` type.

The protocol includes an algorithm for deterministic extraction of public keys
from a given Bitcoin script, based on Miniscript [16].


## Specification

### Commitment procedure

The **committing party**, having a message `msg`, must:
1. Collect all unique public key instances (named **original public key set** 
   `P`) related to a given transaction output, defined as:
   - a single public key for P2PK, P2PKH, P2WPK, P2WPK-in-P2SH type of outputs;
   - an arbitrary public key (even with an unknown corresponding private key),
     if `OP_RETURN` type of `scriptPubkey` is used;
   - an intermediate public key for the Taproot-based (SegWit v1) output;
   - all public keys extracted using 
     [algorithm](#deterministic-public-key-extraction-from-bitcoin-script) from 
     either:
     * `witnessScript` contained in the `witness` stack for native v0 P2WSH
       and legacy P2WSH-in-P2SH SegWit outputs;
     * `redeemScript` contained in top  `scriptSig` for non-SegWit P2SH
     * or bare (custom) script-based `scriptPubkey`
2. Select a single public key `Po` from the set `P` of the original public keys 
   for the tweaking procedure. It is advised that the corresponding private key
   being controlled by the committing party, which will simplify future spending
   of the output.
3. Run [LNPBP-1 commitment procedure](https://github.com/LNP-BP/LNPBPs/blob/master/lnpbp-0001.md#commitment-procedure)
   for the message `msg`, selected public key `Po`, public key set `P` and a
   protocol-specific `tag`, provided by the upstream protocol using this 
   standard.
4. Construct necessary scripts and generate `scriptPubkey` of the required 
   type. If OP_RETURN `scriptPubkey` is used, it MUST be serialized according to
   the following rules:
   - only a single `OP_RETURN` code MUST be present in the `scriptPubkey` and it
     MUST be the first byte of it;
   - it must be followed by 32-byte push of the public key value `P` from the 
     step 2 of the algorithm, serialized according to the rules from [15];
   - if the tweaked public key serialized with bitcoin consensus serialization
     rules does not start with 0x02 as it's first (least significant) byte, the
     procedure must fail; or it may be repeated with a new public key picked at
     step 1.
6. Construct and store an **extra-transaction proof** (ETP), which structure 
   depends on the generated `scriptPubkey` type:
   a) value of `Po`, corresponding to:
      * the original intermediary public key for V1 witness Taproot output;
      * single original public key used in the tweak procedure before the tweak
        was applied;
   b) deterministic script reconstruction data, i.e.:
      * untweaked `redeemScript` (for P2SH outputs), or `witnessScript` 
        (for P2WSH SegWit v0 native or P2WSH-in-P2SH SegWit legacy outputs), 
        constructed using set of the *original public keys*;
      * the "taptweak" hash (i.e. the tagged hash of the Merkle root of the 
        TapScript branches) for v1 witness output (Taproot);
      * for other types of outputs no other data are required for the ETP.


### Reveal procedure

The **reveal protocol** is usually run between the committing and verifying
parties; however it may be used by the committing party to publicly reveal the
proofs of the commitment. These proofs include:
* `scriptPubkey` from the transaction output containing the commitment;
* *original message* `msg` to which the *comitting party* has committed;
* *extra-transaction proof* (ETP), constructed at the 6th step of the
  [commitment protocol](#commitment);
* (optional) proofs that the `scriptPubkey` is a part of the transaction
  included into the bitcoin chain containing the largest known amount of work at
  depth satisfying a *verifying party* security policy (these proofs may be
  reconstructed/verified by the verifying party itself using its trusted Bitcoin
  Core server);


### Verification procedure

The verification process runs by repeating steps of the commitment protocol
using the information provided during the *reveal phase* and verifying the
results for their internal consistency; specifically:
1. Original public key set reconstruction:
   - if *extra-transaction proof* (ETP) provides script data (pt. 6.b from the
     commitment procedure) parse it according to deterministic key collection
     [algorithm](#deterministic-public-key-extraction-from-bitcoin-script)
     and collect all the original public keys from it; fail the verification
     procedure if parsing fails;
   - otherwise use an original public key provided as a part of ETP as a single-
     element set
2. Run LNPBP-1 verification procedure repeating step 3 from the commitment
   procedure, using original public key value from the *extra-transaction proof*
3. Construct `scriptPubkey'`, matching the type of the `scriptPubkey` in the 
   transaction and matching *extra-transaction proof* data using the tweaked 
   version of the public key in the same way as was perfomed at step 4 of the
   commitment procedure. If there can be multiple matching `scriptPubkey` types
   for a given data, construct a variant for each of them.
4. Make sure that one of the `scriptPubkey'` values generated at the  previous
   step, byte by byte matches the `scriptPubkey` provided during the  *reveal
   phase*; otherwise fail the verification.


### Deterministic public key extraction from Bitcoin Script

1. The provided script MUST be parsed with Miniscript [16] parser; if the parser
   fails the procedure MUST fail.
2. Iterate over all branches of the abstract syntax tree generated by the
   Miniscript parser, running the following algorithm for each node:
   - if a public key hash is met (`pk_h` Miniscript command) and it can't be 
     resolved against known public keys or other public keys extracted from the
     script, fail the procedure;
   - if a public key is found (`pk`) add it to the list of the collected public 
     keys;
   - for all other types of Miniscript commands iterate over their branches.
3. Select unique public keys (i.e. if some public key is repeated in different 
   parts of the script/in different script branches, pick a single instance of 
   it). Compressed and uncompressed versions of the same public key must be 
   treated as the same public key under this procedure.
4. If no public keys were found fail the procedure; return the collected keys 
   otherwise.

By "miniscript" we mean usage of `rust-miniscript` library v2.0.0 (commit 
`463fc1eadac2b46de1cd5ae93e8255a2ab34b906`) which may be found at 
<https://github.com/LNP-BP/rust-miniscript/commit/463fc1eadac2b46de1cd5ae93e8255a2ab34b906>


## Compatibility

The proposed cryptographic commitment scheme is fully compatible, and works with
LNPBP-1 standard [12] for constructing cryptographic commitments with a set of 
public keys.

The standard is not compliant with previously used OP_RETURN-based cryptographic
commitments, like OpenTimestamps [1], since it utilises plain value of the
public key with the commitment for the OP_RETURN push data.

The author is not aware of any P2(W)SH or non-OP_RETURN cryptographic commitment
schemes existing before this proposal, and it is highly probable that the
standard is not compatible with ones if they were existing.

The proposed standard is compliant with current Taproot proposal [14], since it
can use intermediate Taproot key to store the commitment.

Standard also should be interoperable with Schnorr's signature proposal [15],
since it uses miniscript supporting detection of public keys serialized with new
BIP-340 rules.


## Rationale

### Continuing support for P2PK outputs

While P2PK outputs are considered obsolete and are vulnerable to a potential
quantum computing attacks, it was decided to include them into the specification
for unification purposes.

### Support OP_RETURN type of outputs

OP_RETURN was originally designed to reduce the size of memory-kept UTXO data;
however it is not recommended to use it since it bloats blockchain space usage.
This protocol provides support for this type of outputs only because some
hardware (like HSM modules or hardware wallets) can't tweak public keys inside
outputs and produce a correct signature after for their spending, and keeping
the commitment in OP_RETURN's can be an only option. For all other cases it is
highly recommended not to use OP_RETURN outputs for the commitments and use
tweaking of other types of outputs instead.

### Unification with OP_RETURN-based commitments

While it is possible to put a deterministic CC commitments into OP_RETURN-based
outputs like with [1], their format was modified for unification purposes with
the rest of the standard. This will help to reduce the verification and
commitment code branching, preventing potential bugs in the implementations.

### Custom serialization of public keys in OP_RETURN

This saves one byte of data per output, which is important in case of blockchain
space.

While we could pick a new BIP340-based rules for public key serialization [15],
this will require software to support new versions of the secp256k1 libraries 
and language-specific wrappers, which may be a big issue for legacy software
usually used by hardware secure modules, which require presence of OP_RETURN's
(see rationale points above).

### Support for pre-SegWit outputs

These types of outputs are still widely used in the industry and it was decided
to continue their support, which may be dropped in a higher-level requirements
by protocols operating on top of LNPBP-2.

### Committing to the sum all public keys present in the script

Having some of the public keys tweaked with the message digest, and some left
intact will make it impossible to define a simple and deterministic commitment
scheme for an arbitrary script and output type that will prevent any potential
double-commitments.

### Deterministic public key extraction for a Bitcoin script

We use determinism of the proposed Miniscript standard to parse the Bitcoin
script in an implementation-idependet way and in order to avoid the requirement
of the full-fledged Bitcoin Script interpreter for the commitment verification.

The procedure fails on any public key hash present in the script, which prevents
multiple commitment vulnerability, when different parties may be provided with
different *extra-transaction proof* (ETP) data, such that they will use
different value for the `S` public key matching two partial set of the public
keys (containing and not containing public keys for the hashed values).

The algorithm does not require that all of the public keys will be used in
signature verification for spending the given transaction output (i.e. public
keys may not be prefixed with `[v]c:` code in Miniscript [16]), so the
committing parties may have a special public key shared across them for
embedding the commitment, without requiring to know the corresponding private
key to spend the output.

### Use of Taproot intermediate public key

With Taproot we can't tweak the public key contained in the `scriptPubkey`
directly since it will invalidate its commitment to the Tapscript and to the
intermediate key, rendering output unspendable. Thus, we put tweak into the
underlying intermediate public key as the only avaliable option.


## Reference implementation

<https://github.com/LNP-BP/rust-lnpbp/blob/master/src/bp/dbc/lockscript.rs>


## Acknowledgements

Authors would like to thank: 
* Giacomo Zucco and Alekos Filini for their initial work on the commitment 
  schemes as a part of early RGB effort [13];
* Dr Christian Decker for pointing out on Lightning Network incompatibility with
  all existing cryptographic commitment schemes.


## References

1. Peter Todd. OpenTimestamps: Scalable, Trust-Minimized, Distributed 
   Timestamping with Bitcoin.
   <https://petertodd.org/2016/opentimestamps-announcement>
2. Peter Todd. Preventing Consensus Fraud with Commitments and Single-Use-Seals.
   <https://petertodd.org/2016/commitments-and-single-use-seals>
3. [Eternity Wall's "sign-to-contract" article](https://blog.eternitywall.com/2018/04/13/sign-to-contract/)
4. Adam Back, Matt Corallo, Luke Dashjr, et al. Enabling Blockchain Innovations 
   with Pegged Sidechains (commit5620e43). Appenxix A. 
   <https://blockstream.com/sidechains.pdf>.
5. <https://exonum.com/doc/version/latest/advanced/bitcoin-anchoring/>
6. Pieter Wuille. Taproot proposal. 
   <https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2019-May/016914.html>
7. Gregory Maxwell. Taproot: Privacy preserving switchable scripting.
   <https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-January/015614.html>
8. Gregory Maxwell. Graftroot: Private and efficient surrogate scripts under the
   taproot assumption.
   <https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-February/015700.html>
9. Andrew Poelstra. Scriptless scripts. 
   <http://diyhpl.us/wiki/transcripts/layer2-summit/2018/scriptless-scripts/>
10. Lightning Network BOLT-3 standard, version 1.0. Sections ["Offered HTLC 
    Outputs"]
    <https://github.com/lightningnetwork/lightning-rfc/blob/v1.0/03-transactions.md#offered-htlc-outputs>
    and ["Received HTLC Outputs"]
    <https://github.com/lightningnetwork/lightning-rfc/blob/v1.0/03-transactions.md#received-htlc-outputs>.
11. Lightning Network BOLT-3 standard, version 1.0. Section ["to_remote Output"]
    https://github.com/lightningnetwork/lightning-rfc/blob/v1.0/03-transactions.md#to_remote-output
12. Maxim Orlovsky, et al. Key tweaking: collision-resistant elliptic 
    curve-based commitments (LNPBP-1 Standard).
    <https://github.com/LNP-BP/lnpbps/blob/master/lnpbp-0001.md>
13. RGB Protocol Specification, version 0.4. "Commitment Scheme" section.
    <https://github.com/rgb-org/spec/blob/old-master/01-rgb.md#commitment-scheme>
14. Pieter Wuille. Taproot: SegWit version 1 spending rules.
    <https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki>
15. Pieter Wuille. Schnorr Signatures for secp256k1.
    <https://github.com/bitcoin/bips/blob/master/bip-0340.mediawiki>
16. Pieter Wuille, Andrew Poelsta. Miniscript. 
    <http://bitcoin.sipa.be/miniscript/>
17. Lightning Network BOLT-3 standard, master branch. Section ["to_remote Output"]
    https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#to_remote-output
18. Lightning Network BOLT-3 standard, master branch. Section ["to_local_anchor and to_remote_anchor"]
    https://github.com/lightningnetwork/lightning-rfc/blob/master/03-transactions.md#to_local_anchor-and-to_remote_anchor-output-option_anchor_outputs

## License

This document is licensed under the Creative Commons CC0 1.0 Universal license.


## Appendix A. Test vectors

### 1. Correct test vectors

TBD

### 2. Invalid test vectors

TBD

### 3. Edge cases: protocol failures

TBD
