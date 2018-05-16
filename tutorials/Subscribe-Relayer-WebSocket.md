In this tutorial, we will show you how you can use the [@0xproject/connect](https://github.com/0xProject/0x.js/tree/development/packages/connect) package in conjunction with [0x.js](https://github.com/0xProject/0x.js/tree/development/packages/0x.js) in order to:

* subscribe to a relayer's orderbook
* receive orderbook updates from a relayer in real time
* take action on the updates

You can find all the `@0xproject/connect` documentation [here](https://0xproject.com/docs/connect).

### Setup

For this tutorial we will be using [TestRPC](https://github.com/ethereumjs/testrpc) as our ethereum node, a local http server, and a local WebSocket server as our relayer. The The [0x starter project](https://github.com/0xProject/0x-starter-project) has everything we need to get started.

Clone the repo:

```
git clone git@github.com:0xProject/0x-starter-project.git
```

Install all the dependencies (we use Yarn: `brew install yarn --without-node`):

```
yarn install
```

Pull the latest TestRPC 0x snapshot with all the 0x contracts pre-deployed and an account with ZRX balance:

```
yarn download_snapshot
```

In a separate terminal, navigate to the project directory and start TestRPC:

```
yarn testrpc
```

In another terminal, navigate to the project directory and start a local http server and local WebSocket server that will act as our relayer, listening to port 3000 and 3001 respectively:

```
yarn api
```

You can now run the first tutorial script. This command will first build and run the `src/tutorials/relayer_websocket/generate_initial_book.ts` script to fill our fake api with orders signed by addresses available from testrpc. Next, it will run the `src/tutorials/relayer_websocket/index.ts` script, which we will take a closer look at below.

```
yarn relayer_websocket:part1
```

If our setup was successful the output of the command should look like this:

```
Listening for ZRX/WETH orderbook...
SNAPSHOT: 9 bids & 9 asks
```

### Importing packages

The first step to interacting with `@0xproject/connect` is to import the following relevant packages:

```javascript
import { ZeroEx, ZeroExConfig } from '0x.js';
import {
    OrderbookChannel,
    OrderbookChannelHandler,
    OrderbookChannelSubscriptionOpts,
    WebSocketOrderbookChannel,
} from '@0xproject/connect';
import * as Web3 from 'web3';

import { CustomOrderbookChannelHandler } from './custom_orderbook_channel_handler';
```

**ZeroEx** is the `0x.js` library, which allows us to interact with the 0x smart contracts and environment. **@0xproject/connect** is a set of tools and types that let us easily interact with relayers that conform to the standard relayer api. **Web3** is the package allowing us to interact with our node and the Ethereum world. **CustomOrderbookChannelHandler** is a class we'll learn how to write later!

### Instantiating ZeroEx and WebSocketOrderbookChannel

First, we create a `ZeroEx` instance with a provider pointing to our local TestRPC node at **http://localhost:8545**. You can read about what providers are [here](#Web3-Provider-Explained). We also pass in a `ZeroExConfig` instance specifying the default testrpc networkId. This allows our `ZeroEx` instance to use the correct addresses of the contracts deployed on that network.

```javascript
// Provider pointing to local TestRPC on default port 8545
const provider = new Web3.providers.HttpProvider('http://localhost:8545');

// Instantiate 0x.js instance
const zeroExConfig: ZeroExConfig = {
    networkId: 50, // testrpc
};
// Instantiate 0x.js instance
const zeroEx = new ZeroEx(provider, zeroExConfig);
```

Next, we create a `WebSocketOrderbookChannel` instance with a url pointing to a local standard relayer api WebSocket server running at **http://localhost:3001**. The `WebSocketOrderbookChannel` is our programmatic gateway to any relayer that conforms to the standard relayer api WebSocket standard.

```javascript
// Instantiate an orderbook channel pointing to a local server on port 3001
const relayerWsApiUrl = 'http://localhost:3001/v0';
const orderbookChannel: OrderbookChannel = new WebSocketOrderbookChannel(relayerWsApiUrl);
```

### Getting the exchange contract addresses

Next, we use `0x.js` to get the address of the exchange contract on the current network.

```javascript
// Get exchange contract address
const EXCHANGE_ADDRESS = await zeroEx.exchange.getContractAddress();
```

### Getting token information

We can use the `tokenRegistry` field of our `ZeroEx` instance to get information related to the WETH and ZRX tokens on the current network. This information is required for interacting with token balances and allowances and for converting token amounts into base unit amounts. Because the `getTokenIfExistsAsync()` method may return `undefined` for inputs that don't represent registered token addresses, we must check for `undefined` results before proceeding. Addresses of other tokens can be acquired though the `tokenRegistry` field of a ZeroEx instance or on [Etherscan](https://etherscan.io/tokens).

```javascript
// Get token information
const wethTokenInfo = await zeroEx.tokenRegistry.getTokenBySymbolIfExistsAsync('WETH');
const zrxTokenInfo = await zeroEx.tokenRegistry.getTokenBySymbolIfExistsAsync('ZRX');

// Check if either getTokenIfExistsAsync query resulted in undefined
if (wethTokenInfo === undefined || zrxTokenInfo === undefined) {
    throw new Error('could not find token info');
}

// Get token contract addresses
const WETH_ADDRESS = wethTokenInfo.address;
const ZRX_ADDRESS = zrxTokenInfo.address;
```

### Deciding our subscription options

If in an application we need exchange functionality between two tokens, we can find a suitable order for our needs by subscribing to our `orderbookChannel`. The first step to doing so is creating an `OrderbookChannelSubscriptionOpts` instance describing the type of subscription we want. The `OrderbookChannelSubscriptionOpts` instance describes the `baseTokenAddress` and `quoteTokenAddress` of the orderbook that we want to subscribe to (learn more about the quote/base token terminology [here](https://en.wikipedia.org/wiki/Currency_pair)). In addition, it includes a `snapshot` property that signals our desire for a current snapshot of the orderbook as well as a `limit` describing how large the snapshot should be.

```javascript
// Generate OrderbookChannelSubscriptionOpts for watching the ZRX/WETH orderbook
const zrxWethSubscriptionOpts: OrderbookChannelSubscriptionOpts = {
    baseTokenAddress: ZRX_ADDRESS,
    quoteTokenAddress: WETH_ADDRESS,
    snapshot: true,
    limit: 20,
};
```

### Subscribing to the orderbook

Now that we have our subscription options, we just need an `OrderbookChannelHandler` instance to handle updates to the orderbook. We instantiate an instance of `CustomOrderbookChannelHandler` which implements `OrderbookChannelHandler` and pass in our `zeroEx` object. In the next section we'll take a closer look at the internals of `CustomOrderbookChannelHandler`. To start the subscription we make a call to the `subscribe()` method of our `orderbookChannel` and pass in our subscription options and handler.

```javascript
// Create a OrderbookChannelHandler to handle messages from the relayer
const orderbookChannelHandler: OrderbookChannelHandler = new CustomOrderbookChannelHandler(zeroEx);

// Subscribe to the relayer
orderbookChannel.subscribe(zrxWethSubscriptionOpts, orderbookChannelHandler);
console.log('Listening for ZRX/WETH orderbook...');
```

### Diving into custom handler

When we subscribed to the orderbook above, we passed in a new instance of `CustomOrderbookChannelHandler` as our `OrderbookChannelHandler`. The `OrderbookChannelHandler` is an object with four properties, each representing a function to be called back when our orderbook channel has an update for us. `onSnapshot()` will be called when the channel has received an orderbook snapshot to deliver to the client in the form of an `OrderbookResponse` instance. `onUpdate()` is called whenever the orderbook has a new `SignedOrder` on the bid or ask side. `onError()` is called whenever there is an issue communicating with the WebSocket. `onClose()` is called whenever the WebSocket has been closed and gives the client a chance to do any cleanup.

In `CustomOrderbookChannelHandler`, we provide implementations of each one of these functions. For each function we log a message in order to provide visual feedback of the subscription to the developer. The implementation of `onSnapshot()` below is what produced the output of `yarn tutorial2:part1` command in our setup. Our `onUpdate()` method contains some extra logic to fill any order at a rate at or above 6 ZRX/WETH.

```javascript
export class CustomOrderbookChannelHandler implements OrderbookChannelHandler  {
    private zeroEx: ZeroEx;
    constructor(zeroEx: ZeroEx) {
        this.zeroEx = zeroEx;
    }
    public onSnapshot(channel: OrderbookChannel, subscriptionOpts: OrderbookChannelSubscriptionOpts,
                      snapshot: OrderbookResponse) {
        // Collect bids and asks from the response log
        const numberOfBids = snapshot.bids.length;
        const numberOfAsks = snapshot.asks.length;
        console.log(`SNAPSHOT: ${numberOfBids} bids & ${numberOfAsks} asks`);
    }
    public async onUpdate(channel: OrderbookChannel, subscriptionOpts: OrderbookChannelSubscriptionOpts,
                          order: SignedOrder) {
        // Get order hash of new order and log
        const orderHash = ZeroEx.getOrderHashHex(order);
        console.log(`NEW ORDER: ${orderHash}`);

        // Look for asks
        if (order.makerTokenAddress === subscriptionOpts.baseTokenAddress) {
            // Calculate the rate of the new order
            const zrxWethRate = order.makerTokenAmount.div(order.takerTokenAmount);
            // If the rate is equal to or better than the rate we are looking for, try and fill it
            const TARGET_RATE = 6;
            if (zrxWethRate.greaterThanOrEqualTo(TARGET_RATE)) {
                const addresses = await this.zeroEx.getAvailableAddressesAsync();
                // This can be any available address of you're choosing, in this example addresses[0] is actually
                // creating and signing the fake orders we're receiving so we need to fill the order with
                // a different address
                const takerAddress = addresses[1];
                const txHash = await this.zeroEx.exchange.fillOrderAsync(
                    order, order.takerTokenAmount, true, takerAddress);
                await this.zeroEx.awaitTransactionMinedAsync(txHash);
                console.log(`ORDER FILLED: ${orderHash}`);
            }
        }
    }
    public onError(channel: OrderbookChannel, subscriptionOpts: OrderbookChannelSubscriptionOpts,
                   err: Error) {
        // Log error
        console.log(`ERROR: ${err}`);
    }
    public onClose(channel: OrderbookChannel, subscriptionOpts: OrderbookChannelSubscriptionOpts) {
        // Log close
        console.log('CLOSE');
    }
}
```

### Filling orders from the subscription

Because there has not been any activity with our local relayer, the script we ran above should still be running with an output of:

```
Listening for ZRX/WETH orderbook...
SNAPSHOT: 9 bids & 9 asks
```

In order to push more updates to our subscription we can run the second tutorial script in another terminal. (We should now have 4 terminals running, one for TestRPC, one for our local relayer api, one for our first tutorial script, and one for our second tutorial script)

```
yarn relayer_websocket:part2
```

This command will build and run the `src/tutorials/relayer_websocket/generate_new_orders_with_interval.ts` script. This script submits new orders signed by the first address provided by TestRPC every 3 seconds. Every third order, the script will increase the ZRX/WETH exchange rate. After the first increase, the new rate should satisfy the target rate defined in `CustomOrderbookChannelHandler` above and we should see this output from our first tutorial script:

```
NEW ORDER: 0x4c6fed03875f00456d71da297881854bb04ae61aa6f21f7057238e2789712160
NEW ORDER: 0x79c7cd027baaf0ab7715b8d18fd46d17b55e599be2fd2ff3c5b1a697c4ad2c60
NEW ORDER: 0x7ff4ca18167cf1cb35a12ac26fed4aa660a92fc080b997d721d3e76e463cb2bb
NEW ORDER: 0xec2294bdd34bade62ebfdd9e572e78d118d26e16717f7f36c2702843f59e5131
ORDER FILLED: 0xec2294bdd34bade62ebfdd9e572e78d118d26e16717f7f36c2702843f59e5131
```

### Wrapping up

Through this tutorial we learned how to:

* subscribe to a relayer's orderbook
* receive orderbook updates from a relayer in real time
* take action on the updates

For more information on how to use `0x.js`, go [here](https://0xproject.com/docs/0x.js).
