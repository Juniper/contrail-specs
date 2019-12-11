# 1. Introduction
Design and implement less resource intensive query engine.

# 2. Problem statement
In current contrail installations, contrail analytics is very resource
intensive. A lot of customers want analytics to be less resource
intensive specially when they don't want all the features provided by
it. Out of all the components, query engine and cassandra were found to
be taking most of the resources. Hence the need to develop Query Engine
Lite.

# 3. Proposed solution
Here are various steps of proposal:
1. Convert those ObjectLogs which are required for basic functionality
   of Contrail to UVEs.
2. Modify Redis UVE update mechanism to store 'stats_history_count'
   instances of all UVEs instead of just 1. Number 'stats_history_count'
   will be configurable, coming from conf file in first cut and later
   from config.
3. Analytics-api will also be responsible for handling stats table
   queries if 'stats_history_count' is non zero in configuration. Instead of sending
   stats queries to query engine, analytics api will query uveserver
   similar to UVE queries. Uveserver, in turn, will get relavant data
   from Redis and send back to analytics-api.
4. It will be optional to choose redis only database at the time of
   provisioning.

## 3.1 Alternatives considered
One other alternative was to implement a new query engine that would
serve out of Redis instead of Cassandra. However, it would require yet
another application to sync with Redis where Analytics-Api is already
doing the same while serving UVEs. Also, it would be little tricky
to switch from Redis mode to Cassandra mode (and vice versa) in case
of a new container for Query Engine Lite.

## 3.2 User workflow impact
Contrail stats will be available for last 'stats_history_count' instances only.
In addition, objectlogs and systemlogs will not be available with Query Engine Lite.

# 4. Implementation
Changes are required in many modules. Here is the breakup of work for
different modules.

## 4.1 Component changes
All the data that is required for basic functionality of contrail need
to be converted to UVE. At this point, only JobManager and Contrail
Security related ObjectLogs and SessionLogs have been identified as the
ones which need to be changed. Those logs are:
```
    EndpointSecurityStats
    SessionEndpointObject
    JobLog
    PRouterOnboardingLog
```
It should be noted that just changing the data type in sandesh file is
enough for this purpose. Here is an example for the same:

```
diff --git a/src/vnsw/agent/uve/interface.sandesh
b/src/vnsw/agent/uve/interface.sandesh
index 65a3067..ca8574c 100644
--- a/src/vnsw/agent/uve/interface.sandesh
+++ b/src/vnsw/agent/uve/interface.sandesh
@@ -271,7 +271,7 @@ struct EndpointStats {
  * @type: objectlog
  * @object: virtual-machine-interface
  */
-objectlog sandesh EndpointSecurityStats {
+uve sandesh EndpointSecurityStats {
     /** Name of Virtual Machine Interface */
     1: string name (key="ObjectVMITable");
```

## 4.1 Redis Changes
Redis will have to store history of UVEs rather than just storing
current value. This will require changes in lua script which writes
to redis:

```
diff --git a/contrail-collector/uveupdate.lua
b/contrail-collector/uveupdate.lua
index 662edd5..592d023 100644
--- a/contrail-collector/uveupdate.lua
+++ b/contrail-collector/uveupdate.lua
@@ -46,6 +46,11 @@ redis.call('sadd',_table,key..':'..sm..":"..typ)
 redis.call('zadd',_uves,seq,key)
 redis.call('hset',_values,attr,val)
 redis.call('hset',_values,'__T',ts_string)
+redis.call('lpush',_values..':'..attr,val)
+redis.call('lrem',_values..':'..'__T',0,ts_string)
+redis.call('lpush',_values..':'..'__T',ts_string)
+redis.call('ltrim',_values..':'..attr,0,stats_history_count)
+redis.call('ltrim',_values..':'..'__T',0,stats_history_count)
```

## 4.2 Provisioning Changes
An optional new parameter will be added in analytics_api.conf to
indicate how long the history of UVEs should be kept in Redis. This
should be populated only if Cassandra and Query Engine containers are
not installed. Any value greater than one will result in more memory
usage for Redis so should be avoided if Cassandra is installed.

