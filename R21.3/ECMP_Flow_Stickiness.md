# 1. Introduction

Traditional ECMP loadbalances traffic over a number of available paths towards a destination. When one path fails (scale-down) or a new path gets added (scale-up), the traffic gets re-shuffled over the available number of paths.

Flow Stickiness: Normal ECMP does not provide flow stickiness.  This is because ECMP is implemented as a modulo-N division where N is the number of members in the load balancing group. If the number of members in the load balancing group changes from N to N+1 as a result of a scale-out event, all hashes will now be computed modulo N+1 instead of modulo N, and as a result most flows will be moved to a different path.

This rehashing as mentioned above can be troublesome in data center environments where many servers advertise a “service prefix” to a loadbalancer/gateway in a sort of “anycast” way.

# 2. Problem Statement

Implement the ecmp flow stickiness in such way that we dont rehash the flows on existing paths when some other path fails or a new path is added.
Detailed problem statement at - https://contrail-jws.atlassian.net/browse/CEM-20298

The good thing of flow stickiness is that established sessions to a server wont get rehashed.

The downside of this is that one server would be serving more requests than other servers. (Traditional ECMP would try to achieve equal spread, at the cost of that rehashing).

# 3. Proposed Solution

An ECMP nexthop aka Composite NH maintains list of all its member nexthops. For a given flow, Agent selects one of the members based on hash computed for the flow. The hash computation uses key fields according to algorithm explained above. The hash is used to index the array of members in Composite NH.
Whenever there is add/delete in member nexthops list, it triggers a recreation of Composite NH and eventually a new Composite NH Id.

Flow key is made of < nh-id, src-ip, dst-ip, protocol, src-port, dst-port>. So, whenever there is change in nh-id, it triggers deletion of existing flow and create a new flow. This eventually does the rehashing.

So, to resolve above issue we need to persist the old nh-id and update the member nexthop list. If old nh-id is maintained, then it will not recreate the flow and hence no rehashing.


# 4. Alternatives Considered
Key of composite NH is list of component NH key. So, whenever this list changes due to scale-down or scale-up, it triggers recreation of CompositeNH. One solution would be keep MPLS label + Vrf as CompositeNH key. This way CompositeNH need not be recreated as key is always same. This means that all caller functions where CompositeNH is created also need to be changed. Since, changes will touch multiple CompositeNH sub-types and it needs more testing, this approach was not taken.


# 5. Implementation Details

The implementation can be divided into 2 parts -

## Scale UP

Whenever a new path is added for a route and there are more than one path of type LOCAL_VM_PORT_PEER, it triggers EcmpAddPath. When member NH count increases from n to n+1 where  n > 1, AppendEcmpPath function will be called. Following needs to be done to handle scale-up case -

1. Create a new component nh key list by appending the new member NH to the existing list.
2. Do a RESYNC operation on current Composite NH of ECMP path.
3. As part of RESYNC callback, Update the nexthop->component_nh_key_list with the new one
4. Also, ensure that NH ordering is maintained.

 In this case, since existing NH is resynced, It will not create a new NextHop. So, NH id will maintained and hence flow will be maintained.

## Scale DOWN

Scale down handling will be almost same as Scale Up. However, it needs to ensure component_nh_key_list size should remain same after deleting one path. For example take a composite NH with members A, B, C
in that exact order, If B gets deleted, the new composite NH created should be A <NULL> C in that order,	irrespective of the order user passed it in. This will ensure that flow ecmp_index will continue to point to correct VM.

In general, when ECMP path is changed, Route will be exported to Controller. Controller would then send back this update to Compute and it will update the Bgp Peer Path. Same Composite NH handling needs to be done for this Bgp Peer Path also.

# 6. Performance and Scaling Impact

# 7. Upgrades

No impact

# 8. Deprecations

There are no deprecations when this change is made.

# 9. Dependencies

There are no dependencies for this feature.

# 10. Tests

## 10.1 Unit tests

Following Unit-tests will be added in ecmp_local and ecmp_remote scenarios.

1. Member NH list change from n to n+1 where n > 1. Composite NH and its id should remain same.
2. Member NH list change from n+1 to n where n > 1. Composite NH and its id should remain same.
3. Flow Stickiness
   1. Member NH list change from n to n+1 where n > 1. Flow should not get deleted and flow ecmp_index should remain same.
   2. Member NH list change from n +1 to n where n > 1. Flow should not get deleted and flow ecmp_index should remain same.
4. Non-Composite NH to Composite NH transition. Flow should get reevaluated and should hash to one member NH.
5. Composite NH to Non-Composite NH transition. Flow should get reevaluated and should map to Interface NH.

## 10.2 Feature tests

This feature will be tested on following deployments -
k8s and openshift
Feature test link - https://junipernetworks.sharepoint.com/:x:/s/contrail-india-dev/EVwTkI1U8G5Kszf1b9l1vjcBK5Caj2GOLrQ3QOqllznj8A?e=wYRzuL

# 11. Unsupported scenario
 
 **Setup**: Deployment of 2 pods in 2 computes, one in each.
 
 **Trigger**: Scale-up
 
 **Result**: Flow stickiness may break.
 
 **Reason**: Before scale-up flow will be non-ecmp with respect to the compute forwarding the traffic. After scale-up it will become an ecmp flow which will trigger rehashing.
 
 Flow stickiness only works if flow is ecmp before and after scale up/down.

# 12. Documentation Impact

NA

# 13. References

NA
