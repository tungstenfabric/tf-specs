# 1. Introduction
Contrail support for dataplane learning of MAC-IP bindings reachable over
VM interfaces.
# 2. Problem statement
A VM deployed in open stack hosts multiple pods with their own MAC-IP addresses.
IP addresses for pods are assigned by external applications in the same subnet
range of VM interfaces over which pods are reachable. So ,contrail  does not
have knowledge over the IP addresses of pods hosted by VMs. In order to support
pod to pod communication over open stack managed IaaS network, vrouter  must
learn MAC-IP bindings of pods and advertise them to controller.
The MAC-IP bindings should be advertised to SDN gateway using ipvpn and evpn
address families.
BFD should be supported for pod liveness.
# 3. Proposed solution
Vrouter traps packets to agent for learning unknown MAC-IP addresses
seen on VM interface by spoofing ARP packets. agent processes the MAC-IP
learning events and generates evpn type2 MAC/IP routes, inet routes for
L2/L3 mode and evpn type2 MAC  only routes for L2 only mode on virtual network.
A new type of BFD healthcheck template is required to support BFD session for
target IPs on VM interface. it should be applied on virtual network.
Agent initiates BFD session for new MAC-IP addresses if new type of BFD HC is
enabled on virtual network and monitor target list of health check contains
new IP address.
agent also sends ARP/ND packets periodically to newly learnt IP addresses and
this is used as a fallback mechanism to check pod’s liveness in absence of
BFD session for that IP address.
The learnt routes are deleted in the following cases.

###BFD session failure:
Whenever BFD session goes down for a target IP address on a VM interface,
Agent deletes the corresponding routes from inet and evpn table.
###ARP/ND failure:
agent deletes routes if arp is not resolved after max retries.
###Pod movement(IP address movement):
Actually pod does not move from one VM to other but it can be destroyed in one
VM and created in some other VM with the same IP address. Note that it is a
IP move and not a MAC move. There are two  cases.
####Movement in same compute
Since pod is destroyed and launched in same compute, so compute retract the
older evpn routes for  MAC1/IP1 pair, advertises newer evpn routes for
MAC2/IP1 pair and updated L3VPN route for IP1 to controller.
and update and advertise inet route.
####Movement across computes.
pod1 with MAC1/IP1 in compute1 is destroyed and pod2 is created with
MAC2/IP1 in compute2 in a virtual network VN1. compute3 also subscribed for
VN1(VRF).
1. Based on ARP activity, compute1 first advertises evpn type2 MAC/IP route for
MAC1/IP1, evpn type2 MAC route for MAC1 and L3VPN IP route for IP1 to controller
and compute2 and compute3 receives evpn and L3VPN routes and it derives inet route
and bridge route from evpn route.
2. When pod2 is created, compute2 advertises evpn type2 MAC/IP route for MAC2/IP1,
evpn type2 MAC route for MAC2 and l3vpn route for IP1 to controller.
controller sees l3VPN route for IP1 from compute1 and compute2, it sends ecmp
L3VPN route add request with nexthops compute1 and compute2  to computes.
3. compute1 receives evpn type2 route for MAC2-IP1 and checks whether locally
learnt MAC-IP entry present with same IP. If it is present, it detects that IP
is moved and retracts evpn type2 MAC/IP route for  MAC1/IP1, evpn type2
MAC route for MAC1 and L3VPN route for IP1. Controller sends route deletion
event for evpn type2 MAC/IP route MAC1/IP1, evpn type2 MAC route MAC1 and
route update event for L3VPN route for IP1 with nexthop set to compute2 to
computes. All computes delete EVPN routes and its derived routes corresponding
to MAC1/IP1 pair.
###VMI health check failure/oper state down/VMI deletion:
When VMI’s health check fails or open state is down or VMI is deleted,
pod routes learnt over this VMI are deleted.
Note1: MAC-IP learning should be enabled on VN before launching of pods.
Note2: no validation check for target IPs in monitor list on whether target IPs
are present in same subnet of VN or not.
##3.1 Alternative considered
None
##3.2 API schema changes

1. MAC-IP learning enable/disable option on virtual network
2. new health check template for BFD mintoring for pods
3. virtual netwoek changes for supporting health check object binding on
   virtual network
##3.3 User workflow impact
User need to enable MAC-IP learning on virtual network to use this feature.
User should use new health check template to run BFD for pods.
##3.4 UI changes
1. MAC-IP learning flag for virtual network object.
2. new BFD health check type and monitor target should support list of
    IP addresses.
3. service healthcheck option in virtual network object.
##3.5 Operations and Notification impact
None.
#4. Implementation
##4.1 Work items
###4.1.1 Agent
A new MAC-IP learning module is added to manage the following.
1. process trap entries for MAC-IP learning.
2. creates interface NH and MPLS labels for MAC-IP pair.
2. enqueue route add/del requests to evpn and inet tables.
3. maintains dependency list for Oper DB entries interface, VRF, VN.
4. listens to the evpn route updates to detect pod movement.
5. listen to the healthcheck notifications for deleting routes in case
   of BFD session failures.
6. listen to the ARP notifications for deleting routes if ARP is not resolved
    after max retries.
Changes to inet-evpn peer path to include MAC-IP pair to handle IP movement.
same inet route can be derived from two evpn routes ( MAC1/IP1 and MAC2/IP1).
evpnderivedpath should include mac address also so that inet route for IP1
will have paths derived from MAC1/IP1 and MAC2/IP1 EVPN routes.
Remote MAC address is provided to BFD session so that session does not derive
dest MAC from VM interface.
Add new callback handler for healthcheck service to notify healthcheck failure
to MAC-IP learning module.
maintain a list of monitoring IP addresses per VN so that when new MAC-IP
address is learnt, if new IP is present in the list , then health check is
initiated for that MAC-IP pair.
###4.1.2 Vrouter
1. if MAC-IP learning is enabled on VM interface, the following cases are
handled.
1.1 layer 3 forwarding is enabled on VM interface
do source IP look up in inet/inet6 table for ARP/ND packets.
if entry is found and source MAC address of packet and stiched MAC address
in the route differs, then trap the packet to agent for learning.
if entry is not found, then trap the packet to agent for learning.
1.2 layer 3 forwarding is not enabled on VM interface.
do source MAC lookup in bridge table.
if entry is not found, trap the packet to agent for learning.
#5 Performance and scaling impact
max number of pods supported for compute is 50.
#6 Upgrade
schema changes for this feature are extentions to the objects. so it does
not affect upgrade.
#7 Deprecations
#8 Dependencies
#9 Testing
TBD
#10 Documentation Impact
#11 Caveats
Ipv6 is not supported in R2011 release.
#12 References

   

