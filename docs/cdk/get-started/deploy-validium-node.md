# Prerequisites

## Dependency Checking

| Dependency | Version | Version Check Command |
| --- | --- | --- |
| golang | 1.21.3 | go version |
| make | 4.3 | make —v |
| docker | 24.0.5 | docker -v |
| postgres | 16.1 | psql -V |
| jq |  |  |
| tomql |  |  |

# Setup & Deployment

First, lets navigate back to the working directory we created earlier, `~/cdk-validium`

```bash
cd ~/cdk-validium
```

For this setup, we will also create a directory in `/tmp/` named `cdk` this will house our environment variables and other configs for the deployment

```bash
mkdir /tmp/cdk/
```

Create a `.env` inside `/tmp/cdk/` and fill with your respective values:

```bash
# /tmp/cdk/.env
TEST_ADDRESS=0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE
TEST_PRIVATE_KEY=0x3b01870a8449ada951f59c0275670bea1fc145954ee7cb1d46f7d21533600726 
L1_URL=https://sepolia.infura.io/v3/<YOUR INFURA PROJECT ID>
L1_WS_URL=wss://sepolia.infura.io/ws/v3/<YOUR INFURA PROJECT ID>
```

We will also need our information from `deploy_output.json` inside `~/cdk-validium/cdk-validium-contracts-0.0.2/deployment`. Navigate back to ``~/cdk-validium/cdk-validium-contracts-0.0.2/deployment` and run the following script to fill the required parameters into our `.env`:

```bash
cd ~/cdk-validium/cdk-validium-contracts-0.0.2/deployment
echo "GEN_BLOCK_NUMBER=$(jq -r '.deploymentBlockNumber' deploy_output.json)" >> /tmp/cdk/.env
echo "CDK_VALIDIUM_ADDRESS=$(jq -r '.cdkValidiumAddress' deploy_output.json)" >> /tmp/cdk/.env
echo "POLYGON_ZKEVM_BRIDGE_ADDRESS=$(jq -r '.polygonZkEVMBridgeAddress' deploy_output.json)" >> /tmp/cdk/.env
echo "POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS=$(jq -r '.polygonZkEVMGlobalExitRootAddress' deploy_output.json)" >> /tmp/cdk/.env
echo "CDK_DATA_COMMITTEE_CONTRACT_ADDRESS=$(jq -r '.cdkDataCommitteeContract' deploy_output.json)" >> /tmp/cdk/.env
echo "MATIC_TOKEN_ADDRESS=$(jq -r '.maticTokenAddress' deploy_output.json)" >> /tmp/cdk/.env
```

Source our new environment:

```bash
source /tmp/cdk/.env
```

## 1. Downloading cdk-validium-node & cdk-data-availability

Now clone the `0.0.3` release of `cdk-validium-node`. 

```bash
git clone --depth 1 --branch v0.0.3 https://github.com/0xPolygon/cdk-validium-node.git
```

In addition to `cdk-validium-node`, we also must download and extract version `0.0.3` of `cdk-data-availability.` The release file is available here:

Downloading `cdk-validium-contracts` as a ***`tar.gz`*** and extracting

```bash
~/cdk-validium % curl -L -o cdk-data-availability.tar.gz https://github.com/0xPolygon/cdk-data-availability/archive/refs/tags/v0.0.3.tar.gz
\tar -xzf cdk-data-availability.tar.gz
```

Now we have two new directories in *`cdk-validium/`* named ***`cdk-data-availability-0.0.3` and `cdk-validium-node`***

## 2. Preparing the environment

We will begin our setup in the node directory.

Navigate into the node directory we cloned from the previous step***`cdk-validium-node/`***

```bash
~/cdk-validium % cd cdk-validium-node/
```

### Setup the database

Run the docker command to start an instance of the `psql` database. The database is used for many of the services, such as the node, prover, DAC, and bridge service.

```bash
docker run -e POSTGRES_USER=cdk_user -e POSTGRES_PASSWORD=cdk_password -e POSTGRES_DB=postgres -p 5432:5432 postgres:15
```

Once you have postgres running, validate you have the following setup:

- admin account called: `cdk_user` with a password of `cdk_password`
- postgres server running on `localhost:5432`

You can use the following command to validate the steps, `\q` to exit:

```bash
PGPASSWORD=cdk_password psql -h localhost -U cdk_user -d postgres -p 5432
#=> \q
```

### Provision the database

The `cdk-validium-node` directory contains a script called `single_db_server.sql` that provisions all the databases required for the prover, state, and pool to operate. Run the script to provision all the necessary databases and schemas that will be used for both the prover and node:

```bash
~/cdk-validium/cdk-validium-node % PGPASSWORD=cdk_password psql -h localhost -U cdk_user -d postgres -p 5432 -a -q -f ./db/scripts/single_db_server.sql
```

In addition to the provisions required for the prover and node, another provision is needed for the Data Availability Committee (DAC). We can set this up now for use later.

```bash
PGPASSWORD=cdk_password psql -h localhost -U cdk_user -d postgres -p 5432 -c "CREATE DATABASE committee_db;"
```

Finally, we will provision a database for our bridge service, which we will setup last in this guide.

```bash
PGPASSWORD=cdk_password psql -h localhost -U cdk_user -d postgres -p 5432 -c "CREATE DATABASE bridge_db;"
```

### Configure the prover

To configure our prover, lets copy and modify the contents of `test.prover.config.json` inside `~/cdk-validium/cdk-validium-node/test/config/` to our `/tmp/cdk/` directory we created earlier using the following jq command:

```bash
~/cdk-validium/cdk-validium-node % jq '.aggregatorClientHost = "127.0.0.1" | .databaseURL = "postgresql://cdk_user:cdk_password@localhost:5432/postgres"' ./test/config/test.prover.config.json > /tmp/cdk/test.prover.config.json
```

### Configure the node

To create the configuration files for our node, we first must build the `cdk-validium-node` itself.

```bash
~/cdk-validium/cdk-validium-node % make build
```

Now we can create a keystore. This will be referenced in all of our`config.toml` we create in the next steps. Take the private key we generated and stored in our `.env` earlier and encrypt it with a basic password.

For example:

```bash
./dist/zkevm-node encryptKey --pk=$TEST_PRIVATE_KEY--pw="testonly" --output=/etc/zkevm/account.keystore
find /etc/zkevm/account.keystore -type f -name 'UTC--*' | head -n 1 | xargs -I xxx mv xxx ~/account.key
```

*note: make sure your environment is sourced ie: `source /tmp/cdk/.env`*

The output keystore is now stored in `/tmp/cdk/account.key`

Create a new file `node-config.toml` inside `/tmp/cdk/` and paste the following content:

Example `node-config.toml`:

```bash
#/tmp/cdk/node-config.toml
IsTrustedSequencer = true

