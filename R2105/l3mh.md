**Introduction**
================

Existing Contrail vRouter software design assumes the presence of only one fabric interface on Computes. This, by default, makes Contrail computes to be single homed to a leaf switch in CLOS topology. Even if a Compute has multiple uplink connections to leaf switches, Contrail software does not have the support to use them to provide high availability or load balancing for traffic exiting vRouter.

With this existing design, on multi-homed Computes, to achieve high availability or load balancing, customers are forced to use MC-LAG or bond interfaces on the provisioned Contrail computes. Many customers are reluctant to use MC-LAG due to its complex configurations.

With L3 Multihoming feature, Contrail computes can be multi-homed to provide high availability and load balancing of Contrail traffic without having to require MC-LAG or bond interfaces.

**2. Problem statement**
========================

#### [**[https://contrail-jws.atlassian.net/browse/CEM-326]**](https://contrail-jws.atlassian.net/browse/CEM-326)

Contrail vRouter software's assumption of a single fabric interface for computes forces customers to use the complex MC-LAG/bond interfaces where Computes have multiple uplink connections to leaf switches. This feature addresses this by making vRouter be aware of the multiple uplink connections to provide high availability and load balancing for Contrail traffic exiting Computes.

**3. Proposed solution**
========================

In phase1, vRouter would support multihoming to upto 3 ToR leaf switches. The vRouter computes and ToR switches would be configured with static routing to reach the lo interface of the vRouter. Also only vRouter in kernel mode will be supported in phase1.

Subsequent phases would support running dynamic routing protocols on the compute and vRouter running in DPDK mode (Not part of scope in this document).

3.1 High level solution overview
--------------------------------

-   Customer will create a lo interface on the computes, specifically to be used by Contrail services. He will assign a /32 IP address to
    it from a predefined address pool, manage it and ensure external
    connectivity for it via the GW IPs (Leaf switches).

-   Customer will also ensure connectivity to the controller (in case they are in a different subnet) via the GW IPs (Leaf switches).

-   Customer underlay network configuration remains untouched by Contrail provisioning.

-   Contrail vRouter will use the lo IP address to initiate and terminate vrouter services (XMPP, overlay traffic).

-   Contrail vRouter will retain the vhost0 interface (without IP) to process vhost packets coming from the linux stack destined to controller/other computes' lo interface (for eg: XMPP/VR-VR traffic/Linklocal).

-   Contrail vRouter will use netfilter hooks to capture packets sent using lo interface Src IP and then apply vhost processing to it ensuring XMPP traffic is also load balanced.

-   Contrail vRouter will create a new type of Composite ECMP nexthop containing the Encap NHs to the ToR switches. The overlay routes will point to this ECMP nexthop resulting in load balancing of traffic across the ToR switches.

-   Contrail vRouter Agent will monitor the Encap NHs to the ToR switches by periodically sending ARP requests to the same. In case of ARP failure (either due to link issue/network issue), the Encap NH will be marked down and traffic will go over the remaining NHs in the composite NH.

### 3.1.1 XMPP traffic flow

![](./images/image1.png){width="6.5in" height="3.388888888888889in"}

-   In the above diagram, we have a compute configured with lo:1interface having 8.1.1.1/32 IP address and there is a controller configured with 30.1.1.2 IP address. The compute and controller are connected via an IP CLOS based fabric n/w.

-   The linux routing table on the compute has reachability towards the controller via GW1 and GW2 (which are the next hops for the compute). Similarly, the linux routing table on the controller has
    reachability towards lo:1 of the compute via some GW.

-   Agent resolves the ARP for the GW1 & GW2 and programs a 0/0 default route pointing to composite NH containing Encap NH towards GW1 & GW2 in VRF 0.

-   As part of vhost0 interface addition, vRouter sets up netfilter hook (IP_POST_ROUTING) to capture packets sent with lo:1 Source IP address.

-   Agent programs a L3 receive NH for the lo:1 IP address with OIF pointing to vhost0.

-   Agent binds the XMPP socket using the lo:1 IP address and initiates connection to the XMPP server running on the controller.

-   Due to netfilter hook, vRouter receives these packets and applies vhost interface packet processing.

-   As part of vhost interface packet processing, the vRouter Route table for VRF 0 (underlay VRF) is looked up, the default 0/0 route is hit and the packet is load balanced using the composite NH. Based on the selected Encap NH, appropriate L2 header is added to the packet and sent out.

-   There is no change in packet processing on the controller.

