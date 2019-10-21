# 1. Introduction
REST/gRPC based schema driven api-server is reusable component, which
can be used by various application/microservice in Contrail and Juniper.

This blueprint describes implementing a Generic Api-Server-Framework(ASF).
Using this ASF framework new application/microservice need not implement
the api-server, instead applications can come with the schema and use plugin
interface provided by the ASF to plugin the custom REST/gRPC API's.

# 2. Problem statement
Any new microservice that need to expose northbound API's or API's for
east-west communication with other microservice, are developing their
own schema driver api-server, which results in duplication of code and
increase in maintenance cost.

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

1) Carving out the core functionalities to separate public git repository
All Go api-server packages that can be reused in the ASF should be moved to the ASF repo with change history.
Then they should be decoupled from schema generated code, because it won't be available in framework repository.
This should be done in a way that allows reusing the packages and combining them with custom schema generated code.

For example, `./pkg/db` package and it's subpackages contain various database drivers mixed with contrail schema specific methods.
The decoupling should extract schema independent structs that later can be reused in the client package (for example by struct embedding).

2) Enhance code generation tool for the applications to specify various schema
Current code generation tool was narrowly tailored to generate code based on fixed schema stored in specific directories.
This should be changed to allow users provide their own schema files.
The user must be able to generate the code based on public schema defined outside the project and the private schema files known only in the private context.
Additionally schema tool should support the mutating the public schema in private context.

Those features should be implemented by modifying the schema generation tool (`./pkg/schema/`).

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
about the list of plugin interface provided by the ASF framework.

# 15. References
JIRA story : https://contrail-jws.atlassian.net/browse/CEM-8845

