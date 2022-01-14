# 1. Introduction
Contrail support for dataplane learning of IPVLAN MAC-IP bindings reachable over
VM interfaces.
# 2. Problem statement
A VM deployed in open stack hosts multiple pods with their own IP addresses but
same MAC(IPVLAN subinterface).
IP addresses for pods are assigned by external applications in the same subnet
range of VM interfaces over which pods are reachable. So ,contrail  does not
have knowledge over the IP addresses of pods hosted by VMs. In order to support
pod to pod communication over open stack managed IaaS network, vrouter  must
learn MAC-IP bindings of pods and advertise them to controller.
The MAC-IP bindings should be advertised to SDN gateway using ipvpn and evpn
address families.
BFD should be supported for pod liveness.
# 3. Proposed solution
Contrail supports dataplane learning of MACVLAN MAC-IP bindings reachable over
VM interfaces from R2011 release. Unlike IPVLAN subinterfaces, MACVLAN subinterface
has their own MAC-IP addresses(unique MAC), So contrail restricts learning of multiple
MAC-IP bindings with same MAC.

By removing this restriction, contrail will seamlessly support learning of
IPVLAN MAC-IP bindings reachable over VM interfaces.
## 3.1 Alternative considered
None
## 3.2 API schema changes
None
## 3.3 User workflow impact
User need to enable MAC-IP learning on virtual network to use this feature.
User should use new health check template to run BFD for pods.
## 3.4 UI changes
None
## 3.5 Operations and Notification impact
None
# 4. Implementation
## 4.1 Work items
### 4.1.1 Agent
Agent will start learning MAC-IP(same MAC, different IP) bindings, instead of ignoring.
# 5 Performance and scaling impact
max number of pods supported for compute is 50.
# 6 Upgrade
None
# 7 Deprecations
# 8 Dependencies
# 9 Testing
All tests of MACVLAN MAC-IP learning feature are applicable to IPVLAN MAC-IP
learning feature as well.
MACVLAN MAC-IP learning also needs to  be tested to ensure no regression is
introduced by IPVLAN MAC-IP learning support.

# 10 Documentation Impact
# 11 Caveats
# 12 References
