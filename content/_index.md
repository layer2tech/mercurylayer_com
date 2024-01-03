+++
title = "Mercury Layer"
description = "Mercury Layer Overview"
+++

# Mercury Layer

Mercury Layer is an implementation of a layer-2 statechain protocol that enables off-chain transfer and settlement of Bitcoin outputs that remain under the full custody of the owner at all times, while benefiting from instant and zero cost transactions. The ability to perform this transfer without requiring the confirmation (mining) of on-chain transactions has advantages in a variety of different applications. 

This documentation covers the description of the Mercury Layer architecture and protocol, the specification Mercury API and instructions for the deployment and operation of the separate components of the system. 

## Overview

An *unspent transaction output* (UTXO) is the fundamental object that defines value and ownership in a cryptocurrency such as Bitcoin. A UTXO is identified by a transaction ID (`TxID`) and output index number (`n`) and has two properties: 1. A value (in BTC) and 2. Spending conditions (defined in Script). The spending condition is most commonly defined by a single public key and can only be spent by a transaction signed with the corresponding public key. 

The simplest function of the Mercury layer system is to enable the transfer of the ownership of individual UTXOs controlled by a single public key `P` from one party to another without an on-chain (Bitcoin) transaction (or change in the spending condition). The SE facilitates this change of ownership, but has no way to seize, confiscate or freeze the output. To enable this, the private key (`s`) for `P` (where `P = s.G`) is shared between the SE and the owner, such that neither party ever has knowledge of the full private key (which is `s = s1 + o1` where `s1` is the SE private key share, and `o1` is the owner key share) and so cooperation of the owner and SE is required to spend the UTXO. However, by sharing the secret key in this way, the SE can modify its key share (`s1 -> s2`) so that it combines with a new owner key share (`o2`) with the cooperation of the original owner, but without changing the full shared key (i.e. `s1 + o1 = s2 + o2`) all without any party revealing their key shares or learning the full key. The exclusive control of the UTXO then passes to the new owner without an on-chain transaction, and the SE only needs to be trusted to delete/overwrite the key share corresponding to the previous owner. 

This key update/transfer mechanism is additionally combined with a system of *backup* transactions which can be used to claim the value of the UTXO by the current owner in the case the SE does not cooperate or has disappeared. The backup transaction is cooperatively signed by the current owner and the SE at the point of transfer, paying to an address controlled by the new owner. To prevent a previous owner (i.e. not the current owner) from broadcasting their backup transaction and stealing the deposit, the `nLocktime` value of the transaction is set to a future specified block height. Each time the ownership of the UTXO is transferred, the `nLocktime` is decremented by a specified value, therefore enabling the current owner to claim the coin before any of the previous owners.

<br><br>
<p align="center">
<img src="fig1.png" align="middle" width="60" vspace="20">
</p>

<p align="center">
  Schematic of confirmed funding transaction, and off-chain signed backup transactions with decrementing nLocktime for a sequence of 4 owners.
</p>
<br><br>

The decrementing timelock backup mechanism limits the number of transfers that can be made within the lock-out time. The user is responsible for submitting backup transactions to the Bitcoin network at the correct time, and applications will do this automatically.

The life-cycle of a coin in the statechain, key reassignment and closure is summarised as follows:

1. The first owner initiates a UTXO statechain with the SE by paying BTC to an address where Owner 1 and the SE each have private key shares, both of which are required to spend the UTXO. Additionally, the SE and the initiator cooperate to sign a backup transaction spending the UTXO to a timelocked transaction spending to an address fully controlled by Owner 1 which can be confirmed after the `nLocktime` block height in case the SE stops cooperating.
3. Owner 1 can verifiably transfer ownership of the UTXO to a new party (Owner 2) via a key update procedure that overwrites the private key share of SE that invalidates the Owner 1 private key and *activates* the Owner 2 private key share. Additionally, the transfer incorporates the cooperative signing of a new backup transaction paying to an address controlled by Owner 2 which can be confirmed after a new `nLocktime` block height, which is reduced (by an accepted confirmation interval) from the previous owners backup transaction `nLocktime`.
5. This transfer can be repeated multiple times to new owners as required (up until the most recent recovery `nLocktime` reaches a lower limit determined by the current Bitcoin block height).
6. At any time the most recent owner and SE can cooperate to sign a transaction spending the UTXO to an address of the most recent owner's choice (i.e. closure). 

