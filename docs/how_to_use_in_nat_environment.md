How to deploy BBc-1 v1.0 in cloud and/or NAT environment

# Introduction
Aiming to create a BBc-1 network constructed with nodes working in various environments, this document explains how to deploy and connect BBc-1 node in a cloud and/or NAT environment.


# Expected Environments
We assume that BBc-1 would be deployed onto a (virtual) host working in a network as the following diagram.

```
node 1 [bbc_core]
< ipv4 private address: A.A.A.A, exposed port: P1 >
   │
NAT gateway 1
< ipv4 global address: X.X.X.X, exposed port: P1 (forwarding to node 1) >
   │
The Internet
   │
NAT gateway 2
< IPv4 global address: Y.Y.Y.Y, exposed port: P2 (forwarding to node 2) >
   │
node 2 [bbc_core]
< ipv4 private address: B.B.B.B, exposed port: P2 >
```

For instance of Microsoft Azure, each virtual machine has one virtual network interface in default, and it get assigned a private IPv4 address from NAT gateway due to the security reason.  In such an environment, although it, of course, get assigned unique global IPv4 address, there is no way for such a virtual node to *implicitly* notify external nodes its global IPv4 address in the standard sequence of BBc-1 connection establishment.

So, here we shall explain how to set up BBc-1 instances and connect them with one another by *explicitly* specifying its reachable global IPv4 address.

This procedure can be applied to any kind of NAT environment with appropriate static port forwarding at NAT gateway.

NOTE: *NAPT (dynamic IP masquerading) is not supported at this point (v1.0). We should configure static port forwarding at gateways in private networks.*

