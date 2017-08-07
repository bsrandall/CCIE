#### Overview
Multicast is a bandwidth and resource saving IP application. If a server had to send out separate directed unicast packets for every host that wants to receive the source's data, that put a lot of extra overhead on both the source and the network along the way to the receivers. Multicast solves this by allowing the source to send out one directed source of packets towards a multicast address, and the network efficiently transports these packets to all receivers who have subscribed to the multicast group.

From the client side, it helps to remember that it is the application that sends a join request for a particular multisource group on an as needed basis.

#### Multicast Group Concept
In multicast, a group is an arbitrary group of receivers that want to receive a partiular data stream. Any hosts interested in joining the group must join using IGMP. To get the correct data stream, the host needs to join the correct grtoup.

#### IP Multicast Addresses
The overall Multicast address range is the Class D network of 224.0.0.0 through 239.255.255.255. The first 4 bits of the 32 bit address are always 1110. The Class D block is broken into several subgroups.

##### IP Class D Addresses

###### Reserved Link Local
This range is 224.0.0.0/24. It is used by network protocols for communication on a local segment. We have seen these used previously. For example, the All_OSPF_Routers address is 224.0.0.5 while the OSPF_DR_Routers address is 224.0.0.6. These are as named link-local in scope and are not forwarded by routers as their TTL is set to 1.

###### Globally Scoped
These are the general purpose multicast addresses used to multicast data between organizations and even across the internet (though not really). There are two subgroups within the globally scoped range:

- Source Specific Multicast: These have a 232.0.0.0/8 range. We will see later that these are used for SSM which is an extension of Protocol Independent Multicast (PIM).
- GLOP Addresses: These have a range of 233.0.0.0/8 and are used by organizations that have an assigned AS. The AS of the domain is embedded in the second and third octets of the 233.0.0.0/8 range.

###### Limited Scope
These are also sometimes called administratively scoped. They are used and constrained to one organization.

##### Layer 2 Multicast Addresses

IANA owns a block of Ethernet MAC addresses that they reserve for mapping IP Class D Multicast addresses to Layer 2 Ethernet addresses. The range is 0100.5e00.0000 through 0100.5e7f.ffff. The lower 23 bits of the Class D address map to the lower 23 bits of the Ethernet Multicast MAC address. Since the first 4 bits of a Class D multicast address are always 1110, we are really only losing (32-4 = 28-23 =) 5 bits of IP multicast information. This means that for every Layer 2 Multicast address there are 32 corresponding multicast group IDs that overlap. So when programming application clients for IP multicast, care has to be taken not to overlap IP multicast address assignments.

#### Intradomain Multicast Protocols
This section covers all of the protocols used inside a multicast domain to support multicasting

##### IGMP
IGMP is used to dynamically register individual hosts in a multicast group on a particular *LAN*. Hosts identify the multicast groups they wish to join by sending IGMP messages to their local multicast router.

IGMP is always the language spoken between hosts and routers when it comes to IP multicast.

IGMP messages are encapsulated in IP headers like ICMP (with a protocol number of 2), but unlike ICMP the messages are limited to the local data link. This is guaranteed as the TTL in the IP header is always set to 1.

There are versions 1, 2 , and 3 of IGMP. In IOS you can change the version with the command **ip igmp version**.

###### IGMP Version 1
In Version 1 of IGMP, the following two types of IGMP messages exist:

- Membership query
- Membership report

Multicast reports are sent out by individual hosts indicating their interest to join a particular multicast group. The TCP/IP stack running on a host automatically sends the IGMP Membership report when an application opens a multicast socket.

Multicast sessions are identified in the routers by a (source, group) pair of addresses, where the source is the unicast address of the session’s originator, and the group is the multicast group address. If the local multicast router does not already have knowledge of the multicast session the host wants to join, it sends a request upstream toward the source. The data stream is received, and the router begins forwarding the stream onto the subnet of the host that requested membership.

The destination address of the Membership Report message’s IP header is the group address, and the message also contains the group address. To ensure that the local router receives the unsolicited Membership Report, the host sends one or two more duplicate reports separated by a short interval. RFC 2236 recommends an interval of 10 seconds.

