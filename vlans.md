# VLANs

I've seen lots of confusion around what a VLAN is while browsing various sub-reddits.  The idea is simple at its core, but there's plenty of complexity invovled, and router/switch UIs are generally confusing, which doesn't help things; I'm not surprised that folks get confused.  This is my attempt at explaining the concepts, so that folks can figure out what's really going on.

I'm going to gloss over some details and make some simplifying assumptions.  I'm going to focus on a home network, as if you're reading this for anything enterprise, you should probably hire help.  If someone spots something factually incorrect, let me know.  If you want to go into more detail, read Stevens.

## LAN

Before talking about VLANs, we need to talk about LANs.  The LAN is the local part of the network and there are lots of technical definitions, but I'll just assume it's everything in your house.  You probably have a modem to talk to the internet, but I'll ignore that for now.  Within your house you have a switch.  You may have  a consumer device called a "router", but in some cases that's a misnomer.  I'll come back to routing.  You also have wifi, but I'll start with talking about wired only.

## Layers

You'll hear reference to "layers" in networking a lot.  This refers to the OSI model.  You can look it up.  I'll be talking mostly about layer 2 and layer 3/4, with a bit of layer 7.  If you're a software engineer, you spend most of your time in layer 7, with a bit of layer 3/4.

## Layer 2: switches and MACs

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
192.168.1.228 dev eth0 lladdr b4:b6:86:85:5e:6e STALE
192.168.1.151 dev eth0 lladdr 84:d6:d0:5c:35:e4 STALE
192.168.1.196 dev eth0 lladdr b4:b6:86:85:5e:6e STALE
192.168.1.1 dev eth0 lladdr fc:aa:14:ad:b2:08 REACHABLE
192.168.1.6 dev eth0 lladdr bc:5f:f4:f8:d2:b3 STALE
```

The first column is the IP address, the second is the local device, the third is the MAC address (which linux calls lladdr, or link-local address), and the fourth is the state of the entry.  Entries expire over time and the kernel treats the various states differently.  I won't go into these details.

When the kernel wants to send a packet to a particular IP, it wraps the packet in one or more ethernet frames and sets the dst MAC to the value found in this table.  If there is no entry in this table, the kernel has to learn the MAC, so it sends out an ARP message.

```
$ sudo tcpdump -i eth0 -nn arp
11:27:07.883543 ARP, Request who-has 192.168.1.2 tell 192.168.1.1, length 46
11:27:07.883563 ARP, Reply 192.168.1.2 is-at b4:2e:99:ef:fd:4a, length 28
```

An ARP "who-has" message is an ethernet frame sent to the broadcast MAC, so all machines receive it.  Inside the message, is a request for a particular IP and which IP is asking.  If a particular host has that IP, it sends back a unicast response, which effectively says: "hey, you were looking for this IP, I've got it and this is my MAC".  The host that wanted to send a packet to that IP can update its neighbours table and can now send the ethernet frame.

One interesting detail of this approach is that generally hosts send ARP requests and responses and both traverse the switch.  Therefore a switch can just passively watch ARP traffic as it does the appropriate forwarding and can use this to build up its own table.

## Layer 3: routers and IPs

Within an ethernet frame, we have an IP packet.  IP packets have a src and dst IP (either v4 or v6) and can be routed across networks.  For now, we'll stick to our simple home network.

### CIDRs

I'll talk about v4 for now.  Each IP is a 32 bit number.  Every host on the same network has the same set of leading bits in the IP address.  This is written as `192.168.1.0/24`.  The `/24` means that the first 24 bits (ie 3 \* 8 bits, or first three "octets") define the network, and the remaining bits define the host within the network.  You may have heard of class A/B/C networks, these correspond to `/8`, `/16`, and `/24`, although classes are no longer used.  The prefix can be of any length, but must be a smaller prefix than /24 to be routed on the internet.  RFC1918 defines special prefixes, including `192.168.0.0/16` as "non-routable", which means that those packets cannot be routed onto the internet.  This is why most home networks use a prefix from RFC1918 space.

Typically, the lowest non-zero address in a range is the router, so in `192.168.1.0/24`, that would be `192.168.1.1`.  This is just a convention and the router could have an IP.  Your host in the network probably has a different IP.

### Routing

When the kernel routes a packet off the box, it does a lookup in the routing table to determine which network to use.  Here's my routing table:

```
$ ip -4 route show
default via 192.168.1.1 dev eth0 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.2 
```

The dst of the packet is compared to all the rules in the routing table and the *most specific* prefix is matched.  Most specific means the longest prefix.  So a `/24` would win over a `/22`, and a `/8` would win over a `/0` (the default route is really `0.0.0.0/0`, ie *any* IP).  So, when I send a packet to another host on my local network, ie `192.168.1.101`, the kernel will send that out on the device `eth0.1` and use the src of `192.168.1.2`.  For any other packet, the packet will be routed "via" `192.168.1.1`, which happens to be a router.  The router is the "gateway" to reaching other networks and it knows how to forward the packet appropriately.  More on routing later.

### Address Assignment

On an IPv4 network, IP addresses are typically assigned by a DHCP server.  IPv6 works differently and I'll talk about that elsewhere.  When a host wants an IP address, it sends a DHCP request packet (using UDP) to the broadcast IP address (also using the broadcast MAC).  Hopefully a DHCP server on the network receives that request, it then looks at the IP range it's been configured to manage, selects an IP from that range, and sends a response back.  The response uses the address to be assigned, which will be on the local network, but that the host doesn't know about yet.  But crucially, the ethernet frame dst is the MAC of the NIC where the request came from.  This ensures that the response gets to the requester.  The DHCP response probably also includes information like: which DNS server to use, what DNS domain the machine is part of, what the gateway is, etc.

## A Second Network

Now let's imagine we want to add a second network to the house for guests.  As we have sensitive data on our network (perhaps family photos, or old tax records), we want to completely separate guests from our internal network.  The simplest thing we can do is purchase a second switch and keep everything isolated.  Because the guest network has distinct hardware, all the MACs will be different.  But we need to assign an IP range.  As the two networks are totally separate, we could give them both the same IP range, and things would work just fine.  Each host would only see one instance of `192.168.1.1` and would have a single MAC associated with that.

But what if we wanted to share a device?  An example of this might be a printer that we want to allow guests to use.  The "simplest" thing would be to have the printer connected to a print server machine that has two NICs, one physically connected to each network.

Let's continue with our assumption of both networks using `192.168.1.0/24`.  Say, the printer NIC on the main network is assigned `192.168.1.101` and the printer NIC on the secondary network is assigned `192.168.1.202`.  These are coincidental assignments by DHCP servers on each network.  The rest of the example holds if the IPs are the same, but it's just easier to talk about if the IPs are different.

Now say a host on the main network talks to the printer.  It sets the dst IP of the packet to `192.168.1.101` and the src to itself `192.168.1.2`.  It consults the routing table and finds that the dst is on the local network and it should use device eth0 to send the packet.  It consults its neighbours table and finds the MAC (and maybe updates the table by sending an ARP who-has message).  The packet leaves the machine and hits the switch.  The switch does a lookup of the MAC and forwards the packet out the appropriate port.  The printer receives the frame, sees that the dst MAC is itself, so delivers the IP packet.  The IP packet is also destined to it, so the request is processed.

The printer does stuff and then generates a response packet.  It flips the src and dst IP, then does a route table lookup.  But this is what its route table looks like (for the pedants, I've hand edited this, as I know this isn't quite possible):

```
$ ip -4 route show
default via 192.168.1.1 dev eth0 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.101
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.202
```

As the prefixes match and are equivalent, the response packet could go out to either network.  And that's a big problem.

When you have a single entity that needs to be on multiple networks, you must setup non-overlapping network ranges.  This means that if your main network is `192.168.1.0/24`, then your guest network must be something else, like `192.168.2.0/24` or `10.12.0.0/16`.  
