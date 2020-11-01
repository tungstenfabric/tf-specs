# 1. Introduction
Manage configuration templates that can be applied on one or more of the managed devices.

- Visualize the list of templates in the repository, by name, device_family, date.

- edit/delete/upload of templates.

- Clone a template into a new version and edit it (e.g. add new User inputs, change the snippets).

- Create a New template and specify the User inputs for that template.

Apply those templates on selected list of devices and verify it has been applied correctly.

- Select and apply on a list of devices.

- Run the template, entering relevant inputs on the UI.

- Verify the configuration template has been applied properly on the device.

### Supported Deployment model:
N/A

### Supported Device Models:
    QFX, MX and SRX series.

### Supported Device version:
    18.4R2-S3 and above.

# 2. Problem statement
CFM should support the Creation and deployment of Config template to the managed devices.

# 3. Proposed solution
Proposed solution is a two-step process.

1. Template definition (landing page with CRUD)
2. Template association/deployment.

## 3.1 Template Definition:

User should be able to manage/create the templates which will be achieved in below steps.

- Step1: UI will take the input- name, device family, template content and template-type – on click next or similar action step2.
- Step2: UI should call the backend API `/validate-template` to validate the template and get the variables associated with the provided template input.
- Step3: Store the template content and variables.

## 3.2 Template Association/Deploy to devices:

User should be able to provided values to template and deploy these to device(s).

- Step1: Select the template and show the list of devices to be applied on (filtered based on device family).
- Step2: Provide the values for the variables in a dynamic UI (UI should be rendered based on the variables stored in the step2/3 of template definition).
- Step3: UI should call `push_config_template` job with config_template_uuid, global variable values, list of device_uuid, device level variable values.
- Step4: Job will invoke the an ansible playbook to push the config to device(s) by
  - getting the template content.
  - rendering template content providing global/device level variable values.
  - get the CLI content and push to device(s).

Example of a template:
- Below template configures the dual RE configuration with static content and variables.
   ```jinja
    set groups re0 system host-name {{re0_hostname}}
    set groups re0 interfaces fxp0 unit 0 family inet address {{mgmt_ip}} master-only
    set groups re1 system host-name {{re1_hostname}}
    set groups re1 interfaces fxp0 unit 0 family inet address {{mgmt_ip}} master-only
    set apply-groups re1
    set apply-groups re0
    set system commit synchronize
    ```

- Below are the list of variables parsed from the above template:
    ```jinja
    re0_hostname
    re1_hostname
    mgmt_ip
    ```

# 4. API schema changes
- New API `/validate-template` to get variables associated to the template by providing name, device family, template content and type.
- New schema added for ConfigTemplate object as below in `fabric.xsd`.
```xml
<xsd:element name="config-template" type="ifmap:IdentityType"/>

<xsd:element name="global-system-config-config-template"/>
<!--#IFMAP-SEMANTICS-IDL
          Link('global-system-config-config-template',
             'global-system-config', 'config-template', ['has'], 'optional', 'CRUD',
             'List of config templates.') -->

<xsd:element name="template-name" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('template-name', 'config-template', 'required', 'CRUD',
              'Name of the config template') -->

<xsd:element name="template-content" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('template-content', 'config-template', 'required', 'CRUD',
              'Content of the config template') -->

<xsd:element name="device-family" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('device-family', 'config-template', 'optional', 'CRUD',
              'Device family of the config template') -->

<xsd:element name="template-type" type="TemplateType"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('template-type', 'config-template', 'optional', 'CRUD',
              'Type of the config template') -->

<xsd:element name="device-vars" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('device-vars', 'config-template', 'optional', 'CRUD',
              'Device specific variables associated to the config template') -->

<xsd:element name="global-vars" type="xsd:string"/>
<!--#IFMAP-SEMANTICS-IDL
          Property('global-vars', 'config-template', 'optional', 'CRUD',
              'Global variables associated to the config template') -->

<xsd:simpleType name="TemplateType" default='SET'>
     <xsd:restriction base="xsd:string">
         <xsd:enumeration value="SET"/>
         <xsd:enumeration value="XML"/>
     </xsd:restriction>
</xsd:simpleType>
```

# 5. Alternatives considered
N/A

# 6. UI changes / User workflow impact
- Workflow for creation of template by name, device family, template content, type and variables.
- Grid to list all the templates created.
- Workflow to deploy config template to managed devices by providing global and device level variables.

# 7. Notification impact
N/A

# 8. Provisioning changes
N/A

# 9. Implementation
- A new api `/validate-template` to validate and get the variables associated to the template.
- New changes in fabric.xsd for ConfigTemplate object creation with template-name, template-content, device-family, template-type (SET or XML), device-vars and global-vars.
- New `push_config_template` job which calls ansible role `push_config_template` to push template content to managed devices according to provided global/device variables.

# 11. Performance and scaling impact
N/A

# 12. Upgrade
N/A

# 13. Deprecations
N/A

# 14. Dependencies
N/A

# 15. Testing

## 15.1 Sanity tests
1. Call `/validate-template` api by providing name, device family, template content and type.
2. Create ConfigTemplate object by providing name, device family, template content and type and variables values.
3. Call `push_config_template` job with global variables values for both devices.
4. Job should be succeeded and config correct config should be pushed.

## 15.2 Feature tests
Below is the link to the feature test plan,
https://drive.google.com/file/d/12wyEJqUX16oSHrBKCLIiv0F4Aa3KLLjg/view?usp=sharing

# 16. Documentation Impact
It will be documented as part of release documentation by the doc team.

# Known issues and Caveat
There are no known issues for this functionality.

# 18. References
https://contrail-jws.atlassian.net/browse/CEM-15726
