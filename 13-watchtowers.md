# Watchtower protocol specification (BOLT DRAFT REV.1)

## Overview

All off-chain protocols assume the user remains online and synchronised with the network. To alleviate this assumption, customers can hire a third party watching service (a.k.a Watchtower) to watch the blockchain and respond to channel breaches on their behalf.

At a high level, every time there is a new transfer in the client's lightning channel, the client sends the Watchtower an encrypted penalty transaction and a transaction locator. Internally, the Watchtower maps the transaction locator to the encrypted penalty transaction. If there is a breach in the lightning channel, the Watchtower can identify it with the locator, and use the commitment transaction ID to compute the decryption key. With the decryption key, the tower decrypt the encrypted penalty transaction and broadcast it to the network. Therefore, the Watchtower does not learn any information about the client's channel unless there is a channel breach (channel-privacy). 

Due to replace-by-revocation Lightning channels, the client should send data to the Watchtower for every new update in the channel, otherwise the Watchtower cannot respond to all potential breaches.

Finally, optional extensions can be offered by the Watchtower to provide stronger guarantees to the client, such as a signed receipt for every new job. The rationale for the receipt is to build an _accountable_ Watchtower as the customer can later use it as publicly verifiable evidence if the Watchtower fails to protect them.

The scope of this document includes: 

- A protocol for client/server communication.
- A payment protocol between the customer and Watchtower. 
- How to build appointments for the Watchtower, including key/locator derivation and data encryption.
- A format for the signed receipt. 

The scope of this bolt does not include: 

 - Watchtower server discovery.
 
For the rest of this document we will use server/tower and client/Lightning node indistinguishably.

