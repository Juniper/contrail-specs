# 1. Introduction
This blueprint describes the requirements, design and implementation of ssl support on rest api port of analytics api.

# 2. Problem statement
The connection between Analytics API server and the clients is not encrypted and can pose a security threat.

# 3. Proposed solution
Changes need to be made, both on the server side as well as to the clients which connect to the rest api port of analytics api server(8081)
Provisioning changes need to be made to add corresponding flags in the conf files, of both the analytics api server and the clients.
Those provisioning changes will be used to enable ssl support.
The clients which are connecting to the rest-api post, 8081 are:
1. Service Monitor
2. Contrail-Command

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
9.1. Server side changes to use SSL certificates when SSL is enabled in the config files. This needs changes in the OpServer.
Fields added in the conf files are:
    analytics_api_ssl_enable
    analytics_api_ssl_certfile
    analytics_api_ssl_keyfile
    analytics_api_ssl_ca_cert


9.2. Changes on the clients to use SSL certificates when SSL is enabled in the config files.
    Above fields will be added in the conf files of the respective clients as well.

9.3. Changes in the Entrypoint.sh file of corresponding docker containers.
if is_enabled ${ANALYTICS_API_SSL_ENABLE} ; then
  read -r -d '' analytics_api_certs_config << EOM || true
analytics_api_ssl_enable=${ANALYTICS_API_SSL_ENABLE}
analytics_api_ssl_certfile=${ANALYTICS_API_SERVER_CERTFILE}
analytics_api_ssl_keyfile=${ANALYTICS_API_SERVER_KEYFILE}
analytics_api_ssl_ca_cert=${ANALYTICS_API_SERVER_CA_CERTFILE}
EOM
else
    analytics_api_certs_config=''
fi


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
14.1 Positive tests
Test cases to make sure that when SSL is enabled, clients are able to connect using https and  not able to connect using http.
14.2 Negative tests
Test cases to make sure that when SSL is not enabled, clients are not able to connect using https and are only able to connect using http.
Also, when certificates are missing or incorrect, clients should not be able to connect.


# 15. Documentation Impact

# 16. References