The router periodically sends out an IGMP membership query to verify that at least one host on the subnet is still interested in receiving traffic directed to that group. Each query contains a value called the Max Response Time, which is always 10 seconds in IGMPv1. (specified in units of tenths of a second).  When a host receives a query, it sets a delay timer to a random value between 0 and the Max Response Time. If the timer expires, the host responds to the query with one Membership Report for each group to which it belongs. 

Because the destination of the Membership Report is the group address, other group members that might be on the subnet hear the report in addition to the router. If the host receives a Membership Report for a group before its delay timer expires, it does not send a Membership Report for that group. In this way, the router is informed of the presence of at least one group member on the subnet, without all members flooding the subnet with Reports.

When there is no reply to *three* consecutive IGMP membership queries, the router times out the group and stops forwarding traffic directed toward that group.

IGMPv1 uses only 1 type of Router Query:

- General Query

There is no Group-Specific Query like we will see in IGMPv2 since there is no host Leave Group message.

###### IGMP Version 2
IGMPv2 is backward compatible with Version 1. There are three message types in IGMPv2 that hosts use compared to the one host message types in version 1.

- Version 1 membership report
- Version 2 membership report
- Leave group

Hosts use Version 1 and Version 2 memberships reports as well as leave group messages. Routers use the Membership query message type.

The main difference between version 1 and version 2 is the Leave Group message in version 2. The leave group message allows a host to explicitly inform a router that it is leaving a multicast group. This reduces the leave latency as compared to version 1 with teh result being that unnecessary traffic is stopped much sooner.

When a host leaves a group, it notifies the local router with a Leave Group message. The message contains the address of the group being left, but unlike Membership Report messages, the Leave Group message is addressed to the “all routers on this subnet” address of 224.0.0.2. This is because only the multicast routers on the subnet need to know that the host is leaving; other group members do not.

RFC 2236 recommends that a Leave Group message be sent only if the leaving member was the last host to send a Membership Report in response to a query. As the next section explains, the local router always responds to a Leave Group message by querying for remaining group members. If group members other than the “last responder” leave quietly, the router continues forwarding the session and does not send a query. As a result, a little bandwidth is saved.

IGMPv2 has two messages that routers generate:

- General Query
- Group-Specific Query

The General Query is the message with which the router polls each of its subnets to discover if group members are present and to detect when there are no members of a group left on a subnet. By default, the queries are sent every 60 seconds; the default can be changed to any value between 0 and 65535 seconds with the statement **ip igmp query-interval**. Remember that the query will contain a host interval called the Max Response Time. In IGMPv1 it was set hard to 10 seconds. In IGMPv2, it can be changed with the command T**ip igmp query-max-response-time**.

The General Query message is sent to the “all systems on this subnet” multicast address of 224.0.0.1 and does not contain a reference to any specific group. As a result, the single message polls for Reports from members of all groups that might be active on the subnet. The router tracks known groups and the interfaces attached to subnets with active members, and can be seen with the command **show ip igmp groups**.

When there is no reply to *three* consecutive IGMP membership queries, the router times out the group and stops forwarding traffic directed toward that group.

A Group-Specific Query is used when a router receives a Leave Group message to see if there are any multicast receivers still using the group address from which the Leave Group messsage noted. Unlike a General Query message, a Group-Specific Query does contain the group multicast address and is sent to teh group multicast address.

If the Group-Specific Query were to become lost or corrupted, a remaining group mem- ber on the subnet might not send a Report. As a result, the router would incorrectly con- clude that no group members are on the subnet and stop forwarding the session packets. To protect against this eventuality, the router sends two Group-Specific Queries, sepa- rated by a 1-second interval known as the *Last Member Query* interval.

When a multicast-enabled router first becomes active on a subnet, it assumes that it is the Querier — the router responsible for sending all General and Group-Specific Queries to the subnet—and immediately sends a General Query.

This action serves both to quickly discover the group members active on the subnet and also alerts other multicast routers that may be on the subnet. When there are multiple routers, the rule for electing the Querier is simple: The router whose connected interface has the lowest IP address is the Querier. So when the existing router on the subnet hears the General Query from the new router, it checks the source address. If the address is lower than its own IP address, it relinquishes the role of Querier to the new router. If its own IP address is lower, it continues sending queries. When the new router receives one of these queries, it sees that the old router has a lower IP address and becomes a Non-Querier. Note that for IGMPv1 there is no Querier election process. Instead, the IP multicast routing protocol on teh subnet elects a designated router.