## Table of Contents 
* [Watchtower discovery](#watchtower-discovery)
* [Watchtower services](#watchtower-discovery)
	* [Basic Service](#basic-service)
	* [Extensions](#extensions)
* [User authentication](#user-authentication-and-subscriptions)
	* [The `register_top_up` message](#the-register_top_up-message)
	* [The `subscription_details` message](#the-subscription_details-message)
* [Sending appointments to the tower](#sending-appointments-to-the-tower)
 	* [The `add_update_appointment` message](#the-add_update_appointment-message)
 	* [The `appointment_accepted` message](#the-appointment_accepted-message)
 	* [The `appointment_rejected` message](#the-appointment_rejected-message)
* [Deleting appointments](#deleting-appointments)
	* [The `delete_appointment` message](#the-delete_appointment-message)
	* [The `deletion_accepted` message](#the-deletion_accepted-message)
	* [The `deletion_rejected` message](#the-deletion_rejected-message)
* [Transaction Locator and Encryption Key](#transaction-locator-and-encryption-key)
* [Encryption Algorithms and Parameters](#encryption-algorithms-and-parameters)
* [Payment Modes](#payment-modes)
* [Data serialisation and signing](#data-serialisation-and-signing)
* [No compression of penalty transaction](#no-compression-of-penalty-transaction)
* [Attacks on towers](#attacks-on-towers)

## Watchtower discovery

We have not defined how a client can find a list of servers to hire yet. We assume the client has found a server and the server is offering a watching service. 

## Watchtower services

### Basic Service
The customer can hire the Watchtower to watch for breaches on the blockchain and relay a penalty transaction on their behalf. The customer receives an acknowledgement when the Watchtower has accepted the job, but the hiring protocol does not guarantee the transaction inclusion.

### Extensions
Extensions build on top of the basic service and are optionally provided by the tower. Different extensions can be offered by the tower. For now we are defining a single type of extension: `accountability`.

#### `accountability`

A Watchtower provides a signed receipt to the customer. This is considered reputational accountability as the customer has publicly verifiable cryptographic evidence the Watchtower was hired. The receipt can be used to prove the Watchtower did not relay the penalty transaction on their behalf and/or request a refund.

## User authentication and subscriptions

Upon establishing the first connection with the tower, the client needs to register a public key. The registration aims to give the user access to the tower's services: 

		+-------+                                     +-------+
		|   A   |--(1)--         register        ---->|   B   |
		|       |<-(2)---  subscription_details  -----|       |
		+-------+                                     +-------+
			
			- where node A is 'client' and node B is 'server'
		
### The `register_top_up` message
The `register_top_up` message contains the information required to start the registration of a new public key with the tower, or to top up an existing one.

1. type: ? (`register_top_up`)
2. data:
   * [`point`:`public_key`]
   * [`u32`: `appointment_slots`]
   * [`u32`: `subscription_period`]

#### Rationale

We define appointment (in a lack of a better word) as how the Watchtower is hired by a client for its watching services. Every client updates the state of one of his channels, he will send an appointment containing information related to the update. Check [Sending appointments to the tower](#sending-appointments-to-the-tower) for more on this.

#### Requirements

The client:

- MUST set `public_key` to the public key he wants to register or top up.
- MUST set `appointment_slots` to the appointment slots he is requesting.
- MUST set `subscription_period` to the number of blocks he is asking the tower to watch for channel breaches.

Upon receiving a `register_top_up` message, the server:

- MUST reply with a `subscription_details` message.

### The `subscription_details` message

The `subscription_details` message contains information regarding the subscription the tower is offering to the client, including the subscription payment request if necessary.

1. type: ? (`subscription_details`)
2. data:
	* [`u16`: `appointment_max_size`]
	* [`u32`: `amount_msat`]
3. tlvs: `wt_subscription_tlvs`
4. types:
	1. type: 1 (`subscription_invoice`)
	2. data:
		* [`tu16*byte`: `invoice`]

#### Requirements

The server:

* MUST receive a `register_top_up` message before sending `subscription_details`.
* MUST set `appointment_max_size` to the maximum size allowed for a single appointment.
* MUST set `amount_msat` to the amount to be paid by the user for the service.
* MAY include `subscription_invoice` if it is offering a non-altruistic service. If so:
	* MUST set `invoice` to a [BOLT11](https://github.com/lightningnetwork/lightning-rfc/blob/master/11-payment-encoding.md) invoice.
	
The user:

* MUST have sent `register_top_up` before receiving a `subscription_details` message.

If `subscription_invoice` is set:

* MAY pay the [BOLT11](https://github.com/lightningnetwork/lightning-rfc/blob/master/11-payment-encoding.md) `invoice`.

#### Rationale 

User authentication with the tower is required to allow interaction with the tower beyond simply sending channel updates (like requesting, updating or deleting channel updates). Authentication is also required to account for the amount of data the client is sending to the tower, so subscriptions models can be properly defined.

The mechanism used to authenticate users by the tower is based on message signing and public key recovery. During the registration phase the user will request a public key registration with the tower (`public_key`).

`appointment_slots` and `subscription_period` are requested by the user to reduce the number of messages exchanged by the two parties whilst allowing for high customisation. Otherwise the tower will need to inform the user of what type of services he can apply for. 

`appointment_max_size` defines what is the maximum size of an appointment. The tower is effectively charging for storage over time, so if an appointment exceeds `appointment_max_size` it will be counted as `ceil(len(appointment)/appointment_max_size)`. 

Once the user is registered, the tower will be able to identify him by doing EC recovery on his signed requests. Message signing and EC recover is performed using the current approach followed by [lnd and c-lightning](#data-serialisation-and-signing).

If a user has filled all his appointment slots, or need to keep the data in the tower for longer than the `subscription_period`, he may need to top up his subscription.

For now we have only defined `subscription_invoice` as payment method. Other payment methods can be defined as tlv in the future.

## Sending appointments to the tower

Once the client is registered, he can start sending appointments to the tower:

		+-------+                                     +-------+
		|   A   |--(1)--- add_update_appointment ---->|   B   |
		|       |<-(2)---   accepted/rejected    -----|       |
		+-------+                                     +-------+
		
		- where node A is 'client' and node B is 'server'

### The `add_update_appointment` message

This message contains all the information regarding an appointment between the client and the server.

1. type: ? (`add_update_appointment`)
2. data:
   * [`16*byte`:`locator`]
   * [`u16`: `encrypted_blob_len`]
   * [`encrypted_blob_len*byte`:`encrypted_blob`]
   * [`u16`: `signature_len`]
	* [`signature_len*byte`: `user_signature`]
3. tlvs: `wt_accountability_tlvs`
4. types:
	1. type: 1 (`user_evidence`)
	2. data:
		* [`u64 `:`to_self_delay`]

#### Requirements

The client:

* MUST set `locator` as specified in [Transaction Locator and Encryption Key](#transaction-locator-and-encryption-key).
* MUST set `encrypted_blob` to the encryption of the `penalty_transaction` as specified in [Transaction Locator and Encryption Key](#transaction-locator-and-encryption-key).
* MUST set `encrypted_blob_len` to the length of `encrypted_blob`.
* MUST set `user_signature ` to the appointment signature.
* MUST set `signature_len` to the length of `user_signature`.
* MAY set `to_self_delay` to the `to_self_delay` requested by the client to its peer.

The server:

* MUST compute `public_key` by performing EC recover.
* MUST reject the appointment if the recovered `public_key` does not match with any of the registered ones.
* MAY reject the appointment if `encrypted_blob` has unreasonable size.

If `accountability` is being offered and `to_self_delay` can be found in `add_update_appointment`:

* MUST reject the appointment if `to_self_delay` is too small.
* MAY accept the appointment otherwise.

If the server accepts the appointment:

* MUST send an `appointment_accepted` message.

If the server rejects the appointment:

* MUST send an `appointment_rejected` message.


#### Signing appointment requests

Appointment request must be arranged as follows while serialised for signing:

	locator | encrypted_blob | to_self_delay
	
`to_self_delay` will only be included if it is also included in the request. 

The signature must be performed following [Data serialisation and signing](#data-serialisation-and-signing).

#### Rationale

Users must have preregistered before they can hire the Watchtower, as discussed in [User authentication](#user-authentication-and-subscriptions). Appointments from non-registered users are therefore rejected.

A client may need to update an appointment after having sent it to the tower (for instance to update the fee, change the outputs, etc). The same message can be used to add new appointments or to update existing ones. If two appointments from the same user share a `locator`, the tower should interpret that as an update and override the oldest. `locators` are `128-bit` values so unintended collisions within the same user should be negligible.

The `encrypted_blob` size depends on the encrypted commitment transaction size and the block size of the chosen cipher. Arbitrarily small/big transaction are invalid, meaning that arbitrarily small/big `encrypted_blob`s will, therefore, also be invalid.

The `encrypted_blob` have to be larger than (or equal to):

`cipher_block_size * ceil(minimum_viable_transaction_size / cipher_block_size)`

and smaller than (or equal to):

`cipher_block_size * ceil(maximum_viable_transaction_size / cipher_block_size`) 

`minimum_viable_transaction_size` and `maximum_viable_transaction_size` refer to the minimum/maximum size required to create a valid transaction. 

`encrypted_blob`s outside those boundaries cannot contain valid transactions, so they should be rejected.

A tower should broadcast the penalty transaction right after a breach is seen, but should be also able to bump the fee if necessary. If `to_self_delay` is smaller than expected, then it can lead the tower to fail.
	
### The `appointment_accepted` message

This message contains information about the acceptance of an appointment by the Watchtower.

1. type: ? (`appointment_accepted `)
2. data:
   * [`16*byte `:`locator`]
   * [`u32`: `start_block`]
3. tlvs: `wt_accountability_tlvs`
4. types:
	1. type: 2 (`receipt_signature`)
	2. data:
		* [`tu16*byte`: `tower_signature`]

The server:

* MUST receive `add_update_appointment` before sending an `appointment_accepted` message.
* MUST set the `locator` to match the one received in `add_update_appointment`.
* MUST set `start_block` to the block height where the tower will start looking for breaches.

If `accountability` is being offered and `add_update_appointment` contained `to_self_delay`:

* MUST create a receipt of the appointment.
* MUST set `tower_signature` to the signature of the receipt as defined in [Data serialisation and signing](#data-serialisation-and-signing).

The client:

* MUST fail the connection  if `locator` does not match any of locators the previously sent to the server.

#### Appointment receipt format 

Data must be arranged in the following order to create the receipt:

	locator | encrypted_blob | to_self_delay | user_signature | start_block
	
The receipt must be signed following [Data serialisation and signing](#data-serialisation-and-signing).

#### Rationale

`start_block` is set by the tower so the user knows when the channel update will be covered. 

We assume the tower has a well-known public key and the user is aware of it. The receipt contains, mainly, the information provided by the user. The Watchtower will need to sign the receipt to provide evidence of agreement.

The `user_signature` is included in the receipt to link both the client request and the server response. Otherwise, the tower could sign a receipt with different data that the one sent by the user, and the user would have no way to prove whether that's true or not. By signing the customer signature the tower creates evidence of what the user sent, since the tower cannot forge the client's signature.

### The `appointment_rejected` message

This message contains information about the rejection of an appointment by the Watchtower.

1. type: ? (`appointment_rejected `)
2. data:
   * [`16*byte `:`locator`]
   * [`u16`: `rcode`]
   * [`u16`: `reason_len`
   * [`reason_len*byte`: `reason`]

The server:

* MUST receive `add_update_appointment` before sending an `appointment_rejected` message.
* MUST set the `locator` to match the one received in `add_update_appointment`.
* MUST set `rcode` to the rejection code.
* MAY set an empty `reason` field.
* MUST set `reason_len` to length of `reason`.

#### Rationale

The `appointment_rejected` message follows the approach taken by the `error` message defined in [BOLT#1](https://github.com/lightningnetwork/lightning-rfc/blob/master/01-messaging.md#the-error-message): error codes are mandatory, whereas reasons are optional and implementation dependant.

## Deleting appointments

A client may want to delete appointments from the tower after a breach, or closing a channel:

		+-------+                                    +-------+
		|   A   |--(1)---   delete_appointment  ---->|   B   |
		|       |<-(2)---   accepted/rejected   -----|       |
		+-------+                                    +-------+
		
		- where node A is 'client' and node B is 'server'

### The `delete_appointment` message

1. type: ? (`delete_appointment`)
2. data:
   * [`16*byte `:`locator`]
   * [`u16`: `signature_len`]
	* [`signature_len*byte`: `user_signature`]

The user: 

* MUST set `locator` as specified in [Transaction Locator and Encryption Key](#transaction-locator-and-encryption-key).
* MUST set `user_signature` to the signature of the deletion request.
* MUST set `signature_len` to the length of `user_signature`.

The server:

* MUST compute `public_key` by performing EC recover.
* If the recovered `public_key` has an associated appointment with locator matching `locator`:
	* MUST delete the appointment and send `deletion_accepted`.
* Otherwise:
	* MUST reject the deletion request and send `deletion_rejected`.


#### Signing deletion requests

Deletion requests must contain a signature of the following message:

	"Delete appointment <locator>" 
	
where `<locator>` MUST match `locator`. 

The signature must be performed following [Data serialisation and signing](#data-serialisation-and-signing).

#### Rationale

Freeing expired appointment from the tower (after a channel clousure or breach) is beneficial for both parties. From the tower side, it reduces the load and free space, for the user side, it recovers space that was used by expired appointments, so it can be used to back up new channel updates.

### The `deletion_accepted` message

This message contains information about the acceptance of an appointment deletion by the Watchtower.

1. type: ? (`deletion_accepted`)
2. data:
   * [`16*byte `:`locator`]
3. tlvs: `wt_accountability_tlvs`
4. types:
	1. type: 2 (`receipt_signature`)
	2. data:
		* [`tu16*byte`: `tower_signature`]

The server:

* MUST receive `delete_appointment` before sending an `deletion_accepted` message.
* MUST set the `locator` to match the one received in `delete_appointment`.

If `accountability` is being offered it was requested for the appointment appointment:

* MUST set `tower_signature` to the signature of `user_signature` as specified following [Data serialisation and signing](#data-serialisation-and-signing).

The client:

* MUST fail the connection  if `locator` does not match the `locator` from `delete_appointment`.


### The `deletion_rejected` message

This message contains information about the rejection of an appointment deletion by the Watchtower.

1. type: ? (`deletion_rejected`)
2. data:
   * [`16*byte `:`locator`]
   * [`u16`: `rcode`]
   * [`u16`: `reason_len`
   * [`reason_len*byte`: `reason`]

The server:

* MUST receive `delete_appointment` before sending an `deletion_rejected` message.
* MUST set the `locator` to match the one received in `delete_appointment`.
* MUST set `rcode` to the rejection code.
* MAY set and empty `reason` field.
* MUST set `reason_len` to length of `reason`.

#### Rationale

Deletion rejection should include cases like the appointment cannot be found for the requesting user (either because it never existed, or because it was already deleted).
The same approach as for `appointment_rejected` error messages is followed here.

## Transaction Locator and Encryption Key

Implementations MUST compute the `locator`, `encryption_key` and `encryption_iv` from the commitment transaction as defined below: 

- `locator`: first half of the commitment transaction id (`commitment_txid(0,16]`)
- `encryption_key `: Hash of the commitment transaction id (`SHA256(commitment_txid)`) 
- `encryption_iv`: A 12-byte long 0 (`'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'`)


The server relies on both the encryption key and iv to decrypt the penalty transaction. Furthermore, the transaction locator helps the Watchtower identify a breach transaction on the blockchain. 

## Encryption Algorithms and Parameters

Users and servers MUST use [ChaCha20-Poly1305](https://tools.ietf.org/html/rfc7539) to encrypt / decrypt blobs. 

Sample code (python) for the client to prepare the `encrypted_blob`: 

	from hashlib import sha256
	from binascii import hexlify
	from cryptography.hazmat.primitives.ciphers.aead import ChaCha20Poly1305	
	
	def encrypt(penalty_tx, commitment_txid):
		# The SHA256 of the commitment txid is used as secret key, and 0 (12-byte long) as nonce.
		sk = sha256(commitment_txid).digest()
		nonce = bytearray(12)
	
		# Encrypt the data
		cipher = ChaCha20Poly1305(sk)
		encrypted_blob = cipher.encrypt(nonce=nonce, data=penalty_tx, associated_data=None)
		encrypted_blob = hexlify(encrypted_blob).decode("utf8")
	
		return encrypted_blob
	    
## Payment modes 

The three most common ways of paying a tower are:

**On-chain bounty**. An additional output is created in the penalty transaction that will reward the Watchtower. 

**Micropayments**. A small payment is sent to the Watchtower for every new job (e.g. over the lightning network).

**Subscription**. Watchtower is periodically rewarded / paid for their service to the customer. (e.g. over the lightning network or fiat subscription). 

The bounty approach has the benefit of paying the tower only if the job is done, but lets the customer hire many Watchtowers and only one Watchtower will be rewarded upon collecting the bounty (O(N) storage for each tower). On top of that, the onchain bounty allows a network-wise DoS attack for free on the watchtower network. However, it also has the benefit of allowing fee-bumping via CPFP by including an output dedicated to the tower.

The micropayment approach can be achieved using the same method as the subscription approach but setting the `appointment_slots` to one. Both micropayments and subscriptions are favourable for a Watchtower.

The ideal approach could be something in between. The tower is paid via a subscription to cover the storage costs and making DoS attacks have a financial cost. On top of that, the penalty transactions can include an output for the tower so 
the tower is encouraged to watch for beaches whilst allowing fee-bumping. 

## Data serialisation and signing

Request and receipts are serialised in big-endian, concatenating their values in the given order if they contain more than one value (as is the case of the appointment receipt). The signature algorithm follows the approach developed by lnd and currently implemented by both lnd and c-lightning [#specinatweet](https://twitter.com/rusty_twit/status/1182102005914800128):

Signatures are [zbase32 encoded](https://philzimmermann.com/docs/human-oriented-base-32-encoding.txt) and are performed over the sha256d of the message prefixed by `"Lightning Signed Message:"`, that is:

	zbase32(SigRec(SHA256(SHA256("Lightning Signed Message:" + serialised_data))))
	
For example, for a deletion request of appointment identified by locator `4a5e1e4baab89f3a32518a88c3bc87f6`, the structure will be:

	zbase32(SigRec(SHA256(SHA256("Lightning Signed Message:Delete appointment 4a5e1e4baab89f3a32518a88c3bc87f6))))

## No compression of penalty transaction 

The storage requirements for a Watchtower can be reduced (linearly) by implementing [shachain](https://github.com/rustyrussell/ccan/blob/master/ccan/crypto/shachain/design.txt), therefore storing the parts required to build the transaction and the corresponding signing key instead of the full transaction. For now, we have decided to keep the hiring protocol simple. Storage is relatively cheap and we can revisit this standard if it becomes a problem. 

## Attacks on towers
There are three main factors that define how easy it is, for a malicious user, to attack a tower: the `cost` of hiring the tower, the level of user `privacy` achieved by the service, and who has `access` to the tower's services.

The most vulnerable Watchtower will, therefore, be a cheap, public, and completely privacy preserving tower. Privacy being our mail goal, we have defined parts of this BOLT to prevent cheap attacks, such as favouring subscriptions over single appointments. Here's an example of what subscriptions try to protect from:

### Locator reuse attack

Given a locator `l`, a tower that provides a per-appointment hiring service (appointments can be bought one by one), and complete privacy (no registration), a malicious user could:

* Send `n` appointments to the tower, all identified by `l`
* Trigger a breach by sending a single old commitment (`txfee`)

The tower will need to store all appointments, since it has no clue which of them is the valid one (if any). On the other hand, the cost for the attacker will only be `n * appointment_cost + txfee`.

Upon detecting a breach, the tower will need to decrypt and analyse `n` transactions and left with the decision of which of them to broadcast (if any).

Using subscriptions, the tower will only store a single appointment, since all appointments with the same `l` will be seen as updates. An attacker will need `n` different subscriptions to attempt the same attack. Assuming a subscription has a minimum size of `m` appointments (`m >> 1`), the cost for the attacker will be `n * m * appointment_cost + txfee`. 

Check [Trustless WatchTowers?](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001203.html) for more on this.

[ADD MORE ATTACKS]

## FIXMES

- Define a proper tower discovery.
- None of the message types have been defined (they have been left with ?).
- Define errors (transient vs permanently).
- Extend attacks on towers

## DISCUSS

- The tower may also need to reply with `appointment_slots` during the registration phase so a minimum amount of appointments are paid for. Check [Attacks on towers](#attacks-on-towers). Therefore hiring the tower for a single appointment may be problematic.
- Signature on the deletion acceptance by the server may not be necessary.
- Appointment deletion can be performed in bulk, by allowing sending more than one appointment at a time. That could result in a privacy leak though, since the tower will be able to link what appointments belonged to the same channel.
	- Recognition codes, by ZmnSCPxj, may help here.
- Separate register and top up so proofs can be used for top ups, in a similar way to [Dead Men's Button](https://github.com/joostjager/deadmensbutton)


## Acknowledgments


## Authors

Patrick McCorry, Sergi Delgado, PISA Research. 

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
