# Network Architecture

This document describes the network configuration used by the self-hosted RustDesk environment.

## Overview

The RustDesk Server runs inside an Ubuntu Server virtual machine hosted on Proxmox VE.

The server is accessible both from the local network and from external networks through Dynamic DNS and NAT/Port Forwarding.

```text
External RustDesk Client
        |
        | Internet
        v
   No-IP DDNS
        |
        v
   Public IPv4
        |
        v
      Router
        |
        | NAT / Port Forwarding
        v
192.168.1.202
  rustdesk01
        |
   +----+----+
   |         |
 hbbs       hbbr
   |         |
   +----+----+
        |
        v
Local RustDesk Client
```

## Local Network

The RustDesk Server VM uses the following internal network configuration:

| Parameter       | Value            |
| --------------- | ---------------- |
| Hostname        | `rustdesk01`     |
| IPv4            | `192.168.1.202`  |
| Default Gateway | `192.168.1.254`  |
| Network         | `192.168.1.0/24` |

The server is connected to the local network through the Proxmox bridge using a VirtIO network interface.

## RustDesk Network Ports

The following ports are required by the current RustDesk Server OSS deployment:

| Port  | Protocol | Service | Purpose                          |
| ----- | -------- | ------- | -------------------------------- |
| 21115 | TCP      | hbbs    | NAT type test                    |
| 21116 | TCP      | hbbs    | Connection and TCP hole punching |
| 21116 | UDP      | hbbs    | ID registration and heartbeat    |
| 21117 | TCP      | hbbr    | Relay connections                |

Ports related to the RustDesk Web Client were not exposed because they are not required by this environment.

## Host Firewall

UFW is enabled on the Ubuntu Server.

RustDesk traffic is allowed through the required ports:

```bash
sudo ufw allow 21115/tcp
sudo ufw allow 21116/tcp
sudo ufw allow 21116/udp
sudo ufw allow 21117/tcp
```

SSH access is restricted to the internal network and is not exposed through the Internet router.

The firewall follows the principle of exposing only the services required by the application.

## NAT and Port Forwarding

The RustDesk Server uses a private IPv4 address and therefore cannot be reached directly from the Internet.

The router performs Destination NAT / Port Forwarding from the public IPv4 address to the RustDesk Server.

```text
Public-IP:21115/TCP
        |
        v
192.168.1.202:21115/TCP

Public-IP:21116/TCP
        |
        v
192.168.1.202:21116/TCP

Public-IP:21116/UDP
        |
        v
192.168.1.202:21116/UDP

Public-IP:21117/TCP
        |
        v
192.168.1.202:21117/TCP
```

The SSH port is intentionally not forwarded.

## Dynamic DNS

The Internet connection does not provide a static public IPv4 address.

No-IP Dynamic DNS is used to maintain a hostname associated with the current public IP.

```text
RustDesk Client
      |
      v
DDNS Hostname
      |
      v
No-IP
      |
      v
Current Public IPv4
      |
      v
Router
```

A No-IP Dynamic Update Client runs on `rustdesk01` and automatically updates the DNS record when the public IPv4 address changes.

The DUC runs as a systemd service:

```bash
sudo systemctl status noip-duc
```

The real DDNS hostname and authentication credentials are intentionally excluded from this repository.

## Client Configuration

RustDesk clients are configured using the DDNS hostname instead of the public IPv4 address.

Example:

```text
ID Server:
rustdesk.example.ddns.net

Relay Server:
empty

API Server:
empty

Key:
RustDesk server public key
```

This allows the clients to continue reaching the server even after the ISP changes the public IPv4 address.

## Connection Flow

RustDesk uses `hbbs` to coordinate the connection between clients.

```text
Client A ------> hbbs <------ Client B
                    |
             Connection setup
                    |
                    v
          Direct connection
           when possible
```

When a direct connection cannot be established, `hbbr` can relay the traffic:

```text
Client A ------> hbbr ------> Client B
                  Relay
```

## External Connectivity Validation

The deployment was validated using two different networks.

The notebook was disconnected from the homelab network and connected through an external Internet connection.

The Windows VM remained inside the homelab LAN.

```text
Notebook
External Network
      |
      v
No-IP DDNS
      |
      v
Internet
      |
      v
Home Router
      |
      v
192.168.1.202
RustDesk Server
      |
      v
Windows VM
```

A remote RustDesk session was successfully established.

An external connection reaching `hbbs` was also verified directly on the Ubuntu Server:

```bash
sudo ss -tnp | grep -E '21115|21116|21117'
```

An established connection was observed on TCP port `21116`, confirming communication between the external RustDesk client and the self-hosted `hbbs` service.

## Security Notes

The following information is intentionally not included in this repository:

* Public IPv4 address
* Real Dynamic DNS hostname
* DDNS credentials
* RustDesk private key
* Authentication credentials

Only private RFC1918 addressing used inside the homelab is documented.
