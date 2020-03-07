# Watchtower protocol specification (BOLT DRAFT REV.1)

## Overview

All off-chain protocols assume the user remains online and synchronised with the network. To alleviate this assumption, customers can hire a third party watching service (a.k.a Watchtower) to watch the blockchain and respond to channel breaches on their behalf. 

At a high level, the client sends an encrypted penalty transaction alongside a transaction locator to the Watchtower. Both the encryption key and the transaction locator are derived from the commitment transaction id, meaning that the Watchtower will be able to decrypt the penalty transaction only after the corresponding breach is seen on the blockchain. Therefore, the Watchtower does not learn any information about the client's channel unless there is a channel breach (channel-privacy).

Due to replace-by-revocation Lightning channels, the client should send data to the Watchtower for every new update in the channel, otherwise the Watchtower may not be able to respond to specific breaches. 

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
* [User authentication](#user-authentication)
	* [The `register_top_up` message](#the-register_top_up-message)
	* [The `subscription_deltais` message](#the-subscription_details-message)
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
Extensions build on top of the basic service are optionally provided by the tower. Different kind of extensions can be offered by the tower. For now we are defining a single type of extension `accountability`.

#### `accountability`

A Watchtower provides a signed receipt to the customer. This is considered reputational accountability as the customer has publicly verifiable cryptographic evidence the Watchtower was hired. The receipt can be used to prove the Watchtower did not relay the penalty transaction on their behalf and/or request a refund.

## User authentication

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

### Rationale 

User authentication with the tower is required to allow interaction with the tower beyond simply sending channel updates (like requesting, updating or deleting channel updates). Authentication is also required to account for the amount of data the client is sending to the tower, so subscriptions models can be properly defined.

The mechanism used to authenticate users by the tower is based on message signing and public key recovery. During the registration phase the user will request a public key registration with the tower (`public_key`).

`appointment_slots` and `subscription_period` are requested by the user to reduce the number of messages exchanged by the two parties whilst allowing for high customisation. Otherwise the tower will need to inform the user of what type of services he can apply for. 

`appointment_max_size` defines what is the maximum size of an appointment. The tower is effectively charging for storage over time, so if an appointment exceeds `appointment_max_size` it will be count as `ceil(len(appointment)/appointment_max_size)`. 

Once the user is registered, the tower will be able to identify him by doing EC recovery on his signed requests. Message signing and EC recover is performed using the current approach followed by [lnd and c-lightning](#data-serialisationa-and-signing).

If a user fills all his appointment slots, or need to keep the data in the tower for longer than the `subscription_period`, he may need to top up his subscription.

For now only `subscription_invoice` tlv has been defined as payment method. Other payment methods can be defined as tlv in the future.

## Sending appointments to the tower

Once the client is registered with the tower, he can start backing up state updates by sending appointments to the tower:

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
	1. type: 1. (`user_evidence`)
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

	txlocator | encrypted_blob {| to_self_delay}
	
`to_self_delay` will only be included if it is also included in the request. 

The signature must be performed following [Data serialisation and signing](#data-serialisation-and-signing).

#### Rationale

We define appointment as the way that the Watchtower is hired / requested by a client to do its watching services.

Users must be registered before any service can be provided, as discussed in [User authentication](#user-authentication). Appointments from non-registered users are therefore rejected.

A client may need to update an appointment after having sent it to the tower (for instance to update the fee, change the outputs, etc). The same message can be used to add new appointment or to update existing ones. If two appointments from the same user share a `locator`, the tower should interpret that as an update and override the oldest. `locators` are `128-bit` values so unintended collisions within the same user should be unlikely.

Block ciphers have a size multiple of the block length, which depends on the key size. Therefore the `encrypted_blob` have to be at least as big as:

`cipher_block_size * ceil(minimum_viable_transaction_size / cipher_block_size)`

and at most as big as:

`cipher_block_size * ceil(maximum_viable_transaction_size / cipher_block_size`) 

`minimum_viable_transaction_size` and `maximum_viable_transaction_size` refer to the minimum/maximum size required to create a valid transaction. 

`encrypted_blob`s outside those boundaries cannot contain valid transactions, so they should be rejected.

A tower should broadcast a penalty transaction right after a breach is seen, but should be also able to bump the fee is necessary. A too small `to_self_delay` can make the tower fail to do so. 
	
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

	[txlocator, encrypted_blob, to_self_delay, user_signature, start_block]
	
The receipt must be signed following [Data serialisation and signing](#data-serialisation-and-signing).

#### Rationale

`start_block` sis set by the tower so the user knows when the channel update will be covered. 

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
* MAY set and empty `reason` field.
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

Freeing expired appointment from the tower (after a channel clousure or breach) should be beneficial for both parties. From the tower side, it allows to reduce the load and free space, for the user side, it recovers space that was used by expired appointments, so it can be used to back up new channel updates.

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

The bounty approach has the benefit of paying the tower only if the job is done, but lets the customer hire many Watchtowers and only one Watchtower will be rewarded upon collecting the bounty (O(N) storage for each tower). On top of that, the onchain bounty allows a network-wise DoS attack for free. However, it also has the benefit of allowing fee-bumping via CPFP by including an output dedicated to the tower.

The micropayment approach can be achieved using the same method as the subscription approach but setting the `appointment_slots` to one. Both micropayments and subscriptions are favourable for a Watchtower.

The ideal approach could be something in between. The tower is paid via a subscription to cover the storage costs and making DoS attacks having a financial cost. On top of that, the penalty transactions can include an output for the tower so 
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

## FIXMES

- Define a proper tower discovery.
- None of the message types have been defined (they have been left with ?).
- Define errors (transient vs permanently).
- Add attacks on towers

## DISCUSS

- The tower may also need to reply with `appointment_slots` during the registration phase so a minimum amount of appointments are paid for. Check [Trustless WatchTowers?](https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-April/001203.html). Therefore hiring the tower for a single appointment may be problematic.
- Signature on the deletion acceptance by the server may not be necessary.
- Appointment deletion can be performed in bulk, by allowing sending more than one appointment at a time. That could result in a privacy leak though, since the tower will be able to link what appointments belonged to the same channel.
- Separate register and top up so proofs can be used for top ups, in a similar way to [Dead Men's Button](https://github.com/joostjager/deadmensbutton)


## Acknowledgments


## Authors

Patrick McCorry, Sergi Delgado, PISA Research. 

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
