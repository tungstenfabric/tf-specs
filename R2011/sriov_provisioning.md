# 1. Introduction
In typical Telco Cloud Stack, infrastructure pplication owners provision their applications using high level interfaces. When Management and Network Orchestration(MANO) interface is not available, application owners will use Network Function Virtualization Infrastructure(NFVI) interface to provision and manage workflows.


# 2. Problem statement
Through standard MANO and NFVI interfaces, it should be possible to provision Virtual Machines(VMs) in OpenStack that experience seamless networking across overlay SDN connected VMs and SRIOV connected VMs.

Following is the use case addressed in this story
**use case 1:** Fabric networking for SRIOV VF’s
**use case 2:** Fabric networking for SRIOV VF’s with vRouter Internetworking
-   **use case 2.1:** Virtual Network(VN) does not exist
-   **use case 2.2:** Only Non-provider VN exists


# 3. Proposed solution

## Backgrounds
A Provider VN is created within OpenStack, with a Provider Network, a Physnet, and a VLAN
A VM can be created with a SR-IOV Virtual Function(VF), and a Provider VN

## Solution
-   **use case 1:** Contrail Config retrives VLAN ID and Switch Port information, and create a VPG that holds both. Device Manage consumes the VPG and creates VTEP on the Fabric.
**use case 2:** 
-   **use case 2.1:** A Provider VN is created with required dependencies
-   **use case 2.2:** The Non-provider VN is converted to a Provider VN


# 4. API schema changes
N/A


# 5. Alternatives considered
N/A


# 6. UI changes / User workflow impact

## UI changes
Currently, in UI -> Infracstructure -> Server section, the Physnet Port setting is only configurable by importing CSV files during creation of servers.
Users should be able to configure Physnet Port at any time.


# 7. Notification impact
N/A


# 8. Provisioning changes
N/A


# 9. Implementation

## **use case 1:** Fabric Networking for SRIOV VF’s - Two VMs have been provisioned in the same virtual network, Each VM has been created with an SRIOV VF
-   ### Schema Transformer(ST) received a Creation Event of Virtual Machine Interface(VMI) through its RabbitMQ queue
Schema Transformer receives VMI of created VM.
-   ### ST retrives Physical Network ID and Segmentation ID from VN
The VMI object contains reference to VM's connected VN. Within the VN object, Pyhsical Network ID and Segmentation ID can be found under provider_properties -> physnet.
-   ### ST retrives Host ID of VM's Host Node
Host ID of the Node, which new VM is created on, can be found under VMI -> virtual_machine_interface_bindings -> host_id.
-   ### ST finds VM's SR-IOV Port ID
With Host ID, we can find out which Virtual Router connects to the Node. Within Virtual Router, VM's SR-IOV Port ID can be found under sriov_interface_map with the Physical Network ID just found.
-   ### ST retrives SR-IOV Port's corresponding Port object
VM's Host Node has a list of references to its Ports. VM's SR-IOV Port object can be found in the list with SR-IOV Port ID just found.
-   ### ST retrives Switch Port information
Config retrives Switch Port information under SR_IOV Port Object -> bms_port_info -> local_link_connection.
-   ### ST creates a placeholder VMI which contains Segmentation ID
A placeholder VMI is created to hold the Segmentation ID(VLAN ID) of VM's VN.
-   ### ST creates a VPG object which refers to placeholder VMI, and Swith Port information
-   ### Device Manager consume the VPG and creates VTEP on the Fabric

## **use case 2.1:** Fabric networking for SRIOV VF’s with vRouter Internetworking - VN does not exist
-   ### No VN exists, thus the following workflow
-   ### User creates a Provider VN with Provider Network, Physnet, and VLAN using openstack (horizon or cli)
-   ### User launches vRouter VM in the provider VN using openstack (horizon or cli)
-   ### User launches SRIOV VM in the provider VN using openstack (horizon or cli)
-   ### System follows the same steps as in use case 1

## **use case 2.1:** Fabric networking for SRIOV VF’s with vRouter Internetworking - Only Non-provider VN exists
-   ### Non-provider VN and vRouter VM's already exists in the VN, , thus the following workflow
-   ### User converts the Non-provider VN to Provider VN using contraill (commandUI or vncApi)
-   ### User launches SRIOV VM in the provider VN using openstack (horizon or cli)
-   ### System follows the same steps as in use case 1


# 11. Performance and scaling impact
N/A


# 12. Upgrade
N/A


# 13. Deprecations
N/A


# 14. Dependencies
N/A


# 15. Testing
[https://drive.google.com/file/d/1nMwXTIeGy2qyumMhibVzkgeUGxIcqSUT/view?usp=sharing](https://drive.google.com/file/d/1nMwXTIeGy2qyumMhibVzkgeUGxIcqSUT/view?usp=sharing)


# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.


# 17. Known issues and Caveat
There are no known issues for this functionality.


# 18. References
[https://contrail-jws.atlassian.net/jira/software/c/projects/CEM/issues/CEM-15195](https://contrail-jws.atlassian.net/jira/software/c/projects/CEM/issues/CEM-15195)