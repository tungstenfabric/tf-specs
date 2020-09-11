# 1. Introduction
When users create Virtual Machines(VMs), they can choose to associate VMs with a SR-IOV port.
After creation of VMs, it is expected that VMs can communicate with each other seamlessly through SR-IOV ports.

# 2. Problem statement
Currently, when a VM is created, there is no Virtual Port Group(VPG) that binds its SR-IOV port with the port on physical Switch side.
This blocks the communation between VMs.

Following is the use case addressed in this story
**use case :** Users create VMs and newly created VMs can communicate with each other seamlessly through SR-IOV ports.

# 3. Proposed solution
-   Contrail Config collects VM's SR-IOV port information and its corresponding physical swtich port information.
-   Config creates a new VPG with collected information.

# 4. API schema changes
N/A

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact

## UI changes
Currently, in Command UI Infracstructure/Server section, the port settings is only configurable by importing CSV files upon creation of servers.
Users need to be able to configure ports when perfoming creation or modification.

## User workflow impack
If the UI is changed, user will have somme extra optional fields they can fill in to configure ports of servers.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation

## **use case :** Users create VMs and newly created VMs can communicate with each other seamlessly through SR-IOV ports.
-   Contrail Config received VMI data of created VM
-   Config retrives Node Profile data of VM's locating Node
-   Config retrives Card data of the Node
-   Config collects SR-IOV port and corresponding Switch port information
-   Config creates a new VPG object with found SR-IOV port and corresponding Switch port

## Workflow with proposed design

### Contrail Config received VMI data of created VM
After creation of a VM, Contrail Config receives the VMI information of it from the POST request through /virtual-machine-interface API.

### Config retrives Node Profile data of VM's locating Node
VMI data has reference to the name of the Node. With the Node name, Contrail Config finds out corresponding Node Profile.

### Config retrives Card data of the Node
Node Profile has reference to hardware. Config retrives Card data that back references the same hardware as Node Profile.

### Config collects SR-IOV port and corresponding Switch port information
Card data has key "interface_map" which holds information of all ports.
The port with label "sriov" is the desired SR-IOV port, it also contains information of the corresponding Switch port.

### Config creates a new VPG object with found SR-IOV port and corresponding Switch port


# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing
N/A

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 17. Known issues and Caveat
There are no known issues for this functionality.

# 18. References
[https://contrail-jws.atlassian.net/jira/software/c/projects/CEM/issues/CEM-15195](https://contrail-jws.atlassian.net/jira/software/c/projects/CEM/issues/CEM-15195)