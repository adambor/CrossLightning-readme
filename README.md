# CrossLightning
[![Twitter](https://img.shields.io/twitter/url/https/twitter.com/SolLightning.svg?style=social&label=Follow%20%40SolLightning)](https://twitter.com/CrossLightning)

A fully trustless DEX (decentralized exchange) protocol between Smart chains (EVM, Solana, etc.) <-> Bitcoin (on-chain and lightning). Utilizing submarine swaps (for lightning network swaps) and bitcoin light client - on-chain SPV verification through [bitcoin relay](https://github.com/adambor/crosslightning-readme/blob/main/btcrelay.md) (for on-chain).

**NOTE:** We are not issuing a new wrapped bitcoin token on Smart chains, as that would require overcollateralization (and be exposed to exchange rate and oracle risks), which we deem insecure, therefore CrossLightning is only acting as a cross-chain DEX (Any smart chain token <-> Bitcoin).

## Explainers
- [Bitcoin lightning <-> Smart chain](https://github.com/adambor/CrossLightning-readme/blob/main/swaps/submarine.md)
- [Bitcoin on-chain <-> Smart chain](https://github.com/adambor/CrossLightning-readme/blob/main/swaps/onchain.md)
- [(WIP) Bitcoin single-use-seals tokens (RGB, Taro) <-> Smart chain](https://github.com/adambor/CrossLightning-readme/blob/main/swaps/onchain-singleuseseals.md)

## Navigation
#### Bitcoin relay
- [Solana: Bitcoin relay program](https://github.com/adambor/BTCRelay-Sol)
- [Solana: Bitcoin relay + Watchtower](https://github.com/adambor/BtcRelay-Sol-TS)
- [EVM: Bitcoin relay smart contract](https://github.com/adambor/BTCRelay)
- [EVM: Bitcoin relay + Watchtower](https://github.com/adambor/BtcRelay-EVM-TS)

#### Swaps
- [Solana: Swap on-chain program](https://github.com/adambor/SolLightning-program)
- [Solana: Swap intermediary](https://github.com/adambor/SolLightning-Intermediary-TS)
- [Solana: Swap SDK](https://github.com/adambor/SolLightning-sdk)
- [EVM: Swap smart contract](https://github.com/adambor/crosslightning-libs/blob/main/crosslightning-evm/solidity/SwapContract-dynamic-security-deposit.sol)
- [EVM: Swap intermediary](https://github.com/adambor/evmlightning-intermediary-ts)
- [EVM: Swap SDK](https://github.com/adambor/EVMLightning-sdk)

#### Proof of concept
- [Solana: MVP React web-app utilizing Swap SDK](https://github.com/adambor/SolLightning-PoC)
- [Linea: MVP React web-app utilizing Swap SDK](https://github.com/adambor/LineaLightning-webapp)

## Test the PoC

### Devnet/Testnet
You can access the demo PoC webapps on devnet/testnet:
 - [Solana](https://sollightningv2.z6.web.core.windows.net/)
 - [Linea](https://linealightningtest.z6.web.core.windows.net/)

1. Be sure to switch your wallet to devnet/testnet and have some native currency in, to cover transaction fees.
2. Use a lightning network testnet wallet [here](https://htlc.me/) (you will receive some testnet bitcoin when you create a wallet).
3. You can then try receiving (btcln -> smart chain), and sending (smart chain -> btcln).
4. For testing on-chain, you will need to download a bitcoin testnet wallet (unfortunatelly there is no online testnet web wallet).
5. For sending (smart chain -> btc) you can just try sending to some random testnet address (e.g. mijXVEL3Ko6fuE1p8M42R95ndVhgQGexEo) and then check it on block explorer [here](https://mempool.space/testnet/address/mijXVEL3Ko6fuE1p8M42R95ndVhgQGexEo)

### Mainnet
We are already deployed on mainnet on Solana & Linea (We are early with testing the product, therefore the liquidity is limited and swap sizes are limited to 0.01 BTC):
 - [Solana](https://app.solbtc.org/)
 - [Linea](https://linealightning.z6.web.core.windows.net/)

You can use any bitcoin lightning & on-chain wallet with our solution (please report any issues with specific lightning wallets in the github issues, we previously had troubles with Muun).

## Motivation
We are allowing wallets on any smart chain (solana, evm, etc.) to use existing bitcoin payment infrastructure - being able to seamlessly receive and pay with native bitcoin. Using lightning network, which with CrossLightning can become a common payment protocol not just limited to Bitcoin and turn into global decentralized chain-agnostic payment protocol - you are able to settle (send/receive) lightning invoices in any token on smart chains (USDC, USDT, ETH, SOL).

## Technology
### Bitcoin relay program
Works by storing an SPV (simplified payment verification) copy of bitcoin blockchain on-chain, with an on-chain program that verifies the blockheader consensus rules. Anyone can then prove that he really sent a bitcoin transaction (and it got confirmed - included in a block) with just a merkle proof.
More info [here](https://github.com/adambor/crosslightning-readme/blob/main/btcrelay.md).

### Submarine swaps (lightning network swaps)
Similar to atomic swaps, but exploits the property that lightning network invoice requires the recipient to reveal a pre-image for the payment to confirm. Can improve on security and speed of regular atomic swaps by locking the the swap not till a specific timestamp but till a **Bitcoin Relay** program reaches the specific blockheight, this way both sides use the same time-chain.
More info [here](https://github.com/adambor/crosslightning-readme/blob/main/swaps/submarine.md)

### Proof-time locked contracts (on-chain swaps)
Similar to atomic swaps, but the claimer needs to prove (with merkle proof and **Bitcoin Relay** program) that he sent the desired bitcoin transaction and it confirmed on bitcoin blockchain. Can improve on security and speed of regular atomic swaps by locking the the swap not till a specific timestamp but till a **Bitcoin Relay** program reaches the specific blockheight, this way both sides use the same time-chain.
More info [here](https://github.com/adambor/crosslightning-readme/blob/main/swaps/onchain.md)

## Protocol workings
Parties:
- __Intermediary node__
    - handles the cross-chain swaps
    - needs liquidity on both chains
    - determines prices
    - earns swap fees
    - has reputation (stored on-chain on smart chain) - successful, failed, cooperative closed swaps
- __Relayer/Watchtower__
    - submits bitcoin blockheaders to bitcoin relay program and keep it in sync
    - claims the swaps on behalf of clients
    - earns a small fee for every claimed swap
- __Client__
    - intiates the swap

The protocol is backed by __intermediary nodes__ and __relayers/watchtowers__, which can be run by anyone. There is a registry (for now a github repo, but will move on-chain), of every __intermediary node__ in the network.

Swap initialization:
1. __Client__ fetches __intermediary nodes__ from registry, fetches their fees and reputation
2. Based on these metrics the __client__ chooses an __intermediary node__ he wants to use
3. __Client__ sends a request to the desired __intermediary node__ to initiate the swap, if the node is unresponsive, or returns invalid data he blacklists it internally and proceeds to send the same request to next best __intermediary node__.

In case of Bitcoin on-chain to Solana swaps, the __client__ has to wait for certain number of confirmations on his bitcoin transaction till he is able to claim his funds on Solana (which with bitcoin's 10 minutes blocktime can take a long time). If a __client__ were to go offline during that time and not return before expiration of the PTLC (proof-time locked contract) he would loose his funds, therefore __relayers__ double-down also as __watchtowers__.
1. All __relayers/watchtowers__ observe creation of Bitcoin -> Solana swaps on-chain.
2. They do check if the bitcoin transaction corresponding to any of the currently active swaps did get enough confirmations in the blockheader that they are going to submit to bitcoin relay program.
3. If there is a bitcoin transaction that claims the swap they will claim it on behalf of the __client__, while earning a small fee paid for upfront by the __client__

## Code architecture

CrossLightning uses modular code architecture, abstracting away chain specific functionalities to create re-usable for Btc relay, Intermediary and Swap SDKs. Therefore most of the functionality is included in the [crosslightning-libs](https://github.com/adambor/crosslightning-libs) and [btcrelay-libs](https://github.com/adambor/btcrelay-libs) monorepos containing the libraries. See below a code architecture diagram (the names of the final products correspond to the repository names on github).

![Architecture diagram](https://github.com/adambor/crosslightning-readme/blob/main/crosslightning-architecture.png)


## Contact

Via mail: adamborcany(at)gmail(dot)com
