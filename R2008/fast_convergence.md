**Fast Convergence in Contrail Clusters**

-   **Introduction**

Network Convergence and fast failover have become instrumental in high
performance SP networking thanks to the increasing deployment of
sensitive applications (e.g. real-time). As networking and service
failures are in essence inevitable, the industry introduced a set of
innovations to maximise the availability of services.

Contrail implements a Software Defined Networking solution that provides
Network Virtualisation at Compute level thanks to overlay networking.
However, the solutions proposed by competition (vertical stacks) are
actually based on hybrid approaches mixing virtualized and physical
networking. Hence, Contrail has to compete with PNF-standard convergence
solutions, which over time have onboard dozen of features to meet highly
demanding WAN SLAs of Service Providers (e.g. FRR, LFA, TI-LFA, BFD,
Ethernet OAM\...).

-   **Problem Statement**

Contrail today is not on par with PNF in terms of convergence, while
overlay networking and fabric integration introduce additional
complexity. In the current contrail architecture, vRouter to vRouter
path liveness detection is not present. This causes outages to forward
traffic whenever any vRouter fails (node failures) or reachability among
vRouters break (path failures). 

The objective of the effort here is to achieve sub-second convergence
(0.5 seconds downtime) in Contrail integrations (i.e Gateway + Fabric +
Contrail) for any failure use case. To achieve this, the document
outlines a set of proposals to solve this problem, in multiple phases.
Each phase increases in complexity (and hence resources/time required).
Hence can be done one on top of another in a serialized order.

-   **Reference Architecture**

The figure below illustrates the reference architecture.

![A picture containing computer Description automatically
generated](./images/FastConvergenceRefArch1.png){width="6.5in"
height="4.427083333333333in"}

The gateway connectivity with the Fabric is based on redundant
point-to-point connectivity. The choice for redundant link provides
better BGP control plane stability as the GW Next Hop self operation is
not affected in case of spine or link Failure.

A loopback is defined at Gateway level to handle the Tunnel Endpoint
(i.e. MPLSoUDP). Additionally, this loopback is  the source of the
MP-BGP session toward the Contrail Control Nodes.

![A screenshot of a cell phone Description automatically
generated](./images/FastConvergenceRefArch2.png){width="5.0in"
height="4.361111111111111in"}

Gateway Connectivity with Fabric

In general, routing between gateways and spine devices is either:

-   Static routing at spine (redistributed to the underlay routing
    protocol of the Fabric)

-   Integrated with the Fabric underlay routing protocol such as
    Single-hop eBGP (afi/safi 1/1 or 2/1), ISIS or RIFT (future).

Network signaling in control nodes are implemented as follows:

-   Native MP-BGP for connectivity toward Gateway routers (service
    planes: vpnv4/vpnv6, evpn, route-targets).

-   xmpp connectivity toward Contrail vrouters (i.e. virtualized
    computes based on Contrail). The content of xmpp sessions is
    actually a pseudo-BGP signalling that carries well known BGP
    attributes (communities, preferences, prefixes, mask\...). 

As of today, the connectivity of Compute Node is exclusively based on
LAG. ECMP at vrouter is under investigation: this is not available in
the current code. It is not clear today when this solution will take off
in practice due to the fact that generic compute integrations are
extensively based on LAG.

-   **Problem Analysis**

    -   **Failure Handling **

Once a failure occurs, the following mechanisms work in sequence to
isolate the failure and repair as necessary in order to ensure traffic
blackholing is resolved as quickly as possible.

**Detection**: The first step in resolving a failure is to have an
equipment detect that a failure has occurred (e.g. VM, PNF, Fabric
failure). Once the failure is detected, corrective actions (such as
finding an alternate path and installing it in the forwarding table) can
be done. Detection time is the time taken . Unlike in a physical
environment, where link down events can be  associated with a detection,
virtual environments may usually rely on KeepAlive mechanisms to detect
failures. A notable requirement in providing fast convergence is that
such \"detection time\" be bounded.

**Local repair** (a.k.a Fast Reroute/Fast convergence) Once a failure is
detected, Right after the detection of a failure, the local system can
divert traffic to an alternative path if available (i.e. if the backup
has been previously signalled). At this point, other systems have not
taken any corrective actions and may not simply be aware of the
topological change.