If the Non-Querier does not hear queries from the Querier within a certain period of time known as the Other Querier Present Interval, it concludes that the Querier is no longer present and assumes that role. IOS has a default Other Querier Present Interval of twice the Query Interval, or 120 seconds, and can be changed with the statement **ip igmp querier-timeout**.

All versions of IGMP are backwards compatible. If a mixture of Version 1 and Version 2 members are on the same subnet, the Version 2 members treat both Version 1 and Version 2 Membership reports the same when determining whether to suppress their own Membership Reports. That is, if a Version 2 member hears a Query from the router and subsequently hears a Version 1 Membership Report for its group before its own delay timer expires, it does not send a Membership Report. Version 1 hosts, however, ignore Version 2 messages. Therefore, if a Version 2 Membership Report is sent for a group first, the Version 1 member also sends a report when its delay timer expires. This does not cause problems for the Version 2 host and is important for the Version 2 router so that it is aware of the pres- ence of Version 1 group members.

If a host is running Version 2 and the local router is running Version 1, the IGMPv1 router ignores the Version 2 messages. So when a Version 2 host receives a Version 1 query, it responds with Version 1 Membership Reports. The IGMPv1 query also does not specify a Max Response Time, so the IGMPv2 host uses the fixed Version 1 period of 10 seconds. The host may or may not send Leave Group messages in the presence of Version 1 routers; the IGMPv1 router does not recognize Leave Group messages and ignores them.

If a Version 2 router receives a Version 1 Membership Report, it treats all members of the group as if they are running Version 1. The router ignores Leave Group messages and hence does not send Group-Specific Queries that the Version 1 members would ignore. Instead it sets a timer, known as the Old Host Present Timer. The period of the timer is the same value as the Group Membership Interval. Whenever a new Version 1 Membership Report is received, the timer is reset; if the timer expires, the router concludes that there are no more Version 1 members of the group on the sub- net and reverts to Version 2 messages and procedures.

If Version 1 and Version 2 routers exist on the same subnet, the Version 1 router does not participate in the Querier election process. Because of this, the Version 2 router must behave as a Version 1 router for consistency. There is no automatic conversion to Version 1; the Version 2 router must be manually configured with the ip igmp version 1 statement.

The Router Alert option is set on the IP header of IGMPv2 messages. The Router Alter option is not set with IGMPv1 messages.


###### IGMP Version 3
IGMPv3 officially obsoletes IGMPv2; however, version 2 is still widely used as of this writing and is still the IOS default, so you need to understand both versions.

IGMPv3 adds support for “source filtering,” which enables a multicast receiver host to signal to a router the groups from which it wants to receive multicast traffic, and from which sources this traffic is expected. This is accomplished with only two message types:

- Version 3 membership query
- Version 3 membership report

Version 3 supports both an INCLUDE mode and an EXCLUDE mode in which hosts announce which multicast groups they wish to receive traffic and those from which they do not wish to receive traffic.

The filtering capabilities make ICMPv3 especially suited for Source-Specific Multicast.

Here are some other changes from IGMPv2:

- The Max Response Time variable in Query messages has an exponential range, changing the maximum from 25.5 seconds to approximately 53 minutes.
- Report messages are sent to the well-known IGMP multicast address 224.0.0.22 to help with IGMP snooping.
- Report messages can contain multiple group records.

###### IGMP Snooping


##### Layer 2 Multicast
The default behavior for a Layer 2 switch is to forward all multicast traffic to every port that belongs to the destination LAN on the switch. This behavior reduces the efficiency of the switch, whose purpose is to limit traffic to the ports that need to receive the data.

Three methods efficiently handle IP multicast in a Layer 2 switching environment—Cisco Group Management Protocol (CGMP), IGMP Snooping, and Router-Port Group Management Protocol (RGMP). CGMP and IGMP Snooping are used on subnets that include end users or receiver clients. RGMP is used on routed segments that contain only routers, such as in a collapsed backbone.

###### Cisco Group Management Protocol (CGMP)
CGMP is a Cisco-developed protocol that allows Catalyst switches to leverage IGMP information on Cisco routers to make Layer 2 forwarding decisions. You must configure CGMP on the multicast routers and the Layer 2 switches. The result is that, with CGMP, IP multicast traffic is delivered only to those Catalyst switch ports that are attached to interested receivers. All other ports that have not explicitly requested the traffic will not receive it unless these ports are connected to a multicast router. Multicast router ports must receive every IP multicast data packet.

