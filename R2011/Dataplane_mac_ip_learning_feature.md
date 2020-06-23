# 1. Introduction
Contrail support for dataplane learning of MAC-IP bindings reachable over
VM interfaces.
# 2. Problem statement
A VM deployed in open stack hosts multiple pods with their own MAC-IP addresses.
IP addresses for pods are assigned by external applications in the same subnet
range of VM interfaces over which pods are reachable. So ,contrail  does not
have knowledge over the IP addresses of pods hosted by VMs. In order to support
pod to pod communication over open stack managed IAAS network, vrouter  must
learn MAC-IP bindings of pods and advertise them to controller.
The MAC-IP bindings should be advertised to SDN gateway using IPVPN and evpn
address families.
BFD should be supported for pod liveness.
# 3. Proposed solution
Vrouter traps packets to agent for learning unknown MAC-IP addresses
seen on VM interface by spoofing ARP packets. agent processes the MAC-IP
learning events and generates evpn type 2 MAC/IP routes, inet routes for
L2/L3 mode and evpn type 2 MAC  only routes for L2 only mode on virtual network.
It initiates BFD session for new MAC-IP addresses if health check is enabled on
virtual network and monitor target list of health check contains new IP address.
agent also sends ARP/ND packets periodically to pods and this is used as a
fallback mechanism to check pod’s liveness in absence of BFD session for a pod.
The learnt routes are deleted in the following cases.

###BFD session failure:
Whenever BFD session goes down for a pod, Agent deletes the corresponding routes
from inet and evpn table.
###ARP/ND failure:
agent deletes routes if arp is not resolved after max retries.
###Pod movement:
Actually pod does not move from one VM to other but it can be destroyed in one
VM and created in some other VM with the same IP address. There are two  cases.
####Movement in same compute
Since pod is destroyed and launched in same compute, so compute retract the
older evpn route IP1/MAC1 and advertise newer route with IP1/MAC2
and update and advertise inet route.
####Movement across computes.
pod1 with IP1/MAC1 in compute 1 is destroyed with and pod2 is created with
IP1/MAC2 in compute 2. compute1 advertises evpn MAC/IP route IP/MAC1 to
controller and compute 2 receives this route and derives inet route and
bridge route. When pod2 is created, compute2 advertises evpn route IP1/MAC2
to controller. Compute 1 receives this route and checks whether locally
learnt MAC-IP entry present with same IP. If it is present, it detects that
pod is moved and retracts evpn route IP1/MAC1.
###VMI health check failure/oper state down/VMI deletion:
When VMI’s health check fails or open state is down or VMI is deleted,
pod routes learnt over this VMI are deleted.
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
Changes to inet-evpn peer to include MAC-IP pair to handle pod movement.
same inet route can be derived from two evpn routes ( IP1/MAC1 and IP1/MAC2)
and it is added by two paths. one path peer is inet-evpn with ip1-mac pair
and second path peer is inet-evpn peer with ip1-mac2 pair. when evpn route
IP1/MAC1 is retracted, path added by peer inet-evpn with ip1-mac1 pair
will be deleted.

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
#11 References

   