## 4.3 Analytics Api Changes
If Cassandra and Query Engine are not installed, stats queries should
be served by Analytics-Api. When a query for stats tables is made, it
will query redis for that table and get corresponding xml data as output.
This xml data should be parsed by Analytics-Api to present in the format
similar to the way it is presented today by Query Engine. This will be
done to make sure that user experience is same. This will require xml
parser to be implemented in Analytics-Api. Also, if query is for a specific
duration then analytics api will need to get timestamps of different
writes of corresponding table which are stored in a set. If the
duration falls beyond the time mentioned in timestamp then no result
is returned.

Here is an example of how stats table will look like in redis. Consider
the table 'StatTable.CollectorDbStats.cql_stats'. This table comes from
UVE CollectorDbStatsTrace. It is defined as following in sandesh file:
```
struct CollectorDbStats {
    5: optional cql.DbStats                    cql_stats (tags="")
    6: optional SessionTableDbInfo             session_table_stats (tags="")
}
uve sandesh CollectorDbStatsTrace {
    1: CollectorDbStats data
}
```
Collector stores latest value of UVE CollectorDbStatsTrace in Redis.
With the proposed changes, it will also store the name of table as
'"StatTable:ObjectCollectorInfo:node1.local:node1.local:Analytics:
contrail-collector:0:CollectorDbStats:errors"' for a collector instance.
Name of the source and module is also included in the table name so that
every module will have a separate table. In addition, there will
be corresponding data in xml format which can be retrieved:
```
lrange "StatTable:ObjectCollectorInfo:node1.local:node1.local:Analytics:
        contrail-collector:0:CollectorDbStats:cql_stats" 0 -1
 1) "<cql_stats type=\"struct\" tags=\"\"><DbStats><requests_one_minute_rate
type=\"double\">13.4266</requests_one_minute_rate><stats type=\"struct\"
tags=\"\"><ClusterStats><total_connections
type=\"u64\">4</total_connections><available_connections
type=\"u64\">4</available_connections><exceeded_pending_requests_water_mark
type=\"u64\">0</exceeded_pending_requests_water_mark><exceeded_write_bytes_water_mark
type=\"u64\">0</exceeded_write_bytes_water_mark></ClusterStats></stats><errors
type=\"struct\" tags=\"\"><ClusterErrors><connection_timeouts
type=\"u64\">0</connection_timeouts><pending_request_timeouts
type=\"u64\">0</pending_request_timeouts><request_timeouts
type=\"u64\">0</request_timeouts></ClusterErrors></errors></DbStats></cql_stats>"
    .
    .
    .
    .
    .
    .
10) "<cql_stats type=\"struct\" tags=\"\"><DbStats><requests_one_minute_rate
type=\"double\">13.6303</requests_one_minute_rate><stats type=\"struct\"
tags=\"\"><ClusterStats><total_connections
type=\"u64\">4</total_connections><available_connections
type=\"u64\">4</available_connections><exceeded_pending_requests_water_mark
type=\"u64\">0</exceeded_pending_requests_water_mark><exceeded_write_bytes_water_mark
type=\"u64\">0</exceeded_write_bytes_water_mark></ClusterStats></stats><errors
type=\"struct\" tags=\"\"><ClusterErrors><connection_timeouts
type=\"u64\">0</connection_timeouts><pending_request_timeouts
type=\"u64\">0</pending_request_timeouts><request_timeouts
type=\"u64\">0</request_timeouts></ClusterErrors></errors></DbStats></cql_stats>"
```

When a query is made for table 'StatTable.CollectorDbStats.cql_stats',
analytics-api will search for all the tables in redis with name
'StatTable*CollectorDbStats.cql_stats'. Data is collated from all these
tables and sent out as response.

## 4.4 UI changes
UI will also require changes to suppress objectlogs and systemlogs
features in case of Cassandra not being installed.

# 5. Performance and scaling impact
It should be noted that Query Engine Lite will not work for scale setups
since Redis is an in memory database and its size will increase
exponentially with scale.

# 6. Upgrade
During the upgrade, database installation can not be changed so default
container will be installed if configured already.

# 7. Deprecations
N/A

# 8. Dependencies

## 8.1

# 9. Testing
## 9.1 Unit tests
Unit tests will be added to make sure that stats and uves can be
fetched using Query Engine Lite.

## 9.2 Dev tests

## 9.3 System tests

# 10. Documentation Impact

# 11. References
