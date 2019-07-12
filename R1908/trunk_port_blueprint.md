Blueprint for Neutron Trunk Port support
========================================

Introduction
============

The trunk extension can be used to multiplex packets coming from and
going to multiple neutron logical networks using a single neutron
logical port. A trunk is modeled in neutron as a collection of neutron
logical ports.\
One port, called parent port, must be associated to a trunk and it is
the port to be used to connect instances with neutron.\
A sequence of subports (or sub\_ports) each typically belonging to
distinct neutron networks, is also associated to a trunk, and each
subport may have a segmentation type and ID used to mux/demux the
traffic coming in and out of the parent port.

Problem Statement
=================

There is no contrail backend support for Neutron Trunk Port feature. We
will extend contrail resource VPG(Virtual Port Group) for mapping it to
neutron trunk port. VPG was designed for handling non LCM BMS workflow
and multi VLAN support.

Solution
========

The extension introduces the following resources:

-   trunk. A top level logical entity to model the group of neutron
    logical networks whose traffic flows through the trunk.

-   sub\_port. An association to a neutron logical port with attributes
    segmentation\_id and segmentation\_type.

-   The following Trunk API Calls are added:

1\) GET /v2.0/trunks (List trunks)\
2) POST /v2.0/trunks (Create trunk)\
3) PUT /v2.0/trunks/{trunk\_id}/add\_subports (Add subports to trunk)\
4) PUT /v2.0/trunks/{trunk\_id}/remove\_subports (Delete subports from
trunk)\
5) GET /v2.0/trunks/{trunk\_id}/get\_subports (List subports for trunk)\
6) PUT /v2.0/trunks/{trunk\_id}(Update trunk)\
7) GET /v2.0/trunks/{trunk\_id} (Show trunk)\
8) DELETE /v2.0/trunks/{trunk\_id}(Delete trunk)\
9) GET /v2.0/ports/{port\_id}(Show trunk details)

[API schema changes:]{.underline}

It will require mapping neutron trunk port resource to Contrail resource
VPG which is being currently used as a bond from switch perspective and
a BMS (Bare meta server) view is a trunk port supporting trunking of
multiple vlans.

\<xsd:element name=\"project-virtual-port-group\"/\>

\<!\--\#IFMAP-SEMANTICS-IDL

Link(\'fabric-virtual-port-group\',

\'fabric\', \'virtual-port-group\', \[\'has\'\], \'optional\', \'CRUD\',

\'List of virtual port groups in this fabric.\');

Link(\'project-virtual-port-group\',

\'project\', \'virtual-port-group\', \[\'has\'\], \'optional\',
\'CRUD\',

\'List of virtual port groups/trunk ports in this project.\')

\--\>

\<img src=\"images/vpglinks.png\"\>

\<img src=\"images/legacymodel.png\"\>

\<img src=\"images/trunkmodel.png\"\>

*[Mapping:]{.underline}*

\<img src=\"images/mapping.png\"\>

Neutron trunk object gets mapped to Contrail's Virtual Port Group (VPG)
which was implemented in Contrail 5.1 as a part of non LCM/brownfield
BMS workflow. Trunk api calls get relays to VPG through vnc\_openstack.

Components that gets affected:

\[api-server\]: virtual\_port\_group resource file

vnc\_openstack

Neutron\_plugin\_contrail repo

*[Workflow for trunk:]{.underline}*

-   Create networks

-   Create ports

-   Create trunk

-   Use ports as created to form logical topology of the trunk

-   Boot instance with parent port

-   Adjusting trunk dynamically with add/remove sub ports
