# 1. Introduction
REST/gRPC based schema driven api-server is resusable component, which
can be used by various application/microservice in Contrail and Juniper.

This blueprint describes implementing a Generic Api-Server-Framework(ASF).
Using this ASF framework new application/microservice need not implement
the api-server, instead appications can come with the schema and use plugin
interface provided by the ASF to plugin the custom REST/gRPC API's.

# 2. Problem statement
Any new microservice that need to expose northbond API's or API's for
east-west communcation with other microservice, are developing their
own shema driver api-server, which results in duplication of code and
increase in maintanece cost.

# 3. Proposed solution
Contrail go api-server already implements schema driven api-server,
however it is tightly coupled with contrail config application/microservice.

Proposal is to refactor contrail go api-server and make it as a reusable
generic Api-Server-Framework(ASF).

# 4. API schema changes
N/A

# 5. UI changes / User workflow impact
N/A

# 6. Notification impact
N/A

# 7. Provisioning changes

# 8. Implementation
The main implementation changes include:
1) Carving out the core functionalities to seperate public git repository

2) Enhance code generation tool for the applications to specify various schema

3) Well defined interface to plugin and register schema generated API's

3) Well defined interface to plugin and register custom API's

4) Well defined interface to plugin services to the request pipeline.

# 9. Performance and scaling impact
N/A

# 10. Upgrade
N/A

# 11. Deprecations
N/A

# 12. Dependencies
N/A

# 13. Testing
#### Unit tests
#### Dev tests
#### System tests

# 14. Documentation Impact
The developer guide should be documented in the ASF repo, with detailed explanation
about the list of pulgin inerface provided by the ASF framework.

# 15. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-8845