##  Statechains

The essential function of the Mercury layer system is that it enables 'ownership' (and control) of a UTXO to be transferred between two parties (who don't need to trust each other) via the SE without an on-chain transaction. The SE only needs to be trusted to not store any information about previous key shares and then the transfer of ownership is completely secure, even if the SE was to later get compromised or hacked. At any time the SE can prove that they have the key share for the current owner (and only to the current owner). The current owner is required to sign a *statechain transaction* (`SCTx`) with an owner key to transfer ownership to a new owner (i.e. a new owner key). This means that any theft of the UTXO by the collusion of a corrupt SE and old owner can be independently and conclusively proven.

## Blinding

The Mercury layer server is *blind* - that is the server *does not* and *cannot* know anything that would enable it to identify the coin (UTXO) that it is co-signing for. This prevents any censorship and storage of any identifying data in the server - the server itself is not aware of bitcoin, and does not perform any verifcation of transactions. 

To achieve this the server cannot know or be able to derive in any way the following values:

- The TxID:vout of the statecoin UTXO
- The address (i.e. public key) of the UTXO
- Any signatures added to the bitcoin blockchain for the coin public key.

This means that the server cannot:

- Learn of the shared public key it is co-signing for.
- Learn of the final (unblinded) form of any signatures it co-generates.
- Verify ANY details of backup or closure transactions (as this would reveal the statecoin TxID).

All verification of backup transaction locktime decrementing sequence must be performed client side (i.e. by the receiving wallet). This requires that the full statechain and sequence of backup transactions is sent from sender to receiver and verified for each transfer (this can be done via the server with the data encrypted with the receiver public key). 

The server is no longer able to perform an explicit proof-of-publication of the globally unique coin ownership (via the SMT of coin TxIDs), as it cannot know these. It can however perform a proof-of-publication of its public key shares for each coin it is co-signing (i.e. SX, where X = 1,2,3 ...). These key shares can be used by the current owner to calulate the full shared public key and verify that their ownership is unique.

### Blind two-party Schnorr signatures

Mercury layer employs Schnorr signatures via Taproot addresses for statecoins. To enable a signature to be generated over a shared public key (by the two private key shares of the server and owner) a blinded variant of the Musig2 protocol is employed. In this variant, one of the co-signing parties (the server) does not learn of 1) The full shared public key or 2) The final signature generated. An ephemeral key commitment scheme is employed to ensure Wagner based attacks are not possible. 

### Client transaction verification

In the blinded mercury layer protocol, the server cannot verify what it signs, but can only state HOW MANY unique signatures it has generated for a specific shared key, and it will return this number when queried by a wallet. The wallet will then have to check that every single previous backup transaction signed has been correctly decremented, AND that the total number of value backup transactions it has verified matches the number of signatures the server has co-generated. This will then enable a receiving wallet to verify that no other valid transactions spending the statecoin output exist (given it trusts the server to return the correct number of signatures). 

When it comes to closure, the server can no longer verify that any fee has been added to the closure transaction, and the wallet can just create any transaction it wants to end the chain. In this case, any fee collected by the SE must be done separately to the statecoin initialisation and closure transactions (and be required on initialisation, before the shared key is generated).

### Keyshare publication

The server does not have access to the TxIDs of individual coins along with the user proof keys that it can publish. Instead, it takes each of the current public key shares for each coin in the system and publishes this list. This is then updated with each new coin or coin ownership change. This public key list is then commited to bitcoin via Mainstay for a proof-of-uniqueness. 

To verify the uniqueness of the ownership of the shared public key, the current owner then derives the full shared public key from this commitment and their or key share (P = o1.(s1.G)) and verifies it against the coin. 

# Mercury Layer Protocol

## Preliminaries

The SE and each owner are required to generate private keys securely. Owners are required to verify ownership of UTXOs (this can be achieved via a wallet interface, and requires connection to an Electrum server or fully verifying Bitcoin node). Elliptic curve points (public keys) are depicted as upper case letter, and private keys as lower case letters. Elliptic curve point multiplication (i.e. generation of public keys from private keys) is denoted using the `.` symbol. The generator point of the elliptic curve standard used (e.g. secp256k1) is denoted as `G`. All arithmetic operations on secret values (in Zp) are modulo the field the EC standard.  