[Log]
Environment = "development" # "production" or "development"
Level = "debug"
Outputs = ["stderr"]

[State]
	[State.DB]
	User = "cdk_user"
	Password = "cdk_password"
	Name = "state_db"
	Host = "localhost"
	Port = "5432"
	EnableLog = false	
	MaxConns = 200
	[State.Batch]
		[State.Batch.Constraints]
		MaxTxsPerBatch = 300
		MaxBatchBytesSize = 120000
		MaxCumulativeGasUsed = 30000000
		MaxKeccakHashes = 2145
		MaxPoseidonHashes = 252357
		MaxPoseidonPaddings = 135191
		MaxMemAligns = 236585
		MaxArithmetics = 236585
		MaxBinaries = 473170
		MaxSteps = 7570538

[Pool]
FreeClaimGasLimit = 1500000
IntervalToRefreshBlockedAddresses = "5m"
IntervalToRefreshGasPrices = "5s"
MaxTxBytesSize=100132
MaxTxDataBytesSize=100000
DefaultMinGasPriceAllowed = 0
MinAllowedGasPriceInterval = "5m"
PollMinAllowedGasPriceInterval = "15s"
AccountQueue = 64
GlobalQueue = 1024
	[Pool.EffectiveGasPrice]
		Enabled = false
		L1GasPriceFactor = 0.25
		ByteGasCost = 16
		ZeroByteGasCost = 4
		NetProfit = 1
	    BreakEvenFactor = 1.1			
		FinalDeviationPct = 10
		L2GasPriceSuggesterFactor = 0.5
	[Pool.DB]
	User = "cdk_user"
	Password = "cdk_password"
	Name = "pool_db"
	Host = "localhost"
	Port = "5432"
	EnableLog = false
	MaxConns = 200