When a host joins a multicast group (part A in the figure below), it multicasts an unsolicited IGMP membership report message to the target group (224.1.2.3, in this example). The IGMP report is passed through the switch to the router for normal IGMP processing. The router (which must have CGMP enabled on this interface) receives the IGMP report and processes it as it normally would, but also creates a CGMP join message and sends it to the switch (part B).

{% include image.html file="CGMP.png" %}

The switch receives this CGMP join message and then adds the port to its content-addressable memory (CAM) table for that multicast group. All subsequent traffic directed to this multicast group will be forwarded out the port for that host. The Layer 2 switches were designed so that several destination MAC addresses could be assigned to a single physical port. This allows switches to be connected in a hierarchy and also allows many multicast destination addresses to be forwarded out a single port. The router port also is added to the entry for the multicast group. Multicast routers must listen to all multicast traffic for every group because the IGMP control messages also are sent as multicast traffic. With CGMP, the switch must listen only to CGMP join and CGMP leave messages from the router. The rest of the multicast traffic is forwarded using the CAM table with the new entries created by CGMP.

Note. This is rarely used in modern networks! IGMP Snooping is now the accepted means of running IP multicast in a switched environment.

###### IGMP Snooping
IGMP Snooping is an IP multicast constraining mechanism that runs on a Layer 2 LAN switch. IGMP Snooping requires the LAN switch to examine, or “snoop,” some higher Layer 3 information in the IGMP packets sent between the hosts and the router.

A snooping switch can determine what ports multicast routers are on by listening to Membership Queries; it determines which ports group members are on by listening for Membership Reports. Queries, of course, must be sent to all active ports, so the querier can find group members. But when the switch learns what ports the routers are on, it relays Membership Reports only to those ports.

Under IGMPv1, IGMPv2, and MLDv1, hosts that hear a Membership Report from another host on the subnet suppress their own reports. With snooping, the switch does not forward reports from one member to other members. Each group member must respond to the query individually so that the switch can record the member port. The same applies to group leaves; each host, not being aware of other hosts on its subnet, sends a Leave Group (or MLD Listener Done) message when it leaves a group, and the switch removes the host from its port list for that group.

IGMP Snooping is vendor agnostic, and that is part of the reason it has become the method by which switches constrain IP Multicast traffic to only the necesarry ports.



###### Router-Port Group Management Protocol (RGMP)
CGMP and IGMP Snooping are IP multicast constraining mechanisms designed to work on routed network segments that have active receivers. They both depend on IGMP control messages that are sent between the hosts and the routers to determine which switch ports are connected to interested receivers.

Switched Ethernet backbone network segments typically consist of several routers connected to a switch without any hosts on that segment. Because routers do not generate IGMP host reports, CGMP and IGMP Snooping will not be able to constrain the multicast traffic, which will be flooded to every port on the VLAN. Routers instead generate Protocol Independent Multicast (PIM) messages to join and prune multicast traffic flows at a Layer 3 level.


RGMP is an IP multicast constraining mechanism for router-only network segments. RGMP must be enabled on the routers and on the Layer 2 switches. A multicast router indicates that it is interested in receiving a data flow by sending an RGMP join message for a particular group.

The switch then adds the appropriate port to its forwarding table for that multicast group—similar to the way it handles a CGMP join message. IP multicast data flows will be forwarded only to the interested router ports. When the router no longer is interested in that data flow, it sends an RGMP leave message and the switch removes the forwarding entry.

##### Multicast Distribution Tress
Multicast-capable routers create distribution trees that control the path that IP multicast traffic takes through the network in order to deliver traffic to all receivers. The two basic types of multicast distribution trees are source trees and shared trees.

###### Source Trees
Source trees are the simplest form of a multicast distribution tree. The root is the source and the branches form a spanning tree through the network to the receivers. It is also referred to as a shortest path tree.

{% include image.html file="multicast-source-tree.png" %}

The special notation of (S, G), pronounced “S comma G,” enumerates an SPT where S is the IP address of the source and G is the multicast group address. Using this notation, the SPT for the example shown would be (192.168.1.1, 224.1.1.1).

