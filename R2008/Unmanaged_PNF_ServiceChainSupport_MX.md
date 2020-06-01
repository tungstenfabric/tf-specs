# 1. Introduction
Contrail Fabric Manager (CFM) supports multiple physical roles and routing bridging (overlay) roles on QFX devices and MX devices currently. This Unmanaged PNF service chain functionality is supported for the QFX boxes.

# 2. Problem statement
Users should have option to configure either anycast ip or individual ips in ERB gateways or CRB Gateways
As on today, contrail is pushing only anycast ip on irb interface ip in ERB gateways. The user will have the option to also configure static routes or eBGP routing protocol session on those IRBs. Finally those constructs can be applied on L2 VPGs (access ifls) or Routed VPGs (fabric ports)  

#### Supported Device Models
MX 240, MX 480, MX 960, MX 10003

# 3. Proposed solution
The proposed solution is to use the existing CFM architecture for role based config push to the DC devices and add the role required for MX. CFM already supports this Unmanaged PNF service chain functionality is supported for the QFX boxes. We will be using the same backend code and will do the required jinja template changes to make this functionality available on MX.

# 4. API schema changes
The schema changes are already covered as part of CEM-4455 story and no more changes required.

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
The UI changes are already covered as part of CEM-4455 story and no more changes required for this functionality to support on MX.

# 7. Notification impact
N/A
# 8. Provisioning changes
N/A
# 9. Implementation
As mentioned in the proposed solution the functonality which already added for the QFX support will be used as it is and have the required jinja templates modified to support on MX, mainly the type5 template.


# 10. Performance and scaling impact
N/A

# 11. Upgrade
N/A

# 12. Deprecations
N/A

# 13. Dependencies
N/A
- 
# 14. Testing
## 14.1 Unit tests
The UT cases are alreday covered as part of the QFX support of Unmanaged PNF service chain.

## 14.2 Feature tests
Below is the link to the feature test plan,
https://drive.google.com/file/d/1PBNOAYIGgUuY8HY2jInkuc0WqkshpRr_/view?usp=sharing

# 15. Documentation Impact
It will be documented as part of release documentation by the doc team.

# 16. References
https://contrail-jws.atlassian.net/browse/CEM-13032
https://contrail-jws.atlassian.net/browse/CEM-4455
https://github.com/tungstenfabric/tf-specs/blob/master/R2002/unmanaged_pnf_lr_interconnect.md