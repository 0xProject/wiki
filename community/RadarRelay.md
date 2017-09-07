RadarRelay is a trustless exchange built on the [Øx protocol](/0x).

RadarRelay will allow users to trade ERC-20 tokens (Ethereum network tokens like Augur, OmiseGo, Golem, &etc) without trusting eachother or RadarRelay.

+ [RadarRelay site](https://radarrelay.com/)
+ the team is active on [Twitter](https://twitter.com/RadarRelay), [Medium](https://medium.com/@radarrelay), [Slack](https://radarrelay.slack.com/), and [Reddit](https://www.reddit.com/r/Radar_Relay/)


### RadarRelay Beta

The RadarRelay beta launched on September 1st, 2017 on Ethereum's Kovan Testnet.
+ please read the [RadarRelay beta launch announcement](https://medium.com/@RadarRelay/signal-power-achieved-94470b18156f)
+ this beta is purely for testing purposes using Kovan Testnet tokens
+ the Testnet beta will allow users to experiment with RadarRelay and the devs to fix bugs
+ the RadarRelay beta will launch on Ethereum Mainnet for trading with real money
+ The RadarRelay Mainnet beta release does not yet have a launch date

_Make sure_ you are using the Kovan Testnet in your wallet.  In MetaMask, click on "Mainnet" in the top left-hand corner and use the dropdown to select "Kovan Testnet".


### Acquiring Kovan Testnet Tokens

To try out RadarRelay in the beta, you can get Ethereum Kovan Testnet tokens at the [Øx OTC site][ox-otc].  Each time you click the "Request" button, you'll receive 0.1 Kovan ETH or ZRX tokens.

If the ETH "request" button doesn't seem to be working, there are [other ways to get Kovan Testnet ETH](https://github.com/kovan-testnet/faucet).  If you're a Linux or OSX user and you already have a GitHub account, this method will only take a second:

1. you will need to sign up for a GitHub account to use this method.
2. go here and create a gist: https://gist.github.com/
3. leave everything blank except put your Ethereum wallet address as the text of the gist
4. click 'create public gist'
5. copy the URL of your resulting gist
6. now, open up a terminal and run this: `curl http://github-faucet.kovan.network/url --data "address=THE_URL_OF_YOUR_GIST"`
7. Your 1 ETH will be on its way! You can get another 1 ETH in 24 hours.

Windows users will have to [install `curl`](https://curl.haxx.se/download.html) to use this method.


### Fees for the Beta

The fee rates described below are for the BETA and may be changed for the official launch.  Slightly higher fees for a beta make sense, because it can help keep the exchange volume to reasonable levels for testing purposes.

+ All users pay a gas cost to activate or deactivate a token for trading

**Order Creator fees**
+ creating an order: **no fee**
+ order expires at the time sent during order creation: **no fee**
+ order is filled: Maker fee, 0.45% paid in ZRX to RadarRelay
+ cancel an order early: gas cost

**Order Filler fees**
+ order is filled: Taker fee, 0.70% paid in ZRX to RadarRelay
+ order is filled: gas cost

Your wallet address will execute a transaction on the Ethereum blockchain, and you will use up a bit of Ether to pay gas costs, in the following instances:
+ user activates or deactivates a token for trading (calls function `approve()` for the token's contract)
+ order creator cancels an order early (calls function `cancelOrder()` of the Øx exchange contract)
+ order filler fills the order (calls function `fillOrder()` for the Øx exchange contract)

If you're using Metamask, it will set the gas price you're willing to pay to a default of ~22 gwei. If you're willing to wait a little longer for your transactions to go through, try reducing the gas price to 4 gwei and save yourself some ether.

The Maker and Taker fees are paid to RadarRelay in ZRX tokens. For now, you need to buy some ZRX tokens and put them in your wallet so that you can pay this fee.  In the future, you will have the option of allowing the relay to acquire the ZRX tokens for you so that you don't have to buy and hold them yourself.


### Technical Details

The Øx protocol is primarily powered by two Ethereum smart contracts.  On the Kovan Testnet, these are:
+ Øx exchange: [`0x90Fe2Af704B34E0224bF2299C838E04d4Dcf1364`][add-exchange]
+ ZRX token:   [`0x6Ff6C0Ff1d68b964901F986d4C9FA3ac68346570`][add-zrx]


### Enabling Tokens for Trading

When you use RadarRelay, your address will also transact with the token smart contract for each token that you trade.  A token's Kovan Testnet contract address is **different** from its Mainnet contract address.

On the Kovan Testnet, the token addresses (from the [Etherscan Kovan token search][es-kovan-tokens]) are:

+ *Wrapped Eth WETH* [`0x05d090b51c40b020eab3bfcb6a2dff130df22e9c`][add-weth]
+ *MakerDAO MKR* [`0x1dad4783cf3fe3085c1426157ab175a6119a04ba`][add-mkr]
+ *Øx Project ZRX* [`0x6ff6c0ff1d68b964901f986d4c9fa3ac68346570`][add-zrx]
+ *Augur REP* [`0xb18845c260f680d5b9d84649638813e342e4f8c9`][add-rep]
+ *DigixDAO DGD* [`0xeee3870657e4716670f185df08652dd848fe8f7e`][add-dgd]
+ *Melon MLN* [`0x323b5d4c32345ced77393b3530b1eed0f346429d`][add-mln]
+ *Golem GNT* [`0xef7fff64389b814a946f3e92105513705ca6b990`][add-gnt]

In Metamask, you'll need to add these tokens using the "Add Token" button to get them to show up in the Metamask interface.

Activating a token for trading requires a transaction on the Ethereum blockchain.

What's actually going on in this transaction?  RadarRelay published [a page to describe the "token allowances" process in detail](http://tokenallowance.io).

From the [MakerDAO OasisDEX documentation][makerdao-docs]:
> For the (MakerOTC) Dapp to be able to access your token funds for buying and/or selling, you need to first *grant it access to withdraw from your personal account by setting the allowance*. This will specify the maximum that can be withdrawn, and will be reduced with each withdrawal. You can always update the allowance or disable it completely by setting it to 0. This might seem confusing initially, but *this token access pattern is common practice for all EIP-20 tokens*.

This mirrors the tooltip on the [Øx OTC site][ox-otc]:
> Øx smart contracts require access to your token balances in order to execute trades. Toggling permissions *sets an allowance* for the smart contract so you can start trading that token.

[es-kovan-tokens]: https://kovan.etherscan.io/token-search
[makerdao-docs]: https://github.com/OasisDEX/oasis/wiki#allowance
[ox-otc]: https://0xproject.com/portal/balances
[add-mainnet-weth]: https://etherscan.io/address/0x2956356cd2a2bf3202f771f50d3d14a367b48070
[add-weth]: https://kovan.etherscan.io/address/0x05d090b51c40b020eab3bfcb6a2dff130df22e9c
[add-mkr]: https://kovan.etherscan.io/address/0x1dad4783cf3fe3085c1426157ab175a6119a04ba
[add-rep]: https://kovan.etherscan.io/address/0xb18845c260f680d5b9d84649638813e342e4f8c9
[add-dgd]: https://kovan.etherscan.io/address/0xeee3870657e4716670f185df08652dd848fe8f7e
[add-mln]: https://kovan.etherscan.io/address/0x323b5d4c32345ced77393b3530b1eed0f346429d
[add-gnt]: https://kovan.etherscan.io/address/0xef7fff64389b814a946f3e92105513705ca6b990
[add-exchange]: https://kovan.etherscan.io/address/0x90Fe2Af704B34E0224bF2299C838E04d4Dcf1364#code
[add-zrx]: https://kovan.etherscan.io/address/0x6Ff6C0Ff1d68b964901F986d4C9FA3ac68346570#code
