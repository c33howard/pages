# VLANs

I've seen lots of confusion around what a VLAN is while browsing various sub-reddits.  The idea is simple at its core, but there's plenty of complexity invovled, and router/switch UIs are generally confusing, which doesn't help things; I'm not surprised that folks get confused.  This is my attempt at explaining the concepts, so that folks can figure out what's really going on.

I'm going to gloss over some details and make some simplifying assumptions.  I'm going to focus on a home network, as if you're reading this for anything enterprise, you should probably hire help.  If someone spots something factually incorrect, let me know.  If you want to go into more detail, read Stevens.

## LAN

Before talking about VLANs, we need to talk about LANs.  The LAN is the local part of the network and there are lots of technical definitions, but I'll just assume it's everything in your house.  You probably have a modem to talk to the internet, but I'll ignore that for now.  Within your house you have a "router".  I use quotes, because what folks call routers are not exactly what network engineers call routers.  Part of your home "router" is a switch and part is a wireless (wifi) access point.  I'll come back to both of these, but I'm going to start with just the switch and I'll assume everything is wired for now.

## Layers

You'll hear reference to "layers" in networking a lot.  This refers to the OSI model.  You can look it up.  The short version is that layer 1 is the physical layer and is how bits move from one place to another: wires or wireless.  The next layer is the link layer, it's more or less everything directly connected within your home.  Layer three is the network layer and it's how you can address any endpoint on the internet.  Layer four is how you can transport data to another machine.  Layers five and six are mostly ignored these days.  Layer seven is the application layer and it's how applications interpret the data.  I'll mostly be talking about layers two and three.

## Layer 2: switches and MACs

So, you have a switch.  It has multiple devices connected to it via ethernet cables.  Switches pass ethernet "frames" around.  A frame is *not* a packet, although the two often do map 1:1.  A frame also does not have any IP information in it.  An ethernet frame has source (src) and destination (dst) MAC addresses.  Each ethernet card has a MAC, which you can think of as a serial number installed into the network card (NIC).  When a frame is put on the wire, the dst on the frame is set to the MAC of the target NIC.  All a switch does is store a lookup table of MACs to ports.  When the switch receives a frame, it does a lookup in its table for the dst MAC and sends the frame out the associated port.  When a NIC receives a frame, it accepts frames where the dst is its MAC, and drops other frames.

There's a special MAC address (`ff:ff:ff:ff:ff:ff`) that is the broadcast MAC.  When a switch receives a frame with this MAC as the dst, it forwards it to all ports.  A host that receives a frame where the dst is the broadcast MAC accepts the frame.

A note about terms:

