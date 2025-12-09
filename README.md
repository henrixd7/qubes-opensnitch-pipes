# OpenSnitch Setup Helper Scripts for Qubes OS

**qubes-opensnitch-pipes** resolves the identity crisis when running OpenSnitch in Qubes OS.

By default, connecting multiple AppVMs to a central OpenSnitch UI results in all traffic appearing as "localhost," making it impossible to distinguish between source VMs. This project utilizes helper scripts to tunnel traffic through unique ports and dummy network interfaces, allowing the OpenSnitch UI to correctly identify and filter traffic per VM.

> **⚠️ Status: Experimental**
> This setup is experimental. While it works to separate VM identities, reliability with multiple concurrent nodes is not guaranteed.

## Architecture

This system consists of two components:

1.  **`qubes-opensnitch-piped` (The Server/UI Side):**
    * Runs on the VM hosting the OpenSnitch UI (e.g., `snitch-ui`).
    * Listens on a control port (default `50050`) to assign slots.
    * Creates dummy network devices (e.g., `dummy0` at `127.0.0.2`) for each registered node.
    * Redirects traffic from the node's assigned port to the OpenSnitch UI, preserving a unique IP per node.

2.  **`qubes-opensnitch-pipe` (The Client/Node Side):**
    * Runs on the AppVMs you want to monitor.
    * Connects to the server via Qubes RPC (`qubes.ConnectTCP`).
    * Requests a free port, injects it into `/etc/opensnitchd/default-config.json`, establishes a `socat` pipe, and starts the system daemon.

---

## Prerequisites

* **Qubes OS**
* **OpenSnitch** installed on the TemplateVM used by your nodes and the UI VM.

Remember to disable opensnitch systemd service `systemctl disable opensnitch` and remove `/etc/xdg/autostart/opensnitch_ui.desktop`. We will handle these steps differently later.
---

## 1. Qubes Policy Configuration (dom0)

You must allow TCP connections between your nodes and the UI VM.

Edit `/etc/qubes/policy.d/30-opensnitch.policy` in **dom0**:

```bash
# OpenSnitch node connections
# 50050 is the handshake/control port
qubes.ConnectTCP +50050 @tag:snitch @default allow target=snitch-ui

# 50051+ are the data ports (one per node slot)
qubes.ConnectTCP +50052 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50053 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50054 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50055 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50056 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50057 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50058 @tag:snitch @default allow target=snitch-ui
qubes.ConnectTCP +50059 @tag:snitch @default allow target=snitch-ui
```

*Replace `snitch-ui` with the actual name of your UI AppVM.*

### Tagging VMs
Apply the `snitch` tag to any AppVM that should send data to the UI:

```bash
qvm-tags add [VM_NAME] snitch
```

---

## 2. Server Setup (UI VM)

Configure the AppVM where `opensnitch-ui` will run.

### Configuration Map
Create a config file to map your VMs to specific loopback IP addresses. This ensures that `sys-net` always appears as `127.0.0.3` (for example).

```bash
mkdir -p ~/.config/qubes-opensnitch-piped
nano ~/.config/qubes-opensnitch-piped/config.json
```

**Example `config.json`:**
```json
{
    "vault": "127.0.0.2",
    "sys-net": "127.0.0.3",
    "sys-usb": "127.0.0.4",
    "gpu-personal": "127.0.0.5",
    "sys-protonvpn": "127.0.0.6",
    "test": "127.0.0.7"
}
```

### Enable the Helper Service

Create the user-level systemd service file:
File: `~/.config/systemd/user/qubes-opensnitch-piped.service`

```ini
[Unit]
Description=Qubes OpenSnitch Piped Connector
After=graphical-session.target

[Service]
ExecStart=/usr/local/bin/qubes-opensnitch-piped -ad \
    -c /home/user/.config/qubes-opensnitch-piped/config.json
Restart=on-failure

[Install]
WantedBy=default.target
```

Enable the user-level systemd service for the pipe daemon:
```bash
systemctl --user enable --now qubes-opensnitch-piped
```

### Start OpenSnitch UI
Ensure the standard OpenSnitch UI is listening on all interfaces (or specifically the IPv6 wildcard) so it can accept the forwarded traffic. Add this to your `/rw/config/rc.local` or autostart:

```bash
# Start local daemon
systemctl enable --now opensnitch

# Start UI listening on port 50051
opensnitch-ui --socket "[::]:50051" &
```

