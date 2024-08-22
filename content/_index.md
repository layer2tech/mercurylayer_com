+++
title = "Mercury Layer"
description = "Mercury Layer Overview"
+++

# Mercury Layer

Mercury layer is an implementation of a system the uses a blind co-signing and key-update service to enable statechains on Bitcoin. The statechain protocol allows the transfer of ownership of Bitcoin unspent transaction outputs (UTXOs) that remain under the full custody of the owner at all times, while benefiting from instant and zero cost transactions. The ability to perform this transfer without requiring the confirmation (mining) of on-chain transactions has advantages in a variety of different applications. 

## Overview

The Mercury layer system employs a service provider (the mercury layer blind server) that generates and updates key shares (or key fragments) on request in addition to a count of partial blinded signatures. By updating (‘cycling’) a key share and reporting the number of  partial blinded signatures generated for the share, the ownership of individual UTXOs can be transferred between counterparties instantly and at zero marginal cost in a secure and fully self-custodial way. The blind key-update server never has control or custody, and is never aware of the identity of any specific UTXO. 

This system requires that all bitcoin transaction operations and statechain verification are performed entirely by client-side software. 

## Mercury layer service

The mercury layer service generates a private key share `s1` on initialisation of a session. In order to initialise a session, a client must provide a valid `token_id` which controls access to the service (and is generated separately typically after payment of a fee). 

The client then initialises a session with an `auth_pubkey` which is used to authenticate all subsequent messages with the server. The server responds with the public elliptic curve point corresponding to the session private key share `s1`. 

Once initialised, the server can then perform two operations using the key share: 

- Partial signature generation - the server uses the key share to compute a partial signature from a blinded challenge value provided by the client.
- Key update - an encrypted value is sent to the server and used to update the key share along with a new `auth_pubkey` for the updated key. The previous key share is then deleted securely. 

The server does not ever receive any other information regarding the client state. 

##  Statechain transfers

Using the mercury service, clients can use the key share update rules applied by the server to securely transfer ownership of a Bitcoin UTXO to a new client while maintaining self-custody and without requiring a blockchain transaction. 

This is achieved by depositing an amount of bitcoin to an address which is formed in part from the server public key share, and then requesting a partial signature from the server to either spend the coin or create ‘backup transactions’ to protect against server unavailability (i.e. unilateral on-chain exit). Transfers to new clients are secured by the server key update, enabling the UTXO deposit address to stay the same, while removing the ability of a previous owner to steal the funds. 

This key update mechanism is additionally combined with a system of backup transactions which can be used to claim the value of the UTXO by the current owner in the case the server does not cooperate or has disappeared. The backup transaction is created by the current owner at the point of transfer, paying to an address controlled by the new owner. To prevent a previous owner (i.e. not the current owner) from broadcasting their backup transaction and stealing the deposit, the nLocktime value of the transaction is set to a future specified block height. Each time the ownership of the UTXO is transferred, the `nLocktime` is decremented by a specified value, therefore enabling the current owner to claim the coin before any of the previous owners.

The decrementing timelock backup mechanism limits the number of transfers that can be made within the lock-out time. The user is responsible for submitting backup transactions to the Bitcoin network at the correct time, and applications can do this automatically.

The life-cycle of a coin in the statechain, key reassignment, and closure is summarised as follows:

1. The first owner initiates a statechain by paying an amount of bitcoin to an address where the corresponding public key is formed from both the first owner public key share and the server public key share. The first owner creates a timelocked backup transaction spending the statechain UTXO to an address fully controlled by the first owner which can be confirmed after the `nLocktime` block height in case the server stops cooperating.
2. The owner can verifiably transfer ownership of the UTXO to a new party (Owner 2) via a key update procedure that overwrites the private key share of server that invalidates the first owner private key and activates the new owner private key share. Additionally, the transfer incorporates the signing of a new backup transaction paying to an address controlled by the new owner which can be confirmed after a new nLocktime block height, which is reduced (by an accepted confirmation interval) from the previous owners backup transaction `nLocktime`.
3. This transfer can be repeated multiple times to new owners as required (up until the most recent recovery `nLocktime` reaches a lower limit determined by the current Bitcoin block height).

At any time the most recent owner can create and sign a transaction spending the UTXO to an address of the most recent owner's choice (i.e. closure of the statechain).

<br><br>
<p align="center">
<img src="fig1.png" align="middle" width="60" vspace="20">
</p>

<p align="center">
  Schematic of confirmed funding transaction, and off-chain signed backup transactions with decrementing nLocktime for a sequence of 4 owners.
</p>
<br><br>

## Blinding

The Mercury layer server is *blind* - it is not aware of bitcoin, and does not perform any verifcation of transactions. 

The server cannot know or be able to derive in any way the following values:

