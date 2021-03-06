Ethernet multicast versus IP multicast:
=====================================================

Mostly when we talk multicast, IP (layer 3) multicast is meant and Ethernet (layer 2) multicast is taken for granted. But a clear distinction between the two must be made. They work very different and IP multicast is essentially an application on top of Ethernet multicast. This article tries to describe their properties and how they relate.

Table of contents:
------------------

* [Ethernet multicast](#Ethernet-multicast)
* [Globally unique layer 2 multicast addressing](#Globally-unique-layer-2-multicast-addressing)
* [IP multicast](#IP-multicast)
* [IP multicast addresses](#IP-multicast-addresses)
* [IP multicast address to Ethernet address mapping](#IP-multicast-address-to-Ethernet-address-mapping)
* [IGMP snooping](#IGMP-snooping)


Ethernet multicast:
-------------------
The purpose of Ethernet multicast is to allow a single Ethernet frame to be forwarded to multiple receivers (target a specific group of receivers). The mechanism behind Ethernet multicast is very rudimentary. An Ethernet multicast frame is simply flooded throughout the Ethernet (broadcast) domain. A Ethernet switch that receives a multicast frame will forward the frame out of all its switch ports, except the port the the multicast frame was received (this process is called flooding). 

> Generally this simple multicast flooding rule stays put, but as with many things it's not that simple, it depends as you will learn.

The result is that every host connected to the the broadcast domain will receive a copy of every multicast frame. No matter which host originated the multicast frame or which subset of receivers was targeted. Also notice that layer 2 multicast frames are not routed, they will not leave the broadcast domain. If a host is interested in certain multicast traffic it will program its network stack to forward these multicast packets towards the interested host application (a application can be a low level kernel process or a user messaging application, it can be any process interested in certain multicast traffic). Any other multicast traffic is simply dropped by the Ethernet host. Not only hosts but also network equipment like Ethernet switches can be interested in certain multicast traffic. Ethernet multicast subscription mechanisms like IGMP which are used for IP (layer 3) multicast do not affect Ethernet multicast. Ethernet multicast traffic is send everywhere and the host decides to tune into the group or drop the multicast frame.

Ethernet denotes unicast frames from multicast frames by the individual/group (I/G) bit in the first byte of the Ethernet destination address (MAC address). If the I/G bit is set to one, the frame is marked for multicast processing (flooding). 
> You can easily distinguish multicast frames from unicast frames: if the first byte of the destination MAC address is an even number then it is a unicast frame. If it???s an odd number then it???s a multicast frame. Setting the I/G bit within a source MAC address is not allowed according to the Ethernet standards.


![individual group bit](images/individual-group-bit.png?raw=true)


What about an Ethernet broadcast? A broadcast is a multicast, it is handled exactly the same as multicast by the Ethernet network. It is flooded throughout the Ethernet (broadcast) domain. The difference is at host side. All hosts must listen to broadcast frames (the network stack is programmed to always forward broadcasts into the hosts upper layers for processing). A broadcast is a multicast targeted to all hosts in the Ethernet (broadcast) domain. A broadcast frame is send to the reserved destination MAC address FF-FF-FF-FF-FF-FF.

> A broadcast is a multicast, any host must listen to a multicast with destination address FF-FF-FF-FF-FF-FF.

Globally unique layer 2 multicast addressing:
---------------------------------------------
But what binds a destination multicast address to a specific host application? To make multicast work any application must use its own globally unique multicast address(es). The application must know which multicast address to listen or send to, it is simply hard coded to listen to a certain Ethernet multicast address.  

A Ethernet multicast application (destination) is uniquely identified by its layer 2 multicast address(es). Globally unique multicast addresses can be requested by companies or organizations from the Institute of Electrical and Electronics Engineers (IEEE). A company or organization can request an Organizationally Unique Identifier (OUI) aka vendor block from the IEEE. The IEEE will assign a globally unique 2^25 sized address block. The first 2^24 addresses are used for globally unique unicast addresses. The last 2^24 addresses are used for globally unique multicast addresses.

For example Cisco registered the following OUI: 00-00-0C. Cisco uses the 00-00-0C-xx-xx-xx range for unicast and the 01-00-0C-xx-xx-xx range for Ethernet multicast.

There are some exceptions to the multicast flooding rule. Some Ethernet Multicast frames aren???t flooded throughout the Ethernet broadcast domain. These frames are Ethernet Multicast frames targeted towards bridges (Ethernet switches). These frames are used by various protocols like STP and LACP. A well-known Ethernet multicast range that isn't flooded by 802.1D compliant bridges is 01-80-C2.

The 01-80-C2 multicast range is used by several network protocols developed or under control of IEEE. The following two bullets are copied from the IEEE document ???Standard Group MAC Addresses: A Tutorial Guide???:

* `IEEE 802.1D MAC Bridge Filtered MAC Group Addresses`: 01-80-C2-00-00-00 to 01-80-C2-00-00-0F; MAC frames that have a destination MAC address within this range are not relayed by MAC bridges conforming to IEEE 802.1D.
* `Standard MAC Group Addresses`: 01-80-C2-00-00-10 to 01-80-C2-FF-FF-FF; MAC frames that have a destination MAC address within this range may be relayed by MAC bridges conforming to IEEE 802.1D.

But these aren't the only MAC addresses that aren???t flooded. For example CDP multicast frames (01-00-0C-CC-CC-CC) aren???t flooded, assuming the bridge understands the CDP protocol. 

IP multicast:
-------------
IP multicast utilizes the underlying Ethernet multicast mechanism. IP multicast packets are encapsulated in an Ethernet frame with the I/G bit set. Does this mean that IP multicast packets are simply flooded throughout the Ethernet broadcast domain? It depends, it depends on the scope of the IP multicast address, the IGMP snooping settings of the switches and the availability of a IGMP querier.

The advantage of IP multicast compared to Ethernet multicast is that IP multicast can cross Ethernet (Layer 2) boundaries. IP multicast was developed at the early 1980s at Stanford University. The idea was to extend the Ethernet multicast mechanism to work at the IP layer (layer 3).

The reason for IP multicast is the same than Ethernet multicast: being able to address a subset of hosts identified by a single IP (multicast) address (potentially at a global scale - which never became a reality -). A host subscribed to an IP multicast address (IP multicast group) programs the host network stack to listen to a specific MAC address that represents the IP multicast group. To make this work there must be a mapping between the IP multicast address and the MAC address.

IP multicast addresses:
-----------------------
The Internet Assigned Number Authority (IANA) assigned the class D address space for IP multicast use. The class D range represents IP multicast addresses ranging from 224.0.0.0 through 239.255.255.255.

![IP class D](images/Class-D.png?raw=true)

As you can see class D addresses start with the binary prefix 1110 which leaves room for 28^2 multicast addresses.

The class D multicast range is further subdivided in scopes, these scopes are important to know because multicast traffic in the ???link-scope??? is treated differently by the Ethernet network (if IGMP snooping is turned on) than the ???global scope??? and ???administrative scope???.

* Link scope: 224.0.0.0 - 224.0.0.255
* Global scope: 224.0.1.0 - 238.255.255.255
* Administrative scope: 239.0.0.0 - 239.255.255.255

#### Link scoped (link-local) multicast addresses (224.0.0.0 ??? 224.0.0.255):

This range is reserved for use by network protocols. Link scoped IP multicast packets will never be forwarded from the local Ethernet segment by a (multicast) router. Also, link scoped packets will always be flooded throughout the Ethernet broadcast domain. IGMP snooping will not affect flooding. 

The following list shows some examples of well-known link-local multicast addresses:

* 224.0.0.1 All Hosts multicast group.
* 224.0.0.2 All Routers multicast group.
* 224.0.0.4 DVMRP
* 224.0.0.5 All OSPF Routers

#### Globally scoped multicast addresses (224.0.1.0 ??? 238.255.255.255):

Global scoped multicast IP addresses are meant for global use (public use). Forwarding of Globally scoped multicast packets in an Ethernet network is affected by IGMP snooping.

#### Administratively scoped multicast addresses (239.0.0.0 ??? 239.255.255.255):

These multicast addresses can be used privately (RFC 2365), the idea is the same as private IP addressing described in RFC 1918. Just like global scoped addresses forwarding of these packets in an Ethernet network is affected by IGMP snooping.

IP multicast address to Ethernet address mapping:
--------------------------------------------------
Ethernet forwards IP unicast packets based on their destination MAC. A host looks up the MAC address of the target in its ARP table. For multicast such a mechanism does not exist and it would not work because a multicast sender doesn???t know which hosts are listening (the host just transmits IP multicast packets, it???s the responsibility of the network to deliver multicast packets to the interested receivers).

In IP multicast the IP multicast (group) address is encoded within the MAC address.  This encoded multicast MAC address is used by the host to program its network stack. This mechanism allows multicast applications to listen to a specific IP multicast address.

The following figure shows how the IP multicast address is encoded within the multicast MAC address:

![IP Multicast Ethernet mapping](images/IP-Multicast-Ethernet-mapping.png?raw=true)

If you search the IEEE OUI database for 00-00-5E you will see that this range was registered by the IANA. As described the IEEE assigns next to the 00-00-5E prefix the 01-00-5E prefix for multicast. IANA reserved half of the 01-00-5E multicast range for IP multicast. More specific: 01-00-5E-00-00-00 through 01-00-5E-7F-FF-FF. That is why only 23 bits are be mapped from the IP address (represented by the green boxes and arrows) not 24. The first 25 bits of the MAC address are fixed (represented by the red boxes).

Class D IP addresses start with 1110 (purple boxes) this leaves 28 bits for multicast addresses. To map the full 28 bits into the MAC address, 16 OUI ranges of the size of 2^25 would be needed. The story is told that there was not enough budget to allocate 16 OUIs (source: Developing IP Multicast Networks ISBN-10: 1-58714-289-9). So we ended up with 23 bits instead of 28.

> Personally I find such facts very interesting. Why are things are as they are? The why behind technological and architectural choices help you thoroughly understand a technology.

Because of this budget problem there is 5 bit translation error (the blue boxes). For this reason a single multicast MAC address maps to 2^5 (32) different multicast IP addresses. If there are more multicast groups active on the same VLAN that map to the same MAC address the host CPU will be interrupted unnecessary for multicast packets it did not subscribe to. The host will simply discard the received packets based upon the IP multicast addresses it did not subscribe to.

IGMP snooping:
--------------
The goal of IGMP snooping is to forward IP multicast packets only to interested receivers. The required information is snooped from IGMP protocol packets. Without IGMP snooping enabled, IP based multicast traffic would be simply flooded. With IGMP snooping enabled multicast packets with a multicast group address within the following ranges will be only sent towards interested receivers:

* Global scope: 224.0.1.0 ??? 238.255.255.255
* Administrative scope: 239.0.0.0 ??? 239.255.255.255

This article will not explain how IGMP works, this article shows how IGMP snooping controls the delivery of IP multicast packets in an IGMP snooping enabled Ethernet network.

Multicast router and group member interfaces:
---------------------------------------------
The IGMP snooping process running within an Ethernet switch builds an IGMP snooping table. This table contains information about `multicast router` and `group member` interfaces: 
* `Multicast router interface` entries describe the switch interfaces leading towards a PIM router or IGMP querier. This information is snooped from IGMP queries and/or PIM updates. 
* `Group member interface` entries describe the switch interfaces leading towards interested multicast receivers. This information is snooped from IGMP membership reports, unsolicited IGMP joins and leave massages. 

It is crucial to have an IGMP querier (or PIM router which executes the IGMP queries) running in an IGMP snooping enabled network. 
> Without the querier, hosts will not send membership reports and the snooping table cannot be build (or will time out). The result is that, in an IGMP snooping enabled network, packets addressed within the global scope and administrative scope will be dropped.

#### Formation of multicast router interfaces:

The following figure shows the formation of multicast router interfaces (the red crosses are STP blocked ports). The IGMP querier executes regular IGMP queries. The IGMP query destination address is 224.0.0.1, as described in the previous post multicast packets addressed to 224.0.0.x/24 are always flooded, IGMP snooping does not affect the 224.0.0.x/24 range.

![Multicast router interfaces](images/Multicast-router-interfaces.png?raw=true)

All switch ports where flooded IGMP queries are snooped are listed as multicast router interfaces in the snooping table (purple marked interfaces).

#### Formation of multicast group member interfaces:

Interested hosts will respond with IGMP membership reports to the IGMP queries. Membership reports are forwarded via the multicast router interfaces towards the IGMP querier. The following figure shows the result of this process. The group member interfaces are marked green.

![multicast group member interfaces](images/multicast-group-member-interfaces.png?raw=true)

A forwarding path between the IGMP querier and the multicast receiver emerges. This doesn???t mean that the multicast traffic must be originated by the IGMP querier (PIM router). Multicast sources can be everywhere in the network, in the layer 2 domain or behind a PIM router.

#### Multicast traffic flow in an IGMP enabled network:

It???s important to understand that a multicast source plays no role in the described formation process. A multicast source just starts sending multicast traffic. It is the responsibility of the network to deliver the multicast packets to the interested receivers.

![multicast traffic flow registered and unregistered](images/multicast-traffic-flow-registered-and-unregistered.png?raw=true)

The figure shows the stream of packets (in red) originated by the multicast source (assuming the receiver showed interest via membership reports in the multicast group of the source). To understand the flow we must distinguish between registered and unregistered multicast packets.

* Registered multicast packets are packets for which the multicast group address is listed in the IGMP snooping table (there is interest in this multicast stream).
* Unregistered multicast packets are packets for which the multicast group address is not listed in the IGMP snooping table (there is no interest shown in this multicast stream).

A registered multicast packet is forwarded towards every multicast router interface and out of every interested multicast group member interface. Depending on the switch configuration unregistered multicast packets are either flooded or dropped (filtered) by the switch. Multicast frames are always send to the IGMP querier / PIM router. This enables the PIM router to learn about available multicast sources which then can be advertised throughout the PIM routed domain.
