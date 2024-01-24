# Prerequisites

## Dependency Checking

| Dependency | Version | Version Check Command |
| --- | --- | --- |
| git | ^2 | git —version |
| node | ^20 | git —version |
| npm | ^10 | npm —version |
| foundry | ^0.2 | forge —version |

## Sepolia Access

For Polygon CDK to function properly, it needs connection to a layer 1 EVM blockchain. For the purpose of this guide, we will be deploying the CDK contracts to [Sepolia testnet](https://www.alchemy.com/overviews/sepolia-testnet).

You have a couple options on connecting to the Sepolia testnet

1. Node Provider (easiest)
2. Run your own node (recommended for a production setup)

For the sake of simplicity, we will use a node provider ([Infura](https://www.infura.io)) for this guide.

You can use a different provider by modifying your hardhat config (see here)

The deployment is expected to use up to **2 Sepolia ETH**. You can get Sepolia ETH from public faucets such as:

[Infura Faucet](https://www.infura.io/faucet/sepolia)

[Chainstack Faucet](https://chainstack.com/sepolia-faucet/)

[Quicknode Faucet](https://faucet.quicknode.com/ethereum/sepoli)


# Setup & Deployment

## 1. Downloading cdk-validium-contracts

To begin the setup, lets first create a new directory named `cdk-validium` we can work out of. Initially this directory will house `cdk-validium-contracts`, but we will add the `cdk-validium-node` later on.

```bash
mkdir cdk-validium
cd cdk-validium/
```

Download the ***`0.0.2`*** release from the [cdk-validium-contracts github](https://github.com/0xPolygon/cdk-validium-contracts/releases/tag/v0.0.2-RC1)

It is available in both `.tar.gz` and `.zip` formats

Downloading the `tar.gz` and extracting

```bash
~/cdk-validium % curl -L -o cdk-validium-contracts.tar.gz https://github.com/0xPolygon/cdk-validium-contracts/archive/refs/tags/v0.0.2.tar.gz

tar -xzf cdk-validium-contracts.tar.gz
```

Now we have a new directory in `cdk-validium/` named `cdk-validium-contracts-0.0.2`

## 2. Preparing the environment

Navigate into the recently downloaded directory `cdk-validium-contracts-0.0.2/`

```bash
~/cdk-validium % cd cdk-validium-contracts-0.0.2/
```

### Install the dependencies

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2 % npm install
```

### Create the .env configuration

Copy the environment configuration example into a new `.env` file

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2 % cp .env.example .env
```

There are a few environment variables we have to fill in here:

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/.env
MNEMONIC=""
INFURA_PROJECT_ID=""
ETHERSCAN_API_KEY=""
```

First we will generate a new mnemonic using `cast`:

```bash
cast wallet new-mnemonic --words 12
```

*note: if command **`new-mnemonic`** is not found, update foundry using **`foundryup`***

The output should look like this:

```bash
Phrase:
island debris exhaust typical clap debate exhaust little verify mean sausage entire
Accounts:
- Account 0:
Address:     0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE
Private key: 0x3b01870a8449ada951f59c0275670bea1fc145954ee7cb1d46f7d21533600726
```

Copy and paste the newly generated `Phrase` into the `MNEMONIC` field inside `.env`

Also, **keep track of this generated address and private key**, we will use it for the remainder of the guide.

For this guide we are using Infura as our node provider. You can grab a `Project ID` from Infura [here](https://www.infura.io). Then copy and paste the value into `INFURA_PROJECT_ID`. If you want to use another node provider, see the "Using a different node provider" section below.

**Optionally**, we can verify our contracts using [Etherscan](https://etherscan.io). For this, an API key is required. 

You can grab an API key from [here](https://etherscan.io/myapikey), copy and paste the value into ***`ETHERSCAN_API_KEY`***

Your final `.env` should look similar to this:

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/.env
MNEMONIC="island debris exhaust typical clap debate exhaust little verify mean sausage entire"
INFURA_PROJECT_ID="1234567890abcdefghijklmnop" # or blank if not using Infura
ETHERSCAN_API_KEY="1234567890abcdefghijklmnopqr" # or blank if not verify contracts
```

## 3. Deploying the contracts

First, we must navigate into the `deployment/` directory and create a new `deploy_parameters.json` by copying the example

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2 % cd deployment
~/cdk-validium/cdk-validium-contracts-0.0.2/deployment % cp deploy_parameters.json.example deploy_parameters.json
```

### Configure deployment parameters

There are several fields that need to be changed inside `deploy_parameters.json`.

First, change the value of these fields to the address we generated using cast.

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/deploy_parameters.json
"trustedSequencer": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE"
"trustedAggregator": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE"
"admin": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE"
"cdkValidiumOwner": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE"
"initialCDKValidiumDeployerOwner": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE"
```

Next change the value of these two fields which will ensure compatibility with the `cdk-validium-node` version we will later deploy.

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/deploy_parameters.json
"trustedSequencerURL": "http://127.0.0.1:8123"
"forkID": 6
```

Your complete `deploy_parameters.json` should look similar to this:

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/deploy_parameters.json
{
    "realVerifier": false,
    "trustedSequencerURL": "http://127.0.0.1:8123,
    "networkName": "cdk-validium",
    "version":"0.0.1",
    "trustedSequencer":"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "chainID": 1001,
    "trustedAggregator":"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "trustedAggregatorTimeout": 604799,
    "pendingStateTimeout": 604799,
    "forkID": 6,
    "admin":"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "cdkValidiumOwner": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "timelockAddress": "0x617b3a3528F9cDd6630fd3301B9c8911F7Bf063D",
    "minDelayTimelock": 3600,
    "salt": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "initialCDKValidiumDeployerOwner" :"0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
    "maticTokenAddress":"0x617b3a3528F9cDd6630fd3301B9c8911F7Bf063D",
    "cdkValidiumDeployerAddress":"",
    "deployerPvtKey": "",
    "maxFeePerGas":"",
    "maxPriorityFeePerGas":"",
    "multiplierGas": "",
    "setupEmptyCommittee": true,
    "committeeTimelock": false
}
```

Now we have set the configuration for our contracts, lets deploy!

The first step is deploying and verifying the `CDKValidiumDeployer`, this will be the factory for deterministic contracts, the address of the contracts will depend on the salt and the `initialCDKValidiumDeployerOwner` inside `deploy_parameters.json`.

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2/deployment % npm run deploy:deployer:CDKValidium:sepolia
```

On successful deployment of `CDKValidiumDeployer`, you should see something similar to this:

```bash
cdkValidiumDeployer deployed on:  0x87572242776ccb6c98F4Cf1ff20f7e5a4e4142fF
```

Now we can move forward and deploy the rest of the contract suite:

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2/deployment % npm run deploy:testnet:CDKValidium:sepolia
```

Please note this can take several minutes depending on network conditions.

On successful deployment, a new directory named `deployments` should be created. Inside that directory, another was created with information about your deployment.

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/deployments/sepolia_1705429054/deploy_output.json
{
 "cdkValidiumAddress": "0x37eEBCa90363b0952e03a64979B64AAE3b8C9631",
 "polygonZkEVMBridgeAddress": "0x3485bfA6F27e54a8FF9782032fdbC7C555c178E4",
 "polygonZkEVMGlobalExitRootAddress": "0x8330E90c82F4BDDfa038041B898DE2d900e6246C",
 "cdkDataCommitteeContract": "0xb49d901748c3E278a634c05E0c500b23db992fb0",
 "maticTokenAddress": "0x20db28Bb7C779a06E4081e55808cc5DE35cca303",
 "verifierAddress": "0xb01Be1534d1eF82Ba98DCd5B33A3A331B6d119D0",
 "cdkValidiumDeployerContract": "0x87572242776ccb6c98F4Cf1ff20f7e5a4e4142fF",
 "deployerAddress": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
 "timelockContractAddress": "0xDa476BD0B6A660cd08239dEb620F701877688c6F",
 "deploymentBlockNumber": 5097689,
 "genesisRoot": "0xf07cd7c4f7df7092241ccb2072d9ac9e800a87513df628245657950b3af78f94",
 "trustedSequencer": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
 "trustedSequencerURL": "http://127.0.0.1:8123",
 "chainID": 1001,
 "networkName": "cdk-validium",
 "admin": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
 "trustedAggregator": "0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE",
 "proxyAdminAddress": "0x828E55268524c13dB25FD33f2Be45D2771f1EeA4",
 "forkID": 6,
 "salt": "0x0000000000000000000000000000000000000000000000000000000000000000",
 "version": "0.0.1"
}
```

In addition to `deploy_output.json`, a `genesis.json` should have been generated in `~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/`
Keep track of these two files `genesis.json` and `deploy_output.json` as they are crucial for our next step of setting up and deploying the `cdk-validium-node`.


Congrats! You’ve deployed the CDK Validium contracts!

### Verifying contracts

If deploying to Sepolia, the contracts should be automatically verified based on other live deployments on the network with similar bytecode. If you see that the contracts have not been verified on Etherscan. Run the following commands:

To verify the contract factory:

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2/deployment % npm run verify:deployer:CDKValidium:sepolia
```

To verify the rest of the contract suite:

```bash
~/cdk-validium/cdk-validium-contracts-0.0.2/deployment % npm run verify:CDKValidium:sepolia
```

### Using a different node provider

If you prefer to use a different node provider than Infura, the contents of `~/cdk-validium/cdk-validium-contracts-0.0.2/hardhat.config.js` and `.env` can be modified to fit your provider.

For example using Alchemy:

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/.env
MNEMONIC="island debris exhaust typical clap debate exhaust little verify mean sausage entire"
INFURA_PROJECT_ID="" # or blank if not using Infura
ETHERSCAN_API_KEY="1234567890abcdefghijklmnopqr" # or blank if not verify contracts
ALCHEMY_PROJECT_ID="dGPpsDzM9KpFTEnqMO44rvIXcc0fmgxr" # add this line
```

```bash
# ~/cdk-validium/cdk-validium-contracts-0.0.2/hardhat.config.js
sepolia: {
      url: `https://eth-sepolia.g.alchemy.com/v2/${process.env.ALCHEMY_PROJECT_ID}`, # rpc value changed here
      accounts: {
        mnemonic: process.env.MNEMONIC || DEFAULT_MNEMONIC,
        path: "m/44'/60'/0'/0",
        initialIndex: 0,
        count: 20,
      },
    },
```

### Deployment failure

- Since there are deterministic address you cannot deploy twice on the same network using the same `salt` and `initialCDKValidiumDeployerOwner` inside `deploy_parameters.json`. Changing one of them is enough to make a new deployment.

- It's mandatory to delete the `~/cdk-validium/cdk-validium-contracts-0.0.2/.openzeppelin` upgradability information in order to make a new deployment