[Etherman]
URL = "https://sepolia.infura.io/v3/bd6164d34c324fa08ca5b6dc1d3ed3a2"
ForkIDChunkSize = 20000
MultiGasProvider = false
	[Etherscan]
		ApiKey = ""

[RPC]
Host = "0.0.0.0"
Port = 8123
ReadTimeout = "60s"
WriteTimeout = "60s"
MaxRequestsPerIPAndSecond = 5000
SequencerNodeURI = ""
BatchRequestsEnabled = true
EnableL2SuggestedGasPricePolling = true
	[RPC.WebSockets]
		Enabled = true
		Port = 8133

[Synchronizer]
SyncInterval = "1s"
SyncChunkSize = 100
TrustedSequencerURL = "" # If it is empty or not specified, then the value is read from the smc.
L1SynchronizationMode = "sequential" # "sequential" or "parallel"
	[Synchronizer.L1ParallelSynchronization]
		MaxClients = 10
		MaxPendingNoProcessedBlocks = 25
		RequestLastBlockPeriod = "5s"
		RequestLastBlockTimeout = "5s"
		RequestLastBlockMaxRetries = 3
		StatisticsPeriod = "5m"
		TimeoutMainLoop = "5m"
		RollupInfoRetriesSpacing= "5s"
		FallbackToSequentialModeOnSynchronized = false
		[Synchronizer.L1ParallelSynchronization.PerformanceWarning]
			AceptableInacctivityTime = "5s"
			ApplyAfterNumRollupReceived = 10

[Sequencer]
WaitPeriodPoolIsEmpty = "1s"
LastBatchVirtualizationTimeMaxWaitPeriod = "10s"
BlocksAmountForTxsToBeDeleted = 100
FrequencyToCheckTxsForDelete = "12h"
TxLifetimeCheckTimeout = "10m"
MaxTxLifetime = "3h"
	[Sequencer.Finalizer]
		GERDeadlineTimeout = "2s"
		ForcedBatchDeadlineTimeout = "5s"
		SleepDuration = "100ms"
		ResourcePercentageToCloseBatch = 10
		GERFinalityNumberOfBlocks = 0
		ClosingSignalsManagerWaitForCheckingL1Timeout = "10s"
		ClosingSignalsManagerWaitForCheckingGER = "10s"
		ClosingSignalsManagerWaitForCheckingForcedBatches = "10s"
		ForcedBatchesFinalityNumberOfBlocks = 0
		TimestampResolution = "10s"
		StopSequencerOnBatchNum = 0
	[Sequencer.DBManager]
		PoolRetrievalInterval = "500ms"
		L2ReorgRetrievalInterval = "5s"

[SequenceSender]
WaitPeriodSendSequence = "15s"
LastBatchVirtualizationTimeMaxWaitPeriod = "10s"
MaxTxSizeForL1 = 131072
L2Coinbase = "0xf100D00c376D62682Faf28FeE5cF603AAED75e13"
PrivateKey = {Path = "/tmp/cdk/account.key", Password = "testonly"}

[Aggregator]
Host = "0.0.0.0"
Port = 50081
RetryTime = "5s"
VerifyProofInterval = "10s"
TxProfitabilityCheckerType = "acceptall"
TxProfitabilityMinReward = "1.1"
ProofStatePollingInterval = "5s"
SenderAddress = "0xf100D00c376D62682Faf28FeE5cF603AAED75e13"
CleanupLockedProofsInterval = "2m"
GeneratingProofCleanupThreshold = "10m"

[EthTxManager]
ForcedGas = 0
PrivateKeys = [
	{Path = "/tmp/cdk/account.key", Password = "testonly"}
]

