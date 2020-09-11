# 1. Introduction
In typical Telco Cloud Stack, infrastructure pplication owners provision their applications using high level interfaces. When Management and Network Orchestration(MANO) interface is not available, application owners will use Network Function Virtualization Infrastructure(NFVI) interface to provision and manage workflows.


# 2. Problem statement
Through standard MANO and NFVI interfaces, it should be possible to provision Virtual Machines(VMs) in OpenStack that experience seamless networking across overlay SDN connected VMs and SRIOV connected VMs.

Following is the use case addressed in this story
**use case 1:** Fabric networking for SRIOV VF’s
**use case 2(Uncovered):** Fabric networking for SRIOV VF’s with vRouter Internetworking


# 3. Proposed solution
When creating a VM, the following process is needed:
-   A Virtual Network(VN) within OpenStack is created with a Provider Network, Physnet, and VLAN
-   In OpenStack an SRIOV port is created
-   The VM is launched with the SRIOV port in the VN
Contrail Config retrives ID of the VLAN and Physnet's Port, and create a VPG that holds information of both.
Device Manage creates VTEP on the Fabric. 

# 4. API schema changes
N/A

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact

## UI changes
Currently, in UI -> Infracstructure -> Server section, the Physnet Port setting is only configurable by importing CSV files during creation of servers.
Users should be able to configure Physnet Port at any time.

## User workflow impack
Users will be able to specify Physnet Port by filling in details in the UI during Server creation of modification.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation

## **use case :** Fabric Networking for SRIOV VF’s - Two VMs have been provisioned in the same virtual network, Each VM has been created with an SRIOV VF
-   Contrail Config received VMI data of created VM
-   Config retrives provider properties of the VM's VN
-   Config creates a placeholder VMI object which contains VLAN ID
-   Config retrives Physnet's Port information
-   Config creates a VPG object which refers to placeholder VMI and Physnet's Port
-   Device Manager consume the VPG and creates VTEP on the Fabric

## Workflow with proposed design

### Contrail Config received VMI data of created VM
After creation of a VM, Contrail Config receives the VMI information of it from the POST request through /virtual-machine-interface API.

### Config retrives provider properties of the VN to which the new VM connects
The received VMI data holds reference to the VN that it connects to. Provider properties is a section in the VN, containing the infromation of Segmentation ID(VLAN ID) as well as Physical Network(PN) name for the VN. 

### Config creates a placeholder VMI which contains Segmentation ID
It is also necessary to create a placeholder VMI for holding the VLAN ID information of the VN.

### Config retrives Physnet's Port information
By using the name of the PN, we can find the Physical Interface(PI) it connects to. Within the PI, we can find which Physical/Switch Port the VM's SR-IOV VF connects to.

### Config creates a VPG object which refers to placeholder VMI and Physnet's Port
With the Switch Port information and placeholder VMI containing VLAN ID, we can create a VPG that refers to both.

### Device Manager consume the VPG and creates VTEP on the Fabric


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