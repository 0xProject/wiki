0x Instant is a new product from the 0x core team that offers a convenient way for people to get access to a wide variety of tokens and other crypto-assets in just a few taps. Developers can integrate the free, open source library into their applications or websites in order to both offer seamless access to crypto-assets, as well as gain a new source of revenue, with just a few lines of code.

<div align="center">
    <img src="https://s3.eu-west-2.amazonaws.com/0x-wiki-images/instant_screenshot.png" style="padding-bottom: 20px; padding-top: 20px; max-width: 342px;" width="80%" />
</div>

Check out a live example on [mainnet](http://0x-instant-staging.s3-website-us-east-1.amazonaws.com/) and [kovan](http://0x-instant-staging.s3-website-us-east-1.amazonaws.com/?networkId=42&assetData=0xf47261b00000000000000000000000002002d3812f58e35f0ea1ffbf80a75a38c32175fa&liquiditySource=provided).

### Libraries

0x Instant has two main libraries: the `0x Instant UI component` that users will see and the `Asset Buyer` library, a JavaScript / TypeScript library that abstracts out many of the complexities of sourcing orders and performing market buys. Most developers who want to add simple token access will just use the out-of-the-box package that includes the UI and Asset Buyer, but teams may also write their own custom UI and just plug into the Asset Buyer as they see fit. Check out the `@0x/asset-buyer` documentation [here](https://0xproject.com/docs/asset-buyer).

### Orders

Additionally, 0x Instant requires a source of SignedOrders that users can fill. Most teams will opt to provide a [Standard Relayer API HTTP endpoint](https://0xproject.com/wiki#Questions-About-Instant), but teams may optionally source liquidity themselves and pass in specific [SignedOrders](https://0xproject.com/wiki#Questions-About-Instant) for users to fill.

### Affiliate Fees

As an end host of 0x Instant, you can charge users a fee on all trades made through Instant with the `affiliateFee` option. Simply specify an ethereum address and feePercentage (up to 5%), and a percentage of each transaction will be deposited into the specified address (denominated in ETH).

## 0x Instant UI Integration

### Adding the Instant UI

The 0x Instant UI and Asset Buyer are bundled together in a convenient JS package for you. You can either download and serve the package yourself, or use the CDN-hosted version from 0x.

```html
<head>
    ...
    <script src="https://instant.0xproject.com/instant.js"></script>
    ...
</head>
```

```javascript
zeroExInstant.render(
    {
        // options (see below)
    },
    'body',
);
```

### Options Configuration

0x Instant is highly customizable to fit individual developer needs. Below are the different options that can be passed into the `render()` function above

#### Required

| Option      | Description                                                                                                                                                                                     |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| orderSource | Accepts either a [Standard Relayer API HTTP endpoint](https://0xproject.com/wiki#Questions-About-Instant) or an array of signed 0x [orders](https://0xproject.com/wiki#Questions-About-Instant) |

#### Optional

| Option                         | Description                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| provider                       | An instance of an Ethereum [provider](https://0xproject.com/wiki#Questions-About-Instant). If none is provided, 0x instant will try to grab the injected provider if one exists, otherwise it will suggest the user to install MetaMask                                                                                                                     |
| walletDisplayName              | A display string for the wallet you are connected to. Defaults to our best guess (i.e. MetaMask, Coinbase Wallet) but should be provided if a custom provider is supplied as an optional config.                                                                                                                                                            |
| availableAssetDatas            | An array of [assetDatas](https://0xproject.com/wiki#Questions-About-Instant) that can be purchased through Instant. Defaults to all token pairs from orderSource. Will throw an error if empty.                                                                                                                                                             |
| defaultSelectedAssetData       | The asset that should be opened by default. If this is not provided, Instant will show "Select Token" if there are multiple availableAssetDatas.                                                                                                                                                                                                            |
| defaultAssetBuyAmount          | Pre-fill the amount of tokens to purchase. Defaults to 0.                                                                                                                                                                                                                                                                                                   |
| additionalAssetMetaDataMap     | An object with keys that are assetData strings and values that are objects that adhere to the [AssetMetaData schema](https://0xproject.com/wiki#Questions-About-Instant). The values represent the meta data for that asset. There is an internal mapping for popular tokens that cannot be overriden and only appended to using this configuration option. |
| networkId                      | Id of Ethereum network to connect to. Defaults to 1 (mainnet).                                                                                                                                                                                                                                                                                              |
| affiliateInfo                  | An object specifying what % ETH fee should be added to orders and where the fee should be sent. Max feePercentage is .05 (See examples below)                                                                                                                                                                                                               |
| shouldDisableAnalyticsTracking | An option to turn on / off analytics used to make Instant a better experience. Defaults to false.                                                                                                                                                                                                                                                           |

### Examples

#### Serving Own Liquidity

```javascript
zeroExInstant.render(
    {
        // these can come from your own api, or anywhere
        orderSource: [signedOrder1, signedOrder2],
    },
    'body',
);
```

#### Using All Standard Relayer API Available Assets

Using [/asset_pairs](https://github.com/0xProject/standard-relayer-api/blob/master/http/v2.md#get-v2asset_pairs) to find all \*/WETH pairs

```javascript
zeroExInstant.render(
    {
        orderSource: 'https://api.relayer.com/sra/v2/',
    },
    'body',
);
```

#### Providing your own provider

This will give you more control over what provider is passed in and where RPC calls are directed

```javascript
zeroExInstant.render(
    {
        orderSource: 'https://api.relayer.com/sra/v2/',
        provider: window.ethereum,
        walletDisplayName: 'Trust Wallet',
    },
    'body',
);
```

#### Providing a Default Token (i.e. Open straight into REP or OMG)

```javascript
zeroExInstant.render(
    {
        orderSource: 'https://api.relayer.com/sra/v2/',
        availableAssetDatas: ['0xf47261b04c32345ced77393b3530b1eed0f346429d'],
        defaultSelectedAssetData: '0xf47261b04c32345ced77393b3530b1eed0f346429d',
    },
    'body',
);
```

#### Earning affiliate fees

3% of transaction volume will de deposited into 0x50ff5828a216170cf224389f1c5b0301a5d0a230

```javascript
zeroExInstant.render(
    {
        orderSource: 'https://api.relayer.com/sra/v2/',
        affiliateInfo: {
            feeRecipient: '0x50ff5828a216170cf224389f1c5b0301a5d0a230',
            feePercentage: 0.03,
        },
    },
    'body',
);
```

#### Providing a Custom Token

Your token may not be currently supported by Instant by default. Check [here](https://github.com/0xProject/0x-monorepo/blob/development/packages/instant/src/data/asset_meta_data_map.ts) for a list of supported tokens. Check "What is assetMetaData?" in [the questions section](https://0xproject.com/wiki#Questions-About-Instant) for more information about the object being passed in.

```javascript
zeroExInstant.render(
    {
        // these can contain makerAssetDatas that are not supported by default
        orderSource: [signedOrder1, signedOrder2],
        additionalAssetMetaDataMap: {
            '0xf47261b0000000000000000000000000744d70fdbe2bc4cf95131626614a1764df805b9e': {
                assetProxyId: '0xf47261b0', // ERC20 Proxy Id
                decimals: 18,
                symbol: 'XXX',
                name: 'My Custom Token',
                primaryColor: '#F2F7FF', // Optional
                iconUrl: 'https://cdn.icons.com/my_icon.svg', // Optional
            },
        },
    },
    'body',
);
```
The icon referenced by `iconUrl` will go on top of a 26x26 circle that has `primaryColor` as a background. If an `iconUrl` is not provided, the specified token `symbol` ("XXX" in this case) will be displayed over the circle in white.

## Asset Buyer

Behind the scenes, the `AssetBuyer` class is aggregating liquidity and calculating the orders that need to be filled for a given user request. For teams that require a more custom integration or advanced functionality, they can use the AssetBuyer class directly and skip the Instant UI. Below are the methods you will most likely be interacting with.

```javascript
/**
 * Get a `BuyQuote` containing all information relevant to fulfilling a buy given a desired assetData.
 * You can then pass the `BuyQuote` to `executeBuyQuoteAsync` to execute the buy.
 * @param   assetData           The assetData of the desired asset to buy (for more info: https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md).
 * @param   assetBuyAmount      The amount of asset to buy (in wei).
 * @param   options             Options for the request. See type definition for more information.
 *
 * @return  An object that conforms to BuyQuote that satisfies the request. See type definition for more information.
 */
async getBuyQuoteAsync(assetData: string,
                       assetBuyAmount: BigNumber,
                       options: Partial<BuyQuoteRequestOpts>
                       ): Promise<BuyQuote>;

/**
 * Same as `getBuyQuoteAsync`, except it takes an ERC20 tokenAddress
 */
async getBuyQuoteForERC20TokenAddressAsync(tokenAddress: string,
                                           assetBuyAmount: BigNumber,
                                           options: Partial<BuyQuoteRequestOpts>
                                           ): Promise<BuyQuote>;
/**
 * Given a BuyQuote and desired rate, attempt to execute the buy.
 * @param   buyQuote        An object that conforms to BuyQuote. See type definition for more information.
 * @param   options         Options for the execution of the BuyQuote. See type definition for more information.
 *
 * @return  A promise of the txHash.
 */
async executeBuyQuoteAsync(buyQuote: BuyQuote,
                           options: Partial<BuyQuoteExecutionOpts>
                           ): Promise<string>

```

Where the option types are as follows:

```javascript
/**
 * feePercentage: The affiliate fee percentage. Defaults to 0.
 * shouldForceOrderRefresh: If set to true, new orders and state will be fetched instead of waiting for the next orderRefreshIntervalMs. Defaults to false.
 * slippagePercentage: The percentage buffer to add to account for slippage. Affects max ETH price estimates. Defaults to 0.2 (20%).
 */
export interface BuyQuoteRequestOpts {
    feePercentage: number;
    shouldForceOrderRefresh: boolean;
    slippagePercentage: number;
}

/**
 * ethAmount: The desired amount of eth (in wei) to spend. Defaults to buyQuote.worstCaseQuoteInfo.totalEthAmount.
 * takerAddress: The address to perform the buy. Defaults to the first available address from the provider.
 * gasLimit: The amount of gas to send with a transaction (in Gwei). Defaults to an eth_estimateGas rpc call.
 * gasPrice: Gas price in Wei to use for a transaction
 * feeRecipient: The address where affiliate fees are sent. Defaults to null address (0x000...000).
 */
export interface BuyQuoteExecutionOpts {
    ethAmount?: BigNumber;
    takerAddress?: string;
    gasLimit?: number;
    gasPrice?: BigNumber;
    feeRecipient: string;
    feeRecipient: string;
}
```

`getBuyQuoteAsync()` and `getBuyQuoteForERC20TokenAddressAsync()` are used to retrieve a `BuyQuote` object that contains information about buying a specific amount of asset that can be displayed to the user.

```javascript
/**
 * assetData: String that represents a specific asset (for more info: https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md).
 * assetBuyAmount: The amount of asset to buy (in wei).
 * orders: An array of objects conforming to SignedOrder. These orders can be used to cover the requested assetBuyAmount plus slippage.
 * feeOrders: An array of objects conforming to SignedOrder. These orders can be used to cover the fees for the orders param above.
 * feePercentage: Optional affiliate fee percentage used to calculate the eth amounts above.
 * bestCaseQuoteInfo: Info about the best case price for the asset.
 * worstCaseQuoteInfo: Info about the worst case price for the asset.
 */
export interface BuyQuote {
    assetData: string;
    assetBuyAmount: BigNumber;
    orders: SignedOrder[];
    feeOrders: SignedOrder[];
    feePercentage?: number;
    bestCaseQuoteInfo: BuyQuoteInfo;
    worstCaseQuoteInfo: BuyQuoteInfo;
}

/**
 * assetEthAmount: The amount of eth (in wei) required to pay for the requested asset.
 * feeEthAmount: The amount of eth (in wei) required to pay the affiliate fee.
 * totalEthAmount: the total amount of eth (in wei) required to complete the buy. (Filling orders, feeOrders, and paying affiliate fee)
 */
export interface BuyQuoteInfo {
    assetEthAmount: BigNumber;
    feeEthAmount: BigNumber;
    totalEthAmount: BigNumber;
}
```

`executeBuyQuoteAsync()` is used to actually issue the buy as an Ethereum transaction. It takes a `BuyQuote` object that is obtained from `getBuyQuoteAsync()` as well as the desired rate to try and execute at (this affects how much eth is required to be sent with the transaction as well as how likely the transaction will be successful, the higher the rate, the higher the probability the transaction will succeed)

There are several static `AssetBuyer` factory methods that can be used to construct instances of `AssetBuyer` depending on how you'd like to source liquidity (in memory SignedOrders, or SRA). Unless your situation is specific, it is unlikely that you need to use the constructor directly. Typically, you'll want to source liquidity from an SRA endpoint, in which case you'll want to use the method: `AssetBuyer.getAssetBuyerForStandardRelayerAPIUrl()`. If you already have the orders you want to perform the buy with use `AssetBuyer.getAssetBuyerForProvidedOrders()`.

#### Full AssetBuyer Example:

```javascript
const provider = window.ethereum;
const sraUrl = 'https://api.relayer.com/sra/v2/';
const zrxAssetData = '0xf47261b0000000000000000000000000e41d2489571d322189246dafa5ebde1f4699f498';
const assetBuyer = AssetBuyer.getAssetBuyerForStandardRelayerAPIUrl(provider, sraUrl);
const amountToBuy = new BigNumber(10000000);
const quote = await assetBuyer.getBuyQuoteAsync(zrxAssetData, amountToBuy);
const txHash = await assetBuyer.executeBuyQuoteAsync(quote);
```

As mentioned, you can also use the quote providing and fee abstraction functionality in `AssetBuyer` using your own liquidity. The interface is the same, except the factory method which no longer accepts a standard relayer API url but in memory orders.