**Global repair** (a.k.a network convergence): Global repair happens
after all systems in the network are notified of the topological and
have enforced corrective actions (when appropriate), and the signaling
of the topological changes being ideally propagated by routing
protocols. After "global repair", the network is in a steady state with
a consistent control plane and datapaths.

![A close up of a logo Description automatically
generated](./images/FastConvergenceFailureSequence.png){width="6.5in"
height="1.7916666666666667in"}

Sequence of events after a failure occurs

The availability of the services is bound to timers, network
capabilities and design. From the perspective of service availability,
local convergence can be enough, as long as an alternative forwarding
path has been provided by the network. This is a typical situation in DC
architecture, where ECMP or bond offer native redundant paths.

-   **Failure Types**

From a high level perspective, the spectrum of failures is generally
split into two main types: Edge versus Transit. However, the
introduction of overlay networking introduces another dimension and
cross dependencies, which makes the analysis more complex. 
Additionally, Edge and Transit are not disjoint functions. Indeed, a
Transit node can also be an Edge node, for example in the case of
collapsed micro fabric designs where BMS are connected to spine
switches. The failures can be split into overlay (i.e Virtual Machine or
pods within Virtual Networks) and underlay failures (vrouter tunnel
endpoints) based on where the failure occurs.

-   **Failure Scenarios of Interest**

    1.  **Overlay Failures**

For workload failures in the overlay (virtual-machine or pod), the VMI
is always up and contrail advertises prefixes towards this VMI. The
challenge in this case is mostly about failure detection. In this
failure use case, the Contrail vrouter can detect and propagate any
failure through the local cluster toward Gateways once it is detected
using the existing health- check mechanism. This can be done in
sub-second fashion (with either VRRP timers or BFD). As a result,
workload failures in the overlay are not considered as problematic as of
today. This Use Case requires a minimum level of cooperation from the
VNF to reach sub-second convergence:

-   Support BFD health-check with sub-second timers -(e.g. BFD with
    100\*3 holdtime - ping is too slow)

-   Support VRRP sub-second timer (e.g. 100\*3 timers). This approach is
    less preferred as false positives (split brain) can happen due in
    case of slow underlay failure and convergence (Spine / Leaf / link
    failures).

    ii. **Underlay Failures**

