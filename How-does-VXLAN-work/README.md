How does VXLAN work?
====================

Table of contents:
------------------

* [Introduction](#Introduction)
* [VXLAN terminology](#VXLAN-terminology)
* [VXLAN frame format](#VXLAN-frame-format)
* [VXLAN packet walk](#VXLAN-packet-walk)
* [H1 to VM1 packet walk](#H1-to-VM1-packet-walk)

Introduction:
-------------

VXLAN is an overlay technique (also referred as a tunneling technique). Ethernet frames are encapsulated in a routable VXLAN IP packet (UDP port 4789). The sending host encapsulates the Ethernet frame in a VXLAN packet, the receiving host decapsulates the frame from the VXLAN packet. The encapsulation/decapsulation function is executed by the VXLAN Tunnel End Point (VTEP), this process is completely transparent for the end-hosts.

![VXLAN Topology](images/topology.png?raw=true)


VXLAN will be explained step by step by the figure above. The figure shows two sections:
* The upper section, the network overlay, which is formed by two VXLAN segments identified by VXLAN Network Identifiers (VNIs) 10 and 11.
* The lower section which shows the logical underlying network which is built around three routed VLANs 100,200 and 300.

The network connects four hosts, H1 to H4:
* H1 and H2 are not virtualized and are referred to as “physicals”. 
* H3 and H4 are both hosting two VMs (VM1 to VM4), the “virtuals”. 

There are three VTEPs shown in the drawing:
* VTEP1 and VTEP2, with IP addresses 1.1.1.10 and 2.2.2.10, these VTEPs are embedded within the hypervisor of H3 and H4. 
* VTEP3 with IP address 3.3.3.10 which is part of a switch configuration (this VTEP fulfils a gateway function between VLAN 300 and VXLAN 11).

Further note that:
* VM2 and VM4 are connected to VXLAN 10 (VNI 10, subnet 192.168.10.0/14).
* H1, H2, VM1 and VM3 are connected to VXLAN 11 (VNI 11, subnet 192.168.11.0/24).
* Hosts connected to VXLAN 10 and 11 are not aware that they are communicating over a VXLAN. The VXLAN is a broadcast domain just like a normal VLAN and behaves in exactly the same way.
* The gray router cloud denotes that each VTEP interface must be reachable from every other VTEP interface within the network.
* H3 and H4 could be in the same VLAN, for example both could be in VLAN 100 with IP addresses 1.1.1.10 and 1.1.1.11 (as long as all VTEPs can reach each other it doesn’t matter).
* H3 and/or H4 cannot be in VLAN 300. VTEP3 in VLAN 300 is a VLAN to VXLAN gateway. It encapsulates all Ethernet frames originating from VLAN 300 in a VXLAN packet. H3 and H4 have this functionality build into the hypervisor.

VXLAN terminology:
------------------
The listed VXLAN terminology is directly copied from RFC7348:
* VNI: VXLAN Network Identifier (or VXLAN Segment ID). VXLANs in this example are identified by VNI 10 and 11.
* VTEP: VXLAN Tunnel End Point.  An entity that originates and/or terminates VXLAN tunnels. The VTEPs in this example are identified by IPs: 1.1.1.10, 2.2.2.10 and 3.3.3.10
* VXLAN: Virtual eXtensible Local Area Network
* VXLAN Segment: VXLAN Layer 2 overlay network over which VMs communicate.
* VXLAN Gateway: an entity that forwards traffic between VXLANs. Or between VLANs and VXLANs as is VTEP 3 in this example.

VXLAN frame format:
-------------------

![VXLAN frame format](images/vxlan-frame-format.png?raw=true)

A VXLAN frame consists of the inner encapsulated Ethernet frame and the outer VXLAN, UDP, IP and MAC headers. The purple frame (“Encapsluted Ethernet Frame”) represents the inner (encapsulated) Ethernet frame originated by either H1/2 or VM1/2/3/4. The prepended yellow VXLAN header includes the following information:
* Flags (for future use).
* VNI.
* Reserved space (for future use).

An outer UPD header with the UDP destination port set to 4789. Followed by an outer IP header:
* The IP header source IP address will contain the IP address of the originating VTEP. 
* The destination IP address will be set to the IP of the receiving VTEP (or the receiving multicast group as explained later).

An outer Ethernet header (with trailing FCS, the inner Ethernet frame is transported without FCS) holds the MAC addresses from the sending and receiving VTEPs (or the intermediate layer 3 devices).

VXLAN packet walk:
------------------
Please note that this post explains the most basic implementation of VXLAN (multicast based VXLAN) “Learning by flooding”. More recent implementations of VXLAN include a control plain which enables learning in a more efficient way where flooding can be almost avoided. Examples are:
* Cisco ACI.
* VMware NSX (NSX controllers).
* VXLAN with MP-BGP EVPN.

Each VTEP contains a VTEP mapping table. This table maps the inner destination MAC address to the outer destination IP address. In other words the VTEP table tells the VTEP which MAC addresses (end-hosts) are reachable via which VTEP interface. 

The VTEP builds the mapping table by snooping the ARP process executed by the end-hosts. As you know, a ARP-request packet is a broadcast packet addressed towards FF:FF:FF:FF:FF:FF. The VTEP that receives the ARP broadcast must deliver the broadcast frame to all other VTEPs that share the VNI for which the broadcast was received. The VTEP encapsulates the ARP broadcast frame in a VXLAN multicast packet, the VXLAN packet reaches its destination(s) by utilizing the multicast transport capability of the underlying network. To make this process work, VTEPs must register each of their configured VNIs to a multicast group. All VTEPs share a common list of VNI to multicast group mappings, this list is configured by the administrator. Each VTEP is a multicast source and a multicast receiver at the same time:
* Each VTEP sends regular IGMP membership reports for the interested multicast groups in response to the IGMP querier.
* PIM must be active in the L3 network core to be able to detect multicast sending VTEPs and for transport of the multicast packets towards the interested VTEPs.
If you want to know more about how multicast behaves in an Ethernet network read this article: [Ethernet multicast versus IP multicast](Ethernet-multicast-versus-IP-multicast)

![VXLAN packet walk 1](images/packet-walk-1.png?raw=true)

The figure above shows initial state of the network used in the packet walk below, please note:
* The network administrator configured every VTEP to use the following VTEP to multicast mapping:
  * VNI 10 maps to multicast group 239.10.10.10
  * VNI 11 maps to multicast group 239.11.11.11
* An IGMP querier is active within every VLAN.
* PIM is configured in the L3 network core.
* The red arrows represent the IGMP membership reports send towards the IGMP querier.
* All VTEP tables are empty, there wasn’t any end-host activity until now.

H1 to VM1 packet walk:
----------------------
The following figure shows the order of events in the first stage: learning of the MAC address H1 by VM1, VTEP1 and VTEP2.

![VXLAN packet walk 2](images/packet-walk-2.png?raw=true)

1. H1 initiates communication with VM1, H1 ARPs for the MAC address of VM1.
2. The VXLAN gateway VTEP3 intercepts the ARP broadcast frame encapsulates it in a VXLAN packet and sends it towards multicast group address 239.11.11.11. All VTEPs which subscribed to multicast group 239.11.11.11 will receive the VXLAN encapsulated packet.
3. VTEP1 and VTEP2 will add an entry for host H1 into their mapping tables. From this point VTEP1 and VTEP2 know that host H1 is reachable via VTEP IP address 3.3.3.10. VTEP1 decapsulates the inner Ethernet frame and sends it to all hosts in VXLAN11.
4. The ARP frame is processed by host VM1 which will answer with an ARP-reply (see next drawing).

![VXLAN packet walk 3](images/packet-walk-3.png?raw=true)

Within the second stage (shown in the figure above) VTEP3 and H1 will learn about VM1:

5. ARP-reply is unicasted from host VM1 towards host H1.
6. VTEP1 encapsulates the ARP-reply into a unicast VXLAN frame targeted towards VTEP3 (3.3.3.10). All needed information is available in the VTEP mapping table of VTEP1. 
7. VTEP3 receives the VXLAN frame from VTEP1 and will learn about VM1. The mapping between the MAC address of VM1 and VTEP1 IP 1.1.1.10 is added to the VTEP mapping table of VTEP3.
8. De ARP reply is decapsulated by VTEP3 and received by H1. From now on H1 and VM1 can communicate.

As you can see VXLAN is completely passive, VXLAN snoops into the standard ARP process executed by the hosts. Be aware that all traffic that is unknown e.g. not known within the VTEP table must be multicasted to all VTEPs responsible for that VNI. All end-host Broadcast, Unknown unicast and Multicast (BUM) traffic will be multicasted over the VXLAN infrastructure.