---

## 3. Client Setup (Node VMs)

There are two ways to configure the nodes: via the **TemplateVM** (System-wide) or per **AppVM** (User mode).

### Pre-requisite: Persistence
Since AppVMs reset `/etc` on reboot, you must symlink the rules directory to a persistent location. Run this on your **TemplateVM**:

```bash
mkdir -p /rw/config/opensnitch/rules
rm -rf /etc/opensnitchd/rules
ln -s /rw/config/opensnitch/rules /etc/opensnitchd/rules
```

### Option A: System-wide Service (Recommended for TemplateVMs)
This method allows you to deploy the script in a TemplateVM but only activate it on specific hosts (e.g., `sys-net`).

File: `/etc/systemd/system/qubes-opensnitch-pipe.service`

```ini
[Unit]
Description=Qubes OpenSnitch Pipe
After=graphical-session.target
# Only start on these specific VMs:
ConditionHost=|sys-net
ConditionHost=|sys-usb

[Service]
ExecStart=/usr/local/bin/qubes-opensnitch-pipe -sp /rw/config/opensnitchd/rules
Restart=on-failure
KillMode=process

[Install]
WantedBy=default.target
```
*Note: The `-sp` flag adds a prefix to the rules directory (e.g., `/rw/config/opensnitchd/rules.sys-net`), allowing different rule sets for different VMs.*

### Option B: User Service (AppVM Specific)
If you prefer configuring per AppVM (requires sudo privileges):

File: `~/.config/systemd/user/qubes-opensnitch-pipe.service`

```ini
[Unit]
Description=Qubes OpenSnitch Pipe
After=graphical-session.target

[Service]
ExecStart=/usr/local/bin/qubes-opensnitch-pipe -sp /rw/config/opensnitchd/rules
Restart=on-failure
KillMode=process

[Install]
WantedBy=default.target
```
Enable it: `systemctl --user enable --now qubes-opensnitch-pipe`

---

## Troubleshooting

### On the UI VM (Server)

1.  **Check if UI is listening:**
    ```bash
    sudo ss -tulnp | grep 50051
    # Should show: LISTEN ... opensnitch-ui
    ```

2.  **Check if Piped is running:**
    ```bash
    systemctl --user status qubes-opensnitch-piped
    ```

### On the Node VM (Client)

1.  **Check connection to the pipe:**
    ```bash
    sudo ss -tulnp | grep 50052  # (Or whichever port was assigned)
    ```

2.  **Check the daemon status:**
    ```bash
    systemctl status opensnitch
    ```

### Verification
Run a network request from the Node VM:
```bash
curl [http://example.com](http://example.com)
```
A popup should appear in `snitch-ui` on the Server VM.

> **Note:** ICMP requests (`ping`) do **not** reliably generate events in Qubes networking. Use `curl` or `dig` for testing.

---

## CLI Reference

### qubes-opensnitch-pipe (Client)
```text
usage: qubes-opensnitch-pipe [-h] [-c CONFIG] [-t CONNECT] [-p PORT]
                             [-s RULES] [-sp RULES_PREFIX] [-d RULES_DEST]
                             [-ll {info,warn,error,critical}]

options:
  -h, --help            show this help message and exit
  -c, --config          OpenSnitch config file (where port is injected)
  -t, --connect         IP address for server connection
  -p, --port            Port for server connection
  -s, --rules           Rules directory (copied to --rules-dest)
  -sp, --rules-prefix   Rules directory prefix (adds .$hostname to path)
  -d, --rules-dest      Define OpenSnitch rules directory (default: /etc/opensnitchd/rules)
```

### qubes-opensnitch-piped (Server)
```text
usage: qubes-opensnitch-piped [-h] [-c CONFIG] [-a ADDRESS] [-p PORT]
                              [-l LISTEN] [-lp LISTEN_PORT] [-ad]
                              [-ll {info,warn,error,critical}]

options:
  -h, --help            show this help message and exit
  -c, --config          JSON file path for { "host":"ip_address" } pairs
  -a, --address         Starting IP address (default: 127.0.0.1)
  -p, --port            Starting port for node connections (default: 50051)
  -l, --listen          Listen address for service (default: 127.0.0.1)
  -lp, --listen-port    Listen port for service (default: 50050)
  -ad, --allow-disposables  Allow disposables without address defined in config
```