Failures at the underlay as explained in detail in section 3.2 of
[[https://docs.google.com/document/d/1w\_6gtQsvYG2lXQoCnstxxOazTkFu9nrfOQMXJMwqwyA/edit]{.underline}](https://docs.google.com/document/d/1w_6gtQsvYG2lXQoCnstxxOazTkFu9nrfOQMXJMwqwyA/edit)
can be one of the following

-   under-1: Gateway Failure

-   under-2: Gateway-Spine Link Failure

-   under-3: Spine Failure

-   under-4: Spine-Leaf Link Failure

-   under-5: Leaf Failure

-   under-6: Leaf-compute Link Failure

-   under-7: Compute Link Failure

Amongst the above cases, two Use Cases have been identified as
problematic when deploying Contrail:

-   **under-1**: Gateway Failure (Dual Gateway for HA) 

![A picture containing computer Description automatically
generated](./images/FastConvergenceUnderlayFailure1.png){width="4.138888888888889in"
height="3.1666666666666665in"}

As previously described, the SDN gateway peers via MP-BGP with the
Contrail Control Nodes. As an Option B inter-AS framework is
implemented, the Gateway appears as:

-   A \*Tunnel Endpoint\* (MPLSoUDP, MPLSoGRE or VXLAN) from the
    perspective of vrouters for all prefixes originating beyond or from 
    gateway.

-   An \*Egress PE\* (Next Hop) from the perspective of a Remote PE.

 

Today, in case of a failure of the Gateway Node, the Control Node
requires a BGP hold-time expiration to detect the failure and generate
routing updates to the compute nodes (vrouter). Meanwhile, traffic from
the Compute nodes to the backbone will be subject to ECMP load balancing
to both Gateways. Therefore the convergence time for this failure case
is equal to the BGP hold-time (by default 90 seconds). A dependency on
the BGP Holdtime for convergence is not acceptable in a service provider
environment**.** Note that this timer can technically be decreased to 3
seconds. However, this is not a recommended configuration: BFD is indeed
the right alternative for fast peering failure detection.

-   **under-7**: Compute Node Failure (HA service) 

![A close up of a logo Description automatically
generated](./images/FastConvergenceUnderlayFailure2.png){width="4.138888888888889in"
height="3.1527777777777777in"}

This failure case is different from the previous cases because there is
no native backup path in a standard design. Indeed, in order for
convergence to happen, redundancy must be enforced at service level (i.e
overlay / VNF level) typically through the use of Virtual IP addresses
(i.e. a same VIP is reachable via two separate Virtual Machine
Interfaces hosted on different physical compute nodes). This can be done
in two different ways:

-   Active/Active: ECMP load balancing - a same prefix is advertised
    from different vrouters with identical BGP attributes so as to
    enforce load balancing

-   Single Active: Primary/Backup routing managed via routing
    preferences - this can be done in many ways via BGP, a MED approach
    is chosen in this section: other options such as Local Preferences
    or AS path length are perfectly valid as well

In the case of a Compute failure, the following sequence applies:

-   Expiration of the compute xmpp holdtime: 3\*5= 15 seconds

-   Control Nodes update routing tables (vpnv4, evpn) with deletion

-   Control Nodes propagate deletion of all address reachable via the
    failed Compute (mp\_unreach/withdraw) to Gateways (MP-BGP) and
    Computes involved in the same VNs (xmpp)

-   Gateways and Compute update their FIB. Without any specific feature
    (multipath / PIC-EDGE), this operation can take significant time in
    high scale scenarios as a linear time dependencies applies (JUNOS MX
    has roughly 2-5K prefixes per seconds and vrouter has around 10K
    prefixes per seconds). Optimisations and Requirements are discussed
    in a further section.

<!-- -->

-   **Proposed Solutions**

    -   **Overlay Failures**

For workload failures in the overlay, failure detection becomes
important. As previously mentioned this case has, since the Virtual
Machine Interfaces are always up, this Use Case requires a minimum level
of cooperation from the VNF to reach sub-second convergence:

-   Support BFD health check with sub-second timers  

-   Support VRRP sub-second timer

We can use one of the following modes of operation for failure
detection:

-   VRRP-based (i.e: VNF handle the failure): vrouter detects G-ARP to
    adjust routing so as to identify the primary interface.

> If the VNF implements VRRP as a redundancy framework (Single Active),
> it is possible to manage high-availability through the configuration
> of Allowed Address Pair at Contrail Level. For this purpose, the VRRP
> virtual IP and virtual MACs must be attached to the VNF ports.The
> drawback of this approach is that VRPP Keepalives are carried inside
> the overlay. As a result, VRRP timers must be set to a level higher
> than the worst-case fabric convergence time (which actually may depend
> on the scale of the fabric). If not, the standby VM may detect a VRRP
> hold-time expiration upon the fabric convergence event although the
> active VM is still alive and sending VRRP keepalives: this results in
> a split-brain situation.

-   BFD HealthCheck-based (VNF just needs to reply to Health Checks):
    requires contrail to be aware of the failure: ECMP or
    Active/Standby.

> It is possible to manage redundancy thanks to the Use of vrouter
> HealthChecks. Health checks are packets generated locally by the
> vrouter toward the VNF. A Virtual IP can be configured at Contrail
> level together with an Allowed Address Pair which can cover both ECMP
> and Primary/Backup use cases thanks to the configuration of identical
> or different Local Preferences at Virtual Machine Interface level. 

BFD has become a standard protocol for fast failure detection. This
should be the preferred approach for fast failure detection if supported
by the VNF thanks to its native integration in the contrail control
plane. Any route (local, bgpaas, static, AAP) gets deleted upon failure
of the BFD adjacency in a short timeframe (\<50msec measured in lab in
low \#route scale). 

-   **Underlay Failures**

Section 4 of
[[https://docs.google.com/document/d/1w\_6gtQsvYG2lXQoCnstxxOazTkFu9nrfOQMXJMwqwyA/edit\#heading=h.25287a2sukwd]{.underline}](https://docs.google.com/document/d/1w_6gtQsvYG2lXQoCnstxxOazTkFu9nrfOQMXJMwqwyA/edit#heading=h.25287a2sukwd)
details the different approaches that were considered in solving the
underlay failures. Although more complex compared to option 1 and option
2, option 3 presents the highest level of robustness and convergence
speed thanks to an optimal level of integration between Fabric and
Contrail (event-based). This is the reason why this solution is selected
as the best compromise and detailed in this section.

In parallel, other options such as Option 1 can be tactically deployed
for specific use cases such as public cloud, where compute nodes can be
deployed across several locations.

-   **Solution Description**

> **Phase I: Run BFD (over vHost0 VMI) between vRouter and it's
> connected ToR switch**

In this scheme, a static route is configured in the TOR for the
connected vRouter /32 address and BFD is enabled on it. This route is
redistributed into underlay inet.0 BGP. TOR is also peered (optionally
via Route Reflectors) with each of the contrail control-nodes to track
reachability to each vRouter underlay /32 prefix in default routing
table inet.0

In control-node, today BGP does not do next-hop reachability checks for
VPN routes received over bgp/xmpp. With this scheme, control-node should
do the reachability check for all routes' next-hops (which are typically
/32 compute node IPs) in the default inet.0 table. Only if the nexthop
is resolvable should the route be marked as usable and considered for
best path evaluation.

By doing this, control-node will ensure that VPN routes are advertised
only if /32 route for the vrouter is present in the inet.0 table. This
in turn implies that a compute node's routes are considered usable in
the network only if the compute node is alive and responding (with BFD)

Note: This scheme does not detect breakage in network connectivity
between compute-nodes per-se.

> **Phase I: Make Xmpp hold time configurable**

In order to to have better than 15 seconds detection in case of vrouter
failure, configurable xmpp timers will be made configurable to minimize
downtime as low as 1 second hold-time. It should be noted that low timer
values (below 9s) may not work in high scale scenarios and should be
used with caution.

This will require changes in multiple components including UI, schema,
control, agent, and xmpp library.

Schema Changes:
A knob needs to be added to global system configs indicating whether
fast convergence feature is enabled. In addition, another field is
needed to configure xmpp hold time. These are the changes:

```
diff --git a/schema/vnc_cfg.xsd b/schema/vnc_cfg.xsd
index 97e166b..08086e3 100644
--- a/schema/vnc_cfg.xsd
+++ b/schema/vnc_cfg.xsd
@@ -975,6 +975,20 @@ targetNamespace="http://www.contrailsystems.com/2012/VNC-CONFIG/0">
     </xsd:all>
 </xsd:complexType>

+<xsd:simpleType name="XmppHoldTimeType">
+     <xsd:restriction base="xsd:integer">
+         <xsd:minInclusive value="1"/>
+         <xsd:maxInclusive value="90"/>
+     </xsd:restriction>
+</xsd:simpleType>
+
+<xsd:complexType name="FastConvergenceParametersType">
+    <xsd:all>
+        <xsd:element name="enable" type="xsd:boolean" default="false" description="Enable/Disable knob for all Fast-Convergence parameters to take effect"/>
+        <xsd:element name="xmpp-hold-time" type="XmppHoldTimeType" default="90" description="The negotiated XMPP hold-time (in seconds) for sessions between the control and data plane"/>
+    </xsd:all>
+</xsd:complexType>
+
 <xsd:complexType name="PortTranslationPool">
         <xsd:element name="protocol" type="xsd:string"/>
         <xsd:element name="port-range"    type="PortType"  required='exclusive'
@@ -1059,6 +1073,11 @@ targetNamespace="http://www.contrailsystems.com/2012/VNC-CONFIG/0">
      Property('rd-cluster-seed', 'global-system-config', 'optional', 'CRUD',
               'Used to create collision-free route distinguishers.') -->

+<xsd:element name="fast-convergence-parameters" type="FastConvergenceParametersType"/>
+<!--#IFMAP-SEMANTICS-IDL
+     Property('fast-convergence-parameters', 'global-system-config', 'optional',
+              'CRUD', 'Fast Convergence parameters.') -->
+
 <xsd:element name="ip-fabric-subnets" type="SubnetListType"/>
 <!--#IFMAP-SEMANTICS-IDL
      Property('ip-fabric-subnets', 'global-system-config', 'optional', 'CRUD',
```

XMPP changes:
Today, both xmpp server and clients start with default hold time of
90s. With proposed changes, both Xmpp server and client can be
instantiated with specific hold time out value, default for which is
90s (same as today). New APIs will be added which can be used by server
and client to reset hold timer.

Control changes:
Control node will start receiving xmpp hold time as part of global
system config. If the existing value of hold time is different from the
newly configured value then timers of all existing connections will be
reset with the new timer value. It may be noted that same value will be
propagated to agent node so if the new value is too low then there might
be a flap of xmpp session depending on the load on the system.

Agent Changes:
Similar to control node, agent will also receive xmpp hold time as part
of global system config. Initially all agents will connect to
controllers using default xmpp time out of 90s only. Whenever user
configures different value, timers of all existing connections will be
changed to reflect new value and all the newer connections will be
started with newer value.

UI Changes:
There is a set of parameters added in global system config under the
knob “fast-convergence-parameters”. This will have a few fields
including ‘enable’ and ‘xmpp-time-out’ as mentioned in the previous
section. UI will need to expose all those parameters.


> **Phase II: Run BGP in each compute-node and BFD on top of that BGP
> session**

In this scheme, some sort of BGP daemon is run in each compute node in
order to advertise and receive /32 routes in the underlay table (inet.0)
with the TOR switch. In this proposal, a trimmed down version (bgp only)
of control-node is suggested for running in each compute node. cRPD is
also a potential candidate. E-BGP session is run between each compute
node with its associated TOR(s).

When BGP comes up and vRouter is ready to forward, the self  /32 address
should be advertised over BGP in the underlay inet.0 table. This causes
control-nodes to be able to resolve all VPN routes with next-hop as this
vRouter compute node.h. If the TORs are connected to BGP route
reflectors, then each TOR shall advertise its local paths to reach the
associated compute node as the best path and advertise over bgp to the
route-reflector. RR shall choose one of the paths as best and advertise
to other routers including control-node.

On the receiving side, all the learnt /32 routes (of other compute
nodes) are programmed into the kernel\'s default routing table. vRouter
must always use this updated table while making forwarding decisions.
Also, vRouter should send tunneled packets to other vRouters only if /32
route is indeed present in the kernel table. (and not using the
0.0.0.0/0 route). This helps to converge quickly, especially in cases
when multiple paths exist for a given VPN destination.

Note: It is recommended to implement nexthop tracking in compute-node.
This ensures that whenever /32 nexthop route is deleted, all
destinations that are to be tunneled over to that nexthop are correctly
sent over updated set of available nexthops (excluding the the one that
went down or became unreachable) (aka Prefix Independent Convergence
PIC)

By running BFD over this bgp session, any failure can be detected
quickly.

-   **Implementation Details**

<!-- -->

-   **Control-Node**

    -   **Path Resolver Infrastructure**

Control-node today has a path resolver infrastructure that is capable of
resolving any path in a given table. A PathResolver instance is created
at startup for every BgpTable that tracks all ResolverPaths needing
resolution. It also maintains the list of next-hop IP addresses
(ResolverNextHop) that need to be resolved. 

A ResolverPath keeps a list of resolved BgpPaths it has added. A
resolved path is added for each ecmp eligible BgpPath of the BgpRoute
that tracks the nexthop of ResolverPath. The nexthop is represented by
ResolverNexthop. The resolved paths of the ResolverPath are reconciled
when the update list in the PathResolverPartition is processed. New
resolved paths may get added, existing resolved paths may get updated
and stale resolved paths could get deleted.

The ResolverNexthop maintains a vector of ResolverPathList, one entry
per partition. A ResolverNexthop is created when resolution is requested
for the first BgpPath with the associated IP address. At creation, the
ResolverNexthop is added to the PathResolver\'s
registration/unregistration list so that the ConditionMatch can be added
to the BgpConditionListener.

Each ResolverPathList is a set of ResolverPaths that use the
ResolverNexthop in question. When BgpRoute for the IP address being
tracked changes, the ResolverNexthop is added to the update list in the
PathResolver. The PathResolver processes the entries in this list in the
context of the bgp::ResolverNexthop Task. The action is to trigger
re-evaluation of all ResolverPaths that use the ResolverNexthop.

-   **Proposed changes**

Control-Node today does not do any next-hop resolutions for xmpp vpn
routes (except for those received over BGPaaS sessions). When a route is
received from another agent (via xmpp) or via bgp, next-hops are simply
accepted as is. With this proposal, nexthop resolution must be triggered
in inet.0 table in order to ensure that nexthop (in case it is vrouter
agent) is properly resolvable (and hence reachable). This new behavior
can and will be controlled by a config knob so that we do not have to do
this at all times.

If an agent fails which is typically detected very quickly by using BFD,
TOR BGP withdraws the /32 compute route quickly from all peers including
control-node. Control-node then can mark all routes that have the failed
compute as nexthop as unresolvable and withdraw them from the cluster
from all bgp and xmpp peers to which the routes were advertised before.

**Data Path **

Agent already has support to run single hop BFD over any
virtual-machine-interface (VMI). In this schema, this is extended to run
over vhost0 vmi, typically to the TOR gateway address. In phase I, 

1.  detection is done using BFD over LAG and

2.  Local repair in Datapath is done using Prefix Independent
    Convergence

In phase II, more is required as BGP should be run in each compute.

Fast Convergence in vRouter ===\>  Detection(TableA) + Repair (Table B)

+-----------------------+-----------------------+-----------------------+
| TableA - Detection    |                       |                       |
+=======================+=======================+=======================+
| Contrail feature      | Detection Time        | Remarks               |
+-----------------------+-----------------------+-----------------------+
| BFD over LAG          | 1.  11Milliseconds    | 1.  Dependency on QFX |
|                       |     (\~100msec to     |     support for Micro |
|                       |     300msec).         |     BFD               |
|                       |                       |                       |
|                       | 2.  Dynamically       |                       |
|                       |     Configurable in   |                       |
|                       |     scaled setup      |                       |
+-----------------------+-----------------------+-----------------------+
| Configurable Xmpp     | 1.  Typical Detection | 1\. Detection time    |
| timers                |     is about Seconds  | depends is affected   |
|                       |     (\> \~ 3 sec)     | by the Topology       |
|                       |                       | design.               |
+-----------------------+-----------------------+-----------------------+
| Normal BFD + L3       | 1.  Milliseconds      | Not Scoped            |
| Multihoming           |     (\~100msec to     |                       |
|                       |     300msec).         |                       |
|                       |                       |                       |
|                       | 2.  Dynamically       |                       |
|                       |     Configurable in   |                       |
|                       |     scaled setup      |                       |
+-----------------------+-----------------------+-----------------------+

+-----------------------------------+-----------------------------------+
| Table B - Fast Convergence repair |                                   |
+===================================+===================================+
| Local Repair Type                 | Remarks                           |
+-----------------------------------+-----------------------------------+
| 1.  Prefix Independent            | Will converge Traffic from        |
|     convergence infrastructure in | compute to compute (Intra cluster |
|     vRouter and Agent             | traffic)                          |
|                                   |                                   |
| 2.  Path Resolver Infrastructure  |                                   |
|     in Control Node               |                                   |
+-----------------------------------+-----------------------------------+
| 1.  Path Resolver Infrastructure  | Traffic from MX (Inter cluster    |
|     in Control Node               | traffic)                          |
+-----------------------------------+-----------------------------------+
| 1.  Path Resolver Infrastructure  | Traffic from BMS                  |
|     in Control Node               |                                   |
+-----------------------------------+-----------------------------------+

In the case of LAG where interfaces connected to different TORs (with
same virtual IP) using a bond interface, the agent should set up
separate BGP and BFD sessions to specific TORs associated. This can be
accomplished by sending BFD packets that are bound to specific member
interfaces. This implies that the contrail config model must be extended
to clearly map this relation between member interface and associated
TOR's specific IPs (to which BGP and BFD sessions can be established).

Even if TOR happens to send BFD packets to the agent via fabric (when
the directly connected path fails), because the agent always sends
packets using the member interface, BFD should eventually fail. Only
case this does not work is when (somehow) the link fails in only one
direction from TOR towards the vRouter. I think such a scenario is very
unlikely...

**Prefix Independent Convergence (PIC)**

In order to improve convergence during failure scenarios, prefix
independent convergence can be implemented. In this approach, whenever a
/32 compute-node's next hop address is deleted from the inet.0 (default)
routing table, such nexthops must be marked "down" so that all
associated (vpn) routes that resolve over the failed compute immediately
stop using the failed nexthop. 

In the current vrouter implementation, vrouter flows hold reference to
the composite-NH index and route table reference. When the packets are
flow trapped, flow doesn't have the composite-nexthop information. Flow
holds the composite NH index and uses the VRF to lookup the route for
forwarding the packet. 

Agent creates TunnelNH for remote overlay routes received over the xmpp
message. If the tunnel NH for the remote route is in the same subnet,
ARP is started and this TunnelNH is invalid unless ARP is resolved. Any
changes in the TunnelNH is sent to the vrouter via ksync. Vrouter uses
the TunneNH for validating the source info for the compositeNH.

In the proposed design, when the "fast convergence" knob is set,
control-node advertises underlay /32 route for inet.0. These routes will
be added in the default vrf in the Agent. Agent will now have two paths
for the /32 underlay route, 1) Locally added by ARP and 2) added by BGP
peer. When the /32 underlay route for inet.0 is deleted, Agent will
invalidate the TunnelNH that is dependent on the deleted underlay /32
route and sent to vrouter through ksync.

**Flow mode**

Agent computes the Flows ecmp index for the initial flows and sends the
flow update sandesh message to vrouter. Subsequent of the Flow ecmp
index is done in the vrouter on demand.

ComponentNH deletion in a compositeNH, Agent "punches hole" in
corresponding componentNH position. For example if the compositeNH 
(shown in Fig.2) has NH-A and NH-B, if NH-A is deleted compositeNH will
still have two entries but the NH-A is null. "Punching of the hole" is
done so that the traffic for the existing flows with the ecmp index
pointing to NH-B is not affected.

+--------------------+           +--------------------+
|                    |NH-A Deleted                    |
|    CompositeNH     |           |    CompositeNH     |
|                    |---------->|                    |
|--------------------|---------->|---------------------
|   NH-A  |  NH-B    |           | “Null”  |  NH-B    |
|         |          |           |         |          |
+--------------------+           +--------------------+
                     Fig.2 CompositeNH structure
                                                   

During the forwarding decision of the packet after the route lookup,
vrouter modifies the flow's ecmp index when it is pointing to the "null"
componentNH.

Packets that are flow trapped, the ecmp index will be pointing to the
deleted /32 TunnelNH. If the TunnelNH pointed by the ECMP index is
invalid, flow's ecmp index is updated with available TunnelNH in the
composite-NH structure.The Routes that don\'t have composite-NH,
forwarding decision in vrouter is unchanged and these routes will be
acted when route retract xmpp messages received.

**Packet-mode:**

In this mode, vrouter chooses whichever valid TunnelNH available in the
compositeNH structure. 

**New Flows:**

When the /32 compute node down message is received by Agent,
corresponding TunnelNH is marked invalid. For the new flows, the ECMP
index for the composite NH will point to the valid TunnelNH.

                                                                                          

 +----------------+
 |                |
 |   Agent        |
 |                |
 +----------------+
     |
     | Ksync TunnelNH notification with Valid/Invalid Flag
     |
     |                                   TunnelNH-Valid
+----v-----------+      +-----------------+     +-----------+
|                |      |                 |     |           |
|Pkt Flow Lookup |----->|   RouteLookup   |---->| Forward   |
|                |      |                 |     |           |
+-^--------------+      +-----------------+     +-----------+
  |                                 |
  |   +-----------------------+     |
  |   |                       |     | TunnelNH-Invalid
  +<--| Reprogram Flow Index  |<----+
      |                       |
      +-----------------------+

Note: Valid TunnelNH check done only for composite-NH types
                    Fig.3 PIC in vrouter

**     **                                                                                      

  

**BFD over LAG*

**Interface Device hierarchy in vRouter:**

+-----------------------------------------------------------+ +-------------+
|  +----------------+  +----------------+    +-----------+  | | +--------+  |
|  |                |  |                |    |           |  | | |        |  |
|  |Physical Devices|->|   Bond Device  |--->|  VLAN     |----->| VHOST  |  |
|  |                |  |                |    |           |  | | |        |  |
|  +----------------+  +----------------+    +-----------+  | | +--------+  |
|              Linux Kernel network stack                   | |  vRouter    |
+-----------------------------------------------------------+ +-------------+
     ** Note: skbuff->dev is modified across each device hierarchy **
                   Fig.4 Interface Device hierarchy in vRouter


Design\#1: Set up separate BFD sessions to specific TORs associated.

Design Flow:

1.  Establish a BFD session with each of the TOR. One BFD session per
    TOR. 
2.  BFD traffic from Agent to TOR:

    a. Agent to send BFD session\#1 packets on member-links going to
        TOR1

    b.  Agent to send BFD session\#2 packets on member-links going to
        TOR2 

    c.  BFD session\#1 in TOR1 will go down if the member-links going to
        TOR1 is not reachable. TOR1 removes the static routes and
        control-node does NH reachability checks for this TOR1


2.  BFD traffic from TOR to Agent:

    a.  Can come on any member-links in LAG. If the Agent is down, Both
        TOR1 and TOR2 BFD sessions will be down.  

Challenges:

1.  Deterministic way of establishing a session on TOR with its
    member-links requires that the session ID be tied with TOR. BFD
    being a L3 protocol, if the session is initiated from the TOR the
    packets can reach the Agent from any of the member-links in the LAG.

2.  Determining the number of TORs connected to Agent. 

3.  Impossible to determine member-link interfaces in LAG belonging to
    TOR using LACP.

This design approach is not feasible because:

-   In this design, the problem is with running BFD sessions tied to a
    particular TOR over its member-link in LAG without internal
    knowledge of the LAG structure and it is impossible to resolve
    listed in the above challenges in vRouter. It is not guarantee a
    detection of anything but a full LAG shutdown within the BFD timeout
    period.

-   LACP packets that are exchanged between SystemA and TORs were
    captured on SystemA interfaces, Eth0 and Eth1. By using LACP SystemA
    will not be able to determine the number of TOR switches the SystemA
    is connected to and also impossible to determine the LAG
    member-links.

** **

Design Solution\#2: Using Micro BFD instead of Normal BFD over LAG** **

To overcome the problem of  running BFD sessions tied to a particular
TOR,  we can use the IETF proposed (rfc7130) BFD run on each constituent
link of the LAG. We call the BFD sessions running on each link a "micro
BFD session". Currently Micro BFD is supported in JUNOS for MX platforms
and also standard supported across switch vendors.

In this mode of running BFD on LAG interfaces, independent, asynchronous
mode BFD sessions run on every LAG member link in a LAG bundle. Instead
of a single BFD session monitoring the status of the UDP port,
independent micro BFD sessions monitor the status of individual member
links. 

Each of the micro BFD sessions use their own unique, local discriminator
values, maintain their own set of state variables and have their own
independent state machine. If the micro session goes Down, the LAG
Manager is informed which removes the member link from the LAG.

Each micro BFD session (rfc7130) is a regular [[RFC
5880]{.underline}](http://tools.ietf.org/rfc/rfc5880.txt) compliant BFD
session. Sometimes users gets confused between regular BFD and micro
BFD. To avoid the confusion and misconfiguration in the setup, micro BFD
uses different UDP port than the RFC 5880.

   +----------------+
   |                |
   |   Agent (BFD)  |
   |                |
   +----------------+
       ^      |
       |      |
  +---------------+
  |    |PKT#0 |   |
  +---------------+
 BFD_Rx|      |BFD_Tx
       |      |              +----------------+
       |      v              |                |
   +----------------+        |                |
   |                |<-------|                |
   |   vRouter      |        |  TOR           |
   |                |------->|                |
   +----------------+ uBFD   |                |
                             +----------------+
        Fig.5 Micro BFD Packet Journey in Compute

**BGP Daemon (in each compute)**

In order to make each compute-node participate in the underlay routing,
some dynamic routing-daemon should be run in order to advertise and
learn routes of other compute nodes that reside in the underlay.
Possible options are

1.  BIRD

2.  cRPD

3.  Control-Node BGP

Today, BIRD is already used as part of the contrail multi-cloud
solution. cRPD is a potential candidate too, but it brings in licensing
issues especially when working with purely open-source based solutions.

The current recommendation is to utilize control-node BGP for this
purpose as it natively and seamlessly integrates with the rest of the
contrail infrastructure already.

By using control-node and ifmap (bgp-router) based configuration with
the PhysicalRouter object, bgp-peering can be established seamlessly
between all pairs of computes (VirtualRouters) and their associated TORs
(PhysicalRouters)

**Fabric/Device Manager**

All necessary configurations in the TOR switches such as BFD, BGP,
static routes (towards compute-nodes), etc. must be automatically
configured by the device-manager as applicable. This requires
association between vRouter and its associated TOR(s) PhysicalRouter
objects in configuration.
