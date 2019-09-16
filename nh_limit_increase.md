
# 1. Introduction

# 2. Problem statement

Jira story: https://contrail-jws.atlassian.net/browse/CEM-6025

Contrail today supports a maximum of 64k nexthops in both vrouter and agent.
But there is no validation in agent which prevents the addition of new 
nexthops once this limit is hit. Although nexthop addition in vrouter fails
once the limit is hit, agent still continues to think the nexthop is valid.
This leads to traffic blackholing as was seen in one of the customer
deployments. 
Also, there are no alarms sent when the utilization is sufficiently high.
There is also a genuine reason to increase this nexthop limit due to design
limitation of ECMP nexthops where a set of ECMP child nexthops are allocated
per route instead of per peer.

This document captures the design and changes required to increase this limit
to 1M nexthops (theoritical max limit 2^32-1) and also raise alarms when the
high & max limits are reached.



# 3. Proposed solution

Increase the nexthop limit in agent as well as vrouter along with validation
checks. Generate alarms when the high and max limits are
reached for nexthops and mpls labels.

# 4. Alternatives considered
None

# 5. API schema changes
None

# 6. UI changes
None

# 7. Notification impact

The following new system defined alarms will be added as part of this.

Alarms generated for nexthop and mpls labels usage:

- When usage exceeds a configurable high watermark limit - alarm
  raised with severity level major.

- When usage equals 100% - alarm raised with severity level critical.
  High watermark alarm stays.

- When usage becomes 5% less than max 100% limit - critical alarm
  is cleared. But the high watermark alarm stays.

- When usage becomes 5% less than the configured high watermark limit - high 
  watermark alarm is cleared.
 

# 8. Provisioning changes

There are no provisioning changes required for this feature.
The high alarm limit will be configurable in contrail-vrouter-agent.conf file.
This limit will be same for both nexthops and mpls labels.

# 9. Implementation

## 9.1 Agent

The following changes would be required in agent.

1. Fetch the resource max limits from vrouter during init before config
   is read from the controller.

2. Enforce the max limits for nexthop creation. Fail the transaction if
   nexthop limit is exceeded. In case of failure, the route/mpls label should
   point to discard NH.

3. Read the alarm limit from vrouter agent config file.
   Eg:
   vr_high_watermark = x (Default value is 80).

4. Generate alarms by monitoring the nh and mpls label resource usage against
   the threshold configured. A new struct would be added in VRouterAgent UVE for sending the resource usage.
   Also clear the alarm when the usage goes below (x-5%).

5. Add alarm generation rules in contrail_alarm.py file.

## 9.2 Vrouter

The following changes would be required in vrouter.

1. Change nexthop size from 16 bits to 32 bits.
   Majorly the vr_flow_entry uses src_nh_id as 16 bits.
   Since vr_flow_entry is exactly cache line and 4 MB page size aligned,
   the size of vr_flow_entry struct would be increased to 256 bytes from
   the existing 128 bytes.

2. Other changes where nexthop size is assumed to be 16 bits. These changes
   are mainly in vif and gro structs and flow processing code.

3. Set the default max number of nexthops supported to 512K.

# 10. Performance and scaling impact

1. Although the upper limit for max number of nexthops supported could be
   2^32-1, pratically we would support upto 1M nexthops.

2. Since the flow struct size is increased from 128 bytes to 256 bytes,
   the memory usage for flows would double.

3. There should not be any impact on pps performance. NOTE: This has been
   validated through a private image with vrouter changes.

# 11. Upgrade
No impact

# 12. Deprecations
None

# 13. Dependencies
None

# 14. Security Considerations
There are no security aspects to this change.

# 15. Testing

## 15.1 Agent unit tests

1. Add unit test to read high watermark from contrail-vrouter-agent.conf and
   verify its set correctly in agent. Also verify low watermark in agent is set
   to highwatermark-5 ( this value is in percentage), this is calculated in
   agent.