[L2GasPriceSuggester]
Type = "default"
UpdatePeriod = "10s"
Factor = 0.5
DefaultGasPriceWei = 0
MaxGasPriceWei = 0

[MTClient]
URI  = "localhost:50061"

[Executor]
URI = "localhost:50071"
MaxGRPCMessageSize = 100000000

[Metrics]
Host = "0.0.0.0"
Port = 9091
Enabled = true
ProfilingHost = "0.0.0.0"
ProfilingPort = 6060
ProfilingEnabled = true

[HashDB]
User = "cdk_user"
Password = "cdk_password"
Name = "prover_db"
Host = "localhost"
Port = "5432"
EnableLog = false
MaxConns = 200
```

We will modify the `URL` parameter in`[Etherman]` to the URL of our RPC Provider, along with the parameters `L2Coinbase` in `[SequenceSender]` and `SenderAddress` in `[Aggregator]` to the address we generated earlier. Here’s a script to replace those values automatically using your environment sourced from `/tmp/cdk/`

```bash
tomlq -i -t --arg L1_URL "$L1_URL" '.Etherman.URL = $L1_URL' /tmp/cdk/node-config.toml
tomlq -i -t --arg TEST_ADDRESS "$TEST_ADDRESS" '.SequenceSender.L2Coinbase = $TEST_ADDRESS' /tmp/cdk/node-config.toml
tomlq -i -t --arg TEST_ADDRESS "$TEST_ADDRESS" '.Aggregator.SenderAddress = $TEST_ADDRESS' /tmp/cdk/node-config.toml
```

Now we have to copy and modify the `genesis.json` from our earlier deployment of contracts to include information about our newly configured chain

You can find `genesis.json` inside `~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/`

The information we are going to append:

```bash
#~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/genesis.json
"L1Config": {
  "chainId": 11155111,
  "maticTokenAddress": "0xd76B50509c1693C7BA35514103a0A156Ca57980c",
  "polygonZkEVMAddress": "0x52C8f9808246eF2ce992c0e1f04fa54ec3378dD1",
  "cdkDataCommitteeContract": "0x8346026951978bd806912d0c93FB0979D8E3436a",
  "polygonZkEVMGlobalExitRootAddress": "0xE3A721c20B30213FEC306dd60f6c7F2fCB8b46D2"
},
"genesisBlockNumber": 5098088
```

Here is a script to automate the process:

```bash
jq --argjson data "$(jq '{maticTokenAddress, cdkValidiumAddress, cdkDataCommitteeContract, polygonZkEVMGlobalExitRootAddress, deploymentBlockNumber}' ~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/deploy_output.json)" \
'.L1Config.chainId = 11155111 | 
.L1Config.maticTokenAddress = $data.maticTokenAddress | 
.L1Config.polygonZkEVMAddress = $data.cdkValidiumAddress | 
.L1Config.cdkDataCommitteeContract = $data.cdkDataCommitteeContract | 
.L1Config.polygonZkEVMGlobalExitRootAddress = $data.polygonZkEVMGlobalExitRootAddress | 
.genesisBlockNumber = $data.deploymentBlockNumber' ~/cdk-validium/cdk-validium-contracts-0.0.2/deployment/genesis.json > /tmp/cdk/genesis.json
```

### Configure the DAC

At this point we have setup and provisioned the psql database and configured the zk prover and node. Now let’s configure our Data Availability Comittee

Navigate into `~/cdk-validium/cdk-data-availability-0.0.3` we downloaded in step 1.

Build the DAC

```bash
~/cdk-validium/cdk-data-availability-0.0.3 % make build
```

Now we can create a `dac-config.toml` file inside `/tmp/cdk/` . Copy and paste the following example config from below, then run the sequential `tomlq` script to replace the necessary parameters.

```bash
#~/tmp/cdk/dac-config.toml
PrivateKey = {Path = "/root/account.key", Password = "testonly"}

