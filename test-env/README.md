# Mole's Test Environment

This provides a small envorinment where `mole` functions can be tested and
debugged.

Once created, the test environment will provide a container running a ssh
server and another container running two (2) http servers, so ssh tunnels can
be created using `mole`.

In addition to that, the test environment provides the infrastructure to
analyze the traffic going through the ssh traffic.

## Topology

```ascii
+-------------------+                                 
|                   |
|                   |
|  Local Computer   |
|                   |
|                   |
|                   |
+-----------+-------+                                 
            | 127.0.0.1:22122
            |
            |
            |
            | 
+-----------+-------+          +---------------------------+
|                   |          |                           |
|  mole_ssh         |          |  mole_http                |
|  SSH Server (:22) |          |  HTTP Server #1 (:80)     |
|  (192.168.33.10)  |----------|  HTTP Server #2 (:8080)   |
|                   |          |  (192.168.33.11)          |
|                   |          |                           |
+-------------------+          +---------------------------+
```

## Required Software

You will need the following software installed on your computer to build this
test environment:

* [Docker](https://docs.docker.com/install/)
* ssh
* ssh-keygen
* [Wireshark](https://www.wireshark.org/download.html) (optional)

## Managing the Environment

### Setup

```sh
$ make test-env
```

This builds two docker containers: `mole_ssh` and `mole_http` with a local
network (192.168.33.0/24) that connects them.

`mole_ssh` runs a ssh server listening on port `22`.
This port is published on the local computer using port `22122`, so ssh
connections can be made using address `127.0.0.1:22122`.
All ssh connection to `mole_ssh` should be done using the user `mole` and the
key file located on `test-env/key`
The ssh server is configured to end any connections that is idle for three (3)
or more seconds.

`mole_http` runs two http servers listening on port `80` and `8080`, so clients
would be able to access the using the following urls: `http://192.168.33.11:80/`
and `http://192.168.33.11:8080/`
It also publishes the port `8080` to the host machine to be used as a web server
to test remote port forwarding.

### Teardown

```sh
$ make rm-test-env
```

This will destroy both of the containers that was built by running
`make test-env`: `mole_ssh` and `mole_http`.

The ssh authentication key files, `test-env/key` and `test-env/key,pub` will
**not** be deleted.

## How to use the test environment and mole altogether

### Local Port Forwarding

```sh
$ make test-env
<lots of output messages here>
mole start local \
  --verbose \
  --insecure \
  --source :21112 \
  --source :21113 \
  --destination 192.168.33.11:80 \
  --destination 192.168.33.11:8080 \
  --server mole@127.0.0.1:22122 \
  --key test-env/ssh-server/keys/key \
  --keep-alive-interval 2s
DEBU[0000] using ssh config file from: /home/mole/.ssh/config
DEBU[0000] server: [name=127.0.0.1, address=127.0.0.1:22122, user=mole]
DEBU[0000] tunnel: [channels:[[source=127.0.0.1:21112, destination=192.168.33.11:80] [source=127.0.0.1:21113, destination=192.168.33.11:8080]], server:127.0.0.1:22122]
DEBU[0000] connection to the ssh server is established   server="[name=127.0.0.1, address=127.0.0.1:22122, user=mole]"
DEBU[0000] start sending keep alive packets
INFO[0000] tunnel channel is waiting for connection      destination="192.168.33.11:8080" source="127.0.0.1:21113"
INFO[0000] tunnel channel is waiting for connection      destination="192.168.33.11:80" source="127.0.0.1:21112"
$ curl localhost:21112
:)
$ curl localhost:21113
:)
```

NOTE: If you're wondering about the smile face, that is the response from both 
http servers.

### Remote Port Forwarding

```sh
$ mole start remote \
    --verbose \
    --insecure \
    --source 192.168.33.11:9090 \
    --destination 127.0.0.1:8080 \
    --server mole@127.0.0.01:22122 \
    --key test-env/ssh-server/keys/key \
    --keep-alive-interval 2s
DEBU[0000] using ssh config file from: /home/mole/.ssh/config
DEBU[0000] server: [name=127.0.0.01, address=127.0.0.01:22122, user=mole]
DEBU[0000] tunnel: [channels:[[source=192.168.33.11:9090, destination=127.0.0.1:8080]], server:127.0.0.01:22122]
DEBU[0000] connection to the ssh server is established   server="[name=127.0.0.01, address=127.0.0.01:22122, user=mole]"
DEBU[0000] start sending keep alive packets
INFO[0000] tunnel channel is waiting for connection      destination="127.0.0.1:8080" source="192.168.33.11:9090"
DEBU[0001] connection established                        channel="[source=192.168.33.11:9090, destination=127.0.0.1:8080]"
DEBU[0001] tunnel channel has been established           channel="[source=192.168.33.11:9090, destination=127.0.0.1:8080]" server="[name=127.0.0.01, address=127.0.0.01:22122, user=mole]"
$ docker exec mole_ssh curl -s localhost:9090
:)
```

## How to manage the ssh server instance

```sh
$ docker exec -ti mole_ssh supervisorctl <stop|start|restart> sshd
```

## How to force mole to reconnect to the ssh server

1. Create the mole's test environment

```sh
$ make test-env
<lots of output messages here>
```

2. Start mole

```sh
mole start local \
  --verbose \
  --insecure \
  --source :21112 \
  --source :21113 \
  --destination 192.168.33.11:80 \
  --destination 192.168.33.11:8080 \
  --server mole@127.0.0.1:22122 \
  --key test-env/ssh-server/keys/key \
  --keep-alive-interval 2s
DEBU[0000] using ssh config file from: /home/mole/.ssh/config
DEBU[0000] server: [name=127.0.0.1, address=127.0.0.1:22122, user=mole]
DEBU[0000] tunnel: [channels:[[source=127.0.0.1:21112, destination=192.168.33.11:80] [source=127.0.0.1:21113, destination=192.168.33.11:8080]], server:127.0.0.1:22122]
DEBU[0000] connection to the ssh server is established   server="[name=127.0.0.1, address=127.0.0.1:22122, user=mole]"
DEBU[0000] start sending keep alive packets
INFO[0000] tunnel channel is waiting for connection      destination="192.168.33.11:8080" source="127.0.0.1:21113"
INFO[0000] tunnel channel is waiting for connection      destination="192.168.33.11:80" source="127.0.0.1:21112"
```

3. Kill all ssh processes running on the container holding the ssh server

The container will take care of restarting the ssh server once it gets killed.

```sh
$ docker exec mole_ssh pgrep sshd
8
15
17
$ docker exec mole_ssh kill -9 8 15 17
```

4. The mole's output should be something like the below

```sh
WARN[0019] reconnecting to ssh server                    error=EOF
INFO[0022] tunnel channel is waiting for connection      local="127.0.0.1:21113" remote="192.168.33.11:8080"
INFO[0022] tunnel channel is waiting for connection      local="127.0.0.1:21112" remote="192.168.33.11:80"
```

5. Validate the tunnel is still working

```sh
$ curl 127.0.0.1:21112; curl 127.0.0.1:21113
:)
:)
```

## How to check ssh server logs

```sh
$ docker exec mole_ssh tail -f /var/log/messages
```

## Packet Analisys

If you need to analyze the traffic going through the tunnel, the test
environment provide a handy way to sniff all traffic following the steps below:

```sh
$ make test-env
<lots of output messages here>
$ bash test-env/sniff
tcpdump: listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
```

Wireshark will open and should show all traffic passing through the tunnel.
