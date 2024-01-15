---
title: SSH端口转发
date: 2018-01-30 19:39:35
tags:
- ssh
categories:
- tools
---

```bash
ssh -L port:host:hostport remote
```
将打到本地port的连接 通过 remote 转发到host的hostport端口
```
-L [bind_address:]port:host:hostport
Specifies that the given port on the local (client) host is to be forwarded to the given host and port on the remote side. This works by allocating
a socket to listen to port on the local side, optionally bound to the specified bind_address. Whenever a connection is made to this port, the con‐
nection is forwarded over the secure channel, and a connection is made to host port hostport from the remote machine. Port forwardings can also be
specified in the configuration file. IPv6 addresses can be specified by enclosing the address in square brackets. Only the superuser can forward
privileged ports. By default, the local port is bound in accordance with the GatewayPorts setting. However, an explicit bind_address may be used to
bind the connection to a specific address. The bind_address of “localhost” indicates that the listening port be bound for local use only, while an
empty address or ‘*’ indicates that the port should be available from all interfaces.
```
```
ssh -R port:host:hostport remote
```
监听remote的port端口，将打到port端口的请求通过local转发到host的hostport端口。
```
-R [bind_address:]port:host:hostport

Specifies that the given port on the remote (server) host is to be forwarded to the given host and port on the local side. This works by allocating
a socket to listen to port on the remote side, and whenever a connection is made to this port, the connection is forwarded over the secure channel,
and a connection is made to host port hostport from the local machine.

Port forwardings can also be specified in the configuration file. Privileged ports can be forwarded only when logging in as root on the remote
machine. IPv6 addresses can be specified by enclosing the address in square brackets.

By default, the listening socket on the server will be bound to the loopback interface only. This may be overridden by specifying a bind_address.
An empty bind_address, or the address ‘*’, indicates that the remote socket should listen on all interfaces. Specifying a remote bind_address will
only succeed if the server's GatewayPorts option is enabled (see sshd_config(5)).

If the port argument is ‘0’, the listen port will be dynamically allocated on the server and reported to the client at run time. When used together
with -O forward the allocated port will be printed to the standard output.
```