# Environments
First of all, we need to install several packages into every node (`node 1` and `node 2` in the above diagram) in the exactly same manner as
[Quick Start](https://github.com/beyond-blockchain/bbc1/blob/develop/README.md). Here we shall use `pipenv` as the environment manager for Python 3.

For macOS High Sierra:
```shell
$ brew update
$ brew install python3 pipenv
```

For Ubuntu 17.10:
```shell
$ sudo apt update
$ sudo apt install -y software-properties-common python-software-properties
$ sudo add-apt-repository ppa:pypa/ppa
$ sudo apt update
$ sudo apt install -y git python3 pipenv make
```

# Setup BBc-1
The setup procedure is almost same as well as [Quick Start](https://github.com/beyond-blockchain/bbc1/blob/develop/README.md). At both `node 1` and `node 2`, we should setup the base of BBc-1 as follows. Note that the following procedure shows the case of installation into the native environment.

```shell
# bbc1 source is retrieved
$ git clone https://github.com/beyond-blockchain/bbc1.git
$ cd bbc1
$ git checkout develop

# openssl is retrieved and compiled
$ bash prepare.sh

# install required pypi packages into virutual environment
$ pipenv --python 3 # python3.* will be used for bbc1
$ pipenv install
```
After the above setup procedure, we are ready to run the BBc-1 in a standalone manner. Now you can run BBc-1 standalone by `pipenv run python ./bbc_core.py` at `bbc1/core` directory.

# Configure your domain
In order to connect multiple BBc-1 nodes with each other right after its instantiation, we need to configure the *domain* in the exactly same manner at each nodes.

The minimum configuration would be generated by `utils/bbc_domain_config.py` as the following procedure, where the argument with `-d`, i.e., `da799b...`, is a hexified  sample *domain id* that would be established simultaneously with BBc-1 instantiation.

```shell
$ cd /path/to/bbc1
$ cd utils
$ pipenv run python bbc_domain_config.py -t generate -w ~/.bbc1 # generate config file at ~/.bbc1
$ pipenv run python bbc_domain_config.py -t write -w ~/.bbc1 -d "da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd" # add domain configuration
```

Through this configuration operation, we have the following domain setup in `domains` section of `~/.bbc1/config.json`.

```json:~/.bbc1/config.json
"da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd": {
  "storage": {
    "type": "internal"
  },
  "db": {
    "db_type": "sqlite",
    "db_name": "bbc_ledger.sqlite",
    "replication_strategy": "all",
    "db_servers": [
      {
        "db_addr": "127.0.0.1",
        "db_port": 3306,
        "db_user": "user",
        "db_pass": "pass"
      }
    ]
  }
}
```
The detailed options of `utils/bbc_domain_config.py` is explained [here (bbc1/utils/README.md)](https://github.com/beyond-blockchain/bbc1/blob/develop/utils/README.md).

Your own hexfied domain id can be generated from an arbitrary string through `utils/id_create.py` in the following manner.

```shell
$ cd /path/to/bbc1
$ cd utils
$ pipenv run python id_create.py -s "desired string"
da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd
```

# Run and connect BBc-1 with external nodes

## Start `bbc_core.py`
Now we are ready to create a BBc-1 networks with multiple nodes in multiple distinct networks. To this end, we will execute
```shell
$ pipenv run python ./bbc_core.py -pp <exposed port> --ip4addr <ipv4 address reachable to the node> -w <pre-configured working directory>
```
at each node.

In particular, from the diagram given in the head of this document, we see that `node 1` and `node 2` can be reachable with `X.X.X.X:P1` and `Y.Y.Y.Y:P2`, respectively. Then, we run `bbc_core.py` and instantiate (daemonized) BBc-1 nodes at `node 1` and `node 2` by executing the following commands.

- At `node 1`:
  ```shell
  $ cd /path/to/bbc1
  $ cd bbc1/core
  $ pipenv run python ./bbc_core.py -d -pp P1 --ip4addr X.X.X.X -w ~/.bbc1 -l /var/tmp/bbc1.log
  ```

- At `node 2`:
  ```shell
  $ cd /path/to/bbc1
  $ cd bbc1/core
  $ pipenv run python ./bbc_core.py -d -pp P2 --ip4addr Y.Y.Y.Y -w ~/.bbc1 -l /var/tmp/bbc1.log
  ```

When the initial execution of `bbc_core.py`, the node specific private key, `node_key.pem`, is created and stored under the working directory, i.e., `~/.bbc1` in this example. The private key will be used by management utilities like a node statistics manager to connect the core.

## Connect nodes with each other via `bbc_ping.py`
Then, we connect a node with the other node over the configured domain using `bbc_ping.py` as follows.
```shell
$ pipenv run python bbc_ping.py -k <node private key file> <domain id> <destination address> <destination port>
```

For the case of `node 1` and `node 2` in the exemplary diagram, we execute the following commands.

- At `node 1`:
  ```shell
  $ cd /path/to/bbc1
  $ cd utils
  $ pipenv run python bbc_ping.py -k ~/.bbc1/node_key.pem da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd Y.Y.Y.Y P2
  ```
- At `node 2`:
  ```shell
  $ cd /path/to/bbc1
  $ cd utils
  $ pipenv run python bbc_ping.py ~/.bbc1/node_key.pem da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd X.X.X.X P1
  ```

# Check connection status and statistics
We can verify if the connection is successfully established and still flawlessly connected via `util/bbc_info.py`. If you are on the `node 1`, you can see the list of connected nodes for each domain as follows.

- At `node 1`
  ```shell
  $ cd utils
  $ pipenv run python bbc_info.py -l -k ~/.bbc1/node_key.pem -d da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd
  ====== neighbor list of domain:da799b7ffbf94e7908982c36e7ebfa6a1ae6c9744b4d24d241f711a5d6d0eacd =====
            node_id(4byte), ipv4, ipv6, port, is_domain0
  *myself*  b'c346191c', <your ip address>, ::, 6641, False
            b'b2d94fb6', <node 2 ip address>, ::, 6641, False
  ```

As well as the connection status, we can also see the bbc1 node statistics information using `util/bbc_info.py` as follows. Here we do not need to specify particular domain ids.

- At `node 1`
  ```
  $ pipenv run python bbc_info.py --stat -k ~/.bbc1/node_key.pem
  ------ statistics ------
  {b'client': {b'num_message_receive': 23, b'total_num': 1},
   b'data_handler': {b'exec_sql': 7},
   b'network': {b'domain_ping_send': 2,
                b'neighbor_nodes': 1,
                b'num_domains': 1,
                b'packets_received_by_udp': 6,
                b'send_msg_by_udp4': 6,
                b'sent_data_size': 2151},
   b'topology_manager': {b'NOTIFY_NEIGHBOR_LIST': 4}}
  ```