-   In the incoming direction, the packet from the controller is received on one of the physical interfaces and processed by vRouter.

-   As part of incoming fabric packet processing, vRouter does a route lookup for the Destination IP (lo:1 IP in this case) and gets the receive NH for lo:1 interface with OIF pointing to vhost0. This results in packet getting sent to the linux kernel via the vhost0 interface.

-   The linux kernel finally gives the packet to the Agent socket.

### 3.1.2 VM to VM traffic flow (inter compute)

![](./images/image2.png){width="6.5in" height="3.486111111111111in"}

-   In the above diagram, we have VM1 hosted on Compute1 and VM2 hosted on Compute2.

-   In the VM1 & VM2's VRF, Agent programs a bridge route to the another VM, containing a label allocated by the other compute and a pointer to a special type of Tunnel Nexthop called Tunnel_UL_ECMP nexthop.

-   This special Tunnel Nexthop is marked with the Underlay ECMP flag (UL_ECMP). As part of this nexthop, Agent maintains the Tunnel information (Local Compute lo:1 SrcIP, Remote Compute lo:1 Dst IP, Src Port, Dst Port, Tunnel Encap ) and the L2 encapsulation for all the outgoing interfaces through which the packet may be sent.

-   When VM1 starts traffic towards VM2, a flow gets created.

-   As part of flow creation, Agent programs the underlay ECMP index to be used in case of Tunnel_UL_ECMP nexthop.

-   vRouter uses the flow's underlay ECMP index to select the outgoing interface and L2 Encap information which needs to be added to the encapsulated Tunneled packets when sending them via the Tunnel_UL_ECMP Nexthop.

-   In this way, overlay flows from the VM are load balanced over multiple uplink interfaces.

-   In case the VM interface is operating in packet mode, no flow is created. Instead, when the packet hits Tunnel_UL_ECMP nexthop, vRouter computes the hash of the packet and selects one of the outgoing interfaces through which to send the tunneled packets, achieving load balancing over multiple uplink interfaces.

-   In the Fabric N/W, the packets again get load balanced (due to multiple paths to the lo:1 IP of compute2) and reach the compute2.

-   On Compute2, the incoming packets get decapsulated as usual and the packets get sent to the VM2 based on the label.

-   The traffic flow from VM2 to VM1 is similar to what is described above.

**4. Alternatives considered**
==============================

Instead of using Netfilter hook to capture XMPP and VR to VR Encrypted packets in the outgoing direction to do vhost0 processing, another option was considered, viz to install lo:1/controller subnet routes in the compute routing table which would point to the vhost0 interface explicitly. In this case, these packets would undergo vhost0 processing due to route taking them to the vhost0 interface. However, this option was later dropped since the intention was to not change the existing customer added routes for the lo:1/controller IPs. This option also required vRouter to proxy ARP for the lo:1 ARP requests from the linux stack.

**5. API schema changes**
=========================

**6. UI changes**
=================

**7. Notification impact**
==========================

**8. Provisioning changes**
===========================

**9. Implementation**
=====================

9.1 vRouter Agent
-----------------

### Agent Initialization and Contrail-vrouter-agent.conf Processing

In single homing case, following parameters are defined in contrail-vrouter-agent configuration file -

​	*physical_interface=eth2*

​	*gateway=192.168.196.1*

Since multihoming can support multiple uplink interfaces, above parameters need to be changed to accept a list of ip and interfaces.

​	*physical_interface=eth2, eth3*

​	*gateway=192.168.196.1, 192,168.196.2*

Along with this, loopback ip and name also need to be passed in the contrail-vrouter-agent configuration file. Agent will use this loopback to bind to the Xmpp client. So, going forward all Xmpp packets will be sent/received on this loopback.

Once this is done following needs to be done in agent using these newlist of interfaces -

-   Create these uplink interfaces in vrouter and add them to vrf0

-   Add resolve routes for these interfaces.

-   Add L2 receive routes for uplink physical interface MACs in vrf0 and vrf1.

-   Add routes for gateway ip in vrf0 and trigger arp if necessary.

-   Add a default route in vrf0 with composite NH having gateway list as component NHs.

-   Sandesh changes to reflect these new changes in the Introspect page.

### ARP Handling

When gateway IP is specified in contrail-vrouter-agent.conf, Agent adds a default route to vrf 0 with its path's gateway IP set to the configured value. The path itself does not have any NH set. Agent also
triggers ARP resolution for the gateway IP. While programming this default route to vRouter, Agent programs the route with the resolved ARP NH of gateway IP.

