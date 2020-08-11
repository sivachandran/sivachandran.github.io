---
layout: post
title: "Load Balancing iPerf3 Servers"
featured: true
author: sivachandran
comments: true
tags: [ linux ]
---

Recently I had to set up a TCP load balancer for iperf3 server to allow simultaneous tests from multiple iperf3 clients. [iperf3][1] is a tool to measure network performance, and I used [Balance][2] as the load balancer. This post describes the need and steps to run iperf3 servers behind a TCP load balancer running on the same host.

It all started with the need to measure 4G internet bandwidth at strategic locations across India. I quickly wrote a bash script to run iperf3 client against an iperf3 server hosted by us. The bandwidth measurement results are uploaded to another server in the cloud. A `systemd` timer periodically runs the script.

Everything worked as expected in the beginning. But, soon we started seeing many occurrences of the error `iperf3: error - the server is busy running a test. try again later` all over our results. My investigation reveals that by design, iperf3 server doesn't allow simultaneous tests from more than one client: the actual reason unknown, maybe historic design decision?

The conventional solution to such problem is to run iperf3 server on multiple machines/containers and expose them through a layer 4 load balancer like AWS NLB. But I didn't choose that approach due to the additional cost and complexity. Instead, I chose to use a simple TCP load balancer called [Balance][2].

## The setup

The plan is simple, run multiple instances/processes of iperf3 server in the same host but on different ports. Then we run Balance specifying the load balancing port and the target hosts. In our case, the hostname of our target hosts are `localhost` as both iperf3 servers and the load balancer are running on the same machine. So only the difference in the target hosts is the port. The command-lines for the setup are below.

```bash

$ iperf3 -s -p 2101 &
$ iperf3 -s -p 2102 &
$ iperf3 -s -p 2103 &
$ iperf3 -s -p 2104 &
$ iperf3 -s -p 2105 &
$ balance -f 2100 localhost:2101 localhost:2102 localhost:2103 localhost:2104 localhost:2105

```

Once the setup is ready, I tried testing with iperf3 client running on my laptop. The client connected, but it didn't progress with the testing. The client simply hung. I could also witness the client connection in the servers. But strangely, two servers reported the same client connection. Bit of a web search, I found this [iperf3 issue][4]. Seems iperf3 uses two connections for its operation: one for control, one for data. In the load balancing setup, each of this connection ended up in different iperf3 servers as they balanced on a round-robin basis. This explains why the two iperf3 servers reported the same client connection and why the test didn't progress.

I know AWS load balancers allow us to configure sticky sessions for such use case. Balance doesn't seem to provide similar functionality. But it does support hash-based balancing which uses hashed client address to determine the target host. Though it is not same as a sticky session, it satisfies our need for using the same target host for all connections of a client. The command-line option to enable hash balancing is `%`. So the revised command-line looks like below.

```bash
...
$ balance -f 2100 localhost:2101 localhost:2102 localhost:2103 localhost:2104 localhost:2105 %
```

## Putting it together

Though the above command-lines do the job, they don't provide the flexibility to run/stop them quickly. So I crafted the following bash script to run them with a single command and stop them with __Ctrl+C__ key press. I hope it would be useful for someone.

```bash

#!/bin/bash

trap "kill 0" EXIT

lb_port=5100
port_start=5101
port_end=5105
hosts=""

for (( port=$port_start; port<=$port_end; port++ ))
do
    echo "start iperf3 server on port $port..."
    iperf3 -s -p $port &
    
    hosts="$hosts localhost:$port"
done

```

## Links

1.  [iperf3 project page][1]
2.  [Balance project page][2]
3.  [Balance man page][3]

_This post is originally published [here](https://engineering.qubecinema.com/2020/08/08/load-balancing-iperf3-servers.html)_

[1]: https://iperf.fr/
[2]: https://balance.inlab.net/
[3]: https://linux.die.net/man/1/balance
[4]: https://github.com/esnet/iperf/issues/823
