Brownfield BMS

Introduction
============

To provide support at the contrail config backend for users to configure
and associate port groups of TOR's physical interfaces from UI with the
bond interface of a BMS. To the extension to that user must be able to
configure TOR physical interfaces with multiple VLANs also known as VLAN
trunking (Similar to Neutron's Trunk support).

Problem Statement
=================

The intention here is for user to be able to configure TOR switch's
interfaces without worrying or to have any information about bare metal
servers for non LCM use cases. The workflow for the user should be as
simple as to choose a physical interfaces visible to them from a UI,
then apply to a bond and rest contrail backend should seamlessly take
care of connectivity between BMS server and a switch.

Solution
========

[API schema changes:]{.underline}

It will require to add two new objects/resources in schema to provide
support for this feature to be designed.

1.  PortGroup/Bond/LAG: This resource will have interfaces to the list
    of Physical Interfaces(PI), back ref to a VMI of a BMS server
    instance. There is already a\`

    LAG object on schema we just need to repurpose it for this use case.

    {

-   *bool* bonding\_enabled

-   *dict* { link local information(LLC) }

-   refs/back\_refs (list of \[PhysicalInterfaces\])

-   ref to a Trunk

}

2.  Trunk: It has a primary port uuid as a property and list of
    sub-interfaces with diff. vlan tags.

    Trunk

    {

-   Status

-   tenant\_id

-   sub\_ports = list of dict of { port\_id, segmenation\_type,
    segmentation id/vlanId }

-   port\_id (Trunk Port)

}

[Addition of APIs:]{.underline}

It will require to add APIs for CRUD operations on LAG and Trunk objects
i.e POST/PUT http requests.

*Note: Introducing these schema changes will result in ISSU upgrades as
5.0 doesn't have these objects and these are must needed to support
bonding and vlan trunking of a PI*

The object schema workflow in green is the existing workflow in api
server and red will be the new one with right most side red is the multi
vlan.

![](media/image1.tiff){width="7.480164041994751in"
height="3.8370800524934383in"}

![](media/image2.tiff){width="7.48125in" height="5.016666666666667in"}

![](media/image3.tiff){width="7.5in" height="4.325694444444444in"}

[User workflow:]{.underline}

User will have a screen per VN and VLAN ids with TORs and available
interfaces. They will just require to pick from the available list of
bonds and associate it with a PI interface and then hit apply.

[UI changes/workflow to api server:]{.underline}

1)  Create a trunk port for a BMS instance (This is equivalent of dummy
    VMI that UI is creating right now to support VLAN tagging).

2)  Create a sub interface on a trunk (It's the equivalent of VMI create
    that has LLC information in it).

3)  Create a LAG object with a LLC information and ref (LAG-\>Trunk) to
    a Trunk. If there are more than two physical links set bonding
    enabled to True else False and also make sure the bond names are
    unique (Global SystemConfig)