With L3MH, multiple gateway IPs are specified in contrail-vrouter-agent.conf - one for each physical interface. Agent's Inet4UnicastGatewayRoute::AddChangePathExtended would be modified as
follows -

1.  Trigger ARP resolution for all the gateway IP addresses. Keep the path marked as unresolved.

2.  When Inet4UnicastGatewayRoute::AddChangePathExtended is invoked again after each of the ARP routes get resolved, a composite NH is created/updated with the ARP NH as component nexthop. The path is marked as resolved once all GW IPs have been resolved.

**Existing default route**

​		0.0.0.0 → NULL (GW IP)

​		GW IP → ARP NH

**New default route**

​		0.0.0.0 → CNH  → ARPNH - 1

​									→ ARPNH - 2

​											...

​		GW IP - 1 → ARPNH-1

​		GW IP - 2 → ARPNH-2

​								...

The 0.0.0.0/0 route is made a dependent route of each of the component ARP routes. When an ARP route goes down marking the ARP NH as invalid, the route gets a notification causing it to be updated in vRouter.

The ARPNHs once created during Agent initialization, would remain in place and not get deleted. They get marked invalid on the GWs becoming unreachable and get resurrected once the GWs become reachable again.

### Tunnel NH Changes

In the existing design, TNH holds one encap information corresponding to the DIP of the TNH. The encap information comes from either the ARP NH for the DIP or the ARP NH of GW.

Tunnel NH definition will be modified to hold multiple encapsulation information for the DIP - one for each uplink GW. The encapsulation information consists of the outgoing interface, L2 header and a dependency reference from the ARPNH. In addition, a flag to indicate if the incap info at the index is valid is also included. The number of encap info to hold in TNH is equal to the number of uplink interfaces configured on the Compute.

`struct EncapData { //L3MH`

​			`EncapData(NextHop\* nh, InterfaceConstRef intf) : arp_rt\_(nh), interface\_(intf){}`

​			`DependencyRef\<NextHop, AgentRoute\> arp_rt\_;`

​			`InterfaceConstRef interface\_;`

​			`MacAddress dmac\_;`

​			`bool valid;`

`};`

​	`typedef boost::shared_ptr\<EncapData\> EncapDataPtr;`

​	`typedef std::vector\<EncapDataPtr\> EncapDataList;`

​	`TunnelNH {`

​			`...`

​			`EncapDataList encap_list\_;`

​			`...`

​	`};`

When a Tunnel NH is created, the encap list is populated with the encap data taken from the ARPNHs of GWs. TunnelNH::ChangeEntry will be modified to look up 0.0.0.0 IP in vrf 0 to obtain the composite NH and
its component NHs. The encap info list is populated from the component NHs.

The dependency refs to default route's ARP routes help in receiving change events to ARP routes. If an ARP NH gets marked as invalid, the corresponding encap data in TNH will be marked invalid. Similarly, if an ARP NH gets resurrected, the corresponding encap entry in TNH will be populated with encap information from the ARP NH.

### Flow Stickiness

Since the underlay will have multiple uplink interfaces with L3 Multihoming, it is desirable that all packets belonging to a flow are sent out on the same uplink interface. It ensures that the packets arrive at the destination in order. In the case where flow setting up is triggered from a packet entering a Compute, it is also desirable that the packets belonging to the flow are sent out on the same interface via which the first packet of the flow enters the Compute.

Agent's flow definition will be modified to introduce a new field called "underlay gw index" that indicates the underlay uplink interface to be used to send out packets belonging to non-local flows. When setting up a non-local flow, if L3 Multihoming is enabled, "underaly gw index" for the flow is set according to the following logic -

`if incoming interface is a Fabric interface`

​	`"Underlay gw index" = index of TNH encap corresponding to incoming interface`

`else`

​	`Compute a hash of 5 tuple of the packet to pick an encap Index of the TNH`

The hash used to choose the outgoing interface is the same as the one currently being used in case of ECMP composite nexthop.

In case of overlay ECMP, the remote IP of a flow matches a ECMP composite NH. The hash computed for overlay ECMP will also be used for underlay ECMP.

In case of MPLSoMPLS tunnels, it is possible to have 3 levels of ECMP. Here again, hash computed for a higher level could be used for underlay ECMP.

The three possibilities are depicted below with an example where remote IP of a flow is 1.1.1.2.

**Underlay ECMP only**

1.1.1.2 → TNH (Multiple encaps)

