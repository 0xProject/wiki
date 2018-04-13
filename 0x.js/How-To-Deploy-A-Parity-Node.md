This article will walk you through deploying a [Parity](https://github.com/paritytech/parity) Ethereum node on [Amazon AWS](https://aws.amazon.com) using [Docker Machine](https://docs.docker.com/machine/overview/).

This tutorial assumes that you:

* Have [Docker](https://docs.docker.com/install/) installed
* Have setup an AWS account and have the associated access key and secret.

Before we can go ahead and spin up an EC2 instance, we need to create a security group call "parity-security-group" for our EC2 instance. To do so, log into AWS and open the EC2 service. On the right hand side you should see the "Security Groups" section. Click "Create Security Group" and add the inbound and outbound rules depicted below.

<div align="center">
    <img src="https://s3.eu-west-2.amazonaws.com/0x-wiki-images/inbound.png" style="padding-bottom: 20px; padding-top: 20px" width="80%" />
</div>
<div align="center">
    <img src="https://s3.eu-west-2.amazonaws.com/0x-wiki-images/outbound.png" style="padding-bottom: 20px; padding-top: 20px" width="80%" />
</div>

Next, let's spin up a new EC2 instance with an attached volume. When deploying on Kovan a 16GB SSD should be sufficient, however for mainnet you should attach a 128GB SSD.

Note: replace `${ACCESS_KEY}` and `${ACCESS_SECRET_KEY}` with your AWS credentials.

```
docker-machine create \
--driver amazonec2 \
--amazonec2-security-group parity-security-group \
--amazonec2-instance-type t2.medium \
--amazonec2-region us-east-1 \
--amazonec2-access-key ${ACCESS_KEY} \
--amazonec2-secret-key ${ACCESS_SECRET_KEY} \
--amazonec2-root-size 128 \
parity-node
```

This command creates a T2 Medium EC2 instance with a 128GB volume in the `us-east-1` region.

Once the instance has successfully booted up, let's commandeer the machine:

```
docker-machine env parity-node
```

```
eval $(parity-node)
```

We can now start a docker container running Parity with a single command:

```
docker run -d \
-p 8545:8545 \
--log-opt max-size=100m \
--log-opt max-file=20 \
--name parity-kovan \
-v /home/ubuntu/parity-data:/mnt \
 parity/parity:stable \
--testnet \
--jsonrpc-interface 0.0.0.0 \
--rpccorsdomain '*' \
--jsonrpc-hosts all \
-d /mnt \
--auto-update none \
--no-download \
--tx-queue-gas off \
--tx-queue-size 1000000
```

Since this is a pretty beefy command, let's break it down. The `--log-opt` commands enables the default Docker logger so that it stores up to 20 files of 100 megabytes each. These logs can be inspected by running:

```
docker logs parity-kovan
```

With the `-v` command we mount the Parity node data folder to the external volume. This is important because it means that you won't need to re-sync your node after a restart.

By removing the `--testnet` flag, Parity will run on mainnet. The `--rpccorsdomain` flag ensures that browser-based clients are able to ping your node without CORS errors.

Parity defaults to downloading and installing upgrades as they are published. To avoid this behavior, we need to set the `--no-download` flag and `--auto-update` to `none`.

You should now be able to access your Parity node via it's public DNS endpoint located under EC2's "Instances" section!
