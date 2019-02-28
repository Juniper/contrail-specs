# 1. Introduction

There is currently no way to use TLS 1.2 when deploying Contrail 5.x using 
contrail-ansible-deployer. This document describes the feature to enable the 
ability to configure TLS 1.2 for Openstack and Contrail services using the
contrail-ansible-deployer provisioning scripts.

# 2. Problem statement

Customers require the ability to configure and use TLS 1.2 for contrail services.

# 3. Proposed solution

In previous versions of Contrail (<= 4.x), the following services supported TLS 1.0:

		VNC API (using haproxy)
		XMPP
		Introspect
		WebUI
	
Wherever possible TLS support will be upgraded to TLS 1.2. It 
is also planned to add TLS support to the following external services in Contrail 5.1 and later where possible:
	
		Cassandra
		Kafka
		RabbitMQ
		Zookeeper
		Redis
	

## 3.1 VNC API

The VNC API in previous versions of Contrail did not support TLS natively. The
VNC API traffic was encrypted by having the client talk to the backend through 
a haproxy server and enabling SSL encryptions for the conversation between the 
client and the haproxy server. This was done in order to avoid the performance 
degradation that would be seen if encryption was enabled natively. A 
benchmarking exercise needs to be done to study the performance when TLS is 
enabled natively.

For Contrail 5.1, haproxy will be used to support TLS encryptions for the VNC 
API service. Note that this encryption is only between clients to the haproxy 
service. The backend communication between haproxy and the VNC API service 
will still be unencrypted. The haproxy can be configured to run on the same 
node if unencrypted traffic is not desired on the wire. Native VNC API 
encryption will be enabled in a future release.


## 3.2 XMPP
The libboost library is used to provide SSL encryption services to the XMPP 
server. The version of libboost that comes in Centos 7.5 (which is the 
contrail base container) does not support TLS 1.2. 

Until libboost support for TLS 1.2 is available in Centos, a software workaround
to enable TLS 1.2 encryption is done in the XMPP codebase. Encryption is enabled
using SSLv23 which will use the TLS version that the host openssl package
supports which includes TLS 1.2.

## 3.3 Introspect Service
Support for TLS 1.2 is available and there is code in contrail-ansible-deployer 
that configures required knobs when `contrail_configuration.SSL_ENABLE` is 
set to `True`. 

## 3.3 Sandesh
Support for TLS 1.2 is available and there is code in contrail-ansible-deployer 
that configures required knobs when `contrail_configuration.SSL_ENABLE` is 
set to `True`. 

## 3.3 WebUI Service
Like the VNC API service, for 5.1, encryption for WebUI will be done through a 
Haproxy frontend.


## 3.5 External services that support TLS

### Rabbitmq
Rabbitmq configuration will be enabled in the container as well as in the 
orchestrator deployers and can be selectively enabled/disabled.

### Kafka
Kafka configuration for TLS will be enabled in the container as well as in 
the orchestrator deployers and can be selectively enabled/disabled.

### Cassandra
Cassandra configuration for TLS will be enabled in the container as well as 
in the orchestrator deployers and can be selectively enabled/disabled.

## 3.6 External services that do not support TLS 

### Zookeeper
Zookeeper server TLS support is in Beta phase. Hence support for Zookeeper will
not be added in 5.1 and will be added when TLS support stabilizes for Zookeeper.

On the client side, the software that Contrail utilizes to access the service (python-kazoo 2.5.0-0) 
has no support for TLS. Support for TLS has been added to python-kazoo in the 
latest version according to [this link : (https://github.com/
python-zk/kazoo/issues/382)](https://github.com/python-zk/kazoo/issues/382) 
This version has been released on September 2018. The feasibility of using 
this new version of the kazoo client in Contrail 5.1 packaging needs to be 
explored as there might be other software dependencies. But since the support 
for TLS in the package is relatively very recent, it is proposed to wait till 
the support matures before adopting it.

### Redis
Redis server supports TLS only in their enterprise version and not in the 
opensource version. Hence it is not possible to enable TLS for Redis at this 
point.

If encryption is required for services which cannot natively enable TLS 
encryption, then deployers can use a haproxy container to frontend the 
services and enable encryption on the haproxy endpoints.


# 4. Implementation

All possible TLS configuration can be tied down to a single knob for Contrail:

		contrail_configuraiton:
			SSL_ENABLE: True
			
with further specific configurations which can be turned off if required. 
By default the following knobs are set to True if SSL_ENABLE is set to True as 
above:

		contrail_configuration:
			RABBITMQ_SSL_ENABLE: False
			KAFKA_SSL_ENABLE: False
			CASSANDRA_SSL_ENABLE: False
			INTROSPECT_SSL_ENABLE: False
			XMPP_SSL_ENABLE: False
		
and the following knobs for Openstack:
		
		kolla_config:
			kolla_globals:
				kolla_enable_tls_external: True
				kolla_external_fqdn_cert: <path to cert>
				enable_haproxy: True
				kolla_internal_vip_address: <internal VIP>
				kolla_external_vip_address: <external VIP>

# 5. Performance and scaling impact
There is no known documentation, but apparently some study was done about 
performance hits when enabling TLS natively in the VNC API. This needs to 
be undertaken once again if we aim to support TLS natively and documented.

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

Individual apps will be brought up and tested for ssl support on 
corresponding API endpoints. Some negative test cases will also be 
added to make sure that page access fails if wrong certificate is 
specified by the client.


# 10. Documentation Impact

Documentation about any new configuration knobs to enable/disable 
TLS need to be added.

# 11. References
