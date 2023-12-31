# On-chain swaps between bitcoin and Smart chain utilizing bitcoin relay

Describes a trustless swap between native bitcoin and Smart chain tokens (ERC-20, SPL, native tokens).

Uses [bitcoin relay](https://github.com/adambor/crosslightning-readme/blob/main/btcrelay.md) for proofs that a bitcoin transaction was really sent and confirmed on bitcoin blockchain. This works by storing SPV (simplified payment verification) copy of the bitcoin blockchain on Smart chain, meaning bitcoin blockheaders (with merkle roots) are verified and stored on Smart chain. Anyone can then prove that he sent a bitcoin transaction with a merkle proof.

## PTLC (proof-time locked contract)
Contract similar to HTLC (hash-time locked contract), where claimer needs to provide a proof instead of a secret for a hash. In this case the proof is transaction verification through [bitcoin relay](https://github.com/adambor/crosslightning-readme/blob/main/btcrelay.md).



## Smart chain -> Bitcoin on-chain

### Parties
- **payer** - the one paying in tokens on Smart chain and using intermediary to do the swap
- **intermediary** - handling the swap. receives tokens on smart chain and sends native on-chain bitcoin
- **payee** - recipient of the bitcoin on-chain payment

### Process
1. **Payer** queries the **intermediary** off-chain, sending the payee's bitcoin address and an amount he wishes to send, along with random _nonce N_, **intermediary** returns the network fee, his swap fee needed for the swap and a signed message _Mi (initialize)_ allowing the **payer** to construct a PTLC on Smart chain (this is done prevent DDoS attacks on intermediary, spamming him with requests he is unable to process)
2. **Payer** reviews the returned fees and uses the signed messages _Mi (initialize)_ to construct a PTLC on Smart chain:
	- paying the funds to **intermediary** if he can prove that he sent a pre-agreed amount to **payee**'s address in a bitcoin transaction (tagged with _nonce N_ to prevent replay attacks) that has >=6 confirmations
	- refunding the **payer**, but only after _locktime T_
	- refunding the **payer**, but only with a specific message _Mr (refund)_ signed by **intermediary** (for co-operative close, when payment fails for any reason)

	**NOTE:** If **payer** tries to send same amount to same address twice, even though bitcoin addresses should not be reused this can't be avoided as **payee's** wallet implementation is out of our influence, in this case **intermediary** could use the same transaction proof to claim both of the PTLCs and paying bitcoin transaction just once. To prevent this, it is required that each transaction is tagged with the 7-byte _nonce N_ with least significant 3 bytes prefixed with 0xFF being written as _nSequence_ field for ALL inputs and rest - 4 most significant bytes being treated as integer, adding 500,000,000 to that integer and writing it as _locktime_ field for the transaction.

3. **Intermediary** observes the creation of PTLC on Smart chain and proceeds to send a bitcoin transaction.

##### Successful payment
4. Transaction confirms on bitcoin chain (has >=6 confirmations) **intermediary** proves this via bitcoin relay and claims the funds from the PTLC

##### Failed payment
4. The payment was unsuccessful, maybe **intermediary** ran out of funds in the meantime, or **intermediary** thinks it's not possible to safely send the transaction with pre-agreed fee and have it confirm in under _locktime T_.
5. Upon request by **payer**, **intermediary** creates a specific signed message _Mr (refund)_, allowing the **payer** to refund his funds from the PTLC (cooperative close)

##### Intermediary went offline
4. **Payer** waits till the expiry of _locktime T_ and then refunds his funds back from the PTLC


## Bitcoin on-chain -> Smart chain

### Requirements
- an amount of bitcoin to receive must be known upfront
- for abslute maximum security payee should be online till bitcoin transaction confirms otherwise he could lose funds

### Parties
- **payee** - recipient of the tokens on Smart chain, using intermediary to do the swap
- **intermediary** - handling the swap, sends tokens on smart chain and receives bitcoin on-chain payment
- **payer** - the one paying bitcoin on-chain

### Process
1. **Payee** queries the **intermediary** off-chain, with an amount he wishes the receive and _locktime T_ he wishes to use, **intermediary** returns his swap fee needed for a swap along with a specific message _Mi (initialize)_ signed by **intermediary** allowing payee to create a PTLC on Smart chain with funds pulled from **intermediary's** vault. **Intermediary** can also charge a non-cooperation penalty (security deposit) based on _locktime T_ to disincetivize spamming - in case the swap fails **intermediary** will get this as compensation for locking up his funds for _locktime T_.
2. **Payee** reviews the returned fee and sends a transaction creating PTLC on Smart chain using message _Mi (initialize)_ signed by **intermediary** to pull funds from his vault:
	- paying the funds to **payee** if he can prove that a pre-agreed amount was sent to **intermediary's** address in a bitcoin transaction that has >=6 confirmations
	- refunding the **intermediary**, but only after _locktime T_
	- refunding the **intermediary**, but this can only be intiated by **payee**, (for co-operative close, when payee wishes to cancel the payment), this way **payee** can get part of his non-refundable fee back

	**NOTE:** Here the replay protection needs to be handled by the **intermediary** who needs to make sure to generate a new address for every swap (if he doesn't, he is the one losing money), as we cannot possibly influence fields such as _nSequence_ and _locktime_ in bitcoin transaction sent by payer because his wallet implementation is outside our influence.

##### Successful payment
3. Transaction confirms on bitcoin chain (has >=6 confirmations) **payee** proves this via bitcoin relay and claims the funds from the PTLC

##### Failed payment
3. No payment arrived in **intermediary's** wallet address until the _locktime T_, therefore **intermediary** can refund his funds back from the PTLC, while keeping the non-cooperation penalty (security deposit).


## Watchtowers
For Bitcoin on-chain -> Smart chain swaps, the payee needs to be online in time after the bitcoin transaction confirms on bitcoin blockchain, should he wait too long before claiming the funds, the PTLC might expire and he looses the funds.
### Solutions
1. Having the locktimes for the PTLCs very long (1 week or 1 month)
    - not ideal, since it will make the swaps very capital inefficient for intermediaries.
2. Delegate this act of claiming to a third party, a Watchtower, which will be always online and claims the swap for us as soon as it confirms on bitcoin chain
    - requires us to trust the watchtower we use and also somehow incentivize it

An approach for using watchtower was choosen for CrossLightning.

We can reduce the trust aspect by using multiple watchtowers (requiring just 1 out of N honest party), further we can incentivize watchtowers by paying a small fee to whoever can claim the PTLC on Smart chain on behalf of the claimer. We can then merge this watchtower implementation with BTC Relay implementation, allowing the relayers to become watchtowers to earn fees by claiming the swaps on behalf of the claimers.


### Parties
- **claimer** - recipient of the tokens from PTLC on the Smart chain
- **watchtower** - claiming the tokens to claimer's account, on claimer's behalf

### Process
1. **Watchtower** observes an event of creation of PTLC on-chain (PTLC **claimer** must explicitly opt-in for this feature by providing a txoHash - tx output hash when creating a PTLC)
2. **Watchtower** starts checking if subsequent bitcoin blocks contain the required transaction output specified by txoHash
3. If the transaction was found, the **watchtower** waits till that transaction gets required number of confirmations on bitcoin blockchain
4. **Watchtower** claims the funds from the PTLC to the **claimer**'s account, and receives a watchtower fee from the PTLC (which was initially paid by the **claimer**).

### Security considerations
Requires at least 1 honest relayer in the network to be online before the PTLC expires, otherwise the claimer may lose his funds if he doesn't claim them himself.