**Underlay and Overlay ECMP**

1.1.1.2 → CNH

​						TNH1 (Multiple encaps)

​						TNH2 (Multiple encaps)

**Under and Overlay ECMP with MPLSoMPLS**

1.1.1.2 → CNH1

​						TNH1 (Multiple encaps)

​						TNH2 (Multiple encaps)	

​					CNH2

​						TNH1 (Multiple encaps)

​						TNH2 (Multiple encaps)

### XMPP

With L3MH changes, XMPP client running on agent will bind to new loopback interface.

`Ip4Address local_endpoint;`

`if (!agent_->tor_agent_enabled() && !agent_->isMockMode()) {`

​		`local_endpoint = agent_->**compute_node_ip**();`

​		`xmpp_cfg->local_endpoint.address(AddressFromString(`

​		`local_endpoint.to_string(), &ec));`

`}`

In case of L3MH, compute_node_ip() functions should return loopback address.

### Impact of vhost0 Changes

With L3MH changes and introduction of lo interface for running host services, vhost0 interface will cease to have an IP address. Existing design also had vhost0 interface borrow physical interface's MACaddress. With L3MH, vhost0 will be assigned the VRRP GW mac of 00:00:5e:00:01:00.

Following Agent functionality relied on vhost0's MAC -

1.  EVPN Service Chaining

2.  VxLAN Routing

3.  Various Services Protocols

4.  ARP module

5.  Flow Handler

#### EVPN Service Chaining and VxLAN Routing

EVPN Service Chaing and VxLAN routing make use of vhost0's mac as destination MAC in overlay ethernet header. When a VxLAN packet needing routing is received by vRouter, it matches vhost0's mac route pointing to a Receive NH. In suches cases, vRouter performs a further IP lookup on the destination IP to forward the packet.

With L3MH, 00:00:5e:00:01:00 mac will be used in place of the physical interface MAC which vhost0 had borrowed. Since we already have a route for 00:00:5e:00:01:00 pointing to Receive NH, vRouter's behavior is not expected to change.

#### Various Services Protocols

DHCP, DNS and ICMP handlers use vhost0's mac as source MAC for the packets they generate and transmit. Since these packets are destined to VMIs, 00:00:5e:00:01:00 MAC should suffice.

#### ARP module

All instances of ARP NHs where vhost0 is being used as NH's interface will be replaced to use lo interface.

#### Flow Handler

Agent's Flow handler marks a flow as L3 flows if the interface received in Agent header of flow trap is vhost0. vRouter will continue to set the interface as vhost0 for all flow traps where it was setting agent header's interface as vhost0.

### Undelay packets, Packet mode and BUM

Since flows are not created for underlay and BUM packets, and when packet mode is enabled, the outgoing uplink interface for a packet is not determined by Agent stored in Flow object. Instead, it will be computed by vRouter for each packet.

9.2 vRouter
-----------

### 9.2.1 Fabric Vif interface changes

-   A new VIF interface will be created for each of the fabric interfaces handled by vRouter (Max of 3). Each of these interfaces will go through the existing initialization sequence.

-   A new counter will be added to the global vrouter struct which will keep track of the fabric interfaces added/deleted to the vrouter. Based on this counter, a new API **is_vrouter_multihomed()** is
    added which will return TRUE (if num phy int \> 1) or FALSE. This API will be used in other places to determine if vrouter is running in multihomed mode.

-   There is also one global variable **router-\>vr_eth_if** which currently stores the single fabric interface associated with vrouter. This needs to be changed into an array of size 3 to store the mulitple fabric interfaces. This is mainly used in context of MTU change notification for vhost0 and adjusting MSS in TCP packets.

### 9.2.2 vhost0 Vif interface changes

-   In the multihomed vrouter case, the vhost0 will not have an IP address assigned to it. Also the mac assigned to the vhost0 interface will be the VRRP mac 00:00:5e:00:01:01 instead of the fabric interface mac.

-   The vif interface for vhost0 and fabric interface have a **vif_bridge** element. This vif is used mainly to xconnect packets from vhost0 to fabric interface and viceversa. As part of mulithoming, since there could be more than one fabric interface,
    the **vif_bridge** needs to be made as an array of vifs.

-   In case of vhost0 vif, the **vif_bridge\[0]** will contain all the fabric interfaces. In case of a fabric interface vif, the **vif_bridge\[0\]** will point to the vhost0. In non multihomed case, only the vif_bridge\[0\] will be used for the fabric interface as well.