- The TxID:vout of the statecoin UTXO
- The address (i.e. public key) of the UTXO
- Any signatures added to the bitcoin blockchain for the coin public key.

This means that the server cannot:

- Learn of the shared public key it is co-signing for.
- Learn of the final (unblinded) form of any signatures it co-generates.
- Verify ANY details of backup or closure transactions (as this would reveal the statecoin TxID).

All verification of backup transaction locktime decrementing sequence must be performed client side (i.e. by the receiving client). This requires that the full statechain and sequence of backup transactions is sent from sender to receiver and verified for each transfer (this can be done via the server with the data encrypted with the receiver public key). 

### Blind two-party Schnorr signatures

Mercury layer employs Schnorr signatures via Taproot addresses for statecoins. To enable a signature to be generated over a shared public key (by the two private key shares of the server and owner) a blinded variant of the Musig2 protocol is employed. In this variant, one of the co-signing parties (the server) does not learn of 1) The full shared public key or 2) The final signature generated. 

### Client transaction verification

In the blinded mercury layer protocol, the server cannot verify what it signs, but can only state **HOW MANY** unique signatures it has generated for a specific shared key, and it will return this number when queried. The client will then have to check that every single previous backup transaction signed has been correctly decremented, AND that the total number of value backup transactions it has verified matches the number of blind partial signatures the server has generated. This will then enable a receiving client to verify that no other valid transactions spending the statecoin output exist. 

When it comes to closure the client can just create any transaction it wants to end the chain. 

# Mercury Layer Protocol

## Preliminaries

The server and each owner are required to generate private keys securely. Owners are required to verify ownership of UTXOs (this can be achieved via a client interface, and requires connection to an Electrum server or fully verifying Bitcoin node). Elliptic curve points (public keys) are depicted as upper case letter, and private keys as lower case letters. Elliptic curve point multiplication (i.e. generation of public keys from private keys) is denoted using the `.` symbol. The generator point of the elliptic curve standard used (e.g. secp256k1) is denoted as `G`. All arithmetic operations on secret values (in Zp) are modulo the field the EC standard.  

In addition, a public key encryption scheme is required for blinded private key information sent between parties. This should be compatible with the EC keys used for signatures, and ECIES is used. The notation for the use of ECIES operations is as follows: `Enc(m,K)` denotes the encryption of message `m` with public key `K = k.G` and `Dec(m,k)` denotes the decryption of message `m` using private key `k`.

All transactions are created and signed using segregated witness, which enables input transaction IDs to be determined before signing and prevents their malleability. 

## Initiation

A user wants to create a statechain for a specific amount of bitcoin, and they request that the server initialize the process. To begin, the user must provide a valid `token_id` (UUID), which will be listed in the the server token database. This `token_id` is generated by the server on payment of a fee (via a separate lighning or bitcoin payment). 

1. The initiator (Owner 1) generates a private key: `o1` (the UTXO private key share) and `auth_privkey` and `auth_pubkey`.
2. Owner 1 then calculates the corresponding public key of the share `O1`: `O1 = o1.G`
3. Owner 1 requests a key share generation from the server with a valid `token_id` and provides `auth_pubkey` (which is used to authenticate subsquent communication with the server). 
4. The server then generates a private key: `s1`, calculates the corresponding public key and sends it to Owner 1: `S1 = s1.G` along with a `statechain_id` (UUID). The server then stores `s1` indexed with `statechain_id`. 
5. Owner 1 then adds the public key they receive to their own public key to obtain the shared (aggregated) public key `P` (which corresponds to a shared private key of `p = o1 + s1`): `P = O1 + S1`
6. Owner 1 creates and broadcasts a funding transaction (`Tx0`) to pay an amount `A` to the address corresponding to `P`. This defines the UTXO `TxID:vout` (the outpoint). 
8. Owner 1 creates an unsigned *backup transaction* (`Tx1`) that pays the `P` output of `Tx0` to address of `O1`, and sets the `nLocktime` to the initial future block height `h0` (where `h0 = cheight + hinit`, `cheight` is the current Bitcoin block height and `hinit` is the initial locktime specified by the server).
9. Owner 1 the utilises the server to generate a valid signature on `Tx1` as follows:

### Signature generation

To generate a signature on `Tx1`, the owner first computes the sighash `m1`. 
Owner 1 then generates a random ephemeral nonce `r2_1` and blinding nonce `b1` and computes `R2_1 = r2_1.G`
Owner 1 then requests a partial signature from the server which generates a random `r1_1` and computes `R1_1 = r1_1.G`. `R1_1` is returned to Owner 1. 
Owner 1 then computes `R_1 = R1_1 + r2_1.G + b1.P`, `e1 = SHA256(P||R_1||m1)` and `c1 = e1 + b1` and sends `c1` to the server. 
The server then computes the partial signature `sig1_1 = r1_1 + c1.s1` and sends to Owner 1. 
Owner 1 computes `sig2_1 = r2_1 + c1.o1` and `sig_1 = sig1_1 + sig2_1`. The full signature `(sig_1,R_1)` is then added to `Tx1`. 