The (S, G) notation implies that a separate SPT exists for each individual source sending to each group—which is correct. For example, if Host B is also sending traffic to group 224.1.1.1 and Hosts A and C are receivers, a separate (S, G) SPT would exist with a notation of (192.168.2.2, 224.1.1.1).

###### Shared Trees
Unlike source trees that have their root at the source, shared trees use a single common root placed at some chosen point in the network. This shared root is called a rendezvous point (RP).

{% include image.html file="multicast-shared-tree.png" %}

The figure above shows a shared tree for the group 224.2.2.2 with the root located at Router D. This shared tree is unidirectional. Source traffic is sent towards the RP on a source tree. The traffic is then forwarded down the shared tree from the RP to reach all of the receivers (unless the receiver is located between the source and the RP, in which case it will be serviced directly).

In this example, multicast traffic from the sources, Hosts A and D, travels to the root (Router D) and then down the shared tree to the two receivers, Hosts B and C. Because all sources in the multicast group use a common shared tree, a wildcard notation written as (\*, G), pronounced “star comma G,” represents the tree. In this case, \* means all sources, and G represents the multicast group. Therefore, the shared tree showed above would be written as (*, 224.2.2.2).

###### Source Trees vs Shared Trees
Both source trees and shared trees are loop-free. Messages are replicated only where the tree branches.

Source trees have the advantage of creating the optimal path between the source and the receivers. This advantage guarantees the minimum amount of network latency for forwarding multicast traffic. However, this optimization comes at a cost: The routers must maintain path information for each source. In a network that has thousands of sources and thousands of groups, this overhead can quickly become a resource issue on the routers. Memory consumption from the size of the multicast routing table is a factor that network designers must take into consideration.

Shared trees have the advantage of requiring the minimum amount of state in each router. This advantage lowers the overall memory requirements for a network that only allows shared trees. The disadvantage of shared trees is that under certain circumstances the paths between the source and receivers might not be the optimal paths, which might introduce some latency in packet delivery.

##### Multicast Forwarding
In unicast routing, traffic is routed through the network along a single path from the source to the destination host. A unicast router does not consider the source address; it considers only the destination address and how to forward the traffic toward that destination. The router scans through its routing table for the destination address and then forwards a single copy of the unicast packet out the correct interface in the direction of the destination.

In multicast forwarding, the source is sending traffic to an arbitrary group of hosts that are represented by a multicast group address. The multicast router must determine which direction is the upstream direction (toward the source) and which one is the downstream direction (or directions). If there are multiple downstream paths, the router replicates the packet and forwards it down the appropriate downstream paths (best unicast route metric)—which is not necessarily all paths.

###### Revers Path Forwarding (RPF)
PIM uses the unicast routing information to create a distribution tree along the reverse path from the receivers towards the source. The multicast routers then forward packets along the distribution tree from the source to the receivers. RPF is a key concept in multicast forwarding. It enables routers to correctly forward multicast traffic down the distribution tree. RPF makes use of the existing unicast routing table to determine the upstream and downstream neighbors. *A router will forward a multicast packet only if it is received on the upstream interface*. This RPF check helps to guarantee that the distribution tree will be loop-free.

When a multicast packet arrives at a router, the router performs an RPF check on the packet. If the RPF check succeeds, the packet is forwarded. Otherwise, it is dropped.

For traffic flowing down a source tree, the RPF check procedure works as follows:

1. The router looks up the source address in the unicast routing table to determine if the packet has arrived on the interface that is on the reverse path back to the source.
2. If the packet has arrived on the interface leading back to the source, the RPF check succeeds and the packet is forwarded.
3. If the RPF check in Step 2 fails, the packet is dropped.

##### Protocol Independent Multicast (PIM)
When we say protocol independent with PIM, we mean that it can utilize whatever unicast routing protocol is used to poulate the unicast routing table, including EIGRP, OSPF, BGP, and static routes. PIM uses this unicast routing informationto perform the multicast forwarding function. Although PIM is called a multicast routing protocol, it actually uses the unicast routing table to perform the RPF check function instead of building up a completely independent multicast routing table. Unlike unicast routing protocols, PIM does not send and receive routing updates between routers.

###### PIM Dense Mode (PIM-DM)
PIM-DM uses a push model to flood multicast traffic to every corner of the network. This push model is a brute force method for delivering data to the receivers. This method would be efficient in certain deployments in which there are active receivers on every subnet in the network.

