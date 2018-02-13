One interesting feature enabled by the 0x protocol is the ability to remove the necessity for a user of a certain dApp or protocol to own it's native token in order to use it. The native token is still required, but it can be acquired seamlessly within the same Ethereum transaction as the action the user wishes to perform.

### Abstracting ZRX fees from 0x trades

Let's start with describing how a relayer can allow order **takers** to pay trading fees in WETH instead of ZRX. The way this is done is by enabling users to batch fill two orders in a single Ethereum transaction. The first will convert a sufficient amount of WETH to ZRX to cover the `takerFee` for the order they want to fill. The second order is the one the user is actually interested in filling.

Once the user has selected an order they want to fill, the relayer will supply the user with a second order exchanging WETH for ZRX at the current market price, without fees. The maker then submits both orders to the blockchain. Since the [batchFillOrders](https://0xproject.com/docs/contracts#batchFillOrders) 0x smart contract method fills orders synchronously, by the time the second order is filled, the user has acquired enough ZRX to pay the transaction fee of that order.

##### Limitations

- Token abstraction cannot be used to pay the `makerFee` if one exists.
- Executing a batch fill costs more gas then a regular fill. As such, frequent traders will still prefer to hold a ZRX balance directly.

### Abstracting native tokens from a dApp/protocol

Token abstraction applied to other dApps and protocols have the same end-effect but work slightly differently.

Anyone can write and deploy a smart contract that implements token abstraction for an existing smart contract with a native token. The wrapper contract would accept a valid 0x order that converts another token to the native token along with any arguments required by the primary smart contract.

Let's say someone writes a token abstraction wrapper for the Golem network so that people can pay computation providers in ETH instead of GNT. They would do the following steps:

1. Fetch market orders exchanging WETH to GNT from several relayers using the [Standard Relayer API](https://blog.0xproject.com/introducing-the-0x-standard-relayer-api-8a37bd90a3e).
2. Call the Golem wrapper contract, passing in an 0x order exchanging sufficient WETH for GNT to perform a computation on the Golem network.

The smart contract would then do the following:

1. Wrap the ETH into WETH ERC20-compliant tokens.
2. Call `fillOrKillOrder` on the 0x smart contract passing in the supplied order.
3. Send the GNT to the user.
4. Call the Golem smart contract on behalf of the user, spending the newly acquired GNT.

In this way, the user owns GNT just in time for the transaction, and in the exact amount required for payment.

##### Example code

```
import "./Exchange.sol";
import "./tokens/EtherToken.sol";

contract GolemWrapper {
    struct Order {
        address maker;
        address taker;
        address makerToken;
        address takerToken;
        address feeRecipient;
        uint makerTokenAmount;
        uint takerTokenAmount;
        uint makerFee;
        uint takerFee;
        uint expirationTimestampInSec;
        uint salt;
        uint8 v;
        bytes32 r;
        bytes32 s;
        bytes32 orderHash;
    }

    function useGolemWithEth(
        address[5] orderAddresses,
        uint[6] orderValues,
        uint8 v,
        bytes32 r,
        bytes32 s,
    ) {
            // Marshal args into order instance
            order = Order({
                maker: orderAddresses[0],
                taker: orderAddresses[1],
                makerToken: orderAddresses[2],
                takerToken: orderAddresses[3],
                feeRecipient: orderAddresses[4],
                makerTokenAmount: orderValues[0],
                takerTokenAmount: orderValues[1],
                makerFee: orderValues[2],
                takerFee: orderValues[3],
                expirationTimestampInSec: orderValues[4],
                salt: orderValues[5],
                v: v,
                r: r,
                s: s,
                orderHash: exchange.getOrderHash(orderAddresses, orderValues)
            });

            // Convert the sent ETH into WETH
            uint fillAmount = msg.value;
            ethToken.deposit.value(fillAmount)();

            // Fill the passed-in order or throw if unsuccessful
            exchange.fillOrKillOrder(
                [order.maker, order.taker, order.makerToken, order.takerToken, order.feeRecipient],
                [order.makerTokenAmount, order.takerTokenAmount, order.makerFee, order.takerFee, order.expirationTimestampInSec, order.salt],
                fillAmount,
                order.v,
                order.r,
                order.s
            );

            // Send GNT to user
            Token(order.takerToken).transferFrom(Address(this), msg.sender, order.makerTokenAmount);

            // Call the Golem smart contract using a DELEGATECALL here
    }
}

```

##### Limitations

- This again is more expensive in terms of gas then calling the Golem network directly. As such, it is great for on-boarding new users but frequent users would still prefer to hold a balance of GNT.