[L1]
WsURL = "wss://sepolia.infura.io/ws/v3/bd6164d34c324fa08ca5b6dc1d3ed3a2"
RpcURL = "https://sepolia.infura.io/v3/bd6164d34c324fa08ca5b6dc1d3ed3a2"
CDKValidiumAddress = "0x52C8f9808246eF2ce992c0e1f04fa54ec3378dD1"
DataCommitteeAddress = "0x8346026951978bd806912d0c93FB0979D8E3436a"
Timeout = "3m" # Make sure this value is less than the rootchain-int-ws loadbalancer timeout
RetryPeriod = "5s"

[Log]
Environment = "development" # "production" or "development"
Level = "debug"
Outputs = ["stderr"]

[DB]
User = "cdk_user"
Password = "cdk_password"
Name = "committee_db"
Host = "127.0.0.1"
Port = "5432"
EnableLog = false
MaxConns = 10

[RPC]
Host = "0.0.0.0"
Port = 8444
ReadTimeout = "60s"
WriteTimeout = "60s"
MaxRequestsPerIPAndSecond = 500
SequencerNodeURI = ""
EnableL2SuggestedGasPricePolling = false
	[RPC.WebSockets]
		Enabled = false
```

Replace the values automatically:

```bash
tomlq -i -t --arg L1_URL "$L1_URL" '.L1.RpcURL = $L1_URL' /tmp/cdk/dac-config.toml
tomlq -i -t --arg L1_WS_URL "$L1_WS_URL" '.L1.WsURL = $L1_WS_URL' /tmp/cdk/dac-config.toml
tomlq -i -t --arg CDK_VALIDIUM_ADDRESS "$CDK_VALIDIUM_ADDRESS" '.L1.CDKValidiumAddress = $CDK_VALIDIUM_ADDRESS' /tmp/cdk/dac-config.toml
tomlq -i -t --arg CDK_VALIDIUM_ADDRESS "$CDK_VALIDIUM_ADDRESS" '.L1.DataCommitteeAddress = $CDK_VALIDIUM_ADDRESS' /tmp/cdk/dac-config.toml
```

Now we can update the contracts on Sepolia with information about our DAC

```bash
cast send \
        --legacy \
        --from 0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE\
        --private-key 0x3b01870a8449ada951f59c0275670bea1fc145954ee7cb1d46f7d21533600726 \
        --rpc-url https://sepolia.infura.io/v3/bd6164d34c324fa08ca5b6dc1d3ed3a2 \
        0x8346026951978bd806912d0c93FB0979D8E3436a \
        'function setupCommittee(uint256 _requiredAmountOfSignatures, string[] urls, bytes addrsBytes) returns()' \
        1 \
        '["http://localhost:8444"]' \
        0x8Ea797e7f349dA91078B1d833C534D2c392BB7FE
```

*note: this takes a few minutes*

### Configure Bridge Service

Create a starter bridge config `bridge-config.toml` under `/tmp/cdk`:

`config.toml`:

```bash
[Log]
Level = "info"
Outputs = ["stderr"]

[SyncDB]
Database = "postgres"
User = "cdk_user"
Name = "bridge_db"
Password = "cdk_password"
Host = "localhost"
Port = "5432"
MaxConns = 20

[Etherman]
L1URL = "https://sepolia.infura.io/v3/b27a8be73bcb4bc7a83aada13c65e135"
L2URLs = ["http://localhost:8123"]

[Synchronizer]
SyncInterval = "1s"
SyncChunkSize = 100

[BridgeController]
Store = "postgres"
Height = 32

[BridgeServer]
GRPCPort = "9090"
HTTPPort = "8080"
DefaultPageLimit = 25
MaxPageLimit = 100
BridgeVersion = "v1"
    # Read only
    [BridgeServer.DB]
    Database = "postgres"
    User = "cdk_user"
    Name = "bridge_db"
    Password = "cdk_password"
    Host = "localhost"
    Port = "5432"
    MaxConns = 20