10. `Tx1` is verified and stored by Owner 1. 
11. The server then adds the public key `S1` to the list of current statechain server key shares and publishes. 

## Key Reassignment

Owner 1 wishes to transfer the value of the coin `A` to a new owner (Owner 2). The protocol then proceeds as follows:

### Sender

1. The receiver (Owner 2) generates a statechain private key share `o2`. They then compute the corresponding public key `O2 = o2.G` along with a new `auth_pubkey`. 
2. `O2||auth_pubkey` then represents the Owner 2 'address' and is communicated to Owner 1 (or published) in order for them to 'send' the ownership.
3. Owner 1 then creates a new unsigned backup transaction `Tx2` paying the output of `Tx0` to address of `O2`, and sets the `nLocktime` to `h0 - (n-1)*c` where `c` is the confirmation interval and `n` is the owner number (i.e. 2). 
4. Owner 1 cooperates with server to generate a blind partial signature on `Tx2` as follows:

### Signature generation

To generate a signature on `Tx2`, the owner first computes the sighash `m2`. 
Owner 1 then generates a random ephemeral nonce `r2_2` and blinding nonce `b2` and computes `R2_2 = r2_2.G`
Owner 1 then requests a partial signature from the server which generates a random `r1_2` and computes `R1_2 = r1_2.G`. `R1_2` is returned to Owner 1. 
Owner 1 then computes `R_2 = R1_2 + r2_2.G + b2.P`, `e2 = SHA256(P||R_1||m1)` and `c2 = e2 + b2` and sends `c2` to the server. 
The server then computes `sig1_2 = r1_2 + c2.s1` and sends to Owner 1. 
Owner 1 computes `sig2_2 = r2_2 + c2.o1` and `sig_2 = sig1_2 + sig2_2`. The full signature `(sig_2,R_2)` is then added to `Tx2`.

4. The server generates a random key `x1` and sends it to Owner 1. 
5. Owner 1 then computes the blinded transfer value `t1 = o1 + x1`. 
9. Owner 1 then concatinates the `Tx0` outpoint with the Owner 2 public key (`O2`) and signs it with their key `o1` to generate `SC_sig_1`. 
10. Owner 1 then create a transfer message containing five objects:
	a. All previous signed backup transactions: `Tx1` and `Tx2`
	b. `SC_sig_1`
	c. `t2`
	d. `statechain_id`
11. Owner 1 then encrypts the message with the reciver `auth_pubkey` and sends to the receiver (can be via server relay). 

> At this point the Owner 1 has sent all the information required to complete the reassignment to Owner 2 and is no longer involved in the protocol. Owner 2 then verifies the correctness and validity of the objects. 

### Receiver

1. Receiver (owner 2) decrypts the transfer message with their `auth_privkey`. 
2. Owner 2 verifies that the latest backup transaction pays to `O2` and that the input (`Tx0`) is unspent and pays to `P`. 
3. Owner 2 takes the list of previous `K` backup transactions (`Txi i=1,...,K`) and for each one `i` verifies:
	a. The signature is valid. 
	b. The `nLocktimes` are decremented correctly (i.e. the latest `TxK` is the lowest). 
4. Owner 2 queries server for 1) The total number of signatures generated for `statechain_id`: `N` and 2) Current server public key share: `S1`. 
5. Owner 2 then verifies that `K = N` and then `O1 + S1 = P`

The server key share update then proceeds as follows:

5. Owner 2 then computes `t2 = t1 - o2`. 
6. Owner 2 then sends `t2` to the server. 
7. The server then updates the private key share `s2 = s1 + t2 - x1 = s1 + x1 + o1 - o2 - x1 = s1 + o1 - o2`

> `s2` and `o2` are now key the private key shares of `P = (s2 + o2).G` which remains unchanged (i.e. `s2 + o2 = s1 + o1`), without anyone having learnt the full private key. Provided the server deletes `s1`, then there is no way anyone but the current owner (with `o2`) can spend the output.

8. The server then adds the public key `S2` to the list of active key shares and publishes. 

## Orderly Closure

The current owner of a coin can at any time spend the statechain by simply creating a transaction paying to any specified address. The server cannot identify a closure, but the coin can no longer be transferred to a new owner because the server will have produced an additional signature that cannot be verified as a valid backup by a receiver. 

Closure proceeds as follows:

1. The current owner (e.g. Owner 2) creates an unsigned transaction `TxW` that spends `Tx0` to a closure address `W`.
2. The owner then co-signs this transaction with the server (as above).  `TxW` is broadcast. 
3. The owner then sends the server a closure notification (with their current `statechain_id`) that the coin is withdrawn so the server can remove the coin public from the published key share list. 

