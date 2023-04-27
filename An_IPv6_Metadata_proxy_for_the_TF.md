# An IPv6 metadata service for the TF

## 1. Introduction

Support of IPv6 protocol is a mandatory trait of an every modern network software appliance.

While the Tungsten Fabric supports this protocol in general, some of its components still are IPv4 only.  The TF metadata service is one such example. This TF service accepts requests from VMs and relay them to OpenStack Nova metadata API [MetadataOpenStack] to deliver the metadata [MetadataFAQ] functionality for consumers of a virtualized network. OpenStack Nova metadata API operates both in IPv4 and IPv6 environments (including IPv6-only).

Although, TF Agent is able to communicate with the OpenStack Nova metadata API [MetadataOpenStack] via both IPv4 and IPv6 protocol, it unable to accept requests from VMs via IPv6 protocol. Thus, access to this service in the IPv6-only environment via native TF facilities is impossible. This limitation also detains advent of IPv6 / IPv4 parity in the TF.

To accept metadata requests from virtual machines OpenStack uses port 80/TCP and HTTP protocol on well-known IP addresses:  169.254.169.254 (IPv4 link-local)  and fe80::169.254.169.254 (IPv6 link-local). Using well-known IPv6 link local metadata endpoint fe80::169.254.169.254 (which is an alternative compressed text representation of address fe80::a9fe:a9fe [RFC4291]) was suggested and accepted in OpenStack Neutron specification and implemented in Ussuri release of OpenStack [NeuIPv6].

