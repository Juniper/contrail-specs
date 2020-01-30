# 1. Introduction
During ZTP, if multiple links are present between leaf and spine, there is a need to bundle 
all of those interfaces together as an ae interface, create just one Logical Router and assign it an IP from the fabric subnet. 

# 2. Problem statement
Currently during ZTP, particularly during topology discovery, when  multiple links are present
between leaf and spine, each link is treated seperately and logical routers are created for each of those 
interfaces which should not be the case. 

# 3. Proposed solution
CEM will now discover multiple links existing between leaf and spine and AUTOMATICALLY (without requiring any input from the user) configure them as Aggregated Ethernet interfaces rather than individual interfaces. 
IP addresses assigned to the leaf-spine links are assigned to the AE interface and NOT to the individual links.

# 4. Implementation

1. During ZTP, particularly after all the interfaces have been indentified using lldp, leafs or spines 
   having multiple links across them will be treated differently. 
2. Instead of treating each link individually, an ae bundle willbe created. A particular index for this 
   ae bundle will be fixed. One Logical Router will be created and assigned an IP from the specified fabric subnet. 
3. This will avoid having same multiple bgp instances between a leaf and a spine. 

# 5. API schema changes
No changes 

# 7. UI Changes - ( will update this once the screens are ready)
No changes 

# 8. Testing

# 9. Documentation Impact
