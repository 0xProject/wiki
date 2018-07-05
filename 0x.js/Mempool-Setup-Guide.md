Since 0x trade settlement happens on-chain, trades are first submitted to the Ethereum network mempool before being mined into blocks. If a relayer only updates their orderbook from the information in mined blocks, they will not be using information from transactions submitted to the network awaiting to be mined. With 12 second block-times, there can be a considerable delay between when a transaction is submitted to the mempool and mined. This results in an unpleasant, laggy interface for traders since many will attempt to fill the same order, not knowing others have already submitted valid transactions to fill it. In order to avoid this, applications built on 0x can leverage the mempool and react to submitted valid transactions that have a high likelihood of being included in a subsequent block. This guide will teach you more about mempools and how you can set up Ethereum nodes with mempools closely resembling that of the network.

This article is for advanced users. We assume, that you're already familiar with the following concepts:

* Mempool
* Block reorgs
* Events/Logs
* Docker deployment

We also assume that you are already comfortable using the 0x protocol on the mined block-level and now want to access the mempool in order to give your users a more responsive user experience (or to be a faster trader).

### How it works

There are basically two approaches to extracting data from an Ethereum transaction:

* Pre-execution (static)
    * The benefit to this approach is that the only thing you need is the transaction itself. You can decode it's parameters and know which contract function it calls, together with which arguments.
    * The downside is that we know nothing about contract sub-calls to other contracts and cannot paint a full picture of how the blockchain state will change.
* Post-execution (dynamic)
    * The benefit is that we know exactly what the transaction did and we can extract the event logs from it. These event logs can inform us on what happened, even in contract sub-calls. We assume here that all the state changes we care about have associated log events (balance/allowance changes, order fills/cancels)
    * The downside is that we need to run it through a VM with access to the whole blockchain state, so it's impossible to do it client-side. We need a fully synced Ethereum node.

Given that we want a complete picture of how the blockchain state will change with new transactions, we'll go with the dynamic approach.

### Setting up an accurate mempool

Let's start with describing how to set up an Ethereum node such that it has a mempool containing most transactions that will find their way into the next block. Full nodes with the following characteristics perform better:

* Have been online for a long time
* Have static IP addresses
* Are geographically distributed (closer to where a transaction was submitted)

The first two are easy enough to accomplish. In order to accomplish the last item, we are still waiting on Parity to implement a [feature](https://github.com/paritytech/parity/issues/6869) that would enable nodes to prioritize transaction propagation to reserved peers. This article will be updated once it becomes available.

### Utilizing The mempool

Let's say we now have a fairly accurate mempool and want to extract the events from it. The result of a transaction is deterministic given the existing blockchain state. When a node is mining - it selects a subset of it's mempool transactions and creates an unconfirmed block. Once this is done, it can execute all the transactions in this unconfirmed block under the assumption that this block will be the next mined block and get the corresponding events/logs generated from transaction execution. The reason we can do this is because transaction ordering is well-defined. Miners get to decide on the transaction order but most will use one of the these [approaches](https://ethereum.stackexchange.com/a/6111/6075). They can also choose to be malicious or implement their own custom strategies however. The problem with mining in order to go through this process is that you'd only be getting the logs for the next blocks-worth of transactions and if the mempool is big - you have no way of exploring any transactions that will be included in subsequent blocks. Also, you probably don't actually want to be mining.

The solution is simple: Add the `--force-sealing` Parity flag which commands the node to create pending blocks without actually mining. In addition, we also want to rebuild the block every time we receive a new transaction because it might have a higher gas price then existing transactions and change the transaction execution order for the block. In order to do that, we need to add the `--reseal-on-txs all` flag.

In order to react to all transactions in the mempool (and not just the next block), we need to force our node to create pending blocks that include all the mempool transactions. This can be done in Parity by adding the `--infinite-pending-block` flag.

Now that we have a good understanding of what we want to achieve and what the limitations are, we can begin with the implementation details on how to set up your own infrastructure.

### Node deployment

**Note:** Since Geth does not (at the time of writing) implement some of the feature we need for our setup, we will only be using Parity nodes.

To learn more about how to deploy a Parity node on AWS using Docker, check out [this article](#How-To-Deploy-A-Parity-Node).

Configure your node with the following arguments:

```
docker run -d \
-p 8545:8545 \ // rpc port
-p 30303:30303 \ // p2p port
--log-opt max-size=100m \
--log-opt max-file=20 \
-v /home/ubuntu/parity-data:/mnt \ // mount chain data
parity/parity:v1.9.4 \ // version of parity to run
--jsonrpc-interface all \ // bind on all interfaces
--rpccorsdomain '\*' \ // add CORS headers
--jsonrpc-hosts all \ // allow all hosts
-d /mnt \ // store chain data here
--auto-update none \ // we don't want auto-updates
--no-download \ // and we don't want to download updates either
--tx-queue-gas off \ // our mempool accepts any transaction
--tx-queue-size 1000000 \ // Set the size of the mempool to 1M
--force-sealing \ // generate fake blocks
--reseal-min-period=1 \ // not faster than once per 1ms
--reseal-on-txs all // regenerate block on any new transaction
--infinite-pending-block // increase the pending block to include all pending txs
```

To read more about how to use the mempool to watch an order book check out [this article](#0x-OrderWatcher).
