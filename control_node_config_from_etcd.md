# 1. Introduction
Today, the control-node gets the user configuration from the configuration database (i.e., cassandra) and sends the relevant config data to all the compute nodes that have peered with it (via XMPP). Control-node also consumes some config data (e.g., BGP configuration). With the introduction of ATOM (a new common infra for applications such as contrail), Cassandra will be replaced by MySQL and RabbitMQ, with ETCD. As a result, we need to change the control-node config handling infrastructure to add support for retrieving user configuration from MySQL as well using ETCD.


# 2. Problem statement
Post 4.1 release and until today, the control-node retrieves user configuration by reading the cassandra client database. This is done whenever control-node boots up or restarts. In addition, it subscribes as a RabbitMQ client to listen to any changes (Add/Delete/Update) to the configuration data. As mentioned earlier, with the introduction of a common infra (ATOM) for controllers, contrail will now need the ability to use the new infra. The new infrastructure uses MySQL instead of Cassandra as its database. Also, the RabbitMQ client has been replaced with ETCD which is known to work well with MySQL based systems. ETCD acts as a one-stop shop for listening and retrieving all configuration related changes as opposed to today where we have to subscribe to RabbitMQ to listen for updates while reading bulk user configuration directly from Cassandra.
Accordingly, we will enhance the current control-node behavior to add support for retrieval of user configuration from MySQL as well using ETCD. Existing and new users will have the ability to choose one behavior over the other via a config knob in the configuration file. We will continue to support the cassandra based config handling infrastructure for existing users.

# 3. Proposed solution
We will be adding a new c++ based ETCD client library to retrieve new user-configuration and updates from the golang-based ETCD server. There are two ways in which ETCD clients can communicate with the server to get the data: HTTP based Curl and GRPC. With the proposed solution, we will be using GRPC to communicate with the server. In addition, we will be making changes to existing modules that handle configuration data to add support for retrieving and handling the data from ETCD. There will also be a new knob introduced in the configuration file to allow users to choose between the two.

We do not expect any change to schema, workflow or UI. The new logs for the new modules are TBD.

## 3.1 Alternatives considered
N/A

## 3.2 API schema changes
None

## 3.3 User workflow impact
Users will have the ability to choose either a Cassandra based system or an ETCD based MySQL system via a config knob for handling the user configuration. Other than that there is no workflow impact for the users in terms of the functionality.

## 3.4 UI changes
None

## 3.5 Notification impact
New logs and appropriate UVE changes will be introduced to support the new ETCD based config retrieval feature.


# 4. Implementation
## 4.1 Work items
There will be one new module:

a. c++ based ETCD-Client-Library This module is responsible for interacting with the Golang based ETCD server which retrieves and maintains the user configuration from the MySQL database in JSON format. Support will be added to READ user configuration as well as WATCH for any changes (Add/Delete/Update) to individual configuration items. As mentioned earlier, GRPC will be the communication mechanism used between the ETCD client and server. We will be using the standard proto files available with GRPC.

In addition, the following existing modules will be enhanced to support the new infrastructure:

a. Config-Client-Manager This module oversees the retrieval of user configuration. It currently interacts with the following modules to retrieve and handle user configuration.

(i) Config-Cassandra-Client This module interacts with the cassandra servers. It also performs connection-management and issues read requests actions for the cassandra database holding the user configuration.

(ii) RabbitMQ-Client This module interacts with the RabbitMQ servers. Its function is to retrieve information about the operation (Add/Delete/Change) and the configuration item on which the operation will apply.

(iii) Config-Cassandra-Parser This module parses the configuration received by the cassandra client from the cassandra server.

With the proposed changes, the Config-Client-Manager will also interact with 

(iv) Etcd-Cassandra-Client This module will combines the functionality of both Config-Cassandra-Client and the RabbitMQ-Client. It will interact with the ETCD client library to both retrieve bulk configuration (when control-node comes up or restarts) and watch for any changes (Add/Delete/Update) to get information about the operation and the configuration item on which the operation will apply.

In addition, the Config-Client-Manager will have the ability to choose one infra over the other (Cassandra vs MySQL) based on a config knob in the configuration file.

We will add tests for all the new functionality, change existing tests to work with the new modules, add introspect functionality to peak into the new modules and also add/change introspect functionality that interacts with the new modules.

# 5. Performance and scaling impact
TBD

# 6. Upgrade

The changes are only in the control-node. There should be no operational impact when we replace all control-nodes. In the case, where only a subset of control-nodes are upgraded, the data stored in the older-version and the newer-version control-nodes will be the same.

# 7. Deprecations
None

# 8. Dependencies
None

# 9. Testing
## 9.1 Unit tests
The current automation suite will be enhanced to add tests for the new ETCD based config retrieval system. Efforts will be made to make sure that the test coverage will be on par with the current tests for Cassandra based systems.

## 9.2 Dev tests
TBD

## 9.3 System tests
TBD

# 10. Documentation Impact
TBD

# 11. References
None