In addition, a public key encryption scheme is required for blinded private key information sent between parties. This should be compatible with the EC keys used for signatures, and ECIES is used. The notation for the use of ECIES operations is as follows: `Enc(m,K)` denotes the encryption of message `m` with public key `K = k.G` and `Dec(m,k)` denotes the decryption of message `m` using private key `k`.

All transactions are created and signed using segregated witness, which enables input transaction IDs to be determined before signing and prevents their malleability. 

## Initiation

An user wants to create a statecoin for a specific amount of BTC, and they request that the SE begin initialize the process. To begin, the user must provide a valid `token_id` (UUID), which will be listed in the the server token database. This `token_id` is generated by the SE on payment of a fee (via a separate lighning or bitcoin payment). 

1. The initiator (Owner 1) generates a private key: `o1` (the UTXO private key share).
2. Owner 1 then calculates the corresponding public key of the share `O1` and sends it to the SE: `O1 = o1.G`
3. Owner 1 requests a key share from the SE with a valid `token_id` and amount value `A`. 
4. The SE then generates a private key: `s1` (the SE private key share), calculates the corresponding public key and sends it to Owner 1: `S1 = s1.G` along with a `statechain_id` (UUID). The SE then stores `s1`, `S1` and `A` indexed with `statechain_id`. 
5. Owner 1 then adds the public key they receive by their own public key to obtain the shared (aggregated) public key `P` (which corresponds to a shared private key of `p = o1 + s1`): `P = O1 + S1`
6. Owner 1 creates and broadcasts a funding transaction (`Tx0`) to pay an amount `A` to the address corresponding to `P`. This defines the UTXO `TxID:vout` (the outpoint). 
8. Owner 1 creates an unsigned *backup transaction* (`Tx1`) that pays the `P` output of `Tx0` to address of `O1`, and sets the `nLocktime` to the initial future block height `h0` (where `h0 = cheight + hinit`, `cheight` is the current Bitcoin block height and `hinit` is the initial locktime specified by the server).
9. Owner 1 cooperates with SE to generate a valid signature on `Tx1`

### Signature generation

To generate a signature on `Tx1`, the owner first computes the sighash `m1`. 
Owner 1 then generates a random ephemeral nonce `r2_1` and blinding nonce `b1` and computes `R2_1 = r2_1.G`
Owner 1 then requests a co-signing from the SE sending commitments: `SHA256(R2_1)` and `SHA256(b1)`. 
SE stores the commitments for `statechain_id` and generates a random `r1_1` and computes `R1_1 = r1_1.G`. `R1_1` is returned to Owner 1. 
Owner 1 then computes `R_1 = R1_1 + r2_1.G + b1.P`, `e1 = SHA256(P||R_1||m1)` and `c1 = e1 + b1` and sends `c1` to the SE. 
The SE then computes `sig1_1 = r1_1 + c1.s1` and sends to Owner 1 (and stores `c1` with `statechain_id`). 
Owner 1 computes `sig2_1 = r2_1 + c1.o1` and `sig_1 = sig1_1 + sig2_1`. The full signature `(sig_1,R_1)` is then added to `Tx1`. 

10. `Tx1` is verified and stored by Owner 1, along with the values `b1`, `R1_1` and `e1`. 
11. The SE then adds the public key `S1` to the list of current statechain SE keys and publishes. 

## Key Reassignment

Owner 1 wishes to transfer the value of the coin `A` to a new owner (Owner 2) (as a payment or as part of a swap). The protocol then proceeds as follows:

### Sender

1. The receiver (Owner 2) generates a statechain (proof) private key `o2`. They then compute the corresponding public key `O2 = o2.G`.
2. `O2` then represents the Owner 2 'address' and is communicated to Owner 1 (or published) in order for them to 'send' the ownership.
3. Owner 1 then creates a new unsigned backup transaction `Tx2` paying the output of `Tx0` to address of `O2`, and sets the `nLocktime` to `h0 - (n-1)*c` where `c` is the confirmation interval and `n` is the owner number (i.e. 2). 
4. Owner 1 cooperates with SE to generate a valid signature on `Tx2`

### Signature generation

