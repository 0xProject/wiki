In this tutorial, we will show you how you can use the [@0xproject/connect](https://github.com/0xProject/0x.js/tree/development/packages/connect) package in conjunction with [0x.js](https://github.com/0xProject/0x.js/tree/development/packages/0x.js) in order to:

* ask a relayer for fee information
* submit signed orders to a relayer with appropriate fees
* ask a relayer for a ZRX/WETH orderbook
* find the best orders in the orderbook
* fill orders from the orderbook using `0x.js`

You can find all the `@0xproject/connect` documentation [here](https://0xproject.com/docs/0xjs).

### Setup
---
For this tutorial we will be using [TestRPC](https://github.com/ethereumjs/testrpc) as our ethereum node and a local http service that conforms to the standard relayer api as our relayer. The [Connect starter project](https://github.com/0xProject/connect-starter-project) has everything we need to get started.

Clone the repo:

```
git clone git@github.com:0xProject/connect-starter-project.git
```

Install all the dependencies:

```
npm install
```

Pull the latest TestRPC 0x snapshot with all the 0x contracts pre-deployed and an account with ZRX balance:

```
npm run download_snapshot
```

In a separate terminal, navigate to the project directory and start TestRPC:

```
npm run testrpc
```

In another terminal, navigate to the project directory and start a local http server that conforms to the standard relayer API, listening to port 3000:

```
npm run api
```

You can now run the tutorial script
```
npm run dev
```

### Importing packages
---
The first step to interacting with `@0xproject/connect` is to import the following relevant packages:

```javascript
import * as Web3 from 'web3';
import BigNumber from 'bignumber.js';
import {
    FeesRequest,
    FeesResponse,
    HttpClient,
    OrderbookRequest,
    OrderbookResponse,
} from '@0xproject/connect';
import {
    Order,
    SignedOrder,
    ZeroEx,
} from '0x.js';
```

**Web3** is the package allowing us to interact with our node and the Ethereum world. **BigNumber** is a JavaScript library for arbitrary-precision decimal and non-decimal arithmetic. **@0xproject/connect** is a set of tools and types that let us easily interact with relayers that conform to the standard relayer api. **ZeroEx** is the `0x.js` library, which allows us to interact with the 0x smart contracts and environment.

### Instantiating ZeroEx and HttpRelayerClient
---

First, we create a `ZeroEx` instance with a provider pointing to our local TestRPC node at **http://localhost:8545**. You can read about what providers are [here](https://0xproject.com/wiki#Web3-Provider-Explained).

```javascript
// Provider pointing to local TestRPC on default port 8545
const provider = new Web3.providers.HttpProvider('http://localhost:8545');

// Instantiate 0x.js instance
const zeroEx = new ZeroEx(provider);
```

Next, we create an `HttpRelayerClient` instance with a url pointing to a local standard relayer api http server running at **http://localhost:3000**. The `HttpRelayerClient` is our programmatic gateway to any relayer that conforms to the standard relayer api.

```javascript
// Instantiate relayer client pointing to a local server on port 3000
const relayerApiUrl = 'http://localhost:3000';
const relayerClient = new HttpClient(relayerApiUrl);
```

### Declaring decimals and contract addresses
---
Because there are no decimals in the Ethereum virtual machine (EVM), we need to keep track of how many "decimals" each token possesses. Because we are only interacting with ZRX and ETH in this turorial we can use a shared constant `DECIMALS`. Next, we use `0x.js` to get the addresses of the contracts we care about on the current network. Addresses of other tokens can be acquired though the `tokenRegistry` member of a ZeroEx instance or on [Etherscan](https://etherscan.io/tokens).

```javascript
// The number of decimals ZRX and WETH have
const DECIMALS = 18;

// Get contract addresses
const WETH_ADDRESS = await zeroEx.etherToken.getContractAddressAsync();
const ZRX_ADDRESS = await zeroEx.exchange.getZRXTokenAddressAsync();
const EXCHANGE_ADDRESS = await zeroEx.exchange.getContractAddressAsync();
```

### Setting up accounts
---
Now, we can use `0x.js` to grab available addresses. We assign the first address to a variable `zrxOwnerAddress` because it has a balance of 100000000 ZRX in the snapshot. We assign the rest of the addresses to a variable called `wethOwnerAddresses` because we will generate WETH for these addresses.

``` javascript
// Get all available addresses
const addresses = await zeroEx.getAvailableAddressesAsync();

// Get the first address, this address is preloaded with a ZRX balance from the snapshot
const zrxOwnerAddress = addresses[0];

// Assign other addresses as WETH owners
const wethOwnerAddresses = addresses.slice(1);
```

### Setting unlimited allowance
---
The next step is to allow the 0x protocol to interact with our funds. 0x allows users to trade tokens directly from their accounts without the need to send their tokens anywhere. This is possible thanks to the `approve()` and `transferFrom()` functions that are part of the ERC20 token standard. In order to give the 0x protocol Proxy smart contract access to WETH and ZRX, we need to set *allowances* (you can read about allowances [here](https://tokenallowance.io/)).

```javascript
// Set WETH and ZRX unlimited allowances for all addresses
const setZrxAllowanceTxHashes = await Promise.all(addresses.map(address => {
    return zeroEx.token.setUnlimitedProxyAllowanceAsync(ZRX_ADDRESS, address);
}));
const setWethAllowanceTxHashes = await Promise.all(addresses.map(address => {
    return zeroEx.token.setUnlimitedProxyAllowanceAsync(WETH_ADDRESS, address);
}));
await Promise.all(setZrxAllowanceTxHashes.concat(setWethAllowanceTxHashes).map(tx => {
    return zeroEx.awaitTransactionMinedAsync(tx);
}));
```

### Generating WETH
---
 Another thing we need to do is "convert" ETH to WETH, because ETH is not ERC20 compliant. We iterate over each address in `wethOwnerAddresses` and deposit ETH into the WETH token contract in order to mint WETH (you can read about WETH [here](https://weth.io/)).

```javascript
// Deposit ETH and generate WETH tokens for each address in wethOwnerAddresses
const ethToConvert = ZeroEx.toBaseUnitAmount(new BigNumber(5), DECIMALS); // Number of ETH to convert to WETH
const depositTxHashes = await Promise.all(wethOwnerAddresses.map(address => {
    return zeroEx.etherToken.depositAsync(ethToConvert, address);
}));
await Promise.all(depositTxHashes.map(tx => {
    return zeroEx.awaitTransactionMinedAsync(tx);
}));
```

### Ready to interact with the relayer
---
Now we are ready to start interacting with the relayer! For each address in `wethOwnerAddresses`, we will be getting fee information from the relayer, generating a complete order, signing the order, and submitting the order to the relayer.

```javascript
await Promise.all(wethOwnerAddresses.map(async (address, index) => {
    // Steps for the "Getting fee information from the relayer" and "Signing and submitting an order the the relayer" sections go here
}
```

### Getting fee information from the relayer
---
Different relayers in the 0x ecosystem will implement specific fee structures depending on various factors including orderbook depth, time of day, promotions, etc. The standard relayer api gives relayers the ability to provide fine-grained fee information specific to their service on a per-order basis. Below, we compile an object that conforms to the `FeesRequest` interface. This interface defines a set of information that a relayer needs in order to provide accurate fee information. We then submit this request to the `getFeesAsync()` function of the `relayerClient` and receive a `FeesResponse` instance in response. The `FeesRequest` and `FeesResponse` can be combined to create a complete order with appropriate fees for this relayer.

```javascript
// Programmatically determine the exchange rate based on the index of address in wethOwnerAddresses
const exchangeRate = (index + 1) * 10; // ZRX/WETH
const makerTokenAmount = ZeroEx.toBaseUnitAmount(new BigNumber(5), DECIMALS);
const takerTokenAmount = makerTokenAmount.mul(exchangeRate);

// Generate fees request for the order
const ONE_HOUR_IN_MS = 3600000;
const feesRequest: FeesRequest = {
    exchangeContractAddress: EXCHANGE_ADDRESS,
    maker: address,
    taker: ZeroEx.NULL_ADDRESS,
    makerTokenAddress: WETH_ADDRESS,
    takerTokenAddress: ZRX_ADDRESS,
    makerTokenAmount,
    takerTokenAmount,
    expirationUnixTimestampSec: new BigNumber(Date.now() + ONE_HOUR_IN_MS),
    salt: ZeroEx.generatePseudoRandomSalt(),
};

// Send fees request to relayer and receive a FeesResponse instance
const feesResponse: FeesResponse = await relayerClient.getFeesAsync(feesRequest);

// Combine the fees request and response to form a complete order
const order: Order = {
    ...feesRequest,
    ...feesResponse,
};
```

### Signing and submitting an order the the relayer
---
Now that we created an order, we need to prove that we actually own the address specified in the `maker` field of `order`. To do so, we will sign the order with the corresponding private key and append the signature to our order to form a complete `SignedOrder`. We then submit this order to the `submitOrderAsync()` function of the `relayerClient`. If you are not using TestRPC as your Web3 Provider, you will need to pass a provider to `0x.js` that sends message signing requests to your signing service.

```javascript
// Create orderHash
const orderHash = ZeroEx.getOrderHashHex(order);

// Sign orderHash and produce a ecSignature
const ecSignature = await zeroEx.signOrderHashAsync(orderHash, address);

// Append signature to order
const signedOrder: SignedOrder = {
    ...order,
    ecSignature,
};

// Submit order to relayer
await relayerClient.submitOrderAsync(signedOrder);
```

### Requesting an orderbook
---
If in an application we need exchange functionality between two tokens, we can find a suitable order for our needs using the `getOrderbookAsync()` method of `HttpRelayerClient`. In this example, we use the addresses of the ZRX and WETH tokens we retrieved earlier and use them to generate an `OrderbookRequest` to send to the relayerClient. In response, we get an `OrderbookResponse` containing orders that correspond with the provided `quoteTokenAddress` and `baseTokenAddress` (learn more about the quote/base token terminology [here](https://en.wikipedia.org/wiki/Currency_pair)).

``` javascript
// Generate orderbook request for ZRX/WETH pair
const orderbookRequest: OrderbookRequest = {
    baseTokenAddress: ZRX_ADDRESS,
    quoteTokenAddress: WETH_ADDRESS,
};

// Send orderbook request to relayer
const orderbookResponse: OrderbookResponse = await relayerClient.getOrderbookAsync(orderbookRequest);
```

### Finding the best orders
---
`OrderbookResponse` contains two fields, `bids` and `asks`. `Bids` is a `SignedOrder` array where for each order, the `makerTokenAddress` field is equal to the `quoteTokenAddress` provided by the `OrderbookRequest` and the `takerTokenAddress` field is euqal to `baseTokenAdress`. `Asks` is also a `SignedOrder` array but it is the opposite of `bids`. For each order, the `makerTokenAddress` field is equal to the `quoteTokenAddress` and the `takerTokenAddress` field is equal to `baseTokenAdress`.

``` javascript
// Because we are looking to exchange our ZRX for WETH, we get the bids side of the order book and sort the orders with the best rate first
const sortedBids = orderbookResponse.bids.sort((orderA, orderB) => {
    const orderRateA = (new BigNumber(orderA.makerTokenAmount)).div(new BigNumber(orderA.takerTokenAmount));
    const orderRateB = (new BigNumber(orderB.makerTokenAmount)).div(new BigNumber(orderB.takerTokenAmount));
    return orderRateB.comparedTo(orderRateA);
});

// Find the orders we need in order to fill 300 ZRX
const bidsToBeFilled: SignedOrder[] = [];
let zrxToBeFilled =  ZeroEx.toBaseUnitAmount(new BigNumber(300), DECIMALS);
sortedBids.forEach(bid => {
    if (zrxToBeFilled.greaterThan(0)) {
        bidsToBeFilled.push(bid);
        zrxToBeFilled = zrxToBeFilled.minus(bid.takerTokenAmount);
    }
});

// Calculate and print out the WETH/ZRX exchange rate for each order
const rates = sortedBids.map(order => {
    const rate = (new BigNumber(order.makerTokenAmount)).div(new BigNumber(order.takerTokenAmount));
    return (rate.toString() + ' WETH/ZRX');
});
console.log(rates);
```

### Filling the orders
---
Now that we have the best WETH/ZRX orders that the relayer has to offer, we can use them to exchange some of the ZRX balance of `zrxOwnerAddress` into WETH from the `wethOwnerAddresses`. In this example, we wish to convert 300 ZRX into WETH at the best rate the relayer has to offer. We can do this using the `bidsToBeFilled` array we generated above, and the `fillOrdersUpToAsync()` function of the exchange wrapper provided by our instance of `ZeroEx`. When we print our balances, we can see that we successfully exchanged 300 ZRX for 15 WETH.

``` javascript
// Get balances before the fill
const zrxBalanceBeforeFill = await zeroEx.token.getBalanceAsync(ZRX_ADDRESS, zrxOwnerAddress);
const wethBalanceBeforeFill = await zeroEx.token.getBalanceAsync(WETH_ADDRESS, zrxOwnerAddress);
console.log('ZRX Before: ' + ZeroEx.toUnitAmount(zrxBalanceBeforeFill, DECIMALS).toString());
console.log('WETH Before: ' + ZeroEx.toUnitAmount(wethBalanceBeforeFill, DECIMALS).toString());

// Fill up to 300 ZRX worth of orders from the relayer
const zrxAmount = ZeroEx.toBaseUnitAmount(new BigNumber(300), DECIMALS);
const fillOrderTxHash = await zeroEx.exchange.fillOrdersUpToAsync(bidsToBeFilled, zrxAmount, true, zrxOwnerAddress);
await zeroEx.awaitTransactionMinedAsync(fillOrderTxHash);

// Get balances after the fill
const zrxBalanceAfterFill = await zeroEx.token.getBalanceAsync(ZRX_ADDRESS, zrxOwnerAddress);
const wethBalanceAfterFill = await zeroEx.token.getBalanceAsync(WETH_ADDRESS, zrxOwnerAddress);
console.log('ZRX After: ' + ZeroEx.toUnitAmount(zrxBalanceAfterFill, DECIMALS).toString());
console.log('WETH After: ' + ZeroEx.toUnitAmount(wethBalanceAfterFill, DECIMALS).toString());
```