* unicast: send to a single, specific target
* anycast: send to a single endpoint, but multiple hosts are potential endpoints (think of a call center: you don't care which specific agent you talk to, but you want to talk to a single person)
* multicast: send to everyone of a set of registered listeners
* broadcast: send to everyone

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

`192.168.1.1 dev eth0 lladdr fc:aa:14:ad:b2:08 REACHABLE` reads as: for IP `192.168.1.1` on device `eth0`, this is a `lladdr` (link-local address), with MAC `fc:aa:14:ad:b2:08` and it's currently `REACHABLE`.  Entries in the ARP table expire over time and the kernel treats the various states differently.  I won't go into these details.

When the kernel wants to send a packet to a particular IP, it wraps the packet in one or more ethernet frames and sets the dst MAC to the value found in this table.  If there is no entry in this table, the kernel has to learn the MAC, so it sends out an ARP message.

```
$ sudo tcpdump -i eth0 -e -nn arp
00:01:56.446493 b4:2e:99:ef:fd:4a > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 42: Request who-has 192.168.1.235 tell 192.168.1.2, length 28
00:01:56.610605 6e:36:93:f5:4f:dc > b4:2e:99:ef:fd:4a, ethertype ARP (0x0806), length 60: Reply 192.168.1.235 is-at 6e:36:93:f5:4f:dc, length 46
```

An ARP "who-has" message is an ethernet frame sent to the broadcast MAC, so all machines receive it.  Inside the message, is a request for a particular IP and which IP is asking.  If a particular host has that IP, it sends back a unicast response, which effectively says: "hey, you were looking for this IP, I've got it and this is my MAC".  The host that wanted to send a packet to that IP can update its neighbours table and can now send the ethernet frame.

One interesting detail of this approach is that generally hosts send ARP requests and responses and both traverse the switch.  Therefore a switch can just passively watch ARP traffic as it does the appropriate forwarding and can use this to build up its own table.  If a switch is asked to forward an entry for which it doesn't have an ARP entry, it can send out an ARP request, drop the frame, or send the frame to all ports (ie broadcast it).

## Layer 3: routers and IPs

Within an ethernet frame, we have an IP packet (either v4 or v6).  IP packets have a src and dst IP and can be routed across networks.  For now, we'll stick to our simple home network.

### CIDRs

I'll talk about v4 for now.  Each IP is a 32 bit number.  Every host on the same network has the same set of leading bits in the IP address.  This is written as `192.168.1.0/24`.  The `/24` means that the first 24 bits (ie 3 \* 8 bits, or first three "octets") define the network, and the remaining bits define the host within the network.  You may have heard of class A/B/C networks, these correspond to `/8`, `/16`, and `/24`, although classes are no longer used; instead just say "slash 24".  The prefix can be of any length, but for a home network, you probably want a /24, which gives you 8 bits worth of hosts or 256 IPs.  RFC1918 defines special prefixes, including `192.168.0.0/16` as "non-routable", which means that those packets cannot be routed onto the internet.  This is why most home networks use a prefix from RFC1918 space.  Within `192.168.0.0/16` there are 256 networks of size `/24`.  Two of the most common are `192.168.0.0/24` and `192.168.1.0/24`.  But there's nothing special about those; you can use whatever you'd like within that range, or the two other larger ranges in RFC1918 (`10.0.0.0/8` and `172.16.0.0/12`).

Typically, the lowest non-zero address in a range is the router, so in `192.168.1.0/24`, that would be `192.168.1.1`.  This is just a convention and the router could have an IP.  Your host in the network probably has a different IP.

### Route Table

When the kernel routes a packet off the box, it does a lookup in the routing table to determine which network to use.  Here's my routing table:

```
$ ip -4 route show
default via 192.168.1.1 dev eth0 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.2 
```

The dst of the packet is compared to all the rules in the routing table and the *most specific* prefix is matched.  In this case a "match" is determined by taking the dst IP (say `192.168.1.3`) and, for each routing table entry, taking the prefix length and masking off those bits.  Another way to write `/24` is `255.255.255.0`, so `192.168.1.3 & 255.255.255.0 == 192.168.1.0`.  The result matches the `192.168.1.0/24` in the route table entry, so this is a valid route.  Note that default route can also be written as `0.0.0.0/0`.  And any IP address masked with this is `0.0.0.0`, which is a match, so the default is always an option.  In this example, the `/24` is a more specific match than `/0`, as `24 > 0`, so that's the route that is selected.

This shows that, when I send a packet to another host on my local network, ie `192.168.1.3`, the kernel will send that out on the device `eth0` and use the src of `192.168.1.2`.  For any other packet, the packet will be routed "via" `192.168.1.1`, which happens to be a router.  The router is the "gateway" to reaching other networks and it knows how to forward the packet appropriately.  More on routing later.

### Address Assignment

On an IPv4 network, IP addresses are typically assigned by a DHCP server.  IPv6 works differently and I'll talk about that elsewhere.  When a host wants an IP address, it sends a DHCP request packet (using UDP) to the broadcast IP address (also using the broadcast MAC).  Hopefully a DHCP server on the network receives that request, it then looks at the IP range it's been configured to manage, selects an IP from that range, and sends a response back (reminder, this document is hand-wavy; I'm not going into detail about the DHCP protocol).  The response uses the address to be assigned, which will be on the local network, but that the host doesn't know about yet.  Crucially, the ethernet frame dst is the MAC of the NIC where the request came from.  This ensures that the response gets to the requester.  The DHCP response probably also includes information like: which DNS server to use, what DNS domain the machine is part of, what the gateway is, etc.

```
$ sudo tcpdump -i eth0 -e -nn port 67 or port 68
00:20:30.594658 b4:2e:99:ef:fd:4a > ff:ff:ff:ff:ff:ff, ethertype IPv4 (0x0800), length 342: 0.0.0.0.68 > 255.255.255.255.67: BOOTP/DHCP, Request from b4:2e:99:ef:fd:4a, length 300
00:20:30.595038 fc:aa:14:ad:b2:08 > b4:2e:99:ef:fd:4a, ethertype IPv4 (0x0800), length 342: 192.168.1.1.67 > 192.168.1.2.68: BOOTP/DHCP, Reply, length 300
```

You can, of course, also statically assign IPs.  There are two ways to do this: each host on the network can just manually bring up a network interface with a configured IP.  Hopefully you've picked an unused IP for that machine.  Or, your DHCP server can be configured to always hand out a specific IP address when it sees a DHCP request from a particular MAC.  This second option tends to be more scalable.

### Aside: Duplicate IPs

As an aside, let's think through what happens if two hosts (A, and B) on the same network have the same IP, say `192.168.1.2`.  They both want to use the printer at `192.168.1.3`.  Host A wants to send the first packet to the printer, so it looks up the route and finds that the printer is on the same network and it can send directly.  It then consults its ARP table to determine the dst MAC for the ethernet frame.  There's no entry, so it sends an ARP who-has `192.168.1.3` message to the broadcast MAC.  The printer responds with a reply giving its MAC and IP.  The printer can now also update its ARP table with `192.168.1.2` and the MAC of host A.  Host A can now send its giant document to the printer and packets encapsulated in frames flow between the two.

Meanwhile, host B wants to print.  It performs the same steps as host A and gets to the point where it needs to send an ARP who-has message.  The printer responds, since it has `192.168.1.3`.  Since it just got an ARP from `192.168.1.2` it updates its local ARP table and now `192.168.1.2` has the MAC of host B!  The response packets from the printer to `192.168.1.2` that host A are expecting to see are now redirected to host B.  Host A never receives response packets, so it thinks the printer is offline.  And host B sees all the response frames for host B, which it drops, because the MAC doesn't match.

As far as I'm aware, anycast isn't really a thing on local networks.  But what we've described is a way to "implement" anycast on a local network.  Anycast, as it's used on the internet, is more or less this: duplicate IPs located on different spots on the network, where routing protocols send the packets to one of the duplicates based on a criteria.

## A Second Network

Now let's imagine we want to add a second network to the house for guests.  As we have sensitive data on our network (perhaps family photos, or old tax records), we want to completely separate guests from our internal network.  The simplest thing we can do is purchase a second switch and keep everything isolated.  Because the guest network has distinct hardware, all the MACs will be different.  But we need to assign an IP range.  As the two networks are totally separate, we could give them both the same IP range, and things would work just fine.  Each host would only see one instance of `192.168.1.1` and would have a single MAC associated with that.

But what if we wanted to share a device?  An example of this might be a printer that we want to allow guests to use.  The "simplest" thing would be to have the printer connected to a print server machine that has two NICs, one physically connected to each network.

Let's continue with our assumption of both networks using `192.168.1.0/24`.  Say the printer NIC on the main network is assigned `192.168.1.101` and the printer NIC on the secondary network is assigned `192.168.1.202`.  These are coincidental assignments by DHCP servers on each network and I picked distinct IPs for readability of this example; they could be the same IPs.

Now say a host on the main network talks to the printer.  It sets the dst IP of the packet to `192.168.1.101` and the src to itself `192.168.1.2`.  It consults the routing table and finds that the dst is on the local network and it should use device eth0 to send the packet.  It consults its ARP table and finds the MAC (and maybe updates the table by sending an ARP who-has message).  The packet leaves the machine and hits the switch.  The switch does a lookup of the MAC and forwards the packet out the appropriate port.  The printer receives the frame, sees that the dst MAC is itself, so delivers the IP packet.  The IP packet is also destined to it, so the request is processed.

The printer does stuff and then generates a response packet.  It flips the src and dst IP, then does a route table lookup.  But this is what its route table looks like (for the pedants, I've hand edited this, as I know this isn't quite possible):

```
$ ip -4 route show
default via 192.168.1.1 dev eth0 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.101
192.168.1.0/24 dev eth1 proto kernel scope link src 192.168.1.202
```

As the prefixes match and are equivalent, the response packet could go out to either network.  And that's a big problem.

When you have a single entity that needs to be on multiple networks, you must setup non-overlapping network ranges.  This means that if your main network is `192.168.1.0/24`, then your guest network must be something else, like `192.168.2.0/24` or `10.12.0.0/16`.  As an aside, this is actually a non-trivial problem when a company purchases another and they merge their networks.  Both companies probably used `10.0.0.0/8` before the merger and now they have to renumber their entire network to ensure no overlaps.  IPv6 has an elegant solution to this problem, which I'll talk about later.

Here's what the route table on the printer would look like if the guest network was at `192.168.2.0/24`.

```
$ ip -4 route show
default via 192.168.1.1 dev eth0 
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.101
192.168.2.0/24 dev eth1 proto kernel scope link src 192.168.2.202
```

Now when response packets are routed, the route table can determine which NIC to use for the private network, and which for the guest network, as the IP ranges are distinct.

## Gateways and Internet Access

I've mentioned gateways, the default route, and internet access before.  How does this work?  We use a router.  A "router" is a device that straddles two networks and knows how to pass packets between the two networks.  In a home, the router forwards packets from the local network to the internet, and the response packets from the internet to the local network.  Note that this definition says nothing about wifi, despite what the marketing folks tell you.

### Public Routing

For now, let's pretend that your home isn't using RFC1918 addresses, but is instead using a public range.  We'll use the range `203.0.113.0/24`.  The router is `203.0.113.1` and your workstation is `203.0.113.2`.  The router has two NICs, one for the local network and one for the WAN (wide area network == internet).

The route table on your workstation looks like this:

```
$ ip -4 route show
default via 203.0.113.1 dev eth0
203.0.113.0/24 dev eth0 proto kernel scope link src 203.0.113.2 
```

And the route table on your router looks like this:

```
$ ip -4 route show
default via 198.51.100.1 dev wan0 
198.51.100.1/24 dev wan0 proto kernel scope link src 198.51.100.42
203.0.113.0/24 dev eth0 proto kernel scope link src 203.0.113.1
```

This shows that our internet service provider (ISP) gave us the IP `198.51.100.42`, probably via DHCP from the ISPs DHCP server.  The ISP router of `192.51.100.1` is our gateway and where we send all packets that we don't know how to directly route ourselves.  The eth0 device shows the local network and our local IP.

On our workstation, we now want to do a DNS lookup and we're using Google's public DNS of `8.8.8.8`.  Here's the flow:

1. First we create a packet with a dst of `8.8.8.8` and src of `203.0.113.2`.
2. We look at our route table, and see that 8.8.8.8 only matches the default route, so we send that packet via `203.0.113.1`, which is our router.
3. We look in our ARP table for `203.0.113.1`, send an ARP who-has if necessary, but then see the MAC of the eth0 NIC on the router.
4. We send the frame from our workstation to the router (it may traverse a switch) with a MAC dst of the router and src MAC of ourself.
5. The router receives the frame, sees that the dst MAC is itself and processes the packet.
6. The router looks at the dst IP and sees `8.8.8.8` which isn't itself, and since it's a router, decides to forward the packet.
7. The router consults its routing table and the only match is the default route, so the packet is sent via `198.51.100.1`, which the ISP router.
8. The router looks is its ARP table for `198.51.100.1` and finds the MAC of the ISP router.
9. The frame is sent to the ISP router with the MAC dst of the ISP router we got from our ARP table and the src MAC of wan0 on the router.

This process repeats many times between us and `8.8.8.8` and the reverse happens with the response packet.  You can see the routers between you and a destination by using the traceroute tool.  On your machine, try `traceroute 8.8.8.8`.  (Note that some routers have disabled traceroute packets, so you may see blank lines.  The flag `-n` shows you the raw IPs, instead of the hostnames.)

### Private Routing and NAT

The previous example worked because we were using public IPs.  But we use RFC1918 addresses at home and these aren't routable on the public internet.  (This just means that internet routers are configured to drop all packets with a dst defined by RFC1918.)  Note that this section doesn't really apply to IPv6.  With IPv6, every machine gets a publicly routable IP.  This part of IPv6 is simpler.

So, how can we still use the internet if we're using non-routable IPs in our home network?  We use network address translation (NAT).  Our router gets a single publicly routable IPv4 address from our ISP.  It then "translates" all packets that it forwards to the internet.  It has to write down all the translations it has done, so that response packets can have the reverse translation applied.  The translation is to lie to the rest of the internet and make it appear that the router originated the packet.  Then, the response packet can be addressed to the router.

