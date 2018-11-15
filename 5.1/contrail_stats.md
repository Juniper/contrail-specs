Contrail Stats to External Collector
====================================

# 1. Introduction
Currently sandesh generator sends the stats information in UVE and Objectlogs.
Fields that are tagged become a stat field. The sandesh clients send the stats
message in XML format and it can only be consumed by the contrail-collector. 

# 2. Problem statement
There are many Open Source collectors which can consume the stats messages and
provide analytics on the message. However many of them ingest this information
as JSON since it is less verbose.

# 3. Proposed solution
We propose changes to the sandesh generators which optionally send the UVE
messages in JSON. User can provide the external endpoint, which can be an
ip-address:port in case of a centralized collector or unix domain socket file in
settings with distributed agents residing in the same nodes as the generators.
In both the cases, data will be sent in datagram format.

## 3.1 Alternatives considered
The binary encoding requires message schema to be available at the collector to
decode and XML is verbose, so JSON is a good middle ground and a standard format
for many collectors.

## 3.2 API schema changes
##### None

## 3.3 User workflow impact
This change does not impact existing contrail stats collection. Only if
users specify the external endpoint will the library start sending the stats to
it, and only in addition to (not instead of) the existing stats collection.

## 3.4 UI changes
##### NA

## 3.5 Notification impact
Upon providing the external endpoint, the generators will be sending the message
to it in JSON format in addition to sending to collector in XML format.

# 4. Implementation
## 4.1 Work items
### 4.1.1 Sandesh Library changes
To achieve the JSON encoding, a new parser that implements the
sandesh/protocol/TTransport.h APIs and generates the JSON has to be provided.
The generated Sandesh files (CPP/Python) will call these APIs like the way they
are currently doing for XML and the JSON parser should in turn implement those
APIs to generate equivalent JSON messages. Fields that are currently encoded in
the XML field but not useful are discarded. E.g. fields like the datatype of
each field are not necessary. Also a new stats client has to be added which
opens connection with the external endpoint and sends only the JSON encoded
message to it. The following is an example of a sample sandesh UVE message and
the corresponding JSON generated:
```
struct VrouterFlowRate {
    1: u32 added_flows
}

struct VMIStats {
    1: string name (key="ObjectVMITable")
    2: VrouterFlowRate flow_rate (tags="")
}

uve UVEVMIStats {
    1: VMIStats data
}
```
Corresponding JSON:
```
      "UVEVMIStats": {
        "data": {
          "TYPE": "struct",
          "VAL": {
            "STAT_TYPE": "VMIStats",
            "name": {
              "TYPE": "string",
              "ANNOTATION": {
                "key": "ObjectVMITable"
              },
              "VAL": "default-global-system-config:b7s9:vhost0"
            },
            "flow_rate": {
              "TYPE": "struct",
              "ANNOTATION": {
                "tags": ""
              },
              "VAL": {
                "INSTANCE": "VrouterFlowRate",
                "VAL": {
                  "added_flows": {
                    "TYPE": "u32",
                    "VAL": 0
        } } } } } }
```
### 4.1.2 Provisioning changes
All contrail generators will be required to have a config option to specify
external collector endpoint.

####

# 5. Performance and scaling impact
## None
#### None

## 5.2 Forwarding performance
#### None

# 6. Upgrade
#### None
#### None

# 7. Deprecations
#### None

# 8. Dependencies
#### None

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References
