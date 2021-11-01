# 1. [Introduction]

In scenarios where a BGP/XMPP peer flaps or goes down for some time, the control-node will delete all the routes that have been learned from that peer (BGP/XMPP).This results is disruption of traffic among the worker nodes.

Graceful Restart(GR)/Long-Lived Graceful Restart(LLGR) feature helps in ensuring that the routes learnt are not immediately deleted when the peer(BGP/XMPP) goes down and any temporary flap or restart of the peer does not affect the flow of traffic.

Upon the enablement of GR/LLGR feature,the learnt routes are not deleted when the sessions go DOWN and the routes are also not withdrawn from the advertised peers. During this time duration,the routes are marked as 'STALE'/'LLGR_STALE'.If the sessions come back and if routes are re-learnt, then impact to the network is minimized. If the sessions do not come back up within the configured GR/LLGR timer, control-node deletes the learnt routes from the peer.

GR and LLGR are configurable parameters (values in seconds).


-- Core Team: Sriram Natarajan (snatarajan@juniper.net)

-- JIRA Story: https://contrail-jws.atlassian.net/browse/CEM-23671


# 2. Problem Statement

Although the GR/LLGR features are supported to prolong the routes to be retained as per the configured values,there is no support rendered for the EVPN Type 2 routes. In case of peer(BGP/XMPP) restart/session going DOWN,the EVPN Type 2 routes that have been learned are not marked as STALE when GR/LLGR is configured. Instead these routes are deleted by control-node from the route DB and traffic loss is witnessed for EVPN family of routes. Today,control-node explicitly deletes EVPN routes, even when GR/LLGR is configured.

# 3. Proposed Solution

This feature targets to address the limitation of GR/LLGR support for EVPN Type 2 routes.Post implementation, the behaviour for the EVPN routesis to have the entries marked as STALE and routes not being deleted in accordance to the GR/LLGR timer values configured. This will mark a uniform behaviour of EVPN Type 2 routes with all other address families, in case of STALE scenarios.

  3.1	Limitations
        This feature/user-story explicitly addresses the GR/LLGR support for EVPN based routes ONLY.Support for the ERMVPN/MVPN based routes still remain uncovered.

  3.2	Affected Modules
        Contrail Control-Node

  3.3	Alternatives Considered
        N/A

  3.4	API Schema Changes
        N/A

  3.5	User Work-flow Impact
        N/A

  3.6	UI Changes
        N/A

  3.7	Notification Impact
        N/A


# 4. Implementation

The implementation will focus in extending the GR/LLGR support for the EVPN type 2 routes.

  4.1	Assignee(s)
        -- Sriram Natarajan (snatarajan@juniper.net)

# 5. Performance and Scaling Impact

  5.1	API & Control Plane
        N/A

  5.2	Forwarding Performance
        TBD

# 6.	Upgrade
        TBD

# 7.	Deprecations

# 8.	Dependencies
        TBD

# 9.	Testing

  9.1	Unit Tests

  9.2	Dev Tests

     -- Verification of EVPN based routes not being deleted in the Oper DB
     --	Verification of FIB table in forwarding for ROUTES not deleted (during the CONTROL-COMPUTE association & dissociation scenarios)
     --	Verification of VROUTER CLI to assess the traffic stats & also the associated FLOW related counters

  9.3	Upgrade Tests
        N/A

  9.4	UI Tests
        N/A

# 10. Documentation Impact
      The feature’s overview,usage and its limitation should be documented as part of release documentation.

# 11. References
      To be updated
