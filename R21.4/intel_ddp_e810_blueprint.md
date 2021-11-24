# 1. Introduction
This blue print describes Porting E810 support to Contrail Classic vRouter.
In this document, we will discuss about the E810 NIC support for dpdk vRouter in Contrail Classic
Adapters based on the Intel(R) Ethernet Controller 800 Series require a Dynamic Device Personalization (DDP) package file to enable advanced features

# 2. Problem statement
kernel mode vRouter is known to work with the Intel E800 series NIC, there was work to make this supported with
DPDK vRouter. Customers are now upgrading their hardware, and the older NICs are not even being considered.

Requirement is to support E810 NIC for dpdk vRouter Contrail Classic.

# 3. Proposed solution
  The 800 Series Linux base driver and DPDK driver can both load a specific DDP package to a selected adapter based on the device's serial number. The driver does this by looking for a specific symbolic link package filename containing the selected device's serial number.

  ##3.1 Installation and Loading DDP package :

    1- Download ddp package and extract zip file to obtain package (.pkg) file
    2- Rename the package file with the device serial number in the name.
        Copy the ddp ice package over to /lib/firmware/updates/intel/ice/ddp
        Create a symbolic link with the serial number linking to the package
    3- When the DPDK driver loads, it looks for ice.pkg in /lib/firmware/intel/ice/ddp.
    4- If the driver successfully finds and loads the DDP package, dmesg indicates that the DDP package is successfully loaded.
   ## 3.2 Alternatives considered

   None

   ## 3.3 API schema changes

   Not Applicable

   ## 3.3 User workflow impact

   None

   ## 3.4 UI changes

   None

   ## 3.5 Notification impact

   None

# 4. Implementation
     ##4.1
        Build changes in  Dockerfile of agent-dpdk process to add package in agent-dpdk container
        (Path: tf-container-builder/containers/vrouter/agent-dpdk)
        create a symbolic link with the serial number linking to the package

     ##4.2- Implement Checksum for E810

### Working Mechanism:


# 5. Performance and scaling impact

None (TBD)

    ## 5.1 API and control plane
None

    ## 5.2 Forwarding performance
None

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
Build a binary RPM package of ice driver for supported kernel version

# 9. Testing
## 9.1 Unit tests
    DPDK vRouter Unit testing with updated DDP package for E810
        Verify device serial number and E810 NIC interface is UP
        Verify DDP package loads during device initialization
        Verify driver mode If the DDP package file is missing or incompatible with the driver
        Verify dpdk vRouter stable with ddp package update
## 9.2 Dev tests
## 9.3 Feature tests
Below is the feature/solution testplan.
    Deployment Wih Bond
    Bond+vlan
    9k mtu (lacp fast) ( sanity run for each )
    Bond with LACP and LACP Fast and check covergence
    BOND with Static lag (Active back )
    compute to compute throughput test
    GSO/GRO VM -VM on compute and across
# 10. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 11. Caveats
None

# 12. References
None
