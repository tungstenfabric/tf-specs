
**1. Introduction**

We have support in Contrail to apply routing policies (import) to the routes being imported in the routing table. These routes can be filtered based on Communities, protocol, prefix etc from which they were learnt.

**2. Problem statement**

Routing policy can be applied to routes based on the protocol which indicates how the routes were learnt. One option for protocol is BGP which represents all routes learnt via BGP. This includes both MP-BGP routes as well as BGPaaS routes from VMs. In addition to this, there is also an option to exclude MP-BGP routes and match only on BGPaaS by mentioning sub-protocol as BGPaaS. However, there is no option to filter only on MP-BGP routes and exclude BGPaaS routes. There is a requirement to define an import routing policy that can match only on MP-BGP routes and exclude BGPaaS learnt routes.

# **3. Proposed solution**
# The behavior for BGP protocol filter in the routing-policy needs be altered to be able to filter the routes that are learnt via MP-BGP. Since the current behavior does not allow for the routes to be filtered based on MP-BGP, we can either add a sub-protocol type called MP-BGP under the protocol BGP (similar to BGPaaS) or we can change the implementation of the routing-policy match in control-node to only include MP-BGP routes when BGP protocol filter is applied and not include BGPaaS routes. For BGPaaS route filter, we need to specifically mention the sub-protocol as BGPaaS.

# **4. Implementation**
# Today the BGP protocol filter in the routing-policy is a subset of the routes that are learnt via BGPaaS and MP-BGP. This behavior will be changed to only filter the MP-BGP routes when BGP protocol filter is applied. For BGPaaS routes, we need to apply the subprotocol filter ‘BGPaaS’ in the routing policy.

# **4.1 Current protocol filter for BGP**
#
#
BGPaaS

MP-BGP
# ![](mp-bgp_bgpaas.png)![](mp-bgp_1.png)
#

#
# **4.2 Proposed protocol filter for BGP**
`        `MP-BGP
# ![](mp-bgp_2.png)
#
#
#

# In the proposed solution implementation BGPaaS routes will not be a part of the protocol filter BGP. To filter the BGPaaS routes, we need to add the protocol filter BGPaaS in the routing-policy

# **4.3 Create routing-policy for BGPaaS**
# ![](mp-bgp_prev.png)

# **4.4 Create routing-policy for MP-BGP**
# ![Graphical user interface, application

Description automatically generated](mp-bgp_new_1.png)
#

# **4.5 Create routing-policy for BGP**
# ![Graphical user interface, application

Description automatically generated](mp-bgp_new_2.png)
#

# **5. Performance and scaling impact**

# **6. Upgrade**
None.

# **7. Deprecations**
None.

# **8. Dependencies**
None.

# **9. Testing**
## 9.1 Unit tests
Below unit-test cases are planned to be added.



|**Routing Policy Filter**|**Route Source**|**Accept/Reject Route**|
| :- | :- | :- |
|BGP|BGPaaS|Reject|
|BGP|MP-BGP|Accept|
|BGP|XMPP|Reject|
|BGPaaS|BGPaaS|Accept|
|BGPaaS|MP-BGP|Reject|
|BGP, BGPaaS|BGPaaS|Accept|
|BGP, BGPaaS|MP-BGP|Accept|
|BGP, BGPaaS|XMPP|Reject|


## 9.2 Dev tests
## 9.3 System tests

# **10. Documentation Impact**

# **11. References**

Juniper Business Use Only
![Juniper Business Use Only](mp-bgp_new_3.png)
