# OpenSnitch Setup Helper Scripts for QubesOS

These scripts are designed to help set up OpenSnitch in a QubesOS environment where the UI runs in an isolated VM without network access. Nodes can connect using Qubes/Xen internal mechanisms (socat ConnectTCP pipes) with unique ports and IP addresses.

## Features

- Automates the setup of OpenSnitch nodes in QubesOS
- Allows each node to connect using a unique port
- Sets up networking in the UI VM to handle unique IP addresses for each node
- Simplifies configuration injection and rule management for OpenSnitch
- Uses socat TCP pipes for secure internal communication

## Scripts Overview

### 1. `qubes-opensnitch-piped`

A server-side script that manages connections from multiple nodes.

#### Usage:
```bash
usage: qubes-opensnitch-piped [-h] [-c CONFIG] [-a ADDRESS] [-p PORT]
                              [-l LISTEN] [-lp LISTEN_PORT] [-ad]
                              [-ll {info,warn,error,critical}]

Help OpenSnitch nodes with qubes.ConnectTCP to identify on UI

options:
  -h, --help            show this help message and exit
  -c, --config CONFIG   json file path for acceptable { "host":"ip_address",
                        "host2":"ip_address2" ... } pairs
  -a, --address ADDRESS
                        starting ip address/range (default: 127.0.0.1), this
                        will be incremented, so first address given to client
                        would be 127.0.0.2
  -p, --port PORT       starting port for node connections (default: 50051),
                        this value is incremented for each new connection
  -l, --listen LISTEN   listen address for service (default: 127.0.0.1)
  -lp, --listen-port LISTEN_PORT
                        listen port for service (default: 50050)
  -ad, --allow-disposables
                        allow all disposables without address defined in
                        config
  -ll, --log-level {info,warn,error,critical}
                        Sets output level to...
```

### 2. `qubes-opensnitch-pipe`

A client-side script that helps nodes identify with qubes.ConnectTCP on OpenSnitch.

#### Usage:
```bash
usage: qubes-opensnitch-pipe [-h] [-c CONFIG] [-t CONNECT] [-p PORT]
                             [-s RULES] [-sp RULES_PREFIX] [-d RULES_DEST]
                             [-ll {info,warn,error,critical}]

Client to help nodes to identify with qubes.ConnectTCP on OpenSnitch

options:
  -h, --help            show this help message and exit
  -c, --config CONFIG   OpenSnitch configuration file (where recieved port
                        number will be injected)
  -t, --connect CONNECT
                        IP address for server connection
  -p, --port PORT       port for server connection
  -s, --rules RULES     rules directory (all rules are copied to --rules-dest)
  -sp, --rules-prefix RULES_PREFIX
                        rules directory prefix (".$hostname" will be added to
                        this path)
  -d, --rules-dest RULES_DEST
                        define OpenSnitch rules directory (default:
                        /etc/opensnitchd/rules)
  -ll, --log-level {info,warn,error,critical}
                        Sets output level to...
```

## Requirements

- QubesOS installed and running
- OpenSnitch installed on your system
- Socat package installed (`sudo apt-get install socat` or equivalent for your distribution)

## Usage Example

Edit the `/etc/qubes/policy.d/30-user.policy` and add these lines:

```bash
# OpenSnitch node connections
qubes.ConnectTCP +50050 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50052 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50053 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50054 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50055 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50056 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50057 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50058 @tag:snitch @default allow target=[UI VM NAME]
qubes.ConnectTCP +50059 @tag:snitch @default allow target=[UI VM NAME]
```

Remember to add tags to all node VMs: `qvm-tag [VM] add snitch`

### UI (server) Setup (qubes-opensnitch-piped)
```bash
./qubes-opensnitch-piped -ad
```

### Node (client) Setup (qubes-opensnitch-pipe)
```bash
./qubes-opensnitch-pipe
```

## Configuration Example

Here’s an example configuration file that maps different services/nodes to unique IP addresses:

```json
# Example config.json
{
    "disp": "127.0.0.2",
    "email": "127.0.0.3",
    "gpu-personal": "127.0.0.4",
    "sys-protonvpn": "127.0.0.5",
    "sys-net": "127.0.0.6",
    "sys-usb": "127.0.0.7"
}
```

Each key represents a service or node, and the value is its corresponding IP address.

### Add Scripts to rc.local

Edit the `/rw/config/rc.local` file in your QubesOS VM and add the following lines to ensure the scripts run at startup:

#### For UI VM:
```bash
# UI VM configuration
/usr/local/sbin/qubes-opensnitch-piped -ad -c /home/user/.config/qubes-opensnitch-piped/config.json
```

#### For Node VMs:
```bash
# Node VM configuration (no arguments needed)
/usr/local/sbin/qubes-opensnitch-pipe
```
