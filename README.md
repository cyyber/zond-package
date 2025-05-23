# Zond Package

This is a [Kurtosis][kurtosis-repo] package that will spin up a private Zond testnet over Docker or Kubernetes with multi-client support, Flashbot's `mev-boost` infrastructure for PBS-related testing/validation, and other useful network tools (transaction spammer, monitoring tools, etc). Kurtosis packages are entirely reproducible and composable, so this will work the same way over Docker or Kubernetes, in the cloud or locally on your machine.

You now have the ability to spin up a private Zond testnet or public devnet/testnet with a single command. This package is designed to be used for testing, validation, and development of Zond clients, and is not intended for production use. For more details check network_params.network in the [configuration section](./README.md#configuration).

Specifically, this [package][package-reference] will:

1. Generate Execution Layer (EL) & Consensus Layer (CL) genesis information using [the Zond genesis generator](https://github.com/theQRL/zond-genesis-generator).
2. Configure & bootstrap a network of Zond nodes of *n* size using the genesis data generated above
3. Spin up a [transaction spammer](https://github.com/MariusVanDerWijden/tx-fuzz) to send fake transactions to the network
4. Spin up and connect a [testnet verifier](https://github.com/ethereum/merge-testnet-verifier)
5. Spin up a Grafana and Prometheus instance to observe the network

Optional features (enabled via flags or parameter files at runtime):

* Block until the Beacon nodes finalize an epoch (i.e. finalized_epoch > 0)
* Spin up & configure parameters for the infrastructure behind Flashbot's implementation of PBS using `mev-boost`, in either `full` or `mock` mode. More details [here](./README.md#proposer-builder-separation-pbs-implementation-via-flashbots-mev-boost-protocol).
* Spin up & connect the network to a [beacon metrics gazer service](https://github.com/dapplion/beacon-metrics-gazer) to collect network-wide participation metrics.
* Spin up and connect a [JSON RPC Snooper](https://github.com/ethDreamer/json_rpc_snoop) to the network log responses & requests between the EL engine API and the CL client.
* Specify extra parameters to be passed in for any of the: CL client Beacon, and CL client validator, and/or EL client containers
* Specify the required parameters for the nodes to reach an external block building network
* Generate keystores for each node in parallel

## Quickstart

1. [Install Docker & start the Docker Daemon if you haven't done so already][docker-installation]
2. [Install the Kurtosis CLI, or upgrade it to the latest version if it's already installed][kurtosis-cli-installation]
3. Run the package with default configurations from the command line:

   ```bash
   kurtosis run --enclave my-testnet github.com/theQRL/zond-package
   ```

#### Run with your own configuration

Kurtosis packages are parameterizable, meaning you can customize your network and its behavior to suit your needs by storing parameters in a file that you can pass in at runtime like so:

```bash
kurtosis run --enclave my-testnet github.com/theQRL/zond-package --args-file network_params.yaml
```

Where `network_params.yaml` contains the parameters for your network in your home directory.

#### Run on Kubernetes

Kurtosis packages work the same way over Docker or on Kubernetes. Please visit our [Kubernetes docs](https://docs.kurtosis.com/k8s) to learn how to spin up a private testnet on a Kubernetes cluster.

#### Considerations for Running on a Public Testnet with a Cloud Provider
When running on a public testnet using a cloud provider's Kubernetes cluster, there are a few important factors to consider:

1. State Growth: The growth of the state might be faster than anticipated. This could potentially lead to issues if the default parameters become insufficient over time. It's important to monitor state growth and adjust parameters as necessary.

2. Persistent Storage Speed: Most cloud providers provision their Kubernetes clusters with relatively slow persistent storage by default. This can cause performance issues, particularly with Execution Layer (EL) clients.

3. Network Syncing: The disk speed provided by cloud providers may not be sufficient to sync with networks that have high demands, such as the mainnet. This could lead to syncing issues and delays.

To mitigate these issues, you can use the `el_volume_size` and `cl_volume_size` flags to override the default settings locally. This allows you to allocate more storage to the EL and CL clients, which can help accommodate faster state growth and improve syncing performance. However, keep in mind that increasing the volume size may also increase your cloud provider costs. Always monitor your usage and adjust as necessary to balance performance and cost.

For optimal performance, we recommend using a cloud provider that allows you to provision Kubernetes clusters with fast persistent storage or self hosting your own Kubernetes cluster with fast persistent storage.

### Shadowforking
In order to enable shadowfork capabilities, you can use the `network_params.network` flag. The expected value is the name of the network you want to shadowfork followed by `-shadowfork`. Please note that `persistent` configuration parameter has to be enabled for shadowforks to work! Current limitation on k8s is it is only working on a single node cluster. For example, to shadowfork the Holesky testnet, you can use the following command:
```yaml
...
network_params:
  network: "holesky-shadowfork"
persistent: true
...
```


#### Taints and tolerations
It is possible to run the package on a Kubernetes cluster with taints and tolerations. This is done by adding the tolerations to the `tolerations` field in the `network_params.yaml` file. For example:
```yaml
participants:
  - el_type: gzond
    cl_type: qrysm
global_tolerations:
  - key: "node-role.kubernetes.io/master6"
    value: "true"
    operator: "Equal"
    effect: "NoSchedule"
```

It is possible to define toleration globally, per participant or per container. The order of precedence is as follows:
1. Container (`el_tolerations`, `cl_tolerations`, `vc_tolerations`)
2. Participant (`tolerations`)
3. Global (`global_tolerations`)

This feature is only available for Kubernetes. To learn more about taints and tolerations, please visit the [Kubernetes documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

#### Tear down

The testnet will reside in an [enclave][enclave] - an isolated, ephemeral environment. The enclave and its contents (e.g. running containers, files artifacts, etc) will persist until torn down. You can remove an enclave and its contents with:

```bash
kurtosis enclave rm -f my-testnet
```

## Management

The [Kurtosis CLI](https://docs.kurtosis.com/cli) can be used to inspect and interact with the network.

For example, if you need shell access, simply run:

```bash
kurtosis service shell my-testnet $SERVICE_NAME
```

And if you need the logs for a service, simply run:

```bash
kurtosis service logs my-testnet $SERVICE_NAME
```

Check out the full list of CLI commands [here](https://docs.kurtosis.com/cli)

## Debugging

To grab the genesis files for the network, simply run:

```bash
kurtosis files download my-testnet $FILE_NAME $OUTPUT_DIRECTORY
```

For example, to retrieve the Execution Layer (EL) genesis data, run:

```bash
kurtosis files download my-testnet el-genesis-data ~/Downloads
```

# Basic file sharing

Apache is included in the package to allow for basic file sharing. The Apache service is started when additional services are enabled. It will expose the network-configs directory, which might needed if you want to share the network config publicly.

```yaml
additional_services:
  - apache
```

## Configuration

To configure the package behaviour, you can modify your `network_params.yaml` file. The full YAML schema that can be passed in is as follows with the defaults provided:

```yaml
# Specification of the participants in the network
participants:
  # EL(Execution Layer) Specific flags
    # The type of EL client that should be started
    # Valid values are gzond
  - el_type: gzond

    # The Docker image that should be used for the EL client; leave blank to use the default for the client type
    # Defaults by client:
    # - gzond: qrledger/go-zond:stable
    el_image: ""

    # The log level string that this participant's EL client should log at
    # If this is emptystring then the global `logLevel` parameter's value will be translated into a string appropriate for the client (e.g. if
    # global `logLevel` = `info` then Gzond would receive `3`)
    # If this is not emptystring, then this value will override the global `logLevel` setting to allow for fine-grained control
    # over a specific participant's logging
    el_log_level: ""

    # A list of optional extra env_vars the el container should spin up with
    el_extra_env_vars: {}

    # A list of optional extra labels the el container should spin up with
    # Example; el_extra_labels: {"zond-package.partition": "1"}
    el_extra_labels: {}

    # A list of optional extra params that will be passed to the EL client container for modifying its behaviour
    el_extra_params: []

    # A list of tolerations that will be passed to the EL client container
    # Only works with Kubernetes
    # Example: el_tolerations:
    # - key: "key"
    #   operator: "Equal"
    #   value: "value"
    #   effect: "NoSchedule"
    #   toleration_seconds: 3600
    # Defaults to empty
    el_tolerations: []

    # Persistent storage size for the EL client container (in MB)
    # Defaults to 0, which means that the default size for the client will be used
    # Default values can be found in /src/package_io/constants.star VOLUME_SIZE
    el_volume_size: 0

    # Resource management for el containers
    # CPU is milicores
    # RAM is in MB
    # Defaults to 0, which results in no resource limits
    el_min_cpu: 0
    el_max_cpu: 0
    el_min_mem: 0
    el_max_mem: 0

  # CL(Consensus Layer) Specific flags
    # The type of CL client that should be started
    # Valid values are qrysm
    cl_type: qrysm

    # The Docker image that should be used for the CL client; leave blank to use the default for the client type
    # Defaults by client:
    # - qrysm: qrledger/qrysm:beacon-chain-latest
    cl_image: ""

    # The log level string that this participant's CL client should log at
    # If this is emptystring then the global `logLevel` parameter's value will be translated into a string appropriate for the client (e.g. if
    # global `logLevel` = `info` then Qrysm would receive `info`, etc.)
    # If this is not emptystring, then this value will override the global `logLevel` setting to allow for fine-grained control
    # over a specific participant's logging
    cl_log_level: ""

    # A list of optional extra env_vars the cl container should spin up with
    cl_extra_env_vars: {}

    # A list of optional extra labels that will be passed to the CL client Beacon container.
    # Example; cl_extra_labels: {"zond-package.partition": "1"}
    cl_extra_labels: {}

    # A list of optional extra params that will be passed to the CL client Beacon container for modifying its behaviour
    # If the client combines the Beacon & validator nodes, then this list will be passed to the combined Beacon-validator node
    cl_extra_params: []

    # A list of tolerations that will be passed to the CL client container
    # Only works with Kubernetes
    # Example: el_tolerations:
    # - key: "key"
    #   operator: "Equal"
    #   value: "value"
    #   effect: "NoSchedule"
    #   toleration_seconds: 3600
    # Defaults to empty
    cl_tolerations: []

    # Persistent storage size for the CL client container (in MB)
    # Defaults to 0, which means that the default size for the client will be used
    # Default values can be found in /src/package_io/constants.star VOLUME_SIZE
    cl_volume_size: 0

    # Resource management for cl containers
    # CPU is milicores
    # RAM is in MB
    # Defaults to 0, which results in no resource limits
    cl_min_cpu: 0
    cl_max_cpu: 0
    cl_min_mem: 0
    cl_max_mem: 0

    # Whether to act as a supernode for the network
    # Supernodes will subscribe to all subnet topics
    # This flag should only be used with peerdas
    # Defaults to false
    supernode: false

    # Whether to use a separate validator client attached to the CL client.
    # Defaults to false for clients that can run both in one process
    use_separate_vc: true

  # VC (Validator Client) Specific flags
    # The type of validator client that should be used
    # Valid values are qrysm
    # ( The qrysm validator only works with a qrysm CL client )
    # Defaults to matching the chosen CL client (cl_type)
    vc_type: ""

    # The Docker image that should be used for the separate validator client
    # Defaults by client:
    # - qrysm: qrledger/qrysm:validator-latest
    vc_image: ""

    # The log level string that this participant's validator client should log at
    # If this is emptystring then the global `logLevel` parameter's value will be translated into a string appropriate for the client (e.g. if
    # global `logLevel` = `info` then Qrysm would receive `info`, etc.)
    # If this is not emptystring, then this value will override the global `logLevel` setting to allow for fine-grained control
    # over a specific participant's logging
    vc_log_level: ""

    # A list of optional extra env_vars the vc container should spin up with
    vc_extra_env_vars: {}

    # A list of optional extra labels that will be passed to the validator client validator container.
    # Example; vc_extra_labels: {"zond-package.partition": "1"}
    vc_extra_labels: {}

    # A list of optional extra params that will be passed to the validator client container for modifying its behaviour
    # If the client combines the Beacon & validator nodes, then this list will also be passed to the combined Beacon-validator node
    vc_extra_params: []

    # A list of tolerations that will be passed to the validator container
    # Only works with Kubernetes
    # Example: el_tolerations:
    # - key: "key"
    #   operator: "Equal"
    #   value: "value"
    #   effect: "NoSchedule"
    #   toleration_seconds: 3600
    # Defaults to empty
    vc_tolerations: []

    # Resource management for vc containers
    # CPU is milicores
    # RAM is in MB
    # Defaults to 0, which results in no resource limits
    vc_min_cpu: 0
    vc_max_cpu: 0
    vc_min_mem: 0
    vc_max_mem: 0

    # Count of the number of validators you want to run for a given participant
    # Default to null, which means that the number of validators will be using the
    # network parameter num_validator_keys_per_node
    validator_count: null

    # Whether to use a remote signer instead of the vc directly handling keys
    # Defaults to false
    use_remote_signer: false

  # Remote signer Specific flags
    # The type of remote signer that should be used
    # Valid values are web3signer, clef
    # Defaults to clef
    remote_signer_type: "clef"

    # The Docker image that should be used for the remote signer
    # Defaults to "qrledger/go-zond:alltools-stable"
    remote_signer_image: "qrledger/go-zond:alltools-stable"

    # A list of optional extra env_vars the remote signer container should spin up with
    remote_signer_extra_env_vars: {}

    # A list of optional extra labels that will be passed to the remote signer container.
    # Example; remote_signer_extra_labels: {"zond-package.partition": "1"}
    remote_signer_extra_labels: {}

    # A list of optional extra params that will be passed to the remote signer container for modifying its behaviour
    remote_signer_extra_params: []

    # A list of tolerations that will be passed to the remote signer container
    # Only works with Kubernetes
    # Example: remote_signer_tolerations:
    # - key: "key"
    #   operator: "Equal"
    #   value: "value"
    #   effect: "NoSchedule"
    #   toleration_seconds: 3600
    # Defaults to empty
    remote_signer_tolerations: []

    # Resource management for remote signer containers
    # CPU is milicores
    # RAM is in MB
    # Defaults to 0, which results in no resource limits
    remote_signer_min_cpu: 0
    remote_signer_max_cpu: 0
    remote_signer_min_mem: 0
    remote_signer_max_mem: 0

  # Participant specific flags
    # Node selector
    # Only works with Kubernetes
    # Example: node_selectors: { "disktype": "ssd" }
    # Defaults to empty
    node_selectors: {}

    # A list of tolerations that will be passed to the EL/CL/validator containers
    # This is to be used when you don't want to specify the tolerations for each container separately
    # Only works with Kubernetes
    # Example: tolerations:
    # - key: "key"
    #   operator: "Equal"
    #   value: "value"
    #   effect: "NoSchedule"
    #   toleration_seconds: 3600
    # Defaults to empty
    tolerations: []

    # Count of nodes to spin up for this participant
    # Default to 1
    count: 1

    # Snooper local flag for a participant.
    # Snooper can be enabled with the `snooper_enabled` flag per client or globally
    # Snooper dumps all JSON-RPC requests and responses including BeaconAPI, EngineAPI and ExecutionAPI.
    # Default to null
    snooper_enabled: null

    # Enables Ethereum Metrics Exporter for this participant. Can be set globally.
    # Defaults null and then set to global ethereum_metrics_exporter_enabled (false)
    ethereum_metrics_exporter_enabled: null

    # Enables Xatu Sentry for this participant. Can be set globally.
    # Defaults null and then set to global xatu_sentry_enabled (false)
    xatu_sentry_enabled: null

    # Prometheus additional configuration for a given participant prometheus target.
    # Execution, beacon and validator client targets on prometheus will include this
    # configuration.
    prometheus_config:
      # Scrape interval to be used. Default to 15 seconds
      scrape_interval: 15s
      # Additional labels to be added. Default to empty
      labels: {}

    # A set of parameters the node needs to reach an external block building network
    # If `null` then the builder infrastructure will not be instantiated
    # Example:
    #
    # "relay_endpoints": [
    #  "https://0xdeadbeefcafa@relay.example.com",
    #  "https://0xdeadbeefcafb@relay.example.com",
    #  "https://0xdeadbeefcafc@relay.example.com",
    #  "https://0xdeadbeefcafd@relay.example.com"
    # ]
    builder_network_params: null

    # Participant flag for keymanager api
    # This will open up http ports to your validator services!
    # Defaults null and then set to default global keymanager_enabled (false)
    keymanager_enabled: null

# Participants matrix creates a participant for each combination of EL, CL and VC clients
# Each EL/CL/VC item can provide the same parameters as a standard participant
participants_matrix: {}
  # el:
  #   - el_type: gzond
  #   - el_type: gzond
  # cl:
  #   - cl_type: qrysm
  #   - cl_type: qrysm
  # vc:
  #   - vc_type: qrysm
  #   - vc_type: qrysm


# Default configuration parameters for the network
network_params:
  # Network name, used to enable syncing of alternative networks
  # Defaults to "kurtosis"
  # You can sync any public network by setting this to the network name (e.g. "mainnet", "sepolia")
  # You can sync any devnet by setting this to the network name (e.g. "dencun-devnet-12")
  network: "kurtosis"

  # The network ID of the network.
  network_id: "3151908"

  # The address of the staking contract address
  deposit_contract_address: "Z4242424242424242424242424242424242424242"

  # Number of seconds per slot on the Beacon chain
  seconds_per_slot: 60

  # The number of validator keys that each CL validator node should get
  num_validator_keys_per_node: 64

  # This mnemonic will a) be used to create keystores for all the types of validators that we have and b) be used to generate a CL genesis.ssz that has the children
  # validator keys already preregistered as validators
  preregistered_validator_keys_mnemonic: "veto waiter rail aroma aunt chess fiend than sahara unwary punk dawn belong agent sane reefy loyal from judas clean paste rho madam poor pay convoy duty circa hybrid circus exempt splash"

  # The number of pre-registered validators for genesis. If 0 or not specified then the value will be calculated from the participants
  preregistered_validator_count: 0

  # How long you want the network to wait before starting up
  genesis_delay: 20

  # The gas limit of the network set at genesis
  genesis_gaslimit: 30000000

  # Max churn rate for the network
  # Defaults to 8
  max_per_epoch_activation_churn_limit: 8

  # Churn limit quotient for the network
  # Defaults to 65536
  churn_limit_quotient: 65536

  # Ejection balance
  # Defaults to 16ZND
  # 16000000000 gplanck
  ejection_balance: 16000000000

  # ETH1 follow distance
  # Defaults to 2048
  eth1_follow_distance: 2048

  # The number of epochs to wait validators to be able to withdraw
  # Defaults to 256 epochs ~27 hours
  min_validator_withdrawability_delay: 256

  # The period of the shard committee
  # Defaults to 256 epoch ~27 hours
  shard_committee_period: 256

  # Network sync base url for syncing public networks from a custom snapshot (mostly useful for shadowforks)
  # Defaults to "https://snapshots.ethpandaops.io/"
  # If you have a local snapshot, you can set this to the local url:
  # network_snapshot_url_base = "http://10.10.101.21:10000/snapshots/"
  # The snapshots are taken with https://github.com/ethpandaops/snapshotter
  network_sync_base_url: https://snapshots.theqrl.org/

  # The number of data column sidecar subnets used in the gossipsub protocol
  data_column_sidecar_subnet_count: 128
  # Number of DataColumn random samples a node queries per slot
  samples_per_slot: 8
  # Minimum number of subnets an honest node custodies and serves samples from
  custody_requirement: 4

  # Preset for the network
  # Default: "mainnet"
  # Options: "mainnet", "minimal"
  # "minimal" preset will spin up a network with minimal preset. This is useful for rapid testing and development.
  # 192 seconds to get to finalized epoch vs 1536 seconds with mainnet defaults
  # Please note that minimal preset requires alternative client images.
  # For an example of minimal preset, please refer to [minimal.yaml](.github/tests/minimal.yaml)
  preset: "mainnet"

  # Preloaded contracts for the chain
  additional_preloaded_contracts: {}
  # Example:
  # additional_preloaded_contracts: '{
  #  "Z123463a4B065722E99115D6c222f267d9cABb524":
  #   {
  #     balance: "1ZND",
  #     code: "0x1234",
  #     storage: {},
  #     nonce: 0,
  #     secretKey: "0x",
  #   }
  # }'

  # Repository override for devnet networks
  # Default: ethpandaops
  devnet_repo: ethpandaops

  # A number of prefunded accounts to be created
  # Defaults to no prefunded accounts
  # Example:
  # prefunded_accounts: '{"Z25941dC771bB64514Fc8abBce970307Fb9d477e9": {"balance": "10ZND"}}'
  # 10ZND to the account Z25941dC771bB64514Fc8abBce970307Fb9d477e9
  # To prefund multiple accounts, separate them with a comma
  #
  # prefunded_accounts: '{"Z25941dC771bB64514Fc8abBce970307Fb9d477e9": {"balance": "10ZND"}, "Z4107be99052d895e3ee461C685b042Aa975ab5c0": {"balance": "1ZND"}}'
  prefunded_accounts: {}

  # Maximum size of gossip messages in bytes
  # 10 * 2**20 (= 10485760, 10 MiB)
  # Defaults to 10485760 (10MB)
  gossip_max_size: 10485760



# Global parameters for the network

# By default includes
# - A transaction spammer is launched to fake transactions sent to the network
# - Forkmon for EL will be launched
# - A prometheus will be started, coupled with grafana
# - A beacon metrics gazer will be launched
# - A light beacon chain explorer will be launched
# - Default: []
additional_services:
  - assertoor
  - broadcaster
  - custom_flood
  - tx_spammer
  - el_forkmon
  - blockscout
  - beacon_metrics_gazer
  - dora
  - full_beaconchain_explorer
  - prometheus_grafana
  - dugtrio
  - blutgang
  - forky
  - apache
  - tracoor

# Configuration place for blockscout explorer - https://github.com/blockscout/blockscout
blockscout_params:
  # blockscout docker image to use
  # Defaults to blockscout/blockscout:latest
  image: "blockscout/blockscout:latest"
  # blockscout smart contract verifier image to use
  # Defaults to ghcr.io/blockscout/smart-contract-verifier:latest
  verif_image: "ghcr.io/blockscout/smart-contract-verifier:latest"
  # Frontend image
  # Defaults to ghcr.io/blockscout/frontend:latest
  frontend_image: "ghcr.io/blockscout/frontend:latest"

# Configuration place for dora the explorer - https://github.com/ethpandaops/dora
dora_params:
  # Dora docker image to use
  # Defaults to the latest image
  image: "ethpandaops/dora:latest"
  # A list of optional extra env_vars the dora container should spin up with
  env: {}

# Configuration place for prometheus
prometheus_params:
  storage_tsdb_retention_time: "1d"
  storage_tsdb_retention_size: "512MB"
  # Resource management for prometheus container
  # CPU is milicores
  # RAM is in MB
  min_cpu: 10
  max_cpu: 1000
  min_mem: 128
  max_mem: 2048
  # Prometheus docker image to use
  # Defaults to the latest image
  image: "prom/prometheus:latest"

# Configuration place for grafana
grafana_params:
  # A list of locators for grafana dashboards to be loaded be the grafana service
  additional_dashboards: []
  # Resource management for grafana container
  # CPU is milicores
  # RAM is in MB
  min_cpu: 10
  max_cpu: 1000
  min_mem: 128
  max_mem: 2048
  # Grafana docker image to use
  # Defaults to the latest image
  image: "grafana/grafana:latest"

# Configuration place for the assertoor testing tool - https://github.com/ethpandaops/assertoor
assertoor_params:
  # Assertoor docker image to use
  # Defaults to the latest image
  image: "ethpandaops/assertoor:latest"

  # Check chain stability
  # This check monitors the chain and succeeds if:
  # - all clients are synced
  # - chain is finalizing for min. 2 epochs
  # - >= 98% correct target votes
  # - >= 80% correct head votes
  # - no reorgs with distance > 2 blocks
  # - no more than 2 reorgs per epoch
  run_stability_check: false

  # Check block propöosals
  # This check monitors the chain and succeeds if:
  # - all client pairs have proposed a block
  run_block_proposal_check: false

  # Run normal transaction test
  # This test generates random EOA transactions and checks inclusion with/from all client pairs
  # This test checks for:
  # - block proposals with transactions from all client pairs
  # - transaction inclusion when submitting via each client pair
  # test is done twice, first with legacy (type 0) transactions, then with dynfee (type 2) transactions
  run_transaction_test: false

  # Run all-opcodes transaction test
  # This test generates a transaction that triggers all EVM OPCODES once
  # This test checks for:
  # - all-opcodes transaction success
  run_opcodes_transaction_test: false

  # Run validator lifecycle test (~48h to complete)
  # This test requires exactly 500 active validator keys.
  # The test will cause a temporary chain unfinality when running.
  # This test checks:
  # - Deposit inclusion with/from all client pairs
  # - BLS Change inclusion with/from all client pairs
  # - Voluntary Exit inclusion with/from all client pairs
  # - Attester Slashing inclusion with/from all client pairs
  # - Proposer Slashing inclusion with/from all client pairs
  # all checks are done during finality & unfinality
  run_lifecycle_test: false

  # Run additional tests from external test definitions
  # Entries may be simple strings (link to the test file) or dictionaries with more flexibility
  # eg:
  #   - https://raw.githubusercontent.com/ethpandaops/assertoor/master/example/tests/block-proposal-check.yaml
  #   - file: "https://raw.githubusercontent.com/ethpandaops/assertoor/master/example/tests/block-proposal-check.yaml"
  #     config:
  #       someCustomTestConfig: "some value"
  tests: []


# If set, the package will block until a finalized epoch has occurred.
wait_for_finalization: false

# The global log level that all clients should log at
# Valid values are "error", "warn", "info", "debug", and "trace"
# This value will be overridden by participant-specific values
global_log_level: "info"

# Snooper global flag for all participants
# Snooper can be enabled with the `snooper_enabled` flag per client or globally
# Snooper dumps all JSON-RPC requests and responses including BeaconAPI, EngineAPI and ExecutionAPI.
# Default to false
snooper_enabled: false

# Enables Ethereum Metrics Exporter for all participants
# Defaults to false
ethereum_metrics_exporter_enabled: false

# Parallelizes keystore generation so that each node has keystores being generated in their own container
# This will result in a large number of containers being spun up than normal. We advise users to only enable this on a sufficiently large machine or in the cloud as it can be resource consuming on a single machine.
parallel_keystore_generation: false

# Disable peer scoring to prevent nodes impacted by faults from being permanently ejected from the network
# Default to false
disable_peer_scoring: false

# Whether the environment should be persistent; this is WIP and is slowly being rolled out accross services
# Note this requires Kurtosis greater than 0.85.49 to work
# Defaults to false
persistent: false

# Docker cache url enables all docker images to be pulled through a custom docker registry
# Disabled by default
# Defaults to empty cache url
# Images pulled from dockerhub will be prefixed with "/dh/" by default (docker.io)
# Images pulled from github registry will be prefixed with "/gh/" by default (ghcr.io)
# Images pulled from google registory will be prefixed with "/gcr/" by default (gcr.io)
# If you want to use a local image in combination with the cache, do not put "/" in your local image name
docker_cache_params:
  enabled: false
  url: ""
  dockerhub_prefix: "/dh/"
  github_prefix: "/gh/"
  google_prefix: "/gcr/"

# Supports three valeus
# Default: "null" - no mev boost, mev builder, mev flood or relays are spun up
# "mock" - mock-builder & mev-boost are spun up
# "flashbots" - mev-boost, relays, flooder and builder are all spun up, powered by [flashbots](https://github.com/flashbots)
# "mev-rs" - mev-boost, relays and builder are all spun up, powered by [mev-rs](https://github.com/ralexstokes/mev-rs/)
# "commit-boost" - mev-boost, relays and builder are all spun up, powered by [commit-boost](https://github.com/Commit-Boost/commit-boost-client)
# We have seen instances of multibuilder instances failing to start mev-relay-api with non zero epochs
mev_type: null

# Parameters if MEV is used
mev_params:
  # The image to use for MEV boost relay
  mev_relay_image: flashbots/mev-boost-relay
  # The image to use for the builder
  mev_builder_image: ethpandaops/flashbots-builder:main
  # The image to use for the CL builder
  # mev_builder_cl_image: sigp/lighthouse:latest
  # The image to use for mev-boost
  mev_boost_image: flashbots/mev-boost
  # Parameters for MEV Boost. This overrides all arguments of the mev-boost container
  mev_boost_args: []
  # Extra parameters to send to the API
  mev_relay_api_extra_args: []
  # Extra parameters to send to the housekeeper
  mev_relay_housekeeper_extra_args: []
  # Extra parameters to send to the website
  mev_relay_website_extra_args: []
  # Extra parameters to send to the builder
  mev_builder_extra_args: []
  # Prometheus additional configuration for the mev builder participant.
  # Execution, beacon and validator client targets on prometheus will include this configuration.
  mev_builder_prometheus_config:
    # Scrape interval to be used. Default to 15 seconds
    scrape_interval: 15s
    # Additional labels to be added. Default to empty
    labels: {}
  # Image to use for mev-flood
  mev_flood_image: flashbots/mev-flood
  # Extra parameters to send to mev-flood
  mev_flood_extra_args: []
  # Number of seconds between bundles for mev-flood
  mev_flood_seconds_per_bundle: 15
  # Optional parameters to send to the custom_flood script that sends reliable payloads
  custom_flood_params:
    interval_between_transactions: 1

# Enables Xatu Sentry for all participants
# Defaults to false
xatu_sentry_enabled: false

# Xatu Sentry params
xatu_sentry_params:
  # The image to use for Xatu Sentry
  xatu_sentry_image: ethpandaops/xatu:latest
  # GRPC Endpoint of Xatu Server to send events to
  xatu_server_addr: localhost:8080
  # Enables TLS to Xatu Server
  xatu_server_tls: false
  # Headers to add on to Xatu Server requests
  xatu_server_headers: {}
  # Beacon event stream topics to subscribe to
  beacon_subscriptions:
    - attestation
    - block
    - chain_reorg
    - finalized_checkpoint
    - head
    - voluntary_exit
    - contribution_and_proof

# Apache params
# Apache public port to port forward to local machine
# Default to port None, only set if apache additional service is activated
apache_port: null

# Global tolerations that will be passed to all containers (unless overridden by a more specific toleration)
# Only works with Kubernetes
# Example: tolerations:
# - key: "key"
#   operator: "Equal"
#   value: "value"
#   effect: "NoSchedule"
#   toleration_seconds: 3600
# Defaults to empty
global_tolerations: []

# Global node selector that will be passed to all containers (unless overridden by a more specific node selector)
# Only works with Kubernetes
# Example: global_node_selectors: { "disktype": "ssd" }
# Defaults to empty
global_node_selectors: {}

# Global parameters for keymanager api
# This will open up http ports to your validator services!
# Defaults to false
keymanager_enabled: false

# Global flag to enable checkpoint sync across the network
checkpoint_sync_enabled: false

# Global flag to set checkpoint sync url
checkpoint_sync_url: ""

# Transaction spammer params
tx_spammer_params:
  # The image to use for tx spammer
  image: theqrl/zond-tx-spammer:latest
  # The scenario to use (see https://github.com/theQRL/zond-tx-spammer)
  # Valid scenarios are:
  #  eoatx, zrctx, deploytx, depoy-destruct, gasburnertx
  # Defaults to eoatx
  scenario: eoatx
  # Throughput of tx spammer
  # Defaults to 1000
  throughput: 1000
  # Max pending transactions for tx spammer
  # Defaults to 1000
  max_pending: 1000
  # Max wallets for tx spammer
  # Defaults to 500
  max_wallets: 500
  # Extra parameters to send to tx spammer
  # Defaults to empty
  tx_spammer_extra_args: []

# Zond genesis generator params
zond_genesis_generator_params:
  # The image to use for zond genesis generator
  image: qrledger/qrysm:zond-genesis-generator-latest

# Global parameter to set the exit ip address of services and public ports
port_publisher:
  # if you have a service that you want to expose on a specific interfact; set that IP here
  # if you set it to auto it gets the public ip from ident.me and sets it
  # Defaults to constants.PRIVATE_IP_ADDRESS_PLACEHOLDER
  # The default value just means its the IP address of the container in which the service is running
  nat_exit_ip: KURTOSIS_IP_ADDR_PLACEHOLDER
  # Execution Layer public port exposed to your local machine
  # Disabled by default
  # Public port start defaults to 32000
  # You can't run multiple enclaves on the same port settings
  el:
    enabled: false
    public_port_start: 32000
  # Consensus Layer public port exposed to your local machine
  # Disabled by default
  # Public port start defaults to 33000
  # You can't run multiple enclaves on the same port settings
  cl:
    enabled: false
    public_port_start: 33000
  # Validator client public port exposed to your local machine
  # Disabled by default
  # Public port start defaults to 34000
  # You can't run multiple enclaves on the same port settings
  vc:
    enabled: false
    public_port_start: 34000
  # remote signer public port exposed to your local machine
  # Disabled by default
  # Public port start defaults to 35000
  # You can't run multiple enclaves on the same port settings
  remote_signer:
    enabled: false
    public_port_start: 35000
  # Additional services public port exposed to your local machine
  # Disabled by default
  # Public port start defaults to 36000
  # You can't run multiple enclaves on the same port settings
  additional_services:
    enabled: false
    public_port_start: 36000
```

#### Example configurations

<details>
    <summary>A 5-node Zond network.</summary>

```yaml
participants:
  - el_type: gzond
    cl_type: qrysm
    count: 5
```

</details>

<details>
    <summary>A 2-node gzond/qrysm network with optional services (Grafana, Prometheus, transaction-spammer, EngineAPI snooper, and a testnet verifier)</summary>

```yaml
participants:
  - el_type: gzond
    cl_type: qrysm
    count: 2
snooper_enabled: true
additional_services:
  - prometheus_grafana
ethereum_metrics_exporter_enabled: true
```

</details>

## Custom labels for Docker and Kubernetes

There are 4 custom labels that can be used to identify the nodes in the network. These labels are used to identify the nodes in the network and can be used to run chaos tests on specific nodes. An example for these labels are as follows:

Execution Layer (EL) nodes:

```sh
  "com.kurtosistech.custom.zond-package-client": "gzond",
  "com.kurtosistech.custom.zond-package-client-image": "theqrl-gzond-latest",
  "com.kurtosistech.custom.zond-package-client-type": "execution",
  "com.kurtosistech.custom.zond-package-connected-client": "qrysm",
```

Consensus Layer (CL) nodes - Beacon:

```sh
  "com.kurtosistech.custom.zond-package-client": "qrysm",
  "com.kurtosistech.custom.zond-package-client-image": "theqrl-qrysm-beacon-chain-latest",
  "com.kurtosistech.custom.zond-package-client-type": "beacon",
  "com.kurtosistech.custom.zond-package-connected-client": "gzond",
```

Consensus Layer (CL) nodes - Validator:

```sh
  "com.kurtosistech.custom.zond-package-client": "qrysm",
  "com.kurtosistech.custom.zond-package-client-image": "theqrl-qrysm-validator-latest",
  "com.kurtosistech.custom.zond-package-client-type": "validator",
  "com.kurtosistech.custom.zond-package-connected-client": "gzond",
```

`zond-package-client` describes which client is running on the node.
`zond-package-client-image` describes the image that is used for the client.
`zond-package-client-type` describes the type of client that is running on the node (`execution`,`beacon` or `validator`).
`zond-package-connected-client` describes the CL/EL client that is connected to the EL/CL client.

## Proposer Builder Separation (PBS) emulation

To spin up the network of Zond nodes with an external block building network (using Flashbot's `mev-boost` protocol), simply use:

```
kurtosis run github.com/theQRL/zond-package '{"mev_type": "full"}'
```

Starting your network up with `"mev_type": "full"` will instantiate and connect the following infrastructure to your network:

1. `Flashbot's block builder & CL validator + beacon` - A modified Gzond client that builds blocks. The CL validator and beacon clients are lighthouse clients configured to receive payloads from the relay.
2. `mev-relay-api` - Services that provide APIs for (a) proposers, (b) block builders, (c) data
3. `mev-relay-website` - A website to monitor payloads that have been delivered
4. `mev-relay-housekeeper` - Updates known validators, proposer duties, and more in the background. Only a single instance of this should run.
5. `mev-boost` - open-source middleware instantiated for each EL/Cl pair in the network, including the builder
6. `mev-flood` - Deploys UniV2 smart contracts, provisions liquidity on UniV2 pairs, & sends a constant stream of UniV2 swap transactions to the network's public mempool.

<details>
    <summary>Caveats when using "mev_type": "full"</summary>

* Validators (64 per node by default, so 128 in the example in this guide) will get registered with the relay automatically after the 1st epoch. This registration process is simply a configuration addition to the mev-boost config - which Kurtosis will automatically take care of as part of the set up. This means that the mev-relay infrastructure only becomes aware of the existence of the validators after the 1st epoch.
* After the 3rd epoch, the mev-relay service will begin to receive execution payloads (eth_sendPayload, which does not contain transaction content) from the mev-builder service (or mock-builder in mock-mev mode).
* Validators will start to receive validated execution payload headers from the mev-relay service (via mev-boost) after the 4th epoch. The validator selects the most valuable header, signs the payload, and returns the signed header to the relay - effectively proposing the payload of transactions to be included in the soon-to-be-proposed block. Once the relay verifies the block proposer's signature, the relay will respond with the full execution payload body (incl. the transaction contents) for the validator to use when proposing a SignedBeaconBlock to the network.

</details>

This package also supports a `"mev_type": "mock"` mode that will only bring up:

1. `mock-builder` - a server that listens for builder API directives and responds with payloads built using an execution client
1. `mev-boost` - for every EL/CL pair launched

For more details, including a guide and architecture of the `mev-boost` infrastructure, go [here](https://docs.kurtosis.com/how-to-full-mev-with-ethereum-package/).

## Pre-funded accounts at Genesis

This package comes with [21 prefunded keys for testing](https://github.com/theQRL/zond-package/blob/main/src/prelaunch_data_generator/genesis_constants/genesis_constants.star).

Here's a table of where the keys are used

| Account Index | Component Used In   | Private Key Used | Public Key Used | Comment                     |
|---------------|---------------------|------------------|-----------------|-----------------------------|
| 0             | Builder             | ✅                |                 | As coinbase                |
| 0             | mev_custom_flood    |                   | ✅              | As the receiver of balance |
| 6             | mev_flood           | ✅                |                 | As the contract owner      |
| 7             | mev_flood           | ✅                |                 | As the user_key            |
| 8             | assertoor           | ✅                | ✅              | As the funding for tests   |
| 11            | mev_custom_flood    | ✅                |                 | As the sender of balance   |
| 12            | l2_contracts        | ✅                |                 | Contract deployer address  |
| 13            | tx_spammer             | ✅                |                 | Spams transactions         |

## Developing On This Package

First, install prerequisites:

1. [Install Kurtosis itself][kurtosis-cli-installation]

Then, run the dev loop:

1. Make your code changes
1. Rebuild and re-run the package by running the following from the root of the repo:

   ```bash
   kurtosis run . "{}"
   ```

   NOTE 1: You can change the value of the second positional argument flag to pass in extra configuration to the package per the "Configuration" section above!
   NOTE 2: The second positional argument accepts JSON.

To get detailed information about the structure of the package, visit [the architecture docs](./docs/architecture.md).

When you're happy with your changes:

1. Create a PR
1. Add one of the maintainers of the repo as a "Review Request".
1. Once everything works, merge!

<!------------------------ Only links below here -------------------------------->

[docker-installation]: https://docs.docker.com/get-docker/
[kurtosis-cli-installation]: https://docs.kurtosis.com/install
[kurtosis-repo]: https://github.com/kurtosis-tech/kurtosis
[enclave]: https://docs.kurtosis.com/advanced-concepts/enclaves/
[package-reference]: https://docs.kurtosis.com/advanced-concepts/packages