-   The vrouter vhost interface driver in kernel mode maintains the fabric interface associated with the vhost0 interface in the **vhost_priv** struct. It also contains apis to enable/disable xconnect mode and for that it again uses this information. In the **vhost_priv** struct, an array of fabric interfaces needs to be maintained. These changes are required to be done for both the vhost0 addition and deletion cases.

-   Changes are required in hif_add() to use vhost0 mac instead of fabric interface mac.

### 9.2.3 XConnect mode changes

Xconnect flag is used on the vhost0 VIF and fabric VIF interfaces to indicate that all the traffic will bypass vrouter processing, and packets will be directly enqueued to the cross connected interface. This is mainly used when agent is not alive and vrouter tables may not be available.

It is also used during underlay ARP processing.

#### 9.2.3.1 Xconnect on fabric interface

The following algorithm will be used for incoming packets when Xconnect flag is set on the fabric interface:

-   If is_vrouter_mulithomed(), // checked based on fabric interface count
    -   If ARP req/resp packet, then packet will be sent to the linux stack using the same incoming interface. A new handler hif_rx_pass() will be implemented to return the special code RX_HANDLER_PASS in the linux_rx_handler() so that linux stack processes such packets.

    -   For IP packets, the packet will be enqueued to vhost0 as usual.

-   Else

    -   The vif-\>vif_bridge\[0\] will have the pointer to vhost0 interface and we will directly enqueue the packets to the vhost0 interface using hif_rx().

#### 9.2.3.2 Xconnect on vhost0 interface

The following algorithm will be used for incoming packets when Xconnect flag is set on the vhost0 interface:

-   If is_vrouter_multihomed(), // checked based on fabric interface count
    - If ARP req/resp packets, send the packets to the fabric interface by matching the src mac of the packet or the dst subnet the ARP is destined to.

      // \[TBD\] This may not be required as per the new design of using
      netfilter hook.

    - In case of IP packets, Select a fabric interface by computing a hash of 5 tuple information from the packet.

    - Send the packet out via the selected fabric interface using hif_tx().

-   Else

    -   The vif-\>vif_bridge\[0\] will have a pointer to the fabric interface through which to send the packet out.

### 9.3.4 Nexthop changes

To load balance the tunneled traffic (overlay traffic) across the multiple underlay uplink interfaces, the existing Tunnel NH will be enhanced to do underlay ECMP load balancing.

A new flag to indicate this eg: NH_TUNNEL_UNDERLAY_ECMP will be added and programmed by Agent.

The existing Tunnel nexthop will be modified as given below:-

vr.sandesh

`buffer sandesh vr_nexthop_req {`

​		`list\<I8\> nhr_encap_valid; // max 3 elements; indexed 0,1,2`

​		`list\<i32\> nhr_encap_oif_id; // max 3 elements; indexed 0,1,2`

​		`list\<byte\> nhr_encap; // max 3 \* encap_size elements; indexed 0\*encap_size, ..`

`};`

NOTE:

The indexes will be mapped 1-1 across the 3 fields i.e nhr_encap_valid\[0\] is for nhr_encap_oif_id\[0\] and so on.

In the case of Encap NHs and Tunnel NH without UNDERLAY_ECMP flag, only index 0 will be valid.

In case of Tunnel NH (w UL ECMP), Agent needs to always send the nhr_encap\[\] for all the 3 devices so that indexing of Encap happens properly at vrouter. Invalid nhr_encap can be memset to 0. Also the nhr_encap_oif_id\[\] should be set for all the 3 devices with either a valid vif index or -1 in case of invalid index.

vr_nexthop.h

`// A new flag defined to represent Tunnel nexthop with underlay ECMP`

`#define NH_FLAG_TUNNEL_UNDERLAY_ECMP 0x10000000`

`struct vr_nexthop {`

​			`uint16_t nh_data_size; // This would represent the total size of nh_data array;`

​													`// The size of nh_data will depend on whether it is an`

​													`// Encap NH, Tunnel NH (w/o UL ECMP)`

​													`// Tunnel NH (w UL ECMP)`

​			`struct vr_interface \*nh_dev\[3\]; // The nh_dev will be made an array of max 3 elements`

​			`uint8_t nh_data\[0\]; // Array containing the actual encap data.`

​			`uint8_t nh_encap_valid\[3\]; // This flag will indicate which dev and encap is`

​															`// valid in case of Tunnel NH (w UL ECMP)`

`};`

