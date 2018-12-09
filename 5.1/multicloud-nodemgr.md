# 1. Introduction
Contrail Multicloud is a Contrail feature which enable the user to deploy Contrail across any number of public and/or private clouds. Currently, AWS, GCP and Azure as well as private, on-premise clouds are supported.

Contrail MultiCloud works by creating an underlying network, of which MultiCloud gateways are a crucial part. To provide this network, gateways employ services in form of:
* BGP Bird
* VRRP via Keepalived
* OpenVPN
* StrongSWAN

# 2. Problem statement
After deployment of Contrail MultiCloud the only way to monitor the status of underlying network is to manually SSH onto the machines and check Docker container logs.

At the moment, there is no system which would gather and/or aggregate Multicloud statistics.

The possible data worth monitoring include:
#### Drop/Error rates
Such data can indicate an issue with connection. Error and drop rates should remain as low as possible
#### Packets transmitted/received
This info provides a good overview on traffic. It can be used to detect time of increased activity across clouds.
#### Tunnel status
If a tunnel is broken, direct connection is not possible and has to be done via a hop. This leads to a multiplication of effective traffic transmitted.
#### HA status
If one of gateways go down, the user should be notified of this immediately even if the traffic and connection are effective uninterrupted (For example, in active/active model)
#### Container status
Containers can fail to start for a number of reasons (ex. missing kernel module for IPSec). This should be notified to the user.

# 3. Proposed solution
The proposed solution is to include a new nodemgr module - MultiCloud.

Using standard nodemgr design, it would send relevant UVEs to the analytics server via Sandesh.

## 3.1 Alternatives considered
#### N/A

## 3.2 API schema changes
This proposal includes a few new UVEs. The following Sandesh code snippet provide an exhaustive list of these UVEs.

```
/**
 * Structure definition for holding Interface object details as per Multicloud
 */
struct UveInterfaceMulticloud {
    1: string           name;
    2: i64              bytes_tx;
    3: i64              bytes_rx;
    4: i64              packets_tx;
    5: i64              packets_rx;
}

/**
 * Structure definition for holding Tunnel object details as per Multicloud
 */
struct UveTunnelMulticloud {
    1: string           peer;
    2: bool             is_up;
    3: optional string  since;
}

/**
 * Structure definition for holding HA Status as per Multicloud
 */
struct UveHAMulticloud {
    1: string           network;
    2: bool             is_master;
}

/**
 * Structure definition for holding BGP peering details as per Multicloud
 */
struct UveBirdMulticloud {
    1: string           peer;
    2: bool             is_up;
}

/**
 * Structure definition for delivering route updates that happened since last log as per Multicloud
 */
struct UveRouteUpdate {
    1: i64              timestamp;
    2: string           op;
    3: string           target;
    4: string           iface
    5: optional string  via;
}

/**
 * Structure definition for holding Multicloud object details
 */
struct UveMulticloud {
    /** Configured Name of virtual-machine */
    1: string                name (key="ObjectMCTable")
    /** Value 'true' indicates entry is removed. Holds value of 'false' otherwise*/
    2: optional bool         deleted
    /** List of Interface entries */
    3: optional list<UveInterfaceMulticloud> interface_list; 
    /** List of Tunnel entries */
    4: optional list<UveTunnelMulticloud> tunnel_list;
    /** List of HA networks managed by the gateway */
    5: optional list<UveHAMulticloud> ha_list;
    /** List of Bird peering status of the gateway */
    6: optional list<UveBirdMulticloud> bird_list;
    /** List of route updates on the gateway */
    7: optional list<UveRouteUpdate> route_updates_list;
}

/**
 * @description: Uve for multicloud object
 * @type: uve
 * @object: multicloud
 */
uve sandesh UveMulticloudTrace {
    /** virtual-machine information */
    1: UveMulticloud data;
}
```

## 3.3 User workflow impact
The existing workflow would remain the same.

After integration into UI, user would see current Multicloud status.

## 3.4 UI changes
#### N/A

## 3.5 Notification impact
#### N/A

# 4. Implementation
This implementation mainly uses Python code to extend Contrail NodeMgr. However, there is also an additional UVE and additional Python library requirement.
## 4.1. Schema
The additional UVE is presented in Chapter 3.2.

Additonally, necessary changes were made do `vns.sandesh`, to ensure correct implementation of new module type, "MultiCloud Gateway".
## 4.2. NodeManager mode
If node manager is launched with `multicloud` as nodemgr type, new features will be used. Other modes of operations are not changed.

Please note OS other than CentOS / RedHat / Ubuntu are not supported as MultiCloud gateway at the present time.
## 4.3. Tunnels
There are two types of Tunnels currently supported by Contrail MultiCloud. StrongSWAN and OpenVPN.

Both are represented the same way in UVEs, the only change is the method of data aquisition.

A daemon is periodically checking for the Tunnel status and reports it back in the UVE.
### StrongSWAN
StrongSWAN is checked periodically via issuing a command into its Docker container.

The gathered data is a list of tuples (`peer_ip`, `is_up`, `since`).
### OpenVPN
OpenVPN status is checked by investigating network interfaces data according to the Multicloud configuration files.

The gathered data is a list of tuples (`peer_ip`, `is_up`).

## 4.4. Bird
BGP Bird is monitored by issuing commands into `birdc` inside the Docker container.

The manager investigated BFD sessions and returns a list of tuples (`peer_ip`, `is_up`)

## 4.5. VRRP
VRRP is investigating addresses on loopback network interface to check current status of the gateway. Such an address can indicate the gateway is an active gateway in a network, according to the Multicloud configuration files.

It returns a list of tuples of (`network`, `master_status`).

## 4.6 Route updates
The manager is monitoring for route updates and sends a list of new ones on every UVE update. Both insertion and deletion of routes are included, as well as the instructions.

Returned data is a list of tuples (`timestamp`, `operation`, `target`, `via`, `interface`)

# 5. Performance and scaling impact
## 5.1 API and control plane
#### N/A

## 5.2 Forwarding performance
#### N/A

# 6. Upgrade
#### Backward compatible changes

# 7. Deprecations
#### N/A

# 8. Dependencies
This nodemgr uses Python libraries not used in nodemgr before. 

```
Requires:        python2-pyroute2 >= 0.4.8
```

This is because MultiCloud nodemgr monitors routes.

# 9. Testing
## 9.1 Unit tests
This system was written mostly used TDD, which means there are unit tests testing basic features of many usecases. Since the module interfaces heavily with outside environment, not every possible case is tested.

## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact
UVE schema should be documented, since it can be useful for developers responsible for displaying Analytics data to the Contrail user.

# 11. References
