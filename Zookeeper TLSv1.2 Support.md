
# 1. Introduction
Design and Implement TLSv1.2 Support for Zookeeper.


# 2. Problem statement
The requests from contrail services  to Zookeeper Server are not encrypted and can pose a
security request.

# 3. Proposed solution
Following is the list of contrail services which are client of Zookeeper.

> contrail-analytics-api
> contrail-alarm-gen
> contrail-collector
> Kafka
> kube-manager
> svcmonitor
> config-api
> config-schema
> config-devicemgr
> vcenter

We have to use SSL to encrypt the interaction between above services and Zookeeper.

Here are various steps of this proposal

1. Modify contrail process conf file to include zookeeper ssl related parameters and use them when initiating zookeeper requests.
2. Modify zookeeper provisioning script to create ssl keystore and truststore, and use them zookeeper config file.
3. Modify kafka's zookeeper client properties to use ssl.

## 3.1 Alternatives considered
None

## 3.2 API schema changes
Not Applicable

## 3.3 User workflow impact
Contrail provisioning has to be changed to include Zookeeper SSL parameters. 

## 3.4 UI changes
None

## 3.5 Notification impact
None


# 4. Implementation
## 4.1 Work items
Changes are required in many modules. Here is the breakup of work for different modules.

### 4.1.1 Zookeeper Server

Upgrade to latest available zookeeper >= 3.5.1 which has SSL support.

SSL is only supported on top of Netty communication, which means if we want to use SSL we have to enable Netty by setting this java system property -

`-Dzookeeper.serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory"`

We also need to create keystore and truststore to store ssl private keys and certificates.

`keytool -keystore zookeeper.server.truststore.jks \ 
        -keypass {ZOOKEEPER_KEY_PASSWORD} -storepass {ZOOKEEPER_STORE_PASSWORD} \
        -noprompt \
        -alias CARoot -import -file ca-cert.pem
openssl pkcs12 -export -in server.pem \
        -inkey server-privkey.pem \
        -chain -CAfile ca-cert.pem \
        -password pass:{ZOOKEEPER_KEY_PASSWORD} -name localhost -out TmpFile
keytool -importkeystore -deststorepass {ZOOKEEPER_KEY_PASSWORD} \
        -destkeystore zookeeper.server.keystore.jks \
        -srcstorepass {ZOOKEEPER_STORE_PASSWORD} -srckeystore TmpFile \
        -srcstoretype PKCS12 -alias localhost
keytool -keystore zookeeper.server.keystore.jks \
        -keypass {ZOOKEEPER_KEY_PASSWORD} -storepass ${ZOOKEEPER_STORE_PASSWORD} \
        -noprompt \
        -alias CARoot -import -file ca-cert.pem`

### 4.1.2 Kafka Server

Following needs to be done to enable SSL between kafka and zookeeper - 

1. Upgrade kafka to 2.11-2.4.0. It has Zookeeper 3.5.5 jar files which has TLS support ([KAFKA-8634](https://issues.apache.org/jira/browse/KAFKA-8634))

2. Append following settings in KAFKA_JVM_OPTS

   `-Dzookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty    
    -Dzookeeper.client.secure=true    
    -Dzookeeper.ssl.keyStore.location=zookeeper.server.keystore.jks    
    -Dzookeeper.ssl.keyStore.password=c0ntrail123    
    -Dzookeeper.ssl.trustStore.location=zookeeper.server.truststore.jks    
    -Dzookeeper.ssl.trustStore.password=c0ntrail123`

### 4.1.3 Zookeeper Python Client Library

Upgrade python-kafka library rpm to 2.6.0 or above to support TLS

### 4.1.4 Zookeeper C++ Client Library

Currently we use libzookeeper and libzookeeper-devel rpm to install Zookeeper C++ client bindings. But latest available version of these rpms don't support TLS.

So, we have to download the latest **[zookeeper](https://github.com/apache/zookeeper)** code  and compile c++ bindings ourselves using this link https://zookeeper.apache.org/doc/r3.5.6/zookeeperProgrammers.html#Installation 

### 4.1.5 Provisioning Changes

Addition of following Zookeeper TLS parameters in docker startup scripts.

> zookeeper_ssl_enable = True
> zookeeper_keyfile = /etc/contrail/ssl/private/server-privkey.pem
> zookeeper_certfile = /etc/contrail/ssl/certs/server.pem
> zookeeper_ca_cert=/etc/contrail/ssl/certs/ca-cert.pem


# 5. Performance and scaling impact
Considering that zookeeper is accessed only during process startup or restarts, using SSL
for the zookeeper communication should not cause performance impact.

## 5.1 API and control plane
None

## 5.2 Forwarding performance
None

# 6. Upgrade
One by One upgrade of analytics/config nodes from an older release (which doesn't support Zookeeper TLS) to this release will not work.

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
1. Check relevant SSL configuration options are parsed correctly.
2. Check zookeeper requests with SSL options are invoked and work correctly
3. Negative tests

## 9.2 Dev tests
1. Check provisioning updates the configuration files
2. Check zookeeper communication works with and without SSL being enabled.

## 9.3 System tests



# 10. Documentation Impact


# 11. References

