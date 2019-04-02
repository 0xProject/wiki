This article will walk you through deploying a [Parity](https://github.com/paritytech/parity-ethereum) Ethereum node on [Amazon AWS](https://aws.amazon.com) using [Docker Machine](https://docs.docker.com/machine/overview/).

This tutorial assumes that you:

* Have [Docker](https://docs.docker.com/install/) and [Docker Machine](https://docs.docker.com/machine/install-machine/) installed
* Have setup an AWS account, and have the associated access key and secret, and have the AWS Command Line Interface [installed](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html#cli-quick-configuration).

Before we can go ahead and spin up an EC2 instance, we need to have two things: a Virtual Private Cloud (VPC) and a Security Group.

If you already have a VPC that you want to use, then you'll need to get its VPC ID for use with the commands below, which assume it's stored in environment variable `VPC_ID`.  If you don't yet have a VPC, or if you want to create a new one for this tutorial, use these commands:

```
export VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --query Vpc.VpcId \
    --output text)
aws ec2 create-tags \
    --resources $VPC_ID \
    --tags Key=Name,Value="0x Parity Node Tutorial"
aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.0.0/16 \
    --availability-zone us-east-1a
export GATEWAY_ID=$(aws ec2 create-internet-gateway \
    --query InternetGateway.InternetGatewayId \
    --output text)
aws ec2 attach-internet-gateway \
    --internet-gateway-id $GATEWAY_ID \
    --vpc-id $VPC_ID
aws ec2 create-route \
    --route-table-id \
        $(aws ec2 describe-route-tables \
            --query "RouteTables[?VpcId=='$VPC_ID'].RouteTableId" \
            --output text) \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $GATEWAY_ID
```

We need to create a Security Group called "parity-security-group" for our EC2 instance. You can do this manually via the AWS console -- in the EC2 service, under "Security Groups" section, click "Create Security Group" and add the inbound and outbound rules depicted below -- or you can do it by using the following two commands.

```
export GROUP_ID=$(aws ec2 create-security-group \
    --group-name parity-security-group \
    --description "0x tutorial" \
    --vpc-id $VPC_ID \
    --query GroupId \
    --output text)
for port in 80 22 2376 443 8545 2049; do
    aws ec2 authorize-security-group-ingress \
        --group-id $GROUP_ID \
        --protocol tcp \
        --port $port \
        --cidr 0.0.0.0/0
done
```

<div align="center">
    <img src="https://s3.eu-west-2.amazonaws.com/0x-wiki-images/inbound.png" style="padding-bottom: 20px; padding-top: 20px" width="80%" />
</div>
<div align="center">
    <img src="https://s3.eu-west-2.amazonaws.com/0x-wiki-images/outbound.png" style="padding-bottom: 20px; padding-top: 20px" width="80%" />
</div>

Next, let's spin up a new EC2 instance with an attached volume. When deploying on Kovan a 16GB SSD should be sufficient, however for mainnet you should attach a 128GB SSD.

Note: Replace `$ACCESS_KEY` and `$ACCESS_SECRET_KEY` with your AWS credentials.

```
docker-machine create \
--driver amazonec2 \
--amazonec2-vpc-id $VPC_ID \
--amazonec2-security-group parity-security-group \
--amazonec2-instance-type t2.medium \
--amazonec2-region us-east-1 \
--amazonec2-access-key $ACCESS_KEY \
--amazonec2-secret-key $ACCESS_SECRET_KEY \
--amazonec2-root-size 128 \
parity-node
```

This command creates a T2 Medium EC2 instance with a 128GB volume in the `us-east-1` region.

Once the instance has successfully booted up, let's commandeer the machine:

```
docker-machine env parity-node
```

```
eval $(docker-machine env parity-node)
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
--chain kovan \
--jsonrpc-interface 0.0.0.0 \
--jsonrpc-cors '*' \
--jsonrpc-hosts all \
--auto-update none \
--no-download \
--tx-queue-size 1000000
```

Since this is a pretty beefy command, let's break it down. The `--log-opt` commands enables the default Docker logger so that it stores up to 20 files of 100 megabytes each. These logs can be inspected by running:

```
docker logs parity-kovan
```

With the `-v` command we mount the Parity node data folder to the external volume. This is important because it means that you won't need to re-sync your node after a restart.

By removing the `--testnet` flag, Parity will run on mainnet. The `--rpccorsdomain` flag ensures that browser-based clients are able to ping your node without CORS errors.

Parity defaults to downloading and installing upgrades as they are published. To avoid this behavior, we need to set the `--no-download` flag and `--auto-update` to `none`.

You should now be able to access your Parity node via it's Public IP located under EC2's "Instances" section!  If you need to ssh to the machine, you can use the same command used by `docker-machine`: `ssh -i ~/.docker/machine/machines/parity-node/id_rsa ubuntu@$IP_ADDRESS`.
