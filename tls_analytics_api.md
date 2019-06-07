# 1. Introduction
This blueprint describes the requirements, design and implementation of ssl support on rest api port of analytics api.

# 2. Problem statement
The connection between Analytics API server and the clients is not encrypted and can pose a security threat.

# 3. Proposed solution
Changes need to be made, both on the server side as well as to the clients which connect to the rest api port of analytics api server(8081)
Provisioning changes need to be made to add corresponding flags in the conf files, of both the analytics api server and the clients.
Those provisioning changes will be used to enable ssl support.
The clients which are connecting to the rest-api post, 8081 are:
1. Device Manager
2. Service Monitor
3. Contrail Topology
4. Contrail WebUI

# 4. Alternatives considered
None

# 5. API schema changes
None

# 6. UI changes
None

# 7. Notification impact
None

# 8. Provisioning changes
None

# 9. Implementation
1. Server side changes to use SSL certificates when SSL is enabled in the config files. This needs changes in the OpServer.
2. Changes on the clients to use SSL certificates when SSL is enabled in the config files.
3. Changes in the Entrypoint.sh file of corresponding docker containers.

# 10. Performance and scaling impact
None

# 11. Upgrade
None

# 12. Deprecations
None

# 13. Dependencies
None

# 14. Testing
## Unit tests
Extend exisiting testcases to use SSL

# 15. Documentation Impact

# 16. References
