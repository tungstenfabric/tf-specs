# 1. Introduction

-   This document describes the proposed changes in TF TripleO Heat Templates to 
    introduce deployments’ support for Intel PAC N3000 boards with full hardware 
    flow offload mode. 

-   Blueprint/Feature Lead:  Salvatore Morsa ( <salvatore.morsa@hcl.com> )
-   Core team:
    -   Pattela Jayaragini ( <pattela.j@hcl.com> )
    -   Roberto Pennati ( <roberto.pennati@hcl.com> ) 
-    Jira Story : <https://jira.tungsten.io/browse/TFF-32>


# 2. Problem statement

TF vRouter-DPDK integration with Intel PAC N3000 board ([https://jira.tungsten.io/browse/TFF-22](https://jira.tungsten.io/browse/TFF-22)) 
allows to offload flows processing into hardware (network card). Enabling that 
integration means configuring the host system to support Intel PAC N3000 and 
launching TF vRouter-DPDK with required command line arguments.

To automate the installation process supporting such options requires changes in 
deployment templates/scripts. Initial support will be added to TF TripleO Heat 
Templates. 


# 3. Proposed solution

The proposed solution defines a set of new parameters as a deployment option for 
TF TripleO Heat Templates. Those parameters allow enabling support for configuring 
and running platform in full hardware flow offload mode for Intel PAC N3000 card. 

The goal is to automate the deployment process as much as possible so that 
templates/scripts generate low level inputs for TF vRouter-DPDK agent arguments 
(provided as a vRouter command line argument)

Modified templates allow configuring Intel PAC N3000 card and host operating system, 
including: booting card BMC image, binding card’s PF (physical function) to required 
drivers, creating VFs (virtual functions) and binding them to driver (vfio-pci). 


## 3.1 Alternatives considered

TF vRouter command line arguments specific to the N3000 based offload mode can be 
provided by the user explicitly while setting configuration for deployments by 
adding them to the `ContrailDpdkOptions` parameter for `ContrailDpdk` role. 
Disadvantage of this approach is that it requires knowledge of PCI addresses assigned 
to Intel PAC N3000 board and depending on chosen option may require generation of 
complex vRouter command line arguments which would be error prone. 

TF TripleO Templates support VFs creation (intended for ContrailSriov Role), however, 
they do not provide an option to set a different driver for VF than default. In case 
of TF vRouter with integrated support for Intel PAC N3000 full hardware flow offload 
`vfio-pci` driver for VF should be set (instead of default `virtio-pci` binding). 
Additionally specific drivers are required for physical function (other than default 
ones) which also currently is not supported.


## 3.2 API schema changes

No expected changes to APIs.


## 3.3 User workflow impact

The user (admin deploying platform) can explicitly enable deployment support for Intel 
PAC N3000 boards in full hardware flow offload mode (by default disabled). 

This behaviour is controlled by new parameters for ContrailDpdk Role:

______________________________________________________________________________________
|                                        |                                           |
|  Tripleo Heat Parameters name          |  Parameter Attributes                     |
|  (ContrailDpdk Role)                   |                                           |
|________________________________________|___________________________________________|
|                                        |                                           |
| ContrailSmartNicN3000HwOffloadEnabled  | type: boolean                             |
|                                        | description: main parameter determining   |
|                                        |              if hardware offload for Intel|
|                                        |              PAC N3000 will be enabled.   |
|                                        |              It requires                  |
|                                        |                                           |
|                                        |      ContrailSettings.NIC_OFFLOAD_ENABLE  |
|                                        |                                           |
|                                        |              to set to true (otherwise    |
|                                        |              only slowpath is supported)  |
|                                        | default: false                            |
|----------------------------------------|-------------------------------------------|
|                                        |                                           |
| ContrailSmartNicN3000HwOffloadSettings | type: json (map)                          |
|                                        | description: json formatted map containing|
|                                        |              additional configuration     |
|                                        |              variables. Valid (used) only |
|                                        |              when                         |
|                                        |                                           |
|                                        |   ContrailSmartNicN3000HwOffloadEnabled   |
|                                        |              is set to true               |
|                                        |                                           |
|                                        |  default: {} (empty map, note: “keys”     |
|                                        |           can have their own default      |
|                                        |           values which in such case are   |
|                                        |           used)                           |
--------------------------------------------------------------------------------------

`ContrailSmartNicN3000HwOffloadSettings` contains keys allowing user to specify values 
for optional parameters:
______________________________________________________________________________________
|                                        |                                           |
| ContrailSmartNicN3000HwOffloadSettings |                Description                |
| key name                               |                                           |
|________________________________________|___________________________________________|
|                                        |                                           |
| N3000SriovNumVFs                       | type: number                              |
|                                        | description: number of VFs to be created  |
|                                        |              on Intel PAC N3000 card.     |
|                                        |              Accepted values from 1 to 28 |
|                                        | default: 16                               |
|----------------------------------------|-------------------------------------------|
|                                        |                                           |
| N3000InsertType                        | type: string                              |
|                                        | description: defines how flows are being  |
|                                        |              inserted. Valid values:      |
|                                        |                                           |
|                                        | csr - using synchronous hardware register |
|                                        |       writing through management interface|
|                                        | flr - using implementation specific       |
|                                        |       packets, containing flow details,   |
|                                        |       sent through VF0 slow-path interface|
|                                        |                                           |
|                                        | default value: None (not set, uses vRouter|
|                                        |                default)                   |
|----------------------------------------|-------------------------------------------|
|                                        |                                           |
| N3000PhysQueuesNum                     | type: number                              |
|                                        | description: defines number of queues for |
|                                        |              Intel PAC N3000 card physical|
|                                        |              ports                        |
|                                        | default value: None (not set, uses vRouter|
|                                        |                default)                   |
|----------------------------------------|-------------------------------------------|
|                                        |                                           |
| N3000VFsQueuesNum                      | type: number                              |
|                                        | description: defines number of queues for |
|                                        |              N3000 VFs                    |
|                                        | default value: None (not set, uses vRouter|
|                                        |                default)                   |
|----------------------------------------|-------------------------------------------|
|                                        |                                           |
| N3000StaticLagEnabled                  | type: boolean                             |
|                                        | description: when set to true enables     |
|                                        |              static LAG on physical ports |
|                                        |              of Intel PAC N3000 card      |
|                                        | default value: None (not set, uses vRouter|
|                                        |                default)                   |
--------------------------------------------------------------------------------------

## 3.4 UI changes 

None (no expected changes to UI)


## 3.5 Notification impact

None (no expected changes to notification)


# 4. Implementation

Additional parameters are added into the ContrailDpdk role (described in User workflow 
impact). Parameter values are processed and consumed by templates/scripts for 
ContrailDpdk role run in the pre-network phase ContrailDpdk role is extended with 
optional code allowing to deploy nodes with Intel PAC N3000 support enabled.


## 4.1 Intel PAC N3000 card and host configuration

Added template responsible for Intel PAC N3000 card configuration - booting card image 
and host operating system configuration - binding card’s PF (physical function) to required 
driver, creating VFs (virtual functions) and binding them to driver (vfio-pci).  

To keep desired settings persistent above configuration steps have to be done on each power 
on of node. This is managed by a new systemd service launched when the operating system 
starts. 


## 4.2 TF vRouter command line arguments

Template responsible for handling contrail init container is enhanced with code generating 
command line arguments specific to TF vRouter integrated with Intel PAC N3000 card in full 
hardware flow offload mode. This code is only invoked when Intel PAC N3000 support is 
enabled and based on user provided inputs. All required arguments are generated and passed 
to be used by vRouter.


# 5. Performance and scaling impact

None (no expected changes to performance nor to scaling as all new functionality is optional 
and concerns deployment stage)


# 6. Upgrade 

N/A


# 7. Deprecations 

None


# 8. Dependencies

Customized operating system image for nodes with enabled support for Intel PAC N3000 with 
full hardware flow offload - installed: Intel OPAE drivers and tools + igb_uio driver


# 9. Security Considerations 

None


# 10. Testing 

Testing of deployment RHOSP 16.1 + TF with added modifications for both cases: disabled 
(default) and enabled (explicitly set by user) support for Intel PAC N3000 card in full 
offload mode. 


# 11. Documentation Impact

Documentation describing new parameters and steps required to enable platform (RHOSP16 + 
TF) deployment with support for PAC N3000 board which utilizes full hardware flow offload 
mode.

# 12. References

N/A

