
# 1. Introduction
Design and Implement superspine for 3-tier/5-stage fabric.


# 2. Problem statement
Support for 3-tier/5-stage fabric. Support new layer with devices as super-spine physical role. New super-spine devices
would be behaving either like lean device(IP forwarding) or Route-Reflector acquring and relecting routes for iBGP peers.
Super-spines would be connected to spines in CRB topology or Borderleaves in ERB topology.

# 3. Proposed solution
Following are proposals for additional support 3-tier/5-stage fabric.
        a. New physical role super-spine would be created.
        b. All devices with "super-spine" physical roles can ony have either "lean" or "Route Reflector" as routing bridging role.
           No other role as of today would be supported for physical role "super-spine"
        c. Fabric links should come-up between spine and super-spine or super-spine and border leaf.
        d. ZTP and Brownfield of super-spine supported.

Following table defines "Super-spine" routing bridging roles for individual device type:
        Platform             physical-role               rb-role
        
        QFX10K               Super-spine                 Lean/RR
        QFX5110, QFX5100     Super-spine                 Lean
        QFX5120, QFX5200,    Super-spine                 Lean/RR 
        QFX5210, QFX5220    
        MX                   Super-spine                 Lean/RR

NOte: Hierarchical route reflector functionality for super-spine is not supported with this story.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
Not Applicable

## 3.3 User workflow impact
User will have option to configure new physical role "Super-Spine" and could assign existing routing bridging role as "lean" 
or "Route-Reflector" as per desired topology.

## 3.4 UI changes
None

## 3.5 Notification impact
None


# 4. Implementation
## 4.1 Work items
Device Manager changes: Changes would be regarding defining new physical role "Super-Spine" and associating it with correct device type.
"object_type": "physical-role",
      "objects": [
        {
          "fq_name": [
            "default-global-system-config", "super-spine"
          ],
          "name": "super-spine"
        }

Role Assignment: Fabric links between spine and superspine and super-spine and border lead should be detected and assigned IP address. Currently only fabric links
                 between spine and leaf are detected.

# 5. Performance and scaling impact
No Impact

## 5.1 API and control plane
None

## 5.2 Forwarding performance
None

# 6. Upgrade
No impact.

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
a. Check Role assignment with new physical role "Super-Spine" works with supported platforms.
b. Other than "lean" and "Route-Reflector" no other routing bridging role is supported.
c. CRB topology  
d. ERB topology
e. Interop with collapsed spine.

details of unit testing are at location https://drive.google.com/drive/u/0/folders/1lb2OPqrcrW5I4oXQuzATFoAOYDCZDC4_?ths=true 
# 10. Documentation Impact


# 11. References