## Backup closure

In the case that the server disappears or does not cooperate with the current owner, the current owner can reclaim their funds to an address they control by submitting their backup transaction when the `nLocktime` is reached.

# Token Payment

## Token payment process

This document describes the `/token/token_init` endpoint, which returns a response like the following example:

`https://api.mercurylayer.com/token/token_init`

```
{
  "btc_payment_address": "bc1qu2vs2z58m4dnyzuyk4240wdm2qd9jm827rnrjw",
  "fee": "50",
  "lightning_invoice": "lnbc939970n1pnvtm8ypp525ssrlm7senz7pf9z7h4sx7mz8nd2r7d6nxvldwjw7k2wm7hs8eqdp68psnxv3jxcekytf5x5cnwtf5xp3rwttzxpskxttzxumrgwpc8yer2cej8qcqzzsxqzjcsp5rd992e42ff4my6safcjce3gs0cz74rsuyffhqpragclzzkfw0lcq9qxpqysgq83hpdfxvsgh6ve8afvj38f2uuhg6nzm042sz98ge4qknfrcurlr8n3prw5qtmvjvcc7lpkhrfvm09ex5q3v6aq3tsprq958fakkrljgp2zfy80",
  "processor_id": "bb038002-b7cf-49e7-8425-da4d28e8242e",
  "token_id": "8a32263b-4517-40b7-b0ac-b76488925c28"
}
```

The fee is provided in euros. The invoice can be paid via Lightning Network (LN). To get additional details or to pay on-chain, make a call to:

`https://api.swiss-bitcoin-pay.ch/checkout/bb038002-b7cf-49e7-8425-da4d28e8242e`

with the `processor_id`. This will return the on-chain payment amount (in millisatoshis), as shown in the example below:

```
{
  "id": "bb038002-b7cf-49e7-8425-da4d28e8242e",
  "amount": 93997000,
  "wallet": "041feacad5e94ead9cf1e6b6b3acda14",
  "time": 1724247268,
  "expiry": 1724250868,
  "grossAmount": null,
  "title": "8a32263b-4517-40b7-b0ac-b76488925c28",
  "description": null,
  "status": "open",
  "input": {
    "unit": "EUR",
    "amount": 50
  },
  "paymentDetails": [
    {
      "network": "lightning",
      "paymentRequest": "lnbc939970n1pnvtm8ypp525ssrlm7senz7pf9z7h4sx7mz8nd2r7d6nxvldwjw7k2wm7hs8eqdp68psnxv3jxcekytf5x5cnwtf5xp3rwttzxpskxttzxumrgwpc8yer2cej8qcqzzsxqzjcsp5rd992e42ff4my6safcjce3gs0cz74rsuyffhqpragclzzkfw0lcq9qxpqysgq83hpdfxvsgh6ve8afvj38f2uuhg6nzm042sz98ge4qknfrcurlr8n3prw5qtmvjvcc7lpkhrfvm09ex5q3v6aq3tsprq958fakkrljgp2zfy80",
      "hash": "VSEB/36GZi8FJRevWBvbEebVD83UzM+10nesp2/XgfI="
    },
    {
      "network": "onchain",
      "address": "bc1qu2vs2z58m4dnyzuyk4240wdm2qd9jm827rnrjw"
    }
  ],
  "device": {},
  "extra": {
    "originalHash": "bb038002-b7cf-49e7-8425-da4d28e8242e",
    "tag": "invoice-web"
  },
  "createdAt": 1724247268,
  "delay": 3600,
  "pr": "lnbc939970n1pnvtm8ypp525ssrlm7senz7pf9z7h4sx7mz8nd2r7d6nxvldwjw7k2wm7hs8eqdp68psnxv3jxcekytf5x5cnwtf5xp3rwttzxpskxttzxumrgwpc8yer2cej8qcqzzsxqzjcsp5rd992e42ff4my6safcjce3gs0cz74rsuyffhqpragclzzkfw0lcq9qxpqysgq83hpdfxvsgh6ve8afvj38f2uuhg6nzm042sz98ge4qknfrcurlr8n3prw5qtmvjvcc7lpkhrfvm09ex5q3v6aq3tsprq958fakkrljgp2zfy80",
  "isPaid": false,
  "isExpired": false,
  "hash": "bb038002-b7cf-49e7-8425-da4d28e8242e",
  "fiatAmount": 50,
  "fiatUnit": "EUR",
  "onChainAddr": "bc1qu2vs2z58m4dnyzuyk4240wdm2qd9jm827rnrjw",
  "minConfirmations": 1,
  "isPending": false
}
```

Once the payment is made, calling the `/token/token_verify/<token_id>` endpoint will return `true` or `false`. If `true`, the `token_id` can then be used with the client deposit functions.
