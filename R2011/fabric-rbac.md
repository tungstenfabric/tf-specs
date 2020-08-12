 # 1. Introduction
 All fabric objects currently belong to Cloud admin and are not managed via RBAC. Because of this reason tenant admin cannot independently manage the fabric related resources.

# 2. Problem statement
In some use cases multiple customers/tenants  will be sharing the same fabric. Different overlay resources have to be bound to tenants including Virtual Port-Group.

By default a subtending tenant has no fabrics, devices, nor ports visibility. It must be assigned by the super admin (user that on-boarded the fabrics).    

Following are the three use cases  will be addressed in this story.
 **use case 1 :**  Cloud admin creates VPG and shares it with the tenant
 **use case 2 :**  Cloud admin shares  physical ports with  tenant and tenant creates VPG .
 **use case 3 :**  Cloud admin shares physical router and tenant creates logical router. 

# 3. Proposed solution
-   All fabric objects will be owned by the Cloud admin.    
-   Cloud admin can share objects with other tenants including fabric objects.    
-   Sharing can be done using UI workflow .
-   If a tenant admin is creating an object, that object will be shared with that tenant. Automatic Sharing based on RBAC AUTH_TOKEN of the user will take    
care of by the API server.
-   To support UI workflow, required fabric objects can be shared to tenant admins with “Read” permission .

# 4. API schema changes
   VPG has references to VMI . It will be changed  to VMI to VPG .

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact

N/A
# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation

## **Use case 1 :** Cloud admin creates VPG and shares it with the tenant
1.  Cloud admin shares physical port(s) with a specific tenant.
2.  Tenant admin creates their own VPG using these physical ports.  
3.  Tenant admin applies tenant's security policies and other features on this VPG.  
4.  Tenant admin creates his VLANs (virtual networks) on this VPG.
5.  Only the tenant creating the VPG can see the port(s) and the VPG.

## Workflow with proposed design

**Creation of fabric an onboarding physical routers**
-   Cloud admin will create the fabric and onboard physical routers and physical ports.   
-   Cloud admin will be the owner of fabric objects (fabric, physical routers ,physical ports ) under global system configuration.
    
**VPG creation**
-   Cloud admin will share physical ports with tenant1 . 
-   Tenant1 admin will create VPG1 using the physical ports shared with it.   
-   VPG1 will be owned Cloud admin and automatically shared with tenant1 based on RBAC AUTH_TOKEN of the user .
   
**VLAN association**
-   Tenant admin for tenant1 will create VMI and VN within the project.    
-   Tenant admins for tenant1 will associate VMI (VPG1-10) and VN1 with   VPG1(which result in VLAN association).    
-   VN’s or VMI’s shared to tenant1 also can be associated with VPG1  

**Port and VPG visibility**
-   VPG1 will be visible cloud admin and tenant1 admin.
-   VPG1 won’t be visible to tenant admin.(no sharing)    
-   To support UI workflow we can share Physical port/physical router to tenant1 admin with “Read” permission 

## **Use case 2 :** Cloud admin shares on-boarded physical ports with tenant and tenant creates VPG .

1.  Cloud admin creates a VPG using these physical ports and shares with these tenants.    
2.  Tenant admins create their own VLANs on this VPG.    
3.  All tenants can see the ports and the VPG but not the VLANs created by other tenants.    
4.  VPG related features like SG, port-profile can only be created/updated by Cloud Admin ,
    

## Workflow with proposed design

### Creation of fabric an onboarding physical routers

-   Cloud admin will create the fabric and onboard physical routers and physical ports.
-   Cloud admin will be the owner of fabric objects (fabric, physical routers ,physical ports ) under global system configuration.
    

### VPG creation

-   Cloud admin will create VPG1. VPG1 will be owned by cloud admin.    
-   Cloud admin will share VPG1 with the tenant1 and tenant2.    
-   Sharing can be done using UI workflow or using VNC API’s .    

### VLAN association

-   Tenant admins for tenant1 and tenant2 will create VMI and VN within the project.    
-   Tenant admins will associate VMI and VN with VPG(which result in VLAN association).    

### Port and VPG visibility

-   Physical Ports will visible to Cloud admin and tenants to which these ports are being shared.
-   VPG created will be visible to cloud admin and tenant admin .    
-   Vlan’s created will be visible to the tenant it created. (Because VN and VMI are only visible to tenant with access(owned or shared))
    
-   To support UI workflow, Physical port/Physical router and VPG’s will be shared with “tenant admin” with “Read” permission and with RBAC ACL configured. Only a partial read will be supported for these objects.

## **Use case 3 :** Cloud admin shares physical router and tenant creates logical router.

1.  Cloud admin shares a physical router (PR) with other tenants.    
2.  Tenants create logical routers (LR) on this PR    
3.  PR is visible to all tenants. A tenant can see only their LR’s on this PR.    

## Workflow with proposed design 

Creation of fabric an onboarding physical routers
-   Cloud admin will create the fabric and onboard physical routers and physical ports.    
-   Cloud admin will be the owner of fabric objects (fabric, physical routers ,physical ports ) under global system configuration.    

LR creation

-   Cloud admin will share physical routers with tenant1 .    
-   Sharing can be done using UI workflow or using VNC API’s .    
-   Tenant1 admin will create LR R1. R1 will be owned by cloud administrator and shared with tenant1 .   
-   Public LR and NAT attributes can only be updated by cloud admin . RBAC ACL’s  will be added to enforce this restriction.

LR visibility
-   Physical Routers will visible to Cloud admin and tenants to which these ports are being   shared.
-   R1 will be visible cloud admin and tenant admin .

# 11. Performance and scaling impact
N/A
# 12. Upgrade
During upgrade API server will handle db reconciliation required. because
of schema change.

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing

N/A

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# Known issues and Caveat
There are no known issues for this functionality.

# 18. References
[https://contrail-jws.atlassian.net/browse/CEM-12712](https://contrail-jws.atlassian.net/browse/CEM-12712)
**[https://github.com/Juniper/contrail-controller/wiki/RBAC](https://github.com/Juniper/contrail-controller/wiki/RBAC)**
[https://docs.google.com/document/d/1iM8uKI8h3p4l5Mv5DSjBbFlX9sBAe76iIcYlXCZOv0g/edit#](https://docs.google.com/document/d/1iM8uKI8h3p4l5Mv5DSjBbFlX9sBAe76iIcYlXCZOv0g/edit#)
[https://docs.google.com/document/d/1-4mt48mScxI8WEwAFYGxFapF9I9sXwCZ_RJmhwsggac/edit](https://docs.google.com/document/d/1-4mt48mScxI8WEwAFYGxFapF9I9sXwCZ_RJmhwsggac/edit)