To generate a signature on `Tx2`, the owner first computes the sighash `m2`. 
Owner 1 then generates a random ephemeral nonce `r2_2` and blinding nonce `b2` and computes `R2_2 = r2_2.G`
Owner 1 then requests a co-signing from the SE sending commitments: `SHA256(R2_2)` and `SHA256(b2)`. 
SE stores the commitments for `statechain_id` and generates a random `r1_2` and computes `R1_2 = r1_2.G`. `R1_2` is returned to Owner 1. 
Owner 1 then computes `R_2 = R1_2 + r2_2.G + b2.P`, `e2 = SHA256(P||R_1||m1)` and `c2 = e2 + b2` and sends `c2` to the SE. 
The SE then computes `sig1_2 = r1_2 + c2.s1` and sends to Owner 1 (and stores `c2` with `statechain_id`). 
Owner 1 computes `sig2_2 = r2_2 + c2.o1` and `sig_2 = sig1_2 + sig2_2`. The full signature `(sig_2,R_2)` is then added to `Tx2`.

4. SE generates a random key `x1` and sends it to Owner 1. 
5. Owner 1 then computes `t1 = o1 + x1` and encrypts it with the Owner 2 public key (from the address): `Enc(t1,O2)`
9. Owner 1 then concatinates the `Tx0` outpoint with the Owner 2 public key (`O2`) and signs it with their key `o1` to generate `SC_sig_1`. 
10. Owner 1 then sends Owner 2 a message containing five objects:
	a. All signed backup transactions: `Tx1` and `Tx2`
	b. For each backup transaction signature (`b`,`R2`): `b1`,`b2`,`R2_1` and `R2_2`. 
	c. `SC_sig_1`
	d. `Enc(t2,O2)`
	e. `statechain_id`

> At this point the Owner 1 has sent all the information required to complete the reassignment to Owner 2 and is no longer involved in the protocol. Owner 2 then verifies the correctness and validity of the objects. 

### Receiver

1. Owner 2 verifies that the latest backup transaction pays to `O2` and that the input (`Tx0`) is unspent and pays to `P`. 
2. Owner 2 takes the list of previous `K` backup transactions (`Txi i=1,...,K`) and for each one `i` verifies:
	a. The signature is valid. 
	b. The `nLocktimes` are decremented correctly (i.e. the latest `TxK` is the lowest). 
	c. Retreives `ci`, and commitments `SHA256(R2_i)` and `SHA256(bi)` from the SE. 
	d. Verifies the commitments to `R2_i` and `bi` and verfies that `ci = bi + SHA256(P||R_i||mi)` (where `mi` is the sighash of `Txi`). 
3. Owner 2 queries SE for 1) The total number of signatures generated for `statechain_id`: `N` and 2) Current SE public key: `S1`. 
4. Owner 2 then verifies that `K = N` and then `O1 + S1 = P`

The SE key share update then proceeds as follows:

5. Owner 2 decrypts object `t1`: `Dec(t1,o2)` and then computes `t2 = t1 - o2`. 
6. Owner 2 then sends `t2` to the SE. 
7. SE the updates the private key share `s2 = s1 + t2 - x1 = s1 + x1 + o1 - o2 - x1 = s1 + o1 - o2`

> `s2` and `o2` are now key the private key shares of `P = (s2 + o2).G` which remains unchanged (i.e. `s2 + o2 = s1 + o1`), without anyone having learnt the full private key. Provided the SE deletes `s1`, then there is no way anyone but the current owner (with `o2`) can spend the output.

8. The SE then adds the public key `S2` to the list of active key shares and publishes. 

## Orderly Closure

The current owner of a coin can at any time the statecoin by simply co-signing a transaction paying to any specified address. The server cannot identify a closure, but the coin can no longer be transferred to a new owner because the server will have produced an additional signature that cannot be verified as a valid backup by a receiver. 

Closure proceeds as follows:

1. The current owner (e.g. Owner 2) creates an unsigned transaction `TxW` that spends `Tx0` to a closure address `W`.
2. The owner then co-signs this transaction with the SE (as above).  `TxW` is broadcast. 
3. The owner then sends the SE a closure notification (with their current `statechain_id`) that the coin is withdrawn so the SE can remove the coin public from the published key share list. 

## Backup closure

In the case that the SE disappears or does not cooperate with the current owner, the current owner can reclaim their funds to an address they control by submitting their backup transaction when the `nLocktime` is reached. 
