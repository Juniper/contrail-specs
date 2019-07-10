# 1. Introduction
This blue print describes the requirements, design and implementation of REDIS TLS Support.




# 2. Problem statement
The requests from contrail-analytics clients to Redis server are not encrypted and can pose a
security request.

# 3. Proposed solution
Redis does not support encryption. In order to implement setups where trusted parties can access a
Redis instance over the internet or other untrusted networks, an additional layer of protection should
be implemented. We will use a secure tunneling program called stunnel to encrypt the Redis traffic.
Traffic between Redis clients and servers will be routed through a dedicated SSL encrypted tunnel.

Currently, alarm-gen, analytics-api, collector, query-engine and webui connect to Redis. Out of theses
collector and webui connect to local redis instance listening on localhost 127.0.0.1.
Since, they do not connect to any remote redis instance over internet, there is no need to enable SSL
for these 2 services.

Following change will be needed to enable SSL between Redis clients and servers.

1. All clients, namely, alarm-gen, analytics-api and query-engine will have following conf file changes
   redis_use_ssl=True
   redis_keyfile=/etc/contrail/ssl/private/server-privkey.pem
   redis_certfile=/etc/contrail/ssl/certs/server.pem
   redis_ca_cert=/etc/contrail/ssl/certs/ca-cert.pem

2. Stunnel program needs to run within a new container with following conf parameters.
   cert = /etc/contrail/ssl/certs/ca-cert.pem
   pid = /var/run/stunnel.pid
   foreground = yes
   [redis]
   accept = 10.204.217.62:6379
   connect = 127.0.0.1:6379

   In case of SSL enabled, client of redis-server is stunnel which will always run locally. Stunnel will send the packets to redis
   server on localhost port 6379. So, redis will always listen on localhost:6379. Analytics clients will connect to stunnel on
   public ip-address and port 6379.

   In case of no SSL, behavior will remain unchanged.

3. Alarmgen and Analytics api are using StrictRedis to connect to Redis. It has SSL supports. So, we just need to pass SSL
   parameters when SSL is enabled.

4. Query Engine are using hiredis library to connect to Redis. As of now, hiredis 0.14.0 release is officially available
   but it does not have SSL support. However SSL code is present in latest hiredis github master branch. We have to get an official
   hiredis 0.15.0 release which has SSL support. If latest release is not available, then we can use 0.14.0 release and apply
   SSL patch on top of it.

5. Contrail provisioning will be updated to populate SSL parameters in redis client configuration files. New docker files will
   be added to run stunnel service.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
Not Applicable

## 3.3 User workflow impact
Please see above for the required configuration to be done.

## 3.4 UI changes
None

## 3.5 Notification impact
None


# 4. Implementation
## 4.1 Work items
1. Redis client side changes to use SSL certificates when SSL is enabled. This needs changes in analytics and webui backend.
2. Stunnel docker creation in Redis POD. This needs changes in contrail-container-builder.
3. Deployer changes -
   Priority order for the deployers is the following one:
   RHOSP /TripleO
     -- Stunnel docker will be created if SSL_ENABLE is true in contrail configuration settings.
   OpenShift
   Juju
   Ansible-deployer
      -- Stunnel docker will be created if REDIS_SSL_ENABLE is true in contrail configuration.
   Helm

# 5. Performance and scaling impact
TODO

## 5.1 API and control plane
None

## 5.2 Forwarding performance
None

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
1. Extend existing Redis testcases to use SSL.
2. Stunnel and Redis restart tests.
3. Negative tests.

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