PIM-DM initially floods multicast traffic throughout the network. Routers that have no downstream neighbors prune back the unwanted traffic. This process repeats every 3 minutes.

Routers accumulate state information by receiving data streams through the flood and prune mechanism. These data streams contain the source and group information so that downstream routers can build up their multicast forwarding table. PIM-DM supports only source trees—that is, (S, G) entries—and cannot be used to build a shared distribution tree.

###### PIM Sparse Mode (PIM-SM)
PIM-SM uses a pull model to deliver multicast traffic. Only network segments with active receivers that have explicitly requested the data will receive the traffic.

PIM-SM distributes information about active sources by forwarding data packets on the shared tree. Because PIM-SM uses shared trees (at least, initially), it requires the use of a rendezvous point (RP). The RP must be administratively configured in the network.

Sources register with the RP and then data is forwarded down the shared tree to the receivers. The edge routers learn about a particular source when they receive data packets on the shared tree from that source through the RP. The edge router then sends PIM (S, G) join messages towards that source. Each router along the reverse path compares the unicast routing metric of the RP address to the metric of the source address. If the metric for the source address is better, it will forward a PIM (S, G) join message towards the source. If the metric for the RP is the same or better, then the PIM (S, G) join message will be sent in the same direction as the RP. In this case, the shared tree and the source tree would be considered congruent.

{% include image.html file="pim-sm.png" %}

If the shared tree is not an optimal path between the source and the receiver, the routers dynamically create a source tree and stop traffic from flowing down the shared tree. This behavior is the default behavior in Cisco IOS software. Network administrators can force traffic to stay on the shared tree by using the Cisco IOS **ip pim spt-threshold infinity command**.

PIM-SM scales well to a network of any size, including those with WAN links. The explicit join mechanism will prevent unwanted traffic from flooding the WAN links.

######Bidirectional PIM (Bidir-PIM)
The shared trees that are created in PIM Sparse Mode are unidirectional. This means that a source tree must be created to bring the data stream to the RP (the root of the shared tree) and then it can be forwarded down the branches to the receivers. Source data cannot flow up the shared tree toward the RP—this would be considered a bidirectional shared tree.

In bidirectional mode, traffic is routed only along a bidirectional shared tree that is rooted at the RP for the group. In bidir-PIM, the IP address of the RP acts as the key to having all routers establish a loop-free spanning tree topology rooted in that IP address. This IP address need not be a router address, but can be any unassigned IP address on a network that is reachable throughout the PIM domain.

Bidir-PIM is derived from the mechanisms of PIM sparse mode (PIM-SM) and shares many of the shared tree operations. Bidir-PIM also has unconditional forwarding of source traffic toward the RP upstream on the shared tree, but no registering process for sources as in PIM-SM. These modifications are necessary and sufficient to allow forwarding of traffic in all routers solely based on the (*, G) multicast routing entries. This feature eliminates any source-specific state and allows scaling capability to an arbitrary number of sources.

##### Pragmatic General Multicast (PGM)
PGM is a reliable multicast *transport* protocol for applications that require ordered, duplicate-free, multicast data delivery from multiple sources to multiple receivers. PGM guarantees that a receiver in a multicast group either receives all data packets from transmissions and retransmissions or can detect unrecoverable data packet loss.

PGM is intended as a solution for multicast applications with basic reliability requirements. PGM is better than best effort delivery but not 100% reliable. The specification for PGM is network layer-independent. The Cisco implementation of the PGM Router Assist feature supports PGM over IP.

#### Interdomain Multicast Protocols
These are the protocols that are used to forward multicast traffic between domains.
##### Multiprotocol Border Gateway Protocol (MBGP)
MBGP provides a method for providers to distinguish which route prefixes they will use for performing multicast RPF checks. We saw previously that he RPF check is the fundamental mechanism that routers use to determine the paths that multicast forwarding trees will follow and to successfully deliver multicast content from sources to receivers.

Two path attributes, MP_REACH_NLRI and MP_UNREACH_NLRI, were introduced in BGP4. These new attributes create a simple way to carry two sets of routing information—one for unicast routing and one for multicast routing. The routes associated with multicast routing are used for RPF checking at the interdomain borders.

