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

Following change will be needed to enable SSL between Redis clients and servers.

1. All clients, namely, alarm-gen, analytics-api, collector and query-engine, will have following conf file changes
   redis_use_ssl=True
   redis_keyfile=/etc/contrail/ssl/private/server-privkey.pem
   redis_certfile=/etc/contrail/ssl/certs/server.pem
   redis_ca_cert=/etc/contrail/ssl/certs/ca-cert.pem

2. Stunnel program needs to run within a new container with following conf parameters.
   cert = /etc/contrail/ssl/certs/ca-cert.pem 
   pid = /var/run/stunnel.pid
   foreground = yes
   [redis]
   accept = 0.0.0.0:6666
   connect = 127.0.0.1:6379

   The way this configuration works is when a client on the client host connects to port 6666 locally it will be forwarded 
   through the SSL tunnel that stunnel has created and redirected to redis instance running on server.

3. Alarmgen and Analytics api are using StrictRedis to connect to Redis. It has SSL supports. So, we just need to pass SSL 
   parameters when SSL is enabled.
   
4. Collector and Query Engine are using hiredis library to connect to Redis. In this current version of hiredis 0.11.0, there 
   is no SSL support. So, we have to upgrade to latest relased version 0.14.0 and apply SSL patch on top of it.
   However, in relase 1906 only analytics-api and alarm-gen will be supported to have SSL connection with Redis. So, hiredis 
   work will be taken up post this release.

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
1. Redis client side changes to use SSL certificates when SSL is enabled.
2. Stunnel docker setup.
3. Provisioning changes to update the SSL options. We are planning to support provisioning only via ansible-deployer. 
   Other deployment schemes are not planned.

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