[NetworkConfig]
GenBlockNumber = "5098088"
PolygonZkEVMAddress = "0x52C8f9808246eF2ce992c0e1f04fa54ec3378dD1"
PolygonBridgeAddress = "0x24F2aF81Ae588690C9752A342d7549f58133CE4e"
PolygonZkEVMGlobalExitRootAddress = "0xE3A721c20B30213FEC306dd60f6c7F2fCB8b46D2"
MaticTokenAddress = "0xd76B50509c1693C7BA35514103a0A156Ca57980c"
L2PolygonBridgeAddresses = ["0x24F2aF81Ae588690C9752A342d7549f58133CE4e"]
L1ChainID = 11155111

[ClaimTxManager]
FrequencyToMonitorTxs = "1s"
PrivateKey = {Path = "/tmp/cdk/account.key", Password = "testonly"}
Enabled = true
RetryInterval = "1s"
RetryNumber = 10
```

And replace the values using the `tomlq`script:

```bash
tomlq -i -t --arg L1_URL "$L1_URL" '.Etherman.L1URL = $L1_URL' /tmp/cdk/bridge-config.toml
tomlq -i -t --arg GEN_BLOCK_NUMBER "$GEN_BLOCK_NUMBER" '.NetworkConfig.GenBlockNumber = $GEN_BLOCK_NUMBER' /tmp/cdk/bridge-config.toml
tomlq -i -t --arg CDK_VALIDIUM_ADDRESS "$CDK_VALIDIUM_ADDRESS" '.NetworkConfig.PolygonZkEVMAddress = $CDK_VALIDIUM_ADDRESS' /tmp/cdk/bridge-config.toml
tomlq -i -t --arg POLYGON_ZKEVM_BRIDGE_ADDRESS "$POLYGON_ZKEVM_BRIDGE_ADDRESS" '.NetworkConfig.PolygonBridgeAddress = $POLYGON_ZKEVM_BRIDGE_ADDRESS' /tmp/cdk/bridge-config.toml
tomlq -i -t --arg POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS "$POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS" '.NetworkConfig.PolygonZkEVMGlobalExitRootAddress = $POLYGON_ZKEVM_GLOBAL_EXIT_ROOT_ADDRESS' /tmp/cdk/bridge-config.toml
tomlq -i -t --arg MATIC_TOKEN_ADDRESS "$MATIC_TOKEN_ADDRESS" '.NetworkConfig.MaticTokenAddress = $MATIC_TOKEN_ADDRESS' /tmp/cdk/bridge-config.toml
tomlq -i -t --arg CDK_DATA_COMMITTEE_CONTRACT_ADDRESS "$CDK_DATA_COMMITTEE_CONTRACT_ADDRESS" '.NetworkConfig.L2PolygonBridgeAddresses = [$CDK_DATA_COMMITTEE_CONTRACT_ADDRESS]' /tmp/cdk/bridge-config.toml
```

## 3. Running the components

### Run the prover

Since the prover is large and rather compute expensive to build, we will use a docker container

```bash
docker run -v "/tmp/cdk/test.prover.config.json:/usr/src/app/config.json" -p 50061:50061 -p 50071:50071 --network host hermeznetwork/zkevm-prover:v3.0.2 zkProver -c /usr/src/app/config.json
```

### Run the node

```bash
~/cdk-validium/cdk-validium-node % ./dist/zkevm-node run --network custom --custom-network-file /tmp/cdk/genesis.json --cfg /tmp/cdk/node-config.toml \
	--components sequencer \
	--components sequence-sender \
	--components aggregator \
	--components rpc --http.api eth,net,debug,zkevm,txpool,web3 \
	--components synchronizer \
	--components eth-tx-manager \
	--components l2gaspricer
```

Run the additional approval scripts for node:

```bash
~/cdk-validium/cdk-validium-node % ./dist/zkevm-node approve --network custom \
	--custom-network-file /tmp/cdk/genesis.json \
	--cfg /tmp/cdk/node-config.toml \
	--amount 1000000000000000000000000000 \
	--password "testonly" --yes --key-store-path /tmp/cdk/account.key
```

### Run the DAC

```bash
~/cdk-validium/cdk-data-availability-0.0.3 % ./dist/cdk-data-availability run --cfg /tmp/cdk/dac-config.toml
```

### Run the Bridge Service

```bash
./dist/zkevm-bridge run --cfg /tmp/cdk/bridge-config.toml
```