2. Add unit test to cover cases when watermark is incorrectly configured in
   agent.conf, verify correct values are set in such cases, also verify floating point value is accepted for watermarks. check for cases when
   highwater mark is set greater than 100 (log message use default in such
   case) and calculated low watermark should be greater than 0.

3. Add unit test to verify if high watermark is not set in agent.conf default
   value is used in agent (high watermark=80), we will calculate low water mark
   in agent which should be 75 in this case.

4. Add unit test to configure nexthop limit as 32 add configuration to create
   nexthops, check that VrouterAgent Uve res_table_limit flag is set when
   nexthop count reaches 32, also res_limit should be set, verify new nexthop
   addition fails, check res_table_limit is unset on nexthop deletion when
   nexthop limit count is less than 95 percentage of configured limit.

5. Add unit test to configure nexthop limit as 32, configure watermark to non
   default value add nexthops, when nexthop count becomes greater than or equal
   to 32*high_watermark/100 VrouterAgent res_limit flag is set, delete nexthops
   and check that res_limit flag is unset only when nexthop count is less than
   32*low_watermark/100. calculated low watermark is (high_watermark-5) percentage.

6. Add unit test to configure mpls label limit to 32 add mpls labels, check
   that VrouterAgent uve res_table_limit flag is set when label count equals
   32, also res_limit should be set, verify new label addition fails, check 
   that res_table_limit  is unset on mpls label deletion when label count
   becomes less than 95 percentage of configured limit.

7. Add unit test to configure mpls label limit as 32, configure watermark to
   non default value add labels , when label count becomes greater than or equal to 32*high_watermark/100 VrouterAgent res_limit flag is set, delete
   labels and check that res_limit flag is unset only when label count is less
   than calculated low watermark percentage of configured mpls label limit.

8. Add unit test to configure nexthop count to 32 and non default watermark
   verify that when nexthop count is less than 32*highwatermark/100 
   VrouterAgent uve flags res_limit and res_table_limit is not set.

9. Add unit test to configure mpls label count to 32 and non default watermark,
   verify that when lablel count is less than 32*high_watermark/100 VrouterAgent uve flags res_limit and res_table_limit is not set.



## 15.2 Vrouter unit tests

1. Verify that vrouter comes up properly with default 512K max nexthops and
   vrouter --info shows the correct values.

2. Verity that vrouter comes up properly when max limit is set to 1M nexthops
   and vrouter --info shows the correct values.

3. Verity vif interface creation is successful with 32 bit nh id

4. Verify fabric to VM interface traffic is gro'ed correctly.

5. Verify flow creation and deletion is successful with nexthop id < 64k

6. Verify flow creation and deletion is successful with nexthop id > 64k 

7. Verify nexthop creation fails when the configured NH limit is hit.


## 15.3 System tests

1. Verify we are able to scale to 1M nexthops system wide and there is no 
   traffic blackholing or traffic loss.

2. Verify alarm is generated in UI page with severity major when high
   watermark is hit for nexthop usage.

3. Verify alarm is generated in UI page with severity critical when 100% usage
   is hit for nexthop usage.

4. Verify critical alarm is cleared in UI page when nexthop usage goes to 95%. 
   The high watermark alarm should still be there.

5. Verify alarm is cleared in UI page when nexthop usage goes below 5% of high
   watermark.

6. Verify alarm is generated in UI page with severity major when high
   watermark is hit for mpls labels usage.

7. Verify alarm is generated in UI page with severity critical when 100% usage
   is hit for mpls labels usage.

8. Verify critical alarm is cleared in UI page when usage goes to 95%.
   The high watermark alarm should still be there.

9. Verify alarm is cleared in UI page when usage goes below 5% of high
   watermark.

10. Verify that after the nexthop limit is exceeded, the failure is handled
   gracefully.


# 16. Documentation Impact
The new alarms and config settings for vrouter and agent needs to be documented.


# 17. References

https://contrail-jws.atlassian.net/browse/CEM-6025


