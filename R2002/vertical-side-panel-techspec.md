# 1. Introduction
Following UX Workshop and internal UX reviews, we have agreed to change the Mega Menu in favor of vertical side panel.
This document provides a brief description of the appearance and functionality of the new menu.

The terminology used in the document:
- Tier 1 nodes - categories that are used to group up entities. Examples from the Mega Menu: Infrastructure, Services..
- Tier 2 nodes - entities that are grouped up into categories. Examples from the Mega Menu: Servers, Fabrics..
# 2. Proposed solution
The Mega Menu will be removed and part of the breadcrumb responsible for opening it will be not clickable anymore. Vertical side panel should have the Tier 1 nodes currently available in the Mega Menu.
The sidebar will not contain the Tier 2 nodes, the user will be able to see them by putting the mouse over the Tier 1 category (on hover functionality). The list with the Tier 2 nodes will be displayed on the right side of the panel.
# 3. UI changes

### Modifications to the existing UI workflow
Example of how to open a new page using the side panel step by step. **// TO BE UPDATED**

### General requirements
The user can mark many entities as favorites. 
The list can get really long and depending on screen resolution, it might not fit on the screen. 
If the user will need to scroll the list down, the external applications will be shown always on the bottom and will be no scrolled with other nodes.

Example snapshot: **// TO BE UPDATED**

### Key features
##### Adding Tier 2 nodes to Favorites category
The UI requirements for marking nodes as favorites:
1. Favorites category itself should be visible whatever the user marked any node as favorite one or not.
2. The pin should be visible for every the Tier 2 node when the user put the mouse over it.
3. The pin's color should depend on the current state of Favorites category - if the node is already marked as one of the favorites, the pin should have different styling.
4. The hover styling of the Tier 2 nodes under Favorites category should differ marginally from the hover styling of the Tier 1 nodes.
5. The list of the favorite nodes should be persisted. To achieve this, the browser's local storage will be used.

Example workflow: **// TO BE UPDATED**

##### Reordering Tier 2 nodes under Favorites category
The user should be able to reorder the Tier 2 nodes under Favorites category. It can be achieved by a drag&drop functionality.

Example gif: **// TO BE UPDATED**

##### Search & Highlight for the Tier 2 nodes
The capability for searching and highlighting should be provided. The side panel will have a basic input for queries at the top.

The UI requirements for search&highlight:
1. After providing a query, all the Tier 1 nodes that do not contain the Tier 2 node which matches the query, should be greyed out.
2. All the Tier 2 nodes that match the query, should be highlighted. The styling should match with the one used for the tables.

Example workflow: **// TO BE UPDATED**

# 4. Testing
#### Unit tests
**// TO BE UPDATED**
#### E2E tests
\-

# 5. References
JIRA story - https://contrail-jws.atlassian.net/browse/CEM-9615
