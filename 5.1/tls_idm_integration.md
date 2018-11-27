# 1. Introduction
There is Contrail TLS support in TripleO with manual certificates lifecycle
management. This document describes the feature to enable automatization
of certificates lifecycle management in TripleO via integration with RedHat IDM.

# 2. Problem statement

Customers require the ability to have certificates management via RedHat IDM.
RedHat IDM is the project based of FreeIPA. FreeIPA does not support issuing
of certificates with IPs in the alternative subject names field.
Contrail today uses IPs (on both server and client sides) for connection
negotiation and strict certificate checks from both sides that requires having
IPs in the certificate alternative subject names. This makes impossible
to use FreeIP for Contrail certificates management.

# 3. Proposed solution

Modify Contrail to use FQDNs instead of IPs and add required integration at
the level of Contrail TripleO Heat Templates and Contrail TripleO puppets.

# 4. Implementation

- All Contrail services use FQDNs for connection establishment
- All Contrail services (for server side) that validate clients certificates 
use revers resolving of FQDN by client IP the check host against names in certificate
- Contrail docker containers take FQDNs instead IPs as input
paramaters (all lists _NODES)
- Contrail TripleO Heat Templates pass FQDNs instead of IPs for configuration
of Contrail containers
- Puppet class contrail::certmongeruser (contrail-tripleo-puppet project) that reads
data from hiera and calls ::certmonger
- ContrailCertmongerUser service in Contrail TripleO Heat Templates that is included
into all Contrail TripleO Roles and calls contrail::certmongeruser
- Certificates are issued per Contrail Node and stored into the folder /etc/contrail/ssl,
this folder if mounted to all Contrail containers.
- All Contrail services (including WebUI) use only certificates issued via certmonger.
- Contrail WebUI is available for end-user via Openstack Haproxy, so end-user sees
Haproxy's certificate. Contrail WebUI backend uses own certificate (issued via certmonger).
- Contrail nodes are enrolled in IDM (FreeIPA) the same way as other Openstack nodes
via NovaJoin plugin.

# 5. Performance and scaling impact

N/A

# 6. Upgrade

N/A

# 7. Deprecations

N/A

# 8. Dependencies

N/A

# 9. Testing

Individual apps will be brought up and tested for ssl support on 
corresponding API endpoints. Certificates lifecyle tests are required
(re-issue of certificates in cases of revocation and expiration).

# 10. Documentation Impact

Documentation about any new configuration knobs in
Contrail TripleO Heat Templates project.

# 11. References
