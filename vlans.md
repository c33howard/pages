# VLANs

I've seen lots of confusion around what a VLAN is while browsing various sub-reddits.  The idea is simple at its core, but there's plenty of complexity invovled, and router/switch UIs are generally confusing, which doesn't help things; I'm not surprised that folks get confused.  This is my attempt at explaining the concepts, so that folks can figure out what's really going on.

I'm going to gloss over some details and make some simplifying assumptions.  I'm going to focus on a home network, as if you're reading this for anything enterprise, you should probably hire help.  If someone spots something factually incorrect, let me know.  If you want to go into more detail, read Stevens.

## LAN

Before talking about VLANs, we need to talk about LANs.  The LAN is the local part of the network and there are lots of technical definitions, but I'll just assume it's everything in your house.  You probably have a modem to talk to the internet, but I'll ignore that for now.  Within your house you have a switch.  You may have  a consumer device called a "router", but in some cases that's a misnomer.  I'll come back to routing.  You also have wifi, but I'll start with talking about wired only.

## Layers

You'll hear reference to "layers" in networking a lot.  This refers to the OSI model.  You can look it up.  I'll be talking mostly about layer 2 and layer 3, with a bit of layer 7.  If you're a software engineer, you spend most of your time in layer 7, with a bit of layer 3.

### Layer 2: switches and MACs

So, you have a switch.  It has multiple devices connected to it via ethernet cables.  Switches pass ethernet frames around.  A frame is *not* a packet, although the two often do map 1:1.  A frame also does not have any IP information in it.  An ethernet frame has source (src) and destination (dst) MAC addresses.  Each ethernet card has a MAC, which you can think of as a serial number installed into the network card (NIC).  When a frame is put on the wire, the dst on the frame is set to the MAC of the target NIC.  All a switch does is store a lookup table of MACs to ports.  When the switch receives a frame, it does a lookup in its table for the dst MAC and sends the frame out the associated port.  When a NIC receives a frame, it accepts frames where the dst is its MAC, and drops other frames.

There's a special MAC address (`ff:ff:ff:ff:ff:ff`) that is the broadcast MAC.  When a switch receives a frame with this MAC as the dst, it forwards it to all ports.  A host that receives a frame where the dst is the broadcast MAC handles the frame.

A note about terms:

* broadcast: send to everyone
* unicast: send to a single, specific target
* anycast: send to a single endpoint, but multiple hosts are potential endpoints
* multicast: send to everyone of a set of registered listeners

This leads to two questions:

* How does a machine know what MAC to put in the dst of an ethernet frame?
* How does a switch build up its internal forwarding table?

### ARP

Regular software communicates with either IPv4 or IPv6 addresses.  The OS maintains a table that maps IPs *on the local network* to MACs.  Here's a snippet from my linux machine at the moment:

```
$ ip -4 neigh
192.168.1.228 dev eth0.1 lladdr b4:b6:86:85:5e:6e STALE
192.168.1.151 dev eth0.1 lladdr 84:d6:d0:5c:35:e4 STALE
192.168.1.196 dev eth0.1 lladdr b4:b6:86:85:5e:6e STALE
192.168.1.1 dev eth0.1 lladdr fc:aa:14:ad:b2:08 REACHABLE
192.168.1.6 dev eth0.1 lladdr bc:5f:f4:f8:d2:b3 STALE
```

The first column is the IP address, the second is the local device, the third is the MAC address (which linux calls lladdr, or link-local address), and the fourth is the state of the entry.  Entries expire over time and the kernel treats the various states differently.  I won't go into these details.

When the kernel wants to send a packet to a particular IP, it wraps the packet in one or more ethernet frames and sets the dst MAC to the value found in this table.  If there is no entry in this table, the kernel has to learn the MAC, so it sends out an ARP message.

```
$ sudo tcpdump -i eth0.1 -nn arp
11:27:07.883543 ARP, Request who-has 192.168.1.2 tell 192.168.1.1, length 46
11:27:07.883563 ARP, Reply 192.168.1.2 is-at b4:2e:99:ef:fd:4a, length 28
```

An ARP "who-has" message is an ethernet frame sent to the broadcast MAC, so all machines receive it.  Inside the message, is a request for a particular IP and which IP is asking.  If a particular host has that IP, it sends back a unicast response, which effectively says: "hey, you were looking for this IP, I've got it and this is my MAC".  The host that wanted to send a packet to that IP can update its neighbours table and can now send the ethernet frame.

One interesting detail of this approach is that generally hosts send ARP requests and responses and both traverse the switch.  Therefore a switch can just passively watch ARP traffic as it does the appropriate forwarding and can use this to build up its own table.

### Layer 3: routers and IPs