In case of Encap NH and Tunnel NH (w/o UL ECMP) only nh_dev\[0\] will be valid and nh_data will contain only 1 Encap information.

In case of Tunnel NH (w UL ECMP), there can be more than one valid nh_dev\[\] and nh_data will contain 3 \* 1 Encap information.

The nh_encap_valid\[\] will be used to determine which of the Encap and nh_dev are valid.

At the time of Tunnel NH add/update, the nh_dev\[\] and nh_data will be set accordingly based on nh_encap_valid\[index\] value.

As part of nexthop processing (nh_reach function), the Underlay ECMP flag will be checked in case of Tunnel NH. If it is set, then flow_underlay_ecmp_index will be accessed (in case of flow based forwarding). If flow is valid and flow_underlay_ecmp_index is valid, then the outgoing interface and L2 Encap corresponding to the underlay index will be selected.

The packet will be encapsulated based on the Tunnel information and selected L2 Encap and then sent out via the selected outgoing interface.

In case vrouter is operating in packet mode for this packet (flow_underlay_ecmp_index = -1),

The vrouter will compute a hash on the 5 tuple of the packet and select the outgoing interface and L2 Encap information. Any invalid interfaces will be ignored in the process of selection.

The nexthop function to dump the nexthop in sandesh msgs would also be developed.

### 9.3.5 Flow handling

A new field called **flow_underlay_ecmp_index** (Underlay ECMP index) will be maintained in the flow to select the underlay uplinks for the flow. This would be programmed by Agent during flow creation and updated as and when underlay links go down and come back up or when the GW is unreachable. Vrouter will not do any load balancing in this case. vRouter will use the flow_underlay_ecmp_index to send the packet out.

In case of flows starting from the fabric side, Vrouter can give a hint to agent to indicate the uplink interface on which the packet came in. This can be used by Agent to program the same in flow_underlay_ecmp_index of the flow, so that outgoing packets of the flow also take the same path. Today, for overlay ECMP paths the packet outer source ip is used to select the overlay ECMP tunnel. In case of underlay multihoming, Agent may have to use the outer dst mac of the packet to select the flow_underlay_ecmp_index.

### 9.3.6 BUM traffic handling

The same nexthop used for unicast traffic will be used for BUM case as well. The only difference would be, it will be part of Composite Multicast Nexthop. No additional changes required.

### 9.3.7 CLI changes

-   VIF utility

    -   Changes to take multiple physical interfaces as input at the time of vhost0 vif addition.

-   NH utility

    -   Changes to add Composite ECMP nexthop.

    -   Changes to dump the new nexthop.

-   Flow utility

    -   Changes to dump the additional field in flow.

#### **Describe changes in all affected components such as controller, agent, etc.**

#### **Describe all data model schema changes applicable to this feature**

#### **Describe changes to key data structures**

#### **Describe synchronous/asynchronous nature of any API calls being made if any**

#### **Describe the impact on the feature during process restart if any**

#### **Describe all logs and debug messages associated with this feature**

#### **Describe all high availability considerations**

**10. Performance and scaling impact**
======================================

#### **API and control plane**

#### **Scaling and performance for API and control plane**

#### **Scaling and performance for API and forwarding**

**11. Upgrade**
===============

#### **Describe upgrade impact of the feature, including ISSU**

#### **Schema migration/transition**

**12. Deprecations**
====================

#### **If this feature deprecates any older feature or API then list it here**

**13. Dependencies**
====================

#### **Describe dependent features or components**

#### **Describe any new infrastructure component required**

**14. Security Considerations**
===============================

#### **Describe all security aspects applicable.**

#### **Describe all impact to new/existing third-party code including licensing**

**15. Testing**
===============

#### **Unit tests**

Manual:

1)  Insert vrouter.ko module and check manually if addition and deletion of interfaces is happening for both fabric and vhost interfaces

2)  Enable and disable xconnect mode and check the above

Automation:

1)  Check if addition and deletion of vhost interface and multiple fabric interfaces is happening with cross_connect_idx passed as an array argument

2)  Send a TCP packet with size greater than MTU of the physical interfaces and check if the TCP packet is received at the other end with adjusted MSS which is minimum MTU of all physical interfaces

3)  Send ARP and IP traffic between vhost and fabric interfaces and vice-versa in xconnect mode

4)  Send IP traffic between vhost and fabric interfaces and vice-versa and ARP traffic between fabric and vhost in normal mode.

#### **Dev tests**

#### **System tests**

**16. Documentation Impact**
============================

**17. References**
==================

