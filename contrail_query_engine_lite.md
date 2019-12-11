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
2. Modify Redis UVE update mechanism to store n instances of a
   particular UVE so that it has some history of that UVE. Number n
   will be configurable, coming from conf file in first cut and later
   from config.
3. Analytics-api will also be responsible for handling stats table
   queries if 'n' is non zero in configuration. Instead of sending
   stats queries to query engine, analytics api should query uveserver
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
Contrail stats will be available for last n instances only. In addition,
objectlogs and systemlogs will not be available with Query Engine Lite.

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
+redis.call('ltrim',_values..':'..attr,0,n)
+redis.call('ltrim',_values..':'..'__T',0,n)
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

## 4.4 UI changes
UI will also require changes to suppress objectlogs and systemlogs
features in case of Cassandra not being installed.

# 5. Performance and scaling impact
It should be noted that Query Engine Lite will not work for scale setups
since Redis is in memory database and its size will increase
exponentially with scale.

# 6. Upgrade
During the upgrade, query engine version can not be changed so default
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
