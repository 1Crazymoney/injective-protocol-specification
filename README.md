# Injective Protocol 0.1.0 Specification

## Table of contents

1.  [Architecture](#architecture)
1.  [Sidechain](#sidechain)
    1.  [Decentralized Orderbook](#decentralizedorderbook)
    1.  [Trade Execution Coordinator](#tradeexecutioncoordinator)
1.  [Filter Contract](#filter)
1.  [Interface](#interface)
1.  [Governance](#governance)
1.  [Token Economics](#tokeneconomics)
1.  [Miscellaneous](#miscellaneous)
    1.  [Verifiable Delay Function](#verifiabledelayfunction)



# Architecture

Injective Protocol is a fully decentralized exchange protocol built on top of Ethereum and Cosmos. 

The protocol is comprised of three principal components: 1) the Injective sidechain relayer network, 2) Injective's filter contract (smart contract on Ethereum), and (optionally) 3) a front end interface. In this setup, the front end interface is used to communicate orders to and from the sidechain relayer network which serves as a decentralized orderbook and trade execution coordinator (TEC). The sidechain relayer network aggregates trades in a canonical ordering (preventing front-running) and then submits the trades on Injective's [filter contract](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#filter-contracts) which in turn executes and settles the trades on 0x. 



<img alt="diagram-injective.png" src="https://cl.ly/f9077bd91d24/download/diagram-injective.png" width="700px"/>

# Sidechain

Injective Protocol uses an application-specific sidechain relayer network to maintain a decentralized orderbook, front-running resistant trade execution coordinator, and order matching and execution engine. We refer to our model as **sidechain order relay with on-chain settlement** - a decentralized implementation of the traditionally centralized [off-chain order relay](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md#architecture) used by nearly all central limit order book decentralized exchanges. The sidechain is built on top of Tendermint and the application logic is implemented using the Cosmos-SDK. 

### Decentralized Orderbook

The sidechain maintains a decentralized, censorship-resistant orderbook which stores and relays 

Users submit signed 0x make orders to a relayer (i.e. sidechain node) of their choosing via the relayer's REST endpoint. The relayer performs basic order validation such as checking the user's token balance, approvals, trade signature verification, and trade format integrity. If the conditions are met, then the relayer should relay the make order to the decentralized relay network through a Tendermint transaction. Relayers can (and should) optionally enforce that any orders submitted through their endpoint have the feeRecipientAddress parameter in the 0x order set to their own Ethereum address. This property exists to properly compensate the relayer for sourcing the order (described later) and is easily configurable through the front-end interface.

The transaction containing the order information is relayed throughout the Tendermint based network, and is included in a block over which consensus is reached (thereby updating the sidechain application state which maintains the decentralized orderbook). Now, the order information can be queried by other users through a REST endpoint, which the relayer should also support. 

Another user can then decide to take this order (by submitting a signed 0x order denoting this intention to our sidechain) via our SRA REST endpoint. As before, basic validation on the order is performed and then the transaction denoting the take order intention is then included in the Tendermint block. At the end of every block, a canonical ordering of the take orders is determined, where VDF output size is used to order take orders. (Note: soft-cancel orders also change the canonical ordering of the take orders, but are not discussed here for simplicity purposes). Every predetermined number of sidechain blocks (e.g. 15 sidechain blocks) one "randomly selected" sidechain node is responsible to submitting the aggregated trades over the 15 blocks to Ethereum. The sidechain node should not modify the ordering/existence of the trades to submit to Ethereum, as doing so will result in the node being slashed. The node has the responsibility of pricing the gas sent with the transaction in order to ensure the transaction executes on Ethereum successfully, along with other considerations such as splitting the trades into multiple Ethereum transactions so the gas limit is not exceeded, and resending transactions with higher gas fees if they are stuck in the mempool. 

Submitting the trades to Ethereum means that the node will submit the trades to our filter contract which executes the trades on 0x. As such, the observer pays the Ethereum gas fees for submitting the order. The filter contract is permissioned and only allows stakers to call its methods. The filter contract receives a fee based off the trading pair to compensate relayers and cover the gas costs. The fee amount (denoted in ERC-20 tokens or ETH) is then distributed to the submitting observer as well as the other sidechain node (the exact split is yet to be determined but is calculated based off stake). 

Once the trade settles, Ethereum events (such as those indicating a successful trade) are emitted. The observer (or perhaps any random person) relays this information to the sidechain network through a Tendermint transaction in order to prune the orderbook. There should be some incentive mechanism for this, such as earning some small amount of tokens. The observer can also submit a claim that one of the past relayers did not submit the trades he was responsible for at all/ in the correct order. When this occurs, the sidechain nodes will verify the veracity of the claim and if holds true, then the offending node is slashed. 