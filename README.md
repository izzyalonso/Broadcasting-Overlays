# Broadcasting-Overlays

Checkpoint for the work done during the Spring '21 semester at Pitt.

## Overview

This repo contains a sample p4 file, some instructions on how to get a setup going, and an
explanation on how to get the program to multicast.

## Getting Started

To get started, follow the instructions in
[this link](https://p4.org/p4/getting-started-with-p4.html). When you're done, you'll have a
functioning softswitch. For the purpose of this project I used their three port setup where each
port is connected to a virtual interface.

## The P4 File

The [P4 file]() was using for testing, and thus it only supports UDP, but it could be easily
extended to work with TCP as well. There are two important parts in this program, ingress and
egress. Let's start with ingress. Ingress decides what to do with incoming traffic. In the final
implementation there were going to be 3 things ingress could do:

- Ignore a packet: performs no action, we have no business with this packet, forward as usual.
- Reroute packet: send the packet to a local agent when there's a flow to satisfy.
- **Multicast packet**: send the packet to egress twice with two different egress ports.
- Fake source: make the client believe a packet is coming from its intended destination when in fact
it came from the agent.

Out of all this functionality, we ended up implementing multicast exclusively. The multicast group
is configurable. A packet will be multicast(ed?) to a multicast group based on the destination
address.

Egress' job is to detect duplicate packets and change one of their intended destinations to a
backup spines location. The source and destination addresses are not configurable as of yet.
Instead, we redirect all packets in a configurable egress port to an arbitrary hardcoded address.

## Using the CLI to Configure the Tables

Once you have the switch up and running, you will be able to use the CLI (`simple_switch_CLI`) to
add entries to the ingress and egress tables as well as creating multicast groups.

### Ingress

The first thing we need to do is add a multicast group. A multicast group is associated with a
multicast node, which contains information about which egress ports to multicast the packet to.
To do this we run the following commands:

- `mc_node_create 7 1 2`, where the first number (7) is the rid of the node (I never needed this
value), and any subsequent numbers will be the egress ports to multicast to; 1 and 2 in our case.
This call will return a handle in a message saying `node was created with handle X`. Take note of
the value of X, as you'll need it later.
- `mc_mgrp_create 1`, where the 1 is the multicast group ID. Note that you can choose what this
number is. This is the number the p4 file uses to determine how to multicast a packet.
- `mc_node_associate 1 X`, where the 1 is the number you chose for the multicast group and the X
is the number the `mc_create_node` call returned.

Second, we need to associate an address to a multicast group. This is to say, every time a packet
with a specific destination address passes through ingress, we need to set a multicast group for
the packet. Our P4 program uses an LPM match on an address. To add an entry to the table, run the
following command:

`table_add ipv4_match multicast <ip_address> => 1`

Where ipv4_match is the name of our table, see the ingress block, `multicast` is the name of the
action that will multicast a packet, `<ip_address>` is the destination address we're matching for
multicasting, and the 1 at the end is the multicast group. The 1 will be passed as a parameter to
the `multicast` action.

Once you get to this point, you'll get two packets on egress when you send a packet to the
address you specified.

### Egress

Egress is a little more simple. What it does is looking for a specific egress port in a packet
and redirects it. Egress also has a table to do this, which I uncreatively named `ipv4_match2`.
To add an entry to this table, run the follwoign command:

`table_add ipv4_match2 redirect 2 =>`

Where the 2 after `redirect`, the name of the action, is the egress port whose destination
addresses we want to change. Nore that the redirect action does not take parameters, but you
still need to add the trailing `=>` at the end; otherwise the CLI will bark at you.

## Testing

I recommend you use wireshark to observe the program's behavior.

## Warnings

If you're going to use Ubuntu and a VM, I recommend you give it at least 30 to 35GB of disk. I ran
into space issues almost constantly with a 25GB machine.

Enjoy!

