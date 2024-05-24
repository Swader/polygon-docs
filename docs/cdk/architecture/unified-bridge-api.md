## Introduction

The Unified Bridge API powers the [Polygon portal](https://portal.polygon.technology/) which consists in a set of endpoints that track the status of bridge transactions over all Polygon chains in real time and exposes the data to the user.

The API is built using the [chain indexer framework](../../tools/chain-indexer-framework/overview.md). 

The Unified Bridge API indexes data from Polygon's public chains PoS and zkEVM, Ethereum mainnet and testnets. The service also indexes CDK chains.

## Architecture

![bridge-api-service.png](../../img/cdk/bridge-api-service.png)

[Chain indexer framework](../../tools/chain-indexer-framework/overview.md) is the backend technology that runs the Unified Bridge API. The framework consists of three main components:

### Producer

To index data, the producer reads raw blockchain data from a specific chain and writes it to an [Apache Kafka](https://kafka.apache.org/documentation/) queue.  

The block data indexed by the service can be used for multiple use cases; not just for bridge related scenarios. 

Reorgs are handled at the producer level but only if the reorg height is less than 256 blocks. Anything above that requires the team to manually trigger a resync.

### Transformer

The transformer subscribes to specific topics on the Kafka queue and transforms the data into a different format. 

For the bridge service use case, the transformer filters bridge and claim events coming in from the producer, transforms the information into a more readable format, and pushes it to corresponding Kafka topics.

### Consumer

This component consumes the transformed data from the queue and writes it into a database which is accessible to users via API endpoints.

### Deploying the components

The infrastructure for the three components has to be deployed for each chain. This means that every new CDK chain must have a new producer, transformer, and consumer. Once these are deployed, the API endpoints can start exposing data for the chain. 

## Endpoints

The endpoints are common for all CDK chains. 

### Network ids

Transaction data belonging to a specific chain is identified using either the `destinationNetwork` and `sourceNetwork` parameters which are included in every endpoint response. 

- ETH = 0
- zkEVM = 1
- Astar = 2 (including testnet) 

### Transactions API

This API manages the details of a bridge transactions initiated from, or incoming to, a user’s `walletAddress`. Details include the real time status, the token bridged, the amount, the source and destination chain, etc. For example:

- Testnet: [https://api-gateway.polygon.technology/api/v3/transactions/testnet?userAddress=](https://api-gateway.polygon.technology/api/v3/transactions/testnet?userAddress=)`walletAddress`
- Mainnet: [https://api-gateway.polygon.technology/api/v3/transactions/mainnet?userAddress=](https://api-gateway.polygon.technology/api/v3/transactions/mainnet?userAddress=)`walletAddress`
    
!!! tip
    - Additional filtering can be performed on the transactions API by including the `sourceNetworkIds` and `destinationNetworkIds` query parameters.
    
### Merkle Proof API

This API manages the payload needed to process claims on the destination chain. For example:

- Testnet: [https://api-gateway.polygon.technology/api/v3/merkle-proof/testnet?networkId=](https://api-gateway.polygon.technology/api/v3/merkle-proof/testnet?networkId=sourceNetworkId&depositCount=depositCount)`[sourceNetworkId](https://bridge-api-mainnet-dev.polygon.technology/merkle-proof?networkId=1&depositCount=1)`[&depositCount=](https://api-gateway.polygon.technology/api/v3/merkle-proof/testnet?networkId=sourceNetworkId&depositCount=depositCount)`[depositCount](https://bridge-api-mainnet-dev.polygon.technology/merkle-proof?networkId=1&depositCount=1)`
- Mainnet: [https://api-gateway.polygon.technology/api/v3/merkle-proof/mainnet?networkId](https://api-gateway.polygon.technology/api/v3/merkle-proof/mainnet?networkId)[=`sourceNetworkId`&depositCount=`depositCount`](https://bridge-api-mainnet-dev.polygon.technology/merkle-proof?networkId=1&depositCount=1)
    
!!! tip
    - Use the Transactions API to get `sourceNetworkId` and `depositCount` data for a given bridge transaction. 
    
## Onboarding a new CDK

New CDK chains must supply the following mandatory parameters:

- Public RPC
- Chain ID
- Chain name
- Chain logo
- Dedicated RPC: This is _really important_ as the indexer requires it to index all data coming from the CDK chain.
- Block explorer URL
- Bridging contracts address

!!! important
    - The above are mandatory expectations from any CDK team. 
    - At the time of writing, everything else is handled and deployed by the Polygon team.

## Deeper dive into the bridging workflow

To understand why the API requires the mandatory onboarding parameters, let's examine the workflow of the Unified Bridge API and some of the components it interacts with.

### `lxly.js` client library 

The [LxLy SDK](https://www.npmjs.com/package/@maticnetwork/lxlyjs) is a JavaScript library that contains all prebuilt functions required for initializing and interacting with the bridge contracts; such as the bridge contract address, RPC, and network ID. 

It performs type conversion, formatting, error handling, and other processes to make it easy for a developer to invoke bridge, claim, and many other functions required for bridging. 

The SDK can be used for any compatible chain with no additional changes. 

### Unified Bridge API

The backend service as [explained previously](#unified-bridge-api). 

The repository to spin up this service is not yet open source and is currently fully managed by the Polygon team including all deployment, maintenance, and support. We expect to open source the code soon and require all participants to manage their own deployments.

### Gas station 

In order to estimate the gas price before submitting the transactions, we use the [Polygon gas station](../../tools/gas/polygon-gas-station.md), a lightweight service which gets gas price estimates for a specific chain.

The gas station is currently deployed and maintained by the Polygon team, but can be hosted by anyone as the [code is open source](https://github.com/maticnetwork/maticgasstation).

### Token list

This is a [json-formatted list](https://github.com/maticnetwork/polygon-token-list) containing metadata for all supported tokens. 

It automatically updates with new token details whenever one is bridged for the first time. However, logos or other metadata types need to be added or updated via a PR. 

### Balance API

This service relies on a [balance scanner contract](https://github.com/MyCryptoHQ/eth-scan) deployed on each chain. 

The contract fetches the balance from multiple ERC20 tokens in one single batch call and returns them. Token balance is processed with a price-feed service which attaches the USD value of the tokens before sending to the UI. 

### Merkle Proof Generation API

This is used to process claims on L1 and L2. 

The service can be accessed via the Unified Bridge API endpoint. It relies on the indexed bridge events and Merkle tree data to generate the Merkle proof required to process claims on L1/L2. 

### The auto-claim service

This service polls the Unified Bridge API endpoints to fetch unprocessed claims on a specific chain, and then submits the `claimAsset` transaction for all unprocessed claims. The service takes care of all retry mechanisms and error handling.

Claims are automatically processed on a destination chain so that users don’t have to perform extra steps, or add additional transaction gas fees, to receive their tokens on the destination chain. 

The auto-claim script is not yet open source at the time of writing. 

### How to run the auto-claim script

Clone the [auto-claim service repo](https://github.com/0xPolygon/auto-claim-service) and follow the README instructions making sure to include all required parameters in the `.env` file.