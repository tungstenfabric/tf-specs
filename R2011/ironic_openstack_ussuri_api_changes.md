# 1. Introduction

A BMS server is represented as an Ironic node. In Ussuri release of OpenStack, Ironic node create/update workflow is changed and api-server has adapted to the change in Ironic Workflow.

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#2-problem-statement)2. Problem statement
**Use Case 1: BMS Node Create**
In Earlier releases of OpenStack (For ex. Queens), Ironic port CREATE includes binding profile information needed to create internal virtual-port-group (VPG). VMI creation followed by VPG creation. In Ussuri release of OpenStack, port CREATE doesn't include binding profile info and it won't trigger VPG creation. It will fail BMS node creation state to "clean failed". Expected BMS Ironic state is "Available".
**Use Case 2: BMS Node Delete**
In Earlier releases of OpenStack (For ex. Queens), Ironic port DELETE will trigger VMI and VPG deletion. In Ussuri release of OpenStack, port UPDATE with empty binding profile should trigger physical interfaces removal from VPG. Lastly, VMI DELETE will trigger VPG deletion.
# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#3-proposed-solution)3. Proposed solution
- OpenStack Ussuri has changed Ironic workflow in following ways: 
	- Port CREATE does not include binding profile
		- VMI created with zero mac-address.
		- No VPG creation at this stage.
	- Port UPDATE with binding profile
		- VMI mac address will be update from link local info.
		- Internal VPG creation will be triggered.
	- Port UPDATE with empty binding profile
		- It should trigger PI removal from VPG.
	- VMI DELETE
		- Internal VPG should be deleted.

## [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#backgrounds)Backgrounds

N/A

## [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#solution)Solution


# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#4-api-schema-changes)4. API schema changes

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#5-alternatives-considered)5. Alternatives considered

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#6-ui-changes--user-workflow-impact)6. UI changes / User workflow impact

## [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/sriov_provisioning.md#ui-changes)UI changes

N/A
# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#7-notification-impact)7. Notification impact

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#8-provisioning-changes)8. Provisioning changes

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#9-implementation)9. Implementation
**Use cases:**
 1. UI user can attach PI to VPG before VMI creation. 
	 - If payload doesn't have link_local_info then retrieve it from DB and call _manage_vpg_association
 2. UI user sends PI during VMI creation
     - call _manage_vpg_association
 3. Ironic user doesn't have PI info at the time of VMI create
     - PI info won't be present in the DB or payload
     - Do not call _manage_vpg_association
  4. Non-Ironic or Non-UI/Api-client user can send VMI creation
       - 4a) Without attaching PI info in the payload
	       - If payload doesn't have link_local_info then retrieve it from DB 
	       - call _manage_vpg_association
       - 4b) Without attaching PI to VPG before VMI creation
	       - PI info won't be present in the DB or payload
	       - Do not call _manage_vpg_association

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#11-performance-and-scaling-impact)11. Performance and scaling impact

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#12-upgrade)12. Upgrade

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#13-deprecations)13. Deprecations

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#14-dependencies)14. Dependencies

N/A

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#15-testing)15. Testing
Verified basic port creation and deletion workflow for ironic and non-ironic users.

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#16-documentation-impact)16. Documentation Impact

It will be documented as part of release documentation by the doc team.

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#17-known-issues-and-caveat)17. Known issues and Caveat

OpenStack Train release is not qualified for above api-server changes because of Test Infra Failures.

# [](https://github.com/tungstenfabric/tf-specs/blob/master/R2011/ironic_openstack_ussuri_api_changes.md#18-references)18. References

[https://contrail-jws.atlassian.net/browse/CEM-21810](https://contrail-jws.atlassian.net/browse/CEM-21810)