The main advantage of MBGP is that an internetwork can support noncongruent unicast and multicast topologies. When the unicast and multicast topologies are congruent, MBGP can support different policies for each. Separate BGP routing tables are maintained for the Unicast Routing Information Base (U-RIB) and the Multicast Routing Information Base (M-RIB). The M-RIB is derived from the unicast routing table with the multicast policies applied. RPF checks and PIM forwarding events are performed based on the information in the M-RIB. MBGP provides a scalable policy-based interdomain routing protocol.

##### Multicast Source Discovery Protocol (MSDP)
In the PIM sparse mode model, the router closest to the sources or receivers registers with the RP. The RP knows about all the sources and receivers for any particular group. Network administrators may want to configure several RPs and create several PIM-SM domains. In each domain, RPs have no way of knowing about sources located in other domains. MSDP is an elegant way to solve this problem.

MSDP was developed for peering between Internet service providers (ISPs). ISPs did not want to rely on an RP maintained by a competing ISP to provide service to their customers. MSDP allows each ISP to have its own local RP and still forward and receive multicast traffic to the Internet.

MSDP enables RPs to share information about active sources. RPs know about the receivers in their local domain. When RPs in remote domains hear about the active sources, they can pass on that information to their local receivers and multicast data can then be forwarded between the domains. A useful feature of MSDP is that it allows each domain to maintain an independent RP that does not rely on other domains. MSDP gives the network administrators the option of selectively forwarding multicast traffic between domains or blocking particular groups or sources. PIM-SM is used to forward the traffic between the multicast domains.

The RP in each domain establishes an MSDP peering session using a TCP connection with the RPs in other domains or with border routers leading to the other domains. When the RP learns about a new multicast source within its own domain (through the normal PIM register mechanism), the RP encapsulates the first data packet in a Source-Active (SA) message and sends the SA to all MSDP peers. MSDP uses a modified RPF check in determining which peers should be forwarded the SA messages. This modified RPF check is done at an AS level instead of a hop-by-hop metric. The SA is forwarded by each receiving peer, also using the same modified RPF check, until the SA reaches every MSDP router in the internetwork—theoretically, the entire multicast Internet. If the receiving MSDP peer is an RP, and the RP has a (*, G) entry for the group in the SA (that is, there is an interested receiver), the RP creates (S, G) state for the source and joins to the shortest path tree for the source. The encapsulated data is decapsulated and forwarded down the shared tree of that RP. When the packet is received by the last hop router of the receiver, the last hop router also may join the shortest path tree to the source. The MSDP speaker periodically sends SAs that include all sources within the own domain of the RP.

###### Anycast RP
Anycast RP is a useful application of MSDP that configures a multicast sparse mode network to provide for fault tolerance and load sharing within a single multicast domain.

Two or more RPs are configured with the same IP address (for example, 10.0.0.1) on loopback interfaces. The loopback address should be configured as a host address (with a 32-bit mask). All the downstream routers are configured so that they know that 10.0.0.1 is the IP address of their local RP. IP routing automatically selects the topologically closest RP for each source and receiver. Because some sources use only one RP and some receivers a different RP, MSDP enables RPs to exchange information about active sources. All the RPs are configured to be MSDP peers of each other. Each RP will know about the active sources in the area of the other RP. If any of the RPs fail, IP routing converges and one of the RPs would become the active RP in both areas.

##### Source Specific Multicast
SSM is an extension of the PIM protocol that allows for an efficient data delivery mechanism in one-to-many communications. SSM enables a receiving client, once it has learned about a particular multicast source through a directory service, to then receive content directly from the source, rather than receiving it using a shared RP.

SSM removes the requirement of MSDP to discover the active sources in other PIM domains. An out-of-band service at the application level, such as a web server, can perform source discovery. It also removes the requirement for an RP.

In an SSM-enhanced multicast network, the router closest to the receiver will “see” a request from the receiving application to join to a particular multicast source. The receiver application then can signal its intention to join a particular source by using the INCLUDE mode in IGMPv3.

The multicast router can now send the request directly to the source rather than send the request to a common RP as in PIM sparse mode. At this point, the source can send data directly to the receiver using the shortest path. In SSM, routing of multicast traffic is entirely accomplished with source trees. There are no shared trees and therefore an RP is not required.

SSM solves IP multicast address collision issues associated with one-to-many type applications. Routers running in SSM mode will route data streams based on the full (S, G) address. Assuming that a source has a unique IP address to send on the internet, any (S, G) from this source also would be unique.