The goal of this document to present a concept (as well as it's implementation) of how to add the IPv6 functionality for the TF metadata service. Some ideas of the implementation are borrowed from [NeuIPv6].

Blueprint/Feature Lead: Matvey Kraposhin.

## 2. Problem statement

Modifications to both the controlplane and dataplane of the Tungsten Fabric are required to forward metadata requests and responses  from/to virtual machines using 6-th version of the Internet Protocol.

In a practical sense, the introduction of the IPv6 functionality to the TF metadata service means that it would be possible to use next request within a virtual machine in a TF network [MetadataOpenStack]:

    $ curl -6 http://[fe80::a9fe:a9fe%ethX]:80


Where [MetadataOpenStack,NeuIPv6]:
* fe80::a9fe:a9fe (e.g fe80::169.254.169.254) is a proposed in [NeuIPv6] well known IPv6 link local metadata address, but technically in TF user can choose any address from the link local range fe80::/10;
* ethX is a name of an outgoing Ethernet IPv6-supporting interface in the requesting virtual machine;
* 80 is a port to the metadata service.

## 3. Proposed solution

It is expected that:
* a virtual machine requests for metadata through an IPv6-enabled port with relevant security group options;
* the metadata service proxy server address ((e.g., fe80::a9fe:a9fe) is known to the TF Agent in advance;
* a TF Agent process runs with root privileges in the host OS.

Nowadays, the TF metadata service uses HTTP/TCP/IPv4 stack to communicate with VMs.
General scheme of interaction between a virtual machine and the OpenStack Nova metadata API is preserved in the new version of the proxy server: the TF Agent runs an HTTP proxy server (the metadata proxy server) that listens for incoming requests to a specified address and relays them to the OpenStack Nova metadata API.
However, routing of IPv6 packets between a virtual machine and the metadata proxy server must be implemented in the TF in a different way comparing to its IPv4 counterpart. Namely, IPv4 version of the TF metadata service uses NAT (both source and destination) and PAT to securely identify requesting virtual machine in the Nova database and to forward packets from overlay network to vhost0 interface.

NAT for IPv6 is not implemented in the TF vrouter. But the IPv6 standard introduces link local addresses of interfaces [RFC42911], which are unique by definition and always present for IPv6-enabled Ethernet interfaces. These link local addresses can be used for identification of virtual machines without employment of address translation technique. However, in order to perform forwarding of packets between virtual machines and vhost0 interface, additional tuning of the TF Agent VRF tables is required.

Therefore, next changes of the TF Agent code are introduced:

1. Adaptation of the TF's TCP layer classes (those who are used by an HTTP proxy server) to IPv6 protocol.
2. Advertisement of routes to the metadata proxy server IPv6 prefix (e.g., fe80::a9fe:a9fe/128) in the VRF table where an interface of a requesting virtual machine resides.
3. Advertisement of reciprocal IPv6 link local routes to a metadata receiving response interfaces (e.g., fe80::/10) in corresponding VRF tables.
4. Configuration of the host OS network infrastructure (assignment of IPv6, e.g. fe80::a9fe:a9fe, address to vhost0 interface, adding virtual machine interfaces IPv6 link local and link level addresses to kernel neighbors table on vhost0 interface).

Overall procedure of packet routing & forwarding in case of metadata request via IPv6 protocol is next.
1. TF Agent receives an IPv6 address for the metadata proxy server from the global confab.
2. Using the received IPv6 address, corresponding routes are advertised (e.g., fe80::a9fe:a9fe/128) in all needed VRF tables and the address is added to the vhost0 interface in the host OS.
3. With this routing information a virtual machine can now send an outgoing request to the specified address (e.g., fe80::a9fe:a9fe).
4. The packet with the request from a virtual machine is intercepted by the TF Agent's flow analysis module. If the flow analysis module discovers that a destination address is equal to the specified IPv6 address for the metadata proxy server (e.g., fe80::a9fe:a9fe), it installs corresponding reciprocal routes and updates neighbors table in the host OS.
5. Using the routing information from steps 2-4, metadata request and response packets are forwarded by the TF vrouter between a virtual machine and the metadata proxy server of the TF Agent.

### 3.1 Affected Modules

Next modules of the TF are affected / modified:
- contrail-vrouter-agent (tf-controller repository):
  + TCP session (TcpSession class);
  + TCP server (TcpServer class);
  + Metadata proxy server (MetadataProxy class);
  + Pkt (class PktFlowInfo);
  + InterfaceTable class.
- vrouter.ko (tf-vrouter repository)

### 3.2 Alternatives considered

VR metadata proxy access connectivity could be made using NAT as it is in IPv4 metadata, but NAT for IPv6 is not yet supported in TF forwarder.

### 3.3 API schema changes

No changes.

### 3.4 User workflow impact

User workflow doesn't change: he/she can work with the IPv6 version of the metadata service right after the introduction of source code changes using the same tools he/she used to employ with the IPv4 version.

### 3.5 UI changes

No changes are needed. The IPv6 address for the metadata proxy server is retrieved from the TF GUI by concatenating "fe80::" and IPv4 parts: fe80:: + 169.254.169.254 ==> fe80::a9fe:a9fe .

### 3.6 Operations and Notification impact

To work with the feature an address for the metadata proxy server must be supplied (it is fe80::a9fe:a9fe by default). After that metadata can be requested from any connected virtual machine using ``curl`` or other similar programs.

Warnings and errors are dumped into the standard TF Agent's log file.

### 3.7 Security issues

A set of tests need to be created to prove that no new information security threats are introduced for TF components as well as clients VMs

## 4. Implementation

### 4.1 Assignee(s)

- Overall idea & supervision: Yu. Konovalov.
- Implementation: M. Kraposhin.

### 4.2 Work items

Next parts of control and data plane were modified.

#### 4.2.1 vr_fabric_input

By default in R2011 version of the TF, all IPv6 packets ingressing vhost0 interface are redirected to the underlay Ethernet device. This behavior was changed to account whether vhost0 is in xconnect mode:

    if (pkt->vp_type == VP_TYPE_IP6 && vif_mode_xconnect(vif))
        return vif_xconnect(vif, pkt, fmd);

#### 4.2.2 address_util.h and TcpSession

A new function was introduced to resolve canonical names from IPv6 address:

    ResolveCanonicalNameIPv6

TcpSession class has been modified to account for introduction of ResolveCanonicalNameIPv6 function.

#### 4.2.3 TcpServer

TcpServer class has been modified to enable listening on IPv6 sockets.

#### 4.2.4 InterfaceTable

InterfaceTable class methods have been modified to store a mapping between interfaces (their link local IPv6 addresses) and virtual machines identification information.

#### 4.2.5 PktFlowInfo

PktFlowInfo::IngressProcess method has been modified to intercept requests to the metadata proxy server (fe80::a9fe:a9fe) and to advertise all required routes.

#### 4.2.6 MetadataProxy

MetadataProxy class has been modified to perform required routing tasks:

- advertisement and deletion of routes to the metadata proxy server address (fe80::a9fe:a9fe);
- advertisement and deletion of routes to requesting virtual machines via Ethernet IPv6 link local addresses;
- manipulation of a host OS underlay network system via Netlink messages (vhost0 address and neighbors);
- TF Agents routing tables events handlers and the metadata proxy server address modification (IPv6 address and port).

## 5. Performance and scaling impact

The introduced changes should not affect scaling and performance characteristics of the TF Agent & vrouter.

## 6. Upgrade

N/A

## 7. Deprecations

This feature doesn't deprecate other.

## 8. Dependencies

No components that depends on this feature.

## 9. Testing

### 9.1 Unit tests

Unit tests have been prepared and deployed together with the source code. The tests check basic functionality of all developed methods and classes.

### 9.2 Dev tests

N/A

### 9.3 System tests

The feature was tested using R2011 installation of the TF coupled with the OpenStack platform. These tests included:
1. sequential creation of virtual machines in random IPv6 networks with ping to fe80::a9fe:a9fe;
2. sequential creation of virtual machines in random IPv6 networks with curl to fe80::a9fe:a9fe;
3. parallel curl to fe80::a9fe:a9fe from several virtual machines / interfaces that were created in random IPv6 networks.

## 10. Documentation Impact

Available documentation might require an update to mention the introduced feature. Since settings for the IPv6 metadata service rely only on the IPv4 version of this component, user is not expected to input any additional data. Therefore, present knowledge should be enough to start working with the IPv6 metadata service.

Source code of the feature is self-documented using the Doxygen syntax.

## 11. References

### 11.1 Main references

[MetadataOpenStack] https://docs.openstack.org/nova/latest/user/metadata.html

[NeuIPv6] https://specs.openstack.org/openstack/neutron-specs/specs/ussuri/metadata-add-ipv6-support.html

[MetadataFAQ] https://ibm-blue-box-help.github.io/help-documentation/nova/Metadata_service_FAQ/#q-why-would-you-want-to-use-the-novaneutron-metadata-service

[RFC42911] https://www.rfc-editor.org/rfc/rfc4291

### 11.2 Useful materials:

Metadata Types: http://madorn.com/openstack-metadata-types.html#.ZDQD__ZBxPZ

OpenStack Metadata: http://madorn.com/openstack-metadata-service.html#.ZDQEFfZBxPY

https://github.com/madorn/cloud-init-guide

