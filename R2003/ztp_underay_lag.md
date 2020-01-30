# 1. Introduction
During ZTP, if multiple links are present between leaf and spine, there is a need to bundle 
all of those interfaces together as an ae interface, create just one Logical Interface and assign it an IP from the fabric subnet. 
Underlay BGP will run on the AE interface instead of individual member links.

# 2. Problem statement
Currently during ZTP, particularly during topology discovery, when  multiple links are present
between leaf and spine, each link is treated seperately and logical interfaces are created for each of those 
interfaces which should not be the case.

# 3. Proposed solution
CEM will now discover multiple links existing between leaf and spine and AUTOMATICALLY (without requiring any input from the user) configure them as Aggregated Ethernet interfaces rather than individual interfaces.
IP addresses assigned to the leaf-spine links are assigned to the AE interface and NOT to the individual links.
Underlay BGP will run on the AE interface instead of individual member links.

# 4. Implementation

1. During ZTP, particularly after all the interfaces have been indentified using lldp, leafs or spines 
   having multiple links across them will be treated differently.
2. Instead of treating each link individually, an ae bundle will be created. A particular index for this 
   ae bundle will be fixed. One Logical Interface will be created and assigned an IP from the specified fabric subnet.
3. This will avoid having same multiple bgp instances between a leaf and a spine. After the implementation, a single
   BGP intance will run on the AE interface.
4. Specific configurations such as minimun-link will be set for the AE interface between Leaf/Spine and the value of the 
   minimum-link will be set to the # of links between the two neighbouring switches.
5. By default LACP will be enabled with the minimum links setting with an option to turn it off during ZTP.
6. User will have to specify the underlay MTU for both the ae interface as well as all the links.

# 5. API schema changes
No changes.

# 7. UI Changes - ( will update this once the screens are ready)
No changes.

# 8. Testing

# 9. Documentation Impact
