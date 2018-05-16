In this tutorial, we will show you how you can use the [0x.js](https://github.com/0xProject/0x.js) library to create, validate and fill buy and sell orders via the 0x protocol. You can find all the `0x.js` documentation [here](https://0xproject.com/docs/0x.js).

### Setup

Since 0x.js helps you build apps that interface with Ethereum, you will need to use it in conjunction with an Ethereum node. For development, we recommend using [TestRPC](https://github.com/ethereumjs/testrpc). Since there is some setup required to getting started with TestRPC and the 0x smart contracts, we are going to use a [0x starter project](https://github.com/0xProject/0x-starter-project) which handles a lot of this for us.

Clone the repo:

```
git clone git@github.com:0xProject/0x-starter-project.git
```

Install all the dependencies (we use Yarn: `brew install yarn --without-node`):

```
yarn
```

Pull the latest TestRPC 0x snapshot with all the 0x contracts pre-deployed:

```
yarn download_snapshot
```

In a separate terminal, navigate to the project directory and start TestRPC:

```
yarn testrpc
```

You can now run the code we are going to walk you through in the rest of this tutorial:

```
yarn order_actions
```

### Importing packages

The first step to interact with `0x.js` is to import the following relevant packages:

```javascript
import { DecodedLogEvent, ZeroEx } from '0x.js';
import { BigNumber } from '@0xproject/utils';
import * as Web3 from 'web3';
```

**Web3** is the package allowing us to interact with our node and the Ethereum world. **ZeroEx** is the `0x.js` library, which allows us to interact with the 0x smart contracts and environment. **BigNumber** is a JavaScript library for arbitrary-precision decimal and non-decimal arithmetic.

### Provider and constructor

Now we need to instantiate a zeroEx and HttpProvider. In our case, since we are using our local node, we will use **http://localhost:8545**. You can read about what providers are [here](#Web3-Provider-Explained).

```javascript
// Provider pointing to local TestRPC on default port 8545
const provider = new Web3.providers.HttpProvider('http://localhost:8545');

// Instantiate 0x.js instance
const configs = {
    networkId: TESTRPC_NETWORK_ID,
};
const zeroEx = new ZeroEx(provider, configs);
```

### Declaring decimals and addresses

Since we are dealing with a few contracts, we will specify them now to reduce the syntax load. Fortunately for us, `0x.js` library comes with a bunch of contract addresses that can be useful to have at hand. It also comes with a way to interact with the 0x token registry, which act as a store of vetted tokens and their addresses. One thing that is important to remember is that there are no decimals in the Ethereum virtual machine (EVM), which means you always need to keep track of how many "decimals" each token possesses. Since we will sell some ZRX for some ETH and since they both have 18 decimals, we can use a shared constant.

```javascript
// Number of decimals to use (for ETH and ZRX)
const DECIMALS = 18;

// Addresses
const WETH_ADDRESS = zeroEx.etherToken.getContractAddressIfExists() as string; // The wrapped ETH token contract
const ZRX_ADDRESS = zeroEx.exchange.getZRXTokenAddress(); // The ZRX token contract
// The Exchange.sol address (0x exchange smart contract)
const EXCHANGE_ADDRESS = zeroEx.exchange.getContractAddress();
```

Above, **`EXCHANGE_ADDRESS`** is the [**Exchange.sol**](https://github.com/0xProject/contracts/blob/master/contracts/Exchange.sol) contract address, which is the 0x exchange smart contract which allows the maker to exchange ZRX for WETH with the taker. **`WETH_ADDRESS`** and **`ZRX_ADDRESS`** are the addresses of the tokens we will use in this tutorial. Learn more about what the [ERC20 standard](https://theethereum.wiki/w/index.php/ERC20_Token_Standard) and [WETH](https://weth.io/) are.

### Setting up accounts

TestRPC comes with a few test accounts and tokens, which is quite convenient when prototyping. We can get the list of our account addresses as follows:

```javascript
const accounts = await zeroEx.getAvailableAddressesAsync();
console.log('accounts: ', accounts);
```

giving us:

```javascript
accounts: [
    '0x5409ed021d9299bf6814279a6a1411a7e866a631',
    '0x6ecbe1db9ef729cbe972c83fb886247691fb6beb',
    '0xe36ea790bc9d7ab70c55260c66d52b1eca985f84',
    '0xe834ec434daba538cd1b9fe1582052b880bd7e63',
    '0x78dc5d2d739606d31509c31d654056a45185ecb6',
    '0xa8dda8d7f5310e4a9e24f8eba77e091ac264f872',
    '0x06cef8e666768cc40cc78cf93d9611019ddcb628',
    '0x4404ac8bd8f9618d27ad2f1485aa1b2cfd82482d',
    '0x7457d5e02197480db681d3fdf256c7aca21bdc12',
    '0x91c987bf62d25945db517bdaa840a6c661374402',
];
```

Note that by default only the first address (**Maker**) possesses ZRX tokens. Let's now specify our **Maker** and **Taker** accounts, where the **Maker** is the one creating (or making) the order and the **Taker** is the one filling (or taking) the order.

```javascript
const [makerAddress, takerAddress] = accounts;
```

Another step we need to do is to allow the 0x protocol to interact with our funds. Indeed, 0x allows users to trade their tokens directly from their accounts without the need to send their tokens anywhere. This is possible thanks to the `approve()` and `transferFrom()` functions that are part of the ERC20 token standard. In order to give 0x protocol Proxy smart contract access to funds, we need to set _allowances_ (you can read about allowances [here](https://tokenallowance.io/)).

```javascript
// Unlimited allowances to 0x proxy contract for maker and taker
const setMakerAllowTxHash = await zeroEx.token.setUnlimitedProxyAllowanceAsync(ZRX_ADDRESS, makerAddress);
await zeroEx.awaitTransactionMinedAsync(setMakerAllowTxHash);

const setTakerAllowTxHash = await zeroEx.token.setUnlimitedProxyAllowanceAsync(WETH_ADDRESS, takerAddress);
await zeroEx.awaitTransactionMinedAsync(setTakerAllowTxHash);
```

Now the 0x protocol will be able to allow ZRX/WETH trades between our **Maker** and **Taker**. Another thing we need to do is "convert" ETH to WETH, since ETH is unfortunately not ERC20 compliant. Concretely, "converting" ETH to WETH means that we will deposit some ETH in a smart contract acting as a ERC20 wrapper. In exchange of depositing ETH, we will get some ERC20 compliant tokens called WETH at a 1:1 conversion rate. For example, depositing 10 ETH will give us back 10 WETH and we can revert the process at any time.

```javascript
const ethAmount = new BigNumber(1);
const ethToConvert = ZeroEx.toBaseUnitAmount(ethAmount, DECIMALS); // Number of ETH to convert to WETH

const convertEthTxHash = await zeroEx.etherToken.depositAsync(WETH_ADDRESS, ethToConvert, takerAddress);
await zeroEx.awaitTransactionMinedAsync(convertEthTxHash);
```

At this point, it might be worth mentioning why we need to await all those transactions. Calling an 0x.js function returns immediately after submitting a transaction with a transaction hash, so the user interface (UI) might show some useful information to the user before the transaction is mined (it sometimes takes long time). In our use-case we just want it to be confirmed, which happens immediately on testrpc. It is nevertheless a good habit to interact with the blockchain with these async/await calls.

### Creating an order

Users that create an order are called **Makers** and they need to specify some information in their order so the exchange.sol smart contract knows what to do with them.

```javascript
// Generate order
const order = {
    maker: makerAddress,
    taker: ZeroEx.NULL_ADDRESS,
    feeRecipient: ZeroEx.NULL_ADDRESS,
    makerTokenAddress: ZRX_ADDRESS,
    takerTokenAddress: WETH_ADDRESS,
    exchangeContractAddress: EXCHANGE_ADDRESS,
    salt: ZeroEx.generatePseudoRandomSalt(),
    makerFee: new BigNumber(0),
    takerFee: new BigNumber(0),
    makerTokenAmount: ZeroEx.toBaseUnitAmount(new BigNumber(0.2), DECIMALS), // Base 18 decimals
    takerTokenAmount: ZeroEx.toBaseUnitAmount(new BigNumber(0.3), DECIMALS), // Base 18 decimals
    expirationUnixTimestampSec: new BigNumber(Date.now() + 3600000), // Valid for up to an hour
};
```

where the fields are:

* **maker** : Ethereum address of our **Maker**.
* **taker** : Ethereum address of our **Taker**.
* **feeRecipient** : Ethereum address of our **Relayer** (none for now).
* **makerTokenAddress**: The token address the **Maker** is offering.
* **takerTokenAddress**: The token address the **Maker** is requesting from the **Taker**.
* **exchangeContractAddress** : The exchange.sol address.
* **salt**: Random number to make the order (and therefore its hash) unique.
* **makerFee**: How many ZRX the **Maker** will pay as a fee to the **Relayer**.
* **takerFee** : How many ZRX the **Taker** will pay as a fee to the **Relayer**.
* **makerTokenAmount**: The amount of token the **Maker** is offering.
* **takerTokenAmount**: The amount of token the **Maker** is requesting from the **Taker**.
* **expirationUnixTimestampSec**: When will the order expire (in unix time).

The `NULL_ADDRESS` is used for the `taker` field since in our case we do not care who the taker will be and using `NULL_ADDRESS` will allow anyone to fill our order.

### Signing the order

Now that we created an order as a **Maker**, we need to prove that we actually own the address specified as `makerAddress`. After all, we could always try pretending to be someone else just to annoy an exchange and other traders! To do so, we will sign the orders with the corresponding private key and append the signature to our order.

You can first obtain the order hash with the following command:

```javascript
const orderHash = ZeroEx.getOrderHashHex(order);
```

Now that we have the order hash, we can sign it and append the signature to the order;

```javascript
// Signing orderHash -> ecSignature
const shouldAddPersonalMessagePrefix = false;
const ecSignature = await zeroEx.signOrderHashAsync(orderHash, makerAddress, shouldAddPersonalMessagePrefix);

// Append signature to order
const signedOrder = {
    ...order,
    ecSignature,
};
```

With this, anyone can verify that the signature is authentic and this will prevent any change to the order by a third party. If the order is changed by even a single bit, then the hash of the order will be different and therefore invalid when compared to the signed hash.

Now let's actually verify whether the order we created is valid

```javascript
await zeroEx.exchange.validateOrderFillableOrThrowAsync(signedOrder);
```

If something was wrong with our order, this function would throw an informative error. If it passes, then the order is currently fillable. A relayer should constantly be [pruning their orderbook](#0x-OrderWatcher) of invalid orders using this method.

### Filling the order

Finally, now that we have a valid order, we can try to fill it while acting as a **Taker**. To do so, we first specify a few arguments

```javascript
const shouldThrowOnInsufficientBalanceOrAllowance = true;
const fillTakerTokenAmount = ZeroEx.toBaseUnitAmount(new BigNumber(0.1), DECIMALS);
```

When set to `false`, `shouldThrowOnInsufficientBalanceOrAllowance` will cause the smart contract to verify if the balances or allowances are sufficient and if that's not the case, it will log an error and return the remaining gas to the sender. If set to `true` the fill call will skip this verification step and will throw subsequently (consuming all the gas) if balances or allowances are not sufficient. The former cost slightly more gas for valid orders since it involves an extra verification step, but will save the remaining gas in cases where the fill fails. `fillTakerTokenAmount` is simply the amount of tokens (in our case WETH) the **Taker** wants to fill.

Now let's try to fill the order:

```javascript
const txHash = await zeroEx.exchange.fillOrderAsync(
    signedOrder,
    fillTakerTokenAmount,
    shouldThrowOnInsufficientBalanceOrAllowance,
    takerAddress,
);
```

This function broadcasts the transaction and returns its hash that we can use to get information about the transaction itself, such as when it is successfully mined.

```javascript
const txReceipt = await zeroEx.awaitTransactionMinedAsync(txHash);
console.log('FillOrder transaction receipt: ', txReceipt);
```

printing out the transaction receipt for the successful fillOrder transaction.

Congratulation! You have now successfully created, validated and filled an order using 0x.js and the 0x protocol!
