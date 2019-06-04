---
title: ScalableBFT on Azure
layout: default
---

## Table of Contents

1.  [Ansible and Azure](#ansible-and-azure)
    - [Azure Quick Start](#azure-quick-start)
    - [Ansible Playbooks](#ansible-playbooks)
    - [Launching the Demo](#launching-the-demo)
    - [Virtual Machine Requirements](#vm-requirements)
    - [Network Security Group (NSG) Requirements](#network-security-group-requirements)
    - [Further Reading](#further-reading)
2.  [Kadena Blockchain Documentation](#kadena-blockchain-documentation)
    - [Kadena Demo Quick Start](#kadena-demo-quick-start)
    - [Kadena server and client binaries](#kadena-server-and-client-binaries)
    - [General Considerations](#general-considerations)
    - [Configuration](#configuration)
    - [Interacting With a Running Cluster](#interacting-with-a-running-cluster)
      - [Sample Usage: Payments demo](#sample-usage-running-the-payments-demo-non-private-and-testing-batch-performance)
      - [Sample Usage: Running Pact TodoMVC](#sample-usage-running-pact-todomvc)
    - [Configuration File Documentation](#configuration-file-documentation)

---

# Ansible and Azure

---

## Azure Quick Start

1.  Create a resource Group with a name `kadenaResourceGroup`.
2.  Create a Storage Account with the name `kadenaStorage`.
3.  Spin up a VM with Kadena's ScalableBFT Image with the following requirements. (See [VM Requirements](#vm-requirements)). This will serve as the Ansible monitor VM. <https://docs.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json>
4.  Ensure that the key pair(s) of the monitor and Kadena node VMs are not publicly
    viewable: `chmod 400 /path/to/keypair`. Otherwise, SSH and any service that rely on it (i.e. Ansible)
    will not work. See <https://docs.microsoft.com/en-us/azure/virtual-machines/linux/mac-create-ssh-keys> for setting up SSH keys.
5.  Add the key pair(s) of the monitor and Kadena node VMs to the `ssh-agent`:
    `ssh-add /path/to/keypair`
6.  SSH into the monitor instance using ssh-agent forwarding: `ssh -A <admin-user>@<vm-public-ip>`. The default `<admin-user>` is `ubuntu`.
    This facilitates the Ansible monitor's task of managing different VMs by having access to their key pair.
7.  Once logged into the monitor VM, locate the directory `kadena`.
8.  Grant Ansible the ability to make API calls to Azure on your behalf. You can simply log in with the following command:
    ```
    az login
    ```
9.  Create Azure Active Directory service principal and save credentials file. This is to use Azure's Dynamic Inventory with Ansible. See <https://docs.microsoft.com/en-us/azure/ansible/ansible-manage-azure-dynamic-inventories>. User is required to have a directory role of Application Administrator or higher on Azure Active Directory to perform this. Run the following command.

    ```
    az ad sp create-for-rbac --query '{"client_id": appId, "secret": password, "tenant": tenant}'
    az account show --query '{ subscription_id: id }'
    ```

10. Create credentials file in ~/.azure/credentials path with the following format:

    ```
    [default]
    subscription_id=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    client_id=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    secret=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    tenant=xxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
    ```

11. Edit the `user_vars.yml` and fill in required parameters. There are 4 parameters: `node_image`, `node_count`, `node_region` and `node_size`. `node_image` is the image name of Kadena Scalable Permissioned Blockchain. `node_count` refers to the number of nodes you'd like to spin up. `node_region` is the region of the node VMs, and needs to be the same as the Ansible monitor VM. `node_size` can be decided based on the scale of project. For exploration, we recommend Standard B1s. See below as an example:

        ```
        node_image: Kadena Community Edition, v1.1.4.0
        node_count: 4
        node_region: East US
        node_size: Standard_B1s

        ```

    You are now ready to start using the Ansible playbooks!

## Ansible Playbooks

Playbooks are composed of `plays`, which are then composed of `tasks`. Plays
and tasks are executed sequentially. Ansible playbooks are in YAML format and can be executed as follows:

```
ansible-playbook /path/to/playbook.yml
```

The `ansible/` directory contains the following playbooks:

### `start_instances.yml`

This playbook launches VM's that have the necessary files and directories to run the Kadena Server executable.

### `configure_instances.yml`

This playbook creates a file containing all of their public IP addresses and the default (i.e. SQLite backend) node configurations for each. This list of IP addresses will be located in `ansible/ipAddr.yml`.

### `stop_instances.yml`

This playbook terminates all running Kadena Server VMs and related resources.

### `run_servers.yml`

This playbooks runs the Kadena Server executable. If the servers were already running, it terminates them as well as cleans up their sqlite and log files before launching the server again.
This playbook also updates the server's configuration if it has changed in the specified configuration directory (`conf/`) on the monitor instance.
The Kadena Servers will run for 24 hours after starting. To change this, edit the **Start Kadena Servers** async section in this playbook.

### `get_server_logs.yml`

This playbook retrieves all of the Kadena Servers' logs and sqlite files, deleting all previous retrieved logs.
It stores the logs in `ansible/logs/`.

NB: To change distributed nodes' configuration, run

```
<kadena-directory>$ ./bin/<OS-name>/genconfs --distributed ansible/ipAddr.yml
```

Provide the desired settings when prompted. For more information, refer to the
["Automated configuration generation: `genconfs`"](#configuration) section.

## Launching the Demo

Once you've completed the [Azure Quick Start](#azure-quick-start) instructions, execute the following commands to boot up the ScalableBFT servers and start the kadena-demo:

```
$ cd kadena/
$ ansible-playbook ansible/start_instances.yml
$ ansible-playbook ansible/configure_instances.yml
$ tmux
$ ./ansible/start_demo.sh
```

Press Enter when prompted by `bin/ubuntu-16.04/kadenaclient.sh`.
This will start the Kadena Client and allow you to start interacting with the private blockchain (see the [`kadenaclient` binary explanation](#kadena-server-and-client-binaries) for more details).

For a list of supported interactions, refer to the ["Sample Usage: `[payments|monitor|todomvc]`"](#sample-usage-running-the-payments-demo-non-private-and-testing-batch-performance) section.

To exit the Kadena Client, type `exit`. To kill the tmux sessions, type `tmux kill-session`.

The demo script assumes the following directory structure:

```
$ tree <kadena-directory>
<kadena-directory>
├── ansible
    ├── user_vars.yml
    ├── get_server_logs.yml
    ├── ipAddr.yml		(produced by start_instances.yml)
    ├── run_servers.yml
    ├── start_demo.sh
    ├── start_instances.yml
    ├── configure_instances.yml
    └── stop_instances.yml
└── bin
    └── <OS-name>
        └── <all kadena executables>
```

## VM Requirements

The Ansible monitor VM should be configured as follows:

1.  The monitor VM needs to be created in a resource group with a name `kadenaResourceGroup`.
2.  The resource group, `kadenaResourceGroup` must have a storage account named `kadenastorage`
3.  The monitor VM is built on the image, `Kadena Community Edition, v1.1.4.0`
4.  The monitor VM uses admin name of `ubuntu` and SSH authentication with public key.
5.  The size of monitor VM can be decided by user, but we recommend using a size greater than Standard B2s.
6.  The monitor VM needs to allow Port 22 to manage Kadena Node VMs. See [Network Security Group (NSG) Requirements](#network-security-group-requirements) for more.

## Network Security Group Requirements

Ansible needs to be able to communicate with the Azure VMs it manages, and the Kadena Servers need to communicate
with each other. Therefore, the security group (firewall) assigned to the Kadena Node VMs
should allow for the following:

1.  The Ansible monitor VM (the one running the playbooks) should be able to ssh into
    all of the Kadena Node VMs it will manage. Allow Port 22.
2.  The Kadena Node VMs should be able to communicate via TCP 10000 port.
3.  The Kadena Node VMs should be able to receive HTTP connections via the 8000 port from
    any instance running the Kadena Client.

The Ansible Playbooks will create a Security Group with the requirements for the Kadena Nodes. However, user must specify the Security Group requirement for the Ansible Monitor VM.

## Further Reading

The official guide on how to use Ansible's Azure modules:
<https://docs.ansible.com/ansible/latest/scenario_guides/guide_azure.html?highlight=dynamic%20inventory%20azure>

---

# Kadena Blockchain Documentation

---

Kadena Version: 1.1.x

# Change Log

- Version 1.1.3.0

  - Added MySQL adapter to pact-persist

- Kadena 1.1.2 (June 12th, 2017)
  - Integrated privacy mechanism (on-chain Noise protocol based private channels)
  - Added `par-batch` to REPL
  - Fixed issues with new command forwarding and batching mechanics
  - `local` queries now execute immediately, skipping the write behind's queue
  - Nodes are automatically configured to run on `0.0.0.0`
  - `genconfs` inputs now are reflected in the configuration files
  - Fixed `genconfs --distributed`

# Getting Started

### Dependencies

Required:

- `zeromq >= v4.1.4`
  - OSX: `brew install zeromq`
  - Ubuntu: the `apt-get` version of zeromq v4 is incorrect, you need to build it from source. See the Ubuntu docker file for more information.
- `libz`: usually this comes pre-installed
- `unixodbc == v3.*`
  - OSX: `brew install unixodbc`
  - Ubuntu: refer to docker file
- `MySQL`
- Ubuntu Only:
  - `libsodium`: refer to docker file

Optional:

- `pact == v2.4`: See <https://github.com/kadena-io/pact#installing-pact-with-binary-distributions>.
- `rlwrap`: only used in `kadenaclient.sh` to enable Up-Arrow style history. Feel free to remove it from the script if you'd like to avoid installing it.
- `tmux == v2.0`: only used for the local demo script `<kadena-directory>/bin/<OS-name>/start.sh`.
  A very specific version of tmux is required because features were entirely removed in later version that preclude the script from working.

NB: The docker and script files for installing the Kadena dependencies can be found in `<kadena-directory>/setup`.

### Kadena Demo Quick Start

Quickly launch a local V, see "Sample Usage: `[payments|monitor|todomvc]`" for interactions supported.

#### OSX

```
<kadena-directory>$ tmux
<kadena-directory>$ ./bin/osx/start.sh
```

#### Ubuntu 16.04

```
<kadena-directory>$ tmux
<kadena-directory>$ ./bin/ubuntu-16.04/start.sh
```

# Kadena server and client binaries

### `kadenaserver`

Launch a consensus server node.
On startup, `kadenaserver` will open connections on three ports as specified in the configuration file:
`<apiPort>`, `<nodeId.port>`, `<nodeId.port> + 5000`.
Generally, these ports will default to `8000`, `10000`, and `15000` (see `genconfs` for details).

For information regarding the configuration yaml generally, see the "Configuration Files" section.

```
kadenaserver (-c|--config) [-d|--disablePersistence]

Options:
  -c,--config               [Required] path to server yaml configuration file
  -d,--disablePersistence   [Optional] disable usage of SQLite for on-disk persistence
                                       (higher performance)
```

NB: there is a `zeromq` bug that may cause `kadenaserver` to fail to launch (segfault) ~1% of the time. Once running this is not an issue. If you encounter this problem, please relaunch.

### `kadenaclient` & `kadenaclient.sh`

Launch a client to the consensus cluster.
The client allows for command-line level interaction with the server's REST API in a familiar (REPL-style) format.
The associated script incorporates rlwrap to enable Up-Arrow style history, but is not required.

```
kadenaclient (-c|--config)

Options:
  -c,--config               [Required] path to client yaml configuration file

Sample Usage (found in kadenaclient.sh):
  rlwrap -A bin/kadenaclient -c "conf/$(ls conf | grep -m 1 client)"
```

# General Considerations

### Elections Triggered by a High Load

When running a `local` demo, resource contention can trigger election under a high load when certain configurations are present.
For example, `batch 40000` when the replication per heartbeat is set to +10k will likely trigger an election event.
This is caused entirely by a lack of available CPU being present; one of the nodes will hog the CPU, causing the other nodes to trigger an election.
This should not occur in a distributed setting nor is it a problem overall as the automated handling of availability events are one of the features central to any distributed system.

If you would like to do large scale `batch` tests in a local setting, use `genconfs` to create new configuration files where the replication limit is ~8k.

### Load Testing with Many Clients

If you'll be testing with many (100s to +10k) simultaneous clients please be sure to provision extra CPU's.
In a production setting, we'd expect:

- To use a separate server to collect inbound transactions from the multitude of clients and lump them into a single batch/pipe them over a websocket to the cluster itself so as to avoid needless CPU utilization.
- For clients to connect to different nodes (e.g. all Firm A clients connect to Firm A's nodes, B's to B's, etc.), allowing the nodes themselves to batch/forward commands.

The ability to do either of these is a feature of Kadena -- because commands must have a unique hash and are either (a) signed or (b) fully encrypted, they can be redirected without degrading the security model.

### Replay From Disk

On startup but before `kadenaserver` goes online, it will replay from origin each persisted transaction.
If you would like to start fresh, you will need to delete the SQLite DB's prior to startup.

### Core Count

By default `kadenaserver` is configured use as many cores as are available.
In a distributed setting, this is generally a good default; in a local setting, it is not.
Because each node needs 8 cores to function at peak performance, running multiple nodes locally when clusterSize \* 8 > available cores can cause the nodes to obstruct each other (and thereby trigger an election).

To avoid this, the demo's `start.sh` script restricts each node to 4 cores via the `+RTS -N4 -RTS` flags.
You may use these, or any other flags found in [GHC RTS Options](https://downloads.haskell.org/~ghc/7.10.3/docs/html/users_guide/runtime-control.html#rts-opts-compile-time) to configure a given node should you wish to.

- To set cores to a specific amount, add `+RTS -N[core count] -RTS`.
- To allow kadena to use all available cores, do not specify core count (remove the `+RTS -N[count] -RTS` section, or just the `-N[cores]` if using other runtime settings.)

### Beta Limitations

Beta License instances of Kadena are limited as follows:

- The maximum cluster size is limited to 16
- The maximum number of total committed transactions is limited to 200,000
- The binaries will only run for 90 days
- Consensus level membership & key rotation changes are not available

For a version without any/all of these restrictions, please contact us at [info@kadena.io](mailto:info@kadena.io).

### Azure Marketplace Limitations

The Azure Marketplace listing of Kadena is limited as follows:

- The maximum cluster size is limited to 4

For a version without any/all of these restrictions, please contact us at [info@kadena.io](mailto:info@kadena.io).

# Configuration

### Automated configuration generation: `genconfs`

`kadenaserver` and `kadenaclient` each require a configuration file.
`genconfs` is designed to assist you in quickly (re)generating these files.

It operates in 2 modes:

- `./genconfs` will create a set of config files for a localhost test of kadena.
  It will ask you how many cluster and client nodes you'd like.
- `./genconfs --distributed <cluster-ips file>` will create a set of config files using the IP addresses specified in the files.

In either mode `genconfs` will interactively prompt for settings with recommendations.

```
$ ./genconfs
When a recommended setting is available, press Enter to use it
[FilePath] Which directory should hold the log files and SQLite DB's? (recommended: ./log)

Set to recommended value: "./log"
[FilePath] Where should `genconfs` write the configuration files? (recommended: ./conf)
... etc ...
```

In distributed mode:

```
$ cat server-ips
54.73.153.1
54.73.153.2
54.73.153.3
54.73.153.4
$ ./genconfs --distributed ./server-ips
When a recommended setting is available, press Enter to use it
[FilePath] Which directory should hold the log files and SQLite DB's? (recommended: ./log)
... etc ...
```

For details about what each of these configuration choices do, please refer to the "Configuration Files" section.

# Interacting With a Running Cluster

Interaction with the cluster is performed via the Kadena REST API, exposed by each running node.
The endpoints of interest here support the [Pact REST API](http://pact-language.readthedocs.io/en/latest/pact-reference.html#rest-api) for executing transactional and local commands on the cluster.

### The `kadenaclient` tool

Kadena ships with `kadenaclient`, which is a command-line tool for interacting with the cluster via the REST API.
It is an interactive program or "REPL", similar to the command-line itself. It supports command history,
such that recently-issued commands are accessible via the up- and down-arrow keys, and the history can be
searched with Control-R.

#### Getting help

The `help` command documents all available commands.

```
$ ./kadenaclient.sh
node3> help
Command Help:
sleep [MILLIS]
    Pause for 5 sec or MILLIS
cmd [COMMAND]
    Show/set current batch command
data [JSON]
    Show/set current JSON data payload
load YAMLFILE [MODE]
    Load and submit yaml file with optional mode (transactional|local), defaults to transactional
...
```

#### `server` command

Issue `server` to list all nodes known to the client, and `server NODE` to point the client at
the REST API for NODE.

```
node0> server
Current server: node0
Servers:
node0: 127.0.0.1:8000 ["Alice", sending: True]
node1: 127.0.0.1:8001 ["Bob", sending: True]
node2: 127.0.0.1:8002 ["Carol", sending: True]
node3: 127.0.0.1:8003 ["Dinesh", sending: True]
node0> server node1
node1>
```

#### `load` command

`load` is designed to assist with initializing a new environment on a running blockchain,
accepting a yaml file to instruct how the code and data is loaded.

The "demo" smart contract explores this:

```
$ tree demo
demo
├── demo.pact
├── demo.repl
└── demo.yaml

$ cat demo/demo.yaml
data: |-
  demo-admin-keyset:
    "keys": ["demoadmin"]
    "pred": ">"
codeFile: demo.pact
keyPairs:
  - public: 06c9c56daa8a068e1f19f5578cdf1797b047252e1ef0eb4a1809aa3c2226f61e
    secret: 7ce4bae38fccfe33b6344b8c260bffa21df085cf033b3dc99b4781b550e1e922
batchCmd: |-
  (demo.transfer "Acct1" "Acct2" 1.00)
```

#### Sample Usage: running the payments demo (non-private) and testing batch performance

Launch the client and (optionally) target the leader node (in this case `node0`).
The only reason to target the leader is to forgo the forwarding of new transactions to the leader.
The cluster will handle the forwarding automatically.

```
./kadenaclient.sh
node3> server node0
```

Initialize the chain with the `payments` smart contract, and create the global/non-private accounts.
Note that the load process has an optional feature to set the batch command (see docs for `cmd` in help).
The demo.yaml sets the batch command to
transfer `1.00` between the demo accounts.

```
node0> load demo/demo.yaml
status: success
data: Write succeeded

Setting batch command to: (demo.transfer "Acct1" "Acct2" 1.00)

node0> exec (demo.create-global-accounts)
account      | amount       | balance      | data
---------------------------------------------------------
"Acct1"      | "1000000.0"  | "1000000.0"  | "Admin account funding"
"Acct2"      | "0.0"        | "0.0"        | "Created account"
```

Execute a single dollar transfer and check the balances again with `read-all`. `exec` sends
a command to execute transactionally on the blockchain; `local` queries the local node (here "node0")
to prevent a needless transaction for a query.

```
node0> exec (demo.transfer "Acct1" "Acct2" 1.00)
status: success
data: Write succeeded

node0> local (demo.read-all)
account      | amount       | balance      | data
---------------------------------------------------------
"Acct1"      | "-1.00"      | "999999.00"  | {"transfer-to":"Acct2"}
"Acct2"      | "1.00"       | "1.00"       | {"transfer-from":"Acct1"}
```

Verify that `cmd` is properly setup, and perform a `batch` test.
`batch N` will create N identical transactions, using the command specified in `cmd` for each, and then send them to the cluster via the server specified by `server` (in this case to `node0`).

Once sent, the client will then `listen` for the final transaction, collect and show its timing metrics, and print out the throughput seen in the test (i.e. `"Finished Commit" / N`).
The "First Seen" time is the moment when the targeted server first saw the **batch** of transactions and the "Finished Commit" time delta fully captures the time it took for the replication, consensus, cryptography, and execution of the final transaction (meaning that all previous transactions needed to first be fully executed.)

Some of the metrics may be of interest to you:

```
node0> cmd
(demo.transfer "Acct1" "Acct2" 1.00)
node0> batch 4000
Preparing 4000 messages ...
Sent, retrieving responses
Polling for RequestKey: "b768a85c6e1a06d4cfd9760dd981b675dcd9dc97ee8d7abc756246107f2ea03edd80e10e5168b41ee96a17b098ea3285a0f5ca9c61c4d974a7832e01f354dcf9"
First Seen:          2017-03-19 05:43:14.571 UTC
Hit Turbine:        +24.03 milli(s)
Entered Con Serv:   +39.83 milli(s)
Finished Con Serv:  +52.41 milli(s)
Came to Consensus:  +113.00 milli(s)
Sent to Commit:     +113.94 milli(s)
Started PreProc:    +690.55 milli(s)
Finished PreProc:   +690.66 milli(s)
Crypto took:         115 micro(s)
Started Commit:     +1.51 second(s)
Finished Commit:    +1.51 second(s)
Pact exec took:      179 micro(s)
Completed in 1.517327sec (2637 per sec)
node0> exec (demo.read-all)
account      | amount       | balance      | data
---------------------------------------------------------
"Acct1"      | "-1.00"      | "995999.00"  | {"transfer-to":"Acct2"}
"Acct2"      | "1.00"       | "4001.00"    | {"transfer-from":"Acct1"}
```

If you would like to view the performance metrics from each node in the cluster, this can be done via `pollMetrics <requestKey>`

```
node0> pollMetrics b768a85c6e1a06d4cfd9760dd981b675dcd9dc97ee8d7abc756246107f2ea03edd80e10e5168b41ee96a17b098ea3285a0f5ca9c61c4d974a7832e01f354dcf9
##############  node3  ##############
First Seen:          2017-03-19 05:43:14.571 UTC
Hit Turbine:        +24.03 milli(s)
Entered Con Serv:   +39.83 milli(s)
Finished Con Serv:  +52.41 milli(s)
Came to Consensus:  +113.00 milli(s)
Sent to Commit:     +113.94 milli(s)
Started PreProc:    +690.55 milli(s)
Finished PreProc:   +690.66 milli(s)
Crypto took:         115 micro(s)
Started Commit:     +1.51 second(s)
Finished Commit:    +1.51 second(s)
Pact exec took:      179 micro(s)
##############  node2  ##############
First Seen:          2017-03-19 05:43:14.868 UTC
... etc ...
```

NB: the `Crypto took` metric is only accurate when the concurrency system is set to `Threads`.
All other metrics are always accurate.

#### Sample Usage: Stressing the Cluster

The aforementioned performance test revolves around the idea that, in production, there will be a resource pool that batches new commands and forwards them directly to the Leader for the cluster.
This is, by far, the best architectural setup from a performance and efficiency perspective.
However, if you'd like to test the worst-case setup -- one where new commands are distributed evenly across the cluster and the cluster is forced to forward and batch them as best it can -- you can use `par-batch`.

`par-batch TOTAL_CMD_CNT CMD_RATE_PER_SEC DELAY` works much like `batch` except that `kadenaclient` will evenly distributed the new commands across the entire cluster.
It is creates `TOTAL_CMD_CNT` commands first and then submits portions of the new command pool to each node in individual batches with a `DELAY` millisecond pause between each submission.
Globally, it will achieve the `CMD_RATE_PER_SEC` specified.

For example, on a 4 node cluster `par-batch 10000 1000 200` will submit 50 new commands to each node every 200ms for 10 seconds.

NB: In-REPL performance metrics for this test are inaccurate.
Also, being the worst case architecture means that the cluster will make a best effort at performance but it will not be as high as `batch`.

#### Sample Usage: Running the payments demo with private functionality

Refer to "Entity Configuration" below for general notes on privacy configurations. This demo requires that there be 4 entities configured by the `genconfs` tool, which will name them "Alice", "Bob", "Carol" and "Dinesh". These would correspond to business entities on the blockchain, communicating with private messages over the blockchain. Confirm this setup with the `server` command.

Launch the cluster, and load the demo.yaml file.

```
node3> load demo/demo.yaml
status: success
data: TableCreated

Setting batch command to: (demo.transfer "Acct1" "Acct2" 1.00)
```

Create the private accounts by sending a _private message_ that executes a multi-step _pact_ to create private accounts on each entity. Change to Alice's server (node0) and send a private message to the other 3 participants with the demo code:

```
node3> server node0
node0> private Alice [Bob Carol Dinesh] (demo.create-private-accounts)
```

The `private` command creates an encrypted message, sent from Alice to Bob, Carol and Dinesh. The `create-private-accounts` pact executes a single command on the different servers. To see the results, perform _local queries_ on each node.

```
node0> local (demo.read-all)
account | amount   | balance  | data
-------------------------------------------------
"A"     | "1000.0" | "1000.0" | "Created account"
node0> server node1
node1> local (demo.read-all)
account | amount   | balance  | data
-------------------------------------------------
"B"     | "1000.0" | "1000.0" | "Created account"
node1> server node2
node2> local (demo.read-all)
account | amount   | balance  | data
-------------------------------------------------
"C"     | "1000.0" | "1000.0" | "Created account"
node2> server node3
node3> local (demo.read-all)
account | amount   | balance  | data
-------------------------------------------------
"D"     | "1000.0" | "1000.0" | "Created account"
```

This illustrates how the different servers (which would be presumably behind firewalls, etc) contain different, private data.

Now, execute a confidential transfer between Alice and Bob, transferring money from Alice's account "A" to Bob's account "B".

For this, the pact "payment" is used, which executes the debit step on the "source" entity, and the credit step on the "dest" entity. You can see the function docs by simply executing `payment`:

```
node3> local demo.payment
status: success
data: (TDef defpact demo.payment (src-entity:<i> src:<j> dest-entity:<k> dest:<l>
  amount:<m> -> <n>) "Two-phase confidential payment, sending money from SRC at SRC-ENTITY
  to DEST at DEST-ENTITY.")
```

Set the server to node0 for Alice, and execute the pact to send 1.00 to Bob:

```
node3> server node0
node0> private Alice [Bob] (demo.payment "Alice" "A" "Bob" "B" 1.00)
status: success
data:
  amount: '1.00'
  result: Write succeeded
```

To see the results, issue local queries on the nodes. Note that node2 and node 3 are unchanged:

```
node0> local (demo.read-all)
account | amount  | balance  | data
----------------------------------------------------------------------------
"A"     | "-1.00" | "999.00" | {"tx":5,"transfer-to":"B","message":"Starting pact"}
node0> server node1
node1> local (demo.read-all)
account | amount | balance   | data
-------------------------------------------------------------------------------------
"B"     | "1.00" | "1001.00" | {"tx":5,"debit-result":"Write succeeded","transfer-from":"A"}
node1> server node2
node2> local (demo.read-all)
account | amount   | balance  | data
-------------------------------------------------
"C"     | "1000.0" | "1000.0" | "Created account"
node2> server node3
node3> local (demo.read-all)
account | amount   | balance  | data
-------------------------------------------------
"D"     | "1000.0" | "1000.0" | "Created account"
```

You can also test out the rollback functionality on an error. Mistype the recipient account id (in this case we use "bad" instead of "B"). The pact will execute the debit on Alice/node0; attempt the credit on Bob/node1, failing because of the bad ID; finally the rollback will execute on Alice/node0. Bob's account will be unchanged, while Alice's account will note the rollback with the original tx id of the pact execution.

```
node0> private Alice [Bob] (demo.payment "Alice" "A" "Bob" "bad" 1.00)
status: success
data:
  tx: 7
  amount: '1.00'
  result: Write succeeded

node0> local (demo.read-all)
account | amount | balance  | data
--------------------------------------------
"A"     | "1.00" | "999.00" | {"rollback":7}
node0> server node1
node1> local (demo.read-all)
account | amount | balance   | data
--------------------------------------------------------------------------------------------
"B"     | "1.00" | "1001.00" | {"tx":5,"debit-result":"Write succeeded","transfer-from":"A"}
```

NB: The result of the first send shows you the result of the first part of the multi-phase tx, thus the "success"/"Write succeeded" status. Querying the database reveals the rollback which occurred two transactions later.

#### Sample Usage: Inserting multiple records

You can test inserting multiple records into a sample client database with the following commands:

```
node0> load demo/orders.yaml

This creates an 'orders' table into which records can be inserted.
```

The command:

```
node0> loadMultiple 0 3000 demo/orders.txt
```

will insert 3000 records into the orders table. The file orders.txt serves as a template for the order records, and contains special strings of the form "${count}" that will be replaced with the numbers
from 0 through 2999 as the records are inserted. All 3000 records are sent in a single HTTP 'batch' command.

You can run additional loadMultiple commands, but the initial 'count' (0 in the last example) must be chosen to not overlap with previously inserted rows. So subsequent commands could be:

```
node0> loadMultiple 3000 3000

node0> loadMultiple 6000 3000

node0> loadMultiple 9000 3000
```

etc.

#### Sample Usage: Viewing the Performance Monitor

Each kadena node, while running, will host a performance monitor at the URL `<nodeId.host>:<nodeId.port>/monitor`.

#### Sample Usage: Running Pact TodoMVC

This repo also bundles the [Pact TodoMVC](https://github.com/kadena-io/pact-todomvc). Each Kadena node will host the frontend at `<nodeId.host>:<nodeId.port>/todomvc`. To initialized the `todomvc`:

```
$ cd <kadena-directory>

# launch the cluster

$ ./bin/<OS-name>/kadenaclient.sh
node3> load todomvc/demo.yaml

# go to host:port/todomvc
```

NB: this demo can be run at the same time as the `payments` demo.

#### Sample Usage: Running a cluster on Azure

The Ansible playbooks and scripts we use for testing Kadena on Azure are now available as well, located in
`<kadena-directory>/ansible`. Refer to [Ansible Plabooks](#ansible-playbooks) section for detailed instructions on
how to use these Ansible playbooks and scripts.

#### Sample Usage: Querying the Cluster for Server Metrics

Each `kadenaserver` hosts an instance of `ekg` (a performance monitoring tool) on `<node-host>:<node-port>+80`.
It returns a JSON blob of the latest tracked metrics.
The following shell script extract uses this mechanism for status querying:

```
status)
  for i in `cat kadenaservers.privateIp`;
    do echo $i ;
      curl -sH "Accept: application/json" "$i:10080" | jq '.kadena | {role: .node.role.val, commit_index: .consensus.commit_index.val, applied_index: .node.applied_index.val}' ;
    done
  exit 0
  ;;
```

NB: The `port` that `genconfs` when running in `--distributed` mode is `10000` therefore `ekg` runs on port `10080` on each node.

# Configuration File Documentation

Generally, you won't need to personally edit the configuration files for either the client or server(s), but this information is available should you wish to.
The executable `genconfs` will create the configuration files for you and offer recommended settings based on your choices.

## Server (node) config file

### Node Specific Information

#### Identification

Each consensus node requires a unique Ed25519 keypair and `nodeId`.

```
myPublicKey: 53db73154fbb0c57129a0029439e5fc448e1199b6dcd5601bc08b48c5d9b0058
myPrivateKey: 0c2b9f177cee13c698bec6afe2e635ca244ce402ccbd826a483f25f618beec8f
nodeId:
  alias: node0
  fullAddr: tcp://127.0.0.1:10000
  host: '127.0.0.1'
  port: 10000
```

#### Other Nodes

Each consensus node further requires a map of every other node, as well as their associated public key.

```
otherNodes:
- alias: node1
  fullAddr: tcp://127.0.0.1:10001
  host: '127.0.0.1'
  port: 10001
- alias: node2
  fullAddr: tcp://127.0.0.1:10002
  host: '127.0.0.1'
  port: 10002
- alias: node3
  fullAddr: tcp://127.0.0.1:10003
  host: '127.0.0.1'
  port: 10003

publicKeys:
- - node0
  - 53db73154fbb0c57129a0029439e5fc448e1199b6dcd5601bc08b48c5d9b0058
- - node1
  - 65d59bda770dd6de2b25308b2e039714fec752e42d11af3712159f27e9e295f4
- - node2
  - bd1700e6f206315debabfa5bf42228ed4f9e78cacbffabcca74ff4f67e5ac7a4
- - node3
  - 8d6f928659ea57be2ac19d64af05ca0ccb0f42303f0d668d1263c9a4c8b36925
```

#### Runtime Configuration

Kadena uses SQLite for caching & persisting various by default.
Upon request, Oracle, MS SQL Server, Postgres, and generic ODBC backends are also available.

- `apiPort`: what port to host the REST API identical to the [Pact development server](http://pact-language.readthedocs.io/en/latest/pact-reference.html#rest-api)
- `logDir`: what directory to use for writing the HTTP logs, the Kadena logs, and the various SQLite databases.
- `enableDebug`: should the node write any logs

While this is pretty low level tuning, Kadena nodes can be configured to use different concurrency backends.
We recommend the following defaults but please reach out to us if you have questions about tuning.

```
preProcUsePar: true
preProcThreadCount: 100
```

### Consensus Configuration

These settings should be identical for each node.

- `aeBatchSize:<int>`: This is the maximum number of transactions a leader will attempt to replicate at every heartbeat. It's recommended that this number average out to 10k/s.
- `inMemTxCache:<int>`: How many committed transactions should be kept in memory before only being found on disk. It's recommended that this number be x10-x60 the `aeBatchSize`. This parameter impacts memory usage.
- `heartbeatTimeout:<microseconds>`: How often should the Leader ping its Followers. This parameter should be at least 2x the average roundtrip latency time of the clusters network.
- `electionTimeoutRange:[<min::microseconds>,<max::microseconds>]`: Classic Raft-Style election timeouts.
  - `min` should be >= 5x of `heartbeatTimeout`
  - `max` should be around `min + (heartbeatTimeout*clusterSize)`.

### Entity Configuration and Confidentiality

The Kadena platform uses the [Noise protocol](http://noiseprotocol.org/) to provide the best possible on-chain encryption, as used by Signal, Whatsapp and Facebook. Messages are thoroughly encrypted with perfect forward secrecy, resulting in opaque blobs on-chain that leak no information about contents, senders or recipients.

Configuring for confidential execution employs the notion of "entities", which identify sub-clusters of nodes as belonging to a business entity with private data. Entities maintain keys for encrypting as well as signing.

Within an entity sub-cluster, a single node is configured as a "sending node" which must be used to initiate private messages; this allows other sub-cluster nodes to avoid race conditions surrounding the key "ratcheting" used by the Noise protocol for forward secrecy; this way sub-cluster nodes will stay perfectly in sync and replicate properly.

For a given entity, the `signer` and `local` entries must match for all nodes in the sub-cluster; only one may be designated as `sender` setting that to `true` for that node only. `remotes` list the static public key and the entity name for each remote entity in the cluster.

The `signer` private and public keys are ED25519 signing keys; the `secret` and `public` keys for local ephemeral and static keys, as well as for remote public keys, are Curve25519 Diffie-Hellman keys.

### Performance Considerations

While `genConfs` will make a best guess at what the best configuration for your cluster is based on your inputs, it may be off. To that end, here are some notes if you find yourself seeing unexpected performance numbers.

The relationship of `aeBatchSize` to `heartbeatTimeout` determines the upper bound on performance, specifically `aeBatchSize/heartbeatTimeoutInSeconds = maxTransactionsPerSecond`.
This is because when the cluster has a large number of pending transactions to replicate, it will replicate up to `aeBatchSize` transactions every heartbeat until the cluster has caught up.
Generally, it's best to have `maxTransactionsPerSecond` be 1.5x of the expected performance, which itself is ~8k/second.

Because of the way that we measure performance, which starts from the moment that the cluster's Leader node first sees a transaction to when it fully executes the Pact smart contract (inclusive of the time required for replication, consensus, and cryptography), the logic of the Pact smart contract itself will impact performance.
Thus, executing simple logic like `(+ 1 1)` will achieve 12k commits/second whereas a smart contract with numerous database writes will vary based on the backend used and the complexity of the data model.

## Client (repl) config file

Example of the client (repl) configuration file. `genconfs` will also auto-generate this for you.

```
PublicKey: 53db73154fbb0c57129a0029439e5fc448e1199b6dcd5601bc08b48c5d9b0058
SecretKey: 0c2b9f177cee13c698bec6afe2e635ca244ce402ccbd826a483f25f618beec8f
Endpoints:
  node0: 127.0.0.1:8000
  node1: 127.0.0.1:8001
  node2: 127.0.0.1:8002
  node3: 127.0.0.1:8003
```

(c) Kadena 2017
