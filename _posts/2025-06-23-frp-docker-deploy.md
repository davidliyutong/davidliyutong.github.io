---
layout: post
title: Exposing an Internal SSH Port with FRP and Docker Compose
date: 2025-06-23 00:01:00
description: Run frps and frpc with Docker Compose to reach an internal machine's SSH port through a public server.
tags: frp docker network ssh IT
categories: IT
---

[FRP](https://github.com/fatedier/frp) (Fast Reverse Proxy) lets a machine behind NAT or a firewall expose a service through a server that has a public IP. The internal machine runs the client, `frpc`, which opens an outbound connection to the server, `frps`. The server then forwards traffic arriving on a public port back through that connection. In this setup both sides run in Docker Compose, and the internal machine's SSH port (22) becomes reachable at port 37843 on the public server.

## Ports

| Port  | Where     | Purpose                                                        |
| ----- | --------- | -------------------------------------------------------------- |
| 37000 | frps      | `bindPort`: frpc connects to the server on this port           |
| 7500  | frps      | Web dashboard                                                  |
| 37843 | frps      | `remotePort` of the SSH proxy: users connect here to reach SSH |
| 22    | frpc host | Local SSH daemon that gets exposed                             |

Open 37000 and 37843 (and 7500 if you want to use the dashboard remotely) in the public server's firewall or cloud security group.

## Server (frps)

On the public server, put these two files in the same directory.

`docker-compose.yml`:

```yaml
services:
  frps:
    image: snowdreamtech/frps # or fatedier/frps, which also needs: command: ["-c", "/etc/frp/frps.toml"]
    container_name: frps
    restart: unless-stopped
    ports:
      - "37000:37000" # bindPort (frpc connections)
      - "7500:7500" # dashboard
      - "37843:37843" # exposed SSH port (remotePort)
    volumes:
      - ./frps.toml:/etc/frp/frps.toml
```

`frps.toml`:

```toml
bindPort = 37000
auth.token = "<FRP_AUTH_TOKEN>"

webServer.addr = "0.0.0.0"
webServer.port = 7500
webServer.user = "admin"
webServer.password = "<DASHBOARD_PASSWORD>"
```

Here's what the settings do:

- `bindPort` is the port frps listens on for frpc connections. The compose file has to publish it.
- `auth.token` is a shared secret, and frpc must send the same value. Token authentication is frp's default (`auth.method = "token"`), so you don't need to set the method explicitly.
- `webServer.*` turns on the dashboard, which shows connected clients, proxies and traffic, protected by basic auth. frp binds the dashboard to `127.0.0.1` by default. Inside a container it has to listen on `0.0.0.0`, otherwise the published port 7500 can't reach it. The dashboard is served over plain HTTP. If you don't need it from outside, don't publish 7500, or restrict that port in the firewall.
- Every `remotePort` used by a client proxy must also be published by the frps container, which is why `37843:37843` appears under `ports`. frps opens the listener inside the container, but without the mapping no outside traffic reaches it. Each new proxy needs its own mapping.

The `snowdreamtech/frps` image starts `frps -c /etc/frp/frps.toml` automatically, so mounting the file there is enough. The official `fatedier/frps` image runs plain `frps` with no arguments, so you have to pass the config path through `command` yourself.

Start the server:

```bash
docker compose up -d
```

## Client (frpc)

On the internal machine, create the client files.

`docker-compose.yml`:

```yaml
services:
  frpc:
    image: snowdreamtech/frpc:latest
    container_name: frpc
    restart: unless-stopped
    network_mode: host # host mode recommended
    volumes:
      - ./frpc.toml:/etc/frp/frpc.toml
```

`frpc.toml`:

```toml
serverAddr = "<FRPS_PUBLIC_IP>"  # public IP of the frps server
serverPort = 37000               # must match bindPort in frps.toml
auth.token = "<FRP_AUTH_TOKEN>"

[[proxies]]
name = "ssh"                     # proxy name (any unique name)
type = "tcp"                     # proxy type
localIP = "127.0.0.1"            # local service IP (defaults to 127.0.0.1)
localPort = 22                   # local SSH port
remotePort = 37843               # port opened on frps; users connect here
```

Here's what the settings do:

- `serverAddr` and `serverPort` point at frps. The port must match `bindPort` on the server.
- `auth.token` must be the same as on the server, otherwise frps rejects the login.
- Each `[[proxies]]` entry defines one forwarded service. This `tcp` proxy asks frps to listen on `remotePort` 37843 and forward every connection to `localIP:localPort`, which is the SSH daemon on port 22.
- `network_mode: host` is what makes `127.0.0.1` refer to the host itself. With the default bridge network, `127.0.0.1` would be the frpc container, and port 22 there has nothing listening.

Start the client:

```bash
docker compose up -d
```

## Connecting

Once frpc logs a successful login and starts the `ssh` proxy, you can reach the internal machine through the public server:

```bash
ssh -p 37843 <USER>@<FRPS_PUBLIC_IP>
```
