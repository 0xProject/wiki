A relayer is any party or entity which hosts an off-chain orderbook. They provide a way for users to add, remove and update this orderbook through an API, GUI or both. In doing so, relayers help traders discover counter-parties and ferry cryptographically signed orders between them. Once two parties agree on the terms of an order, the order is settled directly on the Ethereum blockchain via the 0x protocol smart contracts.

### Advantages of decentralized exchange

There are many advantages to a decentralized exchange over centralized exchange. One of the largest problems with centralized Exchanges is that they must hold and secure the funds of all the traders on their platform. This causes there to be a single point of failure that if hacked or mismanaged, means that all these traders could lose their funds. Hundreds of millions have already been stolen in this way. With a decentralized exchange, traders do not need to deposit funds with a centralized entity, but instead can trade directly from their individual wallets (including hardware wallets!). This means the user is in control of their funds at all times, eliminating this single point of failure from the equation.

### 0x protocol overview

In 0x protocol, orders are transported off-chain over any arbitrary medium, massively reducing gas costs and reducing blockchain bloat. Relayers help broadcast orders and collect a fee each time they facilitate a trade. Anyone can build a relayer.

The simplest example of a Relayer is a website allowing users to create, discover and fill orders. The Relayer must build out a UI and host a backend database to provide this functionality.

<div align="center">
    <img src="https://s3.eu-west-2.amazonaws.com/0x-wiki-images/relayer_diagram.png" style="padding-bottom: 20px; padding-top: 20px; max-width: 342px;" width="80%" />
</div>

To simplify the process of interacting with the 0x protocol, we have written a Javascript/Typescript library called [0x.js](http://0xproject.com/docs/0x.js). This library helps relayers interact with the 0x protocol smart contracts through a higher-level, easier to use interface. In addition, it also provides many useful utilities for hashing, signing, validating and serializing 0x orders.

Before getting started with [0x.js](http://0xproject.com/docs/0x.js) and the 0x protocol, it is helpful to introduce a few concepts. There are two parties involved in every trade, a maker and a taker. The maker creates an order for an amount of TokenA in exchange for an amount of TokenB. The makers then submits these to a relayer. Takers discover orders via a relayer and fills them by sending them directly to the 0x protocol smart contracts. The 0x protocol smart contracts performs an atomic swap, exchanging the maker and taker tokens.

### Order

Below is an interface description of the 0x protocol order format:

```typescript
interface Order {
    // Ethereum address of the Maker
    maker: string;
    // Ethereum address of the Taker. If no address specified, anyone can fill the order.
    taker: string;
    // How many ZRX the Maker will pay as a fee to the relayer
    makerFee: BigNumber;
    // How many ZRX the Taker will pay as a fee to the relayer
    takerFee: BigNumber;
    // The amount of tokens the Maker is offering to exchange
    makerTokenAmount: BigNumber;
    // The amount of tokens the Maker is willing to accept in return
    takerTokenAmount: BigNumber;
    // The address of the token the Maker is offering
    makerTokenAddress: string;
    // The address of token the Maker is requesting in return
    takerTokenAddress: string;
    // Random number to guarantee order uniqueness
    salt: BigNumber;
    // The address of the 0x protocol exchange smart contract
    exchangeContractAddress: string;
    // The address that will receive the fees
    feeRecipient: string;
    // When the order will expire (unix timestamp in seconds)
    expirationUnixTimestampSec: BigNumber;
}
```

It is a relayers job to collect cryptographically signed versions of these orders into an off-chain database. This collection of orders is what we refer to as an orderbook. A relayer displays their orderbook to potential takers. The incentive here is for a relayer to collect fees from the orders they host. By being the fee recipient, relayers can earn fees in ZRX tokens.

We have a tutorial on how to [Create, Validate and Fill Orders](https://0xproject.com/wiki#Create,-Validate,-Fill-Order) for when you are ready to jump in and start developing on 0x. This tutorial will take you through setting up [0x.js](http://0xproject.com/docs/0x.js) in order to create, sign and fill a 0x order.

### Relayer strategies

The 0x protocol leaves room for some flexibility on how exactly relayers operate. There are a few different strategies relayers have employed so far, each with their respective advantages and drawbacks. You can read more about the different strategies in the [Relayer Strategies](https://0xproject.com/wiki#Matching) section of the wiki to discover the strategy that suits your needs the best.

### Shared liquidity

Because all relayers represent orders using the 0x protocol order format, an order created on one relayer can be filled by users on another relayer. What this means is that rather then each relayer having a siloed liquidity pool, they can share orders to created a shared liquidity pool. New relayers can bootstrap their liquidity off of existing relayers, immediately becoming an interesting place to trade. It is important to note that fees always go to the relayer where the order was first submitted. We have defined a [Standard Relayer API](https://github.com/0xProject/standard-relayer-api) to help relayers share liquidity and to simplify the integration process for market makers, providing a unified interface for them to build against.

### Pruning your orderbooks

Over time, orders may expire, partially filled, cancelled or no longer fillable. It is best to keep your orderbook free of expired or unfillable orders. Iterating over the orderbook periodically and checking each orders validity is a simple way to accomplish this. For a more advanced and efficient approach, you could use an [OrderWatcher](https://0xproject.com/wiki#0x-OrderWatcher).

### Next steps

Now that you have a high level idea of what a relayer does, it's time to get started learning how to [Create, Validate and Fill Orders](https://0xproject.com/wiki#Create,-Validate,-Fill-Order) on 0x. You may also want to decide on a [Relayer Strategy](https://0xproject.com/wiki#Open-Orderbook), we recommend the open orderbook strategy. If you're looking for more orders to add to your orderbook, take a look at the [Standard Relayer API](https://github.com/0xProject/standard-relayer-api). There are several relayers that have already implemented it and are broadcasting orders. Remember to keep your orderbook free of stale orders by using an [OrderWatcher](https://0xproject.com/wiki#0x-OrderWatcher).
