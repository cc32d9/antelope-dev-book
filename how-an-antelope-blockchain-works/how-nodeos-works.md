# How nodeos works

This chapter gives an overview of blockchain nodes and their work. It is by no means a reference documentation.&#x20;

## Node daemon and its state

`nodeos` is the central part of an Antelope blockchain: it processes the transactions and blocks and makes sure the blockchain adheres to the consensus. It is a Linux daemon program, and it typically runs on Ubuntu bare metal servers.

It has a modular structure with a few mandatory and several optional modules that are activated in the configuration file.

There's a number of configuration options which can be specified in the command line and in `config.ini` file. A few options, such as `--data-dir` and `--config-dir`, can only be specified in the command line. `nodeos --help` will print the full list of supported options. Some of them activate additional modules, and some options are only relevant when a corresponding module is activated.

The central part of a node's data is the blockchain state, and it's stored in `state/` subfolder of the data directory. It is organized as a file-mapped shared memory segment. Within this memory, the node is maintaining several binary trees that keep track of blockchain accounts, smart contract tables, and their indexes. It is seen as one large memory segment by the Linux kernel, and the kernel is doing its best to maintain its mapping to the storage. In each transaction, some data is read and written in the state memory, and it is seen like a completely random access by the kernel.&#x20;

If the state is resided in a normal filesystem, the kernel will try to synchronize all updates with the on-disk file as fast as possible. In a blockchain with persistent activity, this puts a high demand for I/O latency, and rules out any virtualized storage (the virtualized storage always has an unavoidable latency because of the network layer between the CPU and the physical storage). Also, traditional HDD are usually not fast enough. Currently most public blockchains require servers with physically attached SSD or NVME in order to catch up with the blockchain updates.

As the blockchain activity intensifies, the state updates become a serious factor in SSD wearing. A blockchain such as EOS or WAX may render an NVME drive unusable within a year. Also, in such a blockchain as WAX, NVME is becoming the bottleneck in node performance.

In order to circumvent the NVME wearing, `tmpfs` filesystem is widely used on Antelope nodes. It is an in-memory file system, keeping all its content in server RAM and swap space. About half of the state memory is rarely used and can be swapped out without compromising the performance, so the server RAM can be as small as half of the state memory. In addition, the Linux kernel is handling it in a smart way: there is no data duplication, and shared memory client is reading directly from the memory. Also the swap manager is pushing the least accessed pages onto the disk storage, giving more room for frequently accessed ones. On a server with enough RAM and node state in tmpfs, the storage I/O is moderate and not so dangerous for the SSD life anymore.

## Node roles

A blockchain network usually consists of a number of nodes, each serving one or several different roles. The common rule is to run separate nodes for different jobs, although it is possible to combine several roles in one node. The nodes exchange blocks and transactions between each other via the p2p protocol, which will be covered later.

* **Producer nodes:** these nodes are receiving signed transactions over p2p protocol and placing them in blocks. Only one node is producing blocks at a time, and they all sign their blocks according to the schedule. It is also important that the local clock on such a server is synchronized with a reliable source of time. Normally, such nodes are well protected from external traffic as the whole network stability depends on their performance.
* **RPC nodes:** these nodes provide their HTTP RPC interface to the clients. The RPC is typically used for sending transactions to the blockchain and for retrieving the current state, such as data rows in contract tables. A blockchain network operator may provide public nodes for everyone to connect. Also private nodes for backend applications are not uncommon.
* **P2P endpoint nodes:** they accept p2p connections from other nodes. Typically, a blockchain node operator publishes the DNS names and TCP port numbers of p2p nodes so that anyone could connect their own nodes to the network. Sometimes private p2p nodes are utilized between network operators in order to increase the stability. There is also a special kind of nodes which do not allow any speculative transactions on their p2p interface. Such nodes are usually used by block producer organizations for better network performance so that signed blocks are propagated with minimum possible delay.
* **State history nodes:** these nodes run with `state_history_plugin` activated. The plugin exports transaction traces and contract table updates in a data format that is consumed by various history indexing solutions. Normally, such nodes do not expose their RPC or p2p interfaces to public clients.
* **Firehose nodes:** they run with deep mind plugin activated and provide another way of exporting transaction traces and memory updates.

## Node interfaces

A running `nodeos` process is using a number of API protocols in its interaction with external systems. The most commonly used interfaces are listed below.

### HTTP RPC

The [EOSIO Developers Portal](https://docs.eosnetwork.com/) has a full reference of the RPC endpoints. Below we list the most essential ones for a blockchain developer. The node provides a standard HTTP interface at the TCP port specified by the `http-server-address` configuration option. It also supports HTTPS mode, but it's not recommended as SSL termination is more efficiently handled by a separate service, such as nginx.

* `/v1/chain/get_info` shows the state of the blockchain as seen by the node. The most important fields are explained below:
  * `chain_id` is a unique 32-byte identifier that distinguishes the blockchains from one another.
  * `head_block_num` is the latest block number as known by the node.
  * `head_block_time` is the timestamp of the latest known block in UTC timezone. In normal circumstances, it should be within 0.5s from the current time.
  * `last_irreversible_block_num` (LIB) indicates the blockchain finality as seen by the node. If a blockchain is malfunctioning, you would see the LIB stuck and not advancing. Under normal condition, the LIB is within 300-350 blocks from the head block.&#x20;
  * `last_irreversible_block_id`: each block has a unique 32-byte identifier which is equal to the sha256 hash of its content. In order to send a transaction, the client needs to refer to a valid block ID, and the LIB identifier is the most reliable one as it won't be cancelled by a fork.
* `/v1/chain/get_block` takes a block number or block ID as an argument and returns a JSON representation of the block. It's only showing transaction _inputs_ but not transaction _traces_ (see the [transaction lifecycle chapter](life-cycle-of-a-transaction.md) for more details).
* `/v1/chain/push_transaction` and `/v1/chain/send_transaction` are doing the same job by accepting a transaction for further evaluation and broadcast. The difference is in their output: `push_transaction` returns the trace in a hierarchical object, whereas the newer `send_transaction` is returning the trace as a flat list. Also, `send_transaction`  executes faster as it does not need to prepare the hierarchical object.&#x20;
* `/v1/chain/get_table_rows` lets you query the smart contract tables and receive the content of their rows. The API allows specifying the range of values for the primary or secondary key and returns the data in deserialized form using the contract ABI for decoding. The output length is limited by the time of response, so you would get a few dozen rows even if you requested a thousand. The response contains "more" and "next\_key" parameters which allow the client to retrieve more values if there are any in the table.

### P2P interface

The P2P protocol is used by the nodes to exchange all needed information between each other. This includes blocks and speculative transactions as well as control messages.

A node accepts P2P connections on the TCP port specified by `p2p-listen-endpoint` configuration statement. it also connects to other nodes which are listed in `p2p-peer-address` statements.

The communication protocol is symmetrical: it does not matter who connected to whom, and a pair of nodes is communicating in the same way. It means that your node will receive all necessary data if it has only incoming or only outgoing p2p connections.

### State history API

Some nodes may have the `state_history_plugin` activated in order to collect and export the details of all transactions and changes in contract memory. The API is using raw websocket protocol, and the data is exported in serialized form. There are several [open-source libraries and tools](https://cc32d9.medium.com/history-and-notifications-in-eosio-blockchain-8255194af93) for reading and decoding the state history.&#x20;
