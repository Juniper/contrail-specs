## 3.4 UI changes
Option to configure primary and secondary controller in the bgpaas.
UI will show only discovered controll nodes.

## 3.5 Notification impact

# 4. Implementation

## 4.1 Work items

### 4.1.1 Vrouter Agent changes
Contrail-vrouter-agent will get primary and secondary controller from
the attribute of bgpaas-bgp-router node. If VNF uses x.x.x.1, agent will
make the peer with primary controller. If VNF uses x.x.x.2, agent will
make the peer with seconday controller. If there is no controller
configured, it will follow the existing model(choose first xmpp server for
x.x.x.1 and second xmpp server for x.x.x.2)

### 4.1.2 Vrouter changes
There is no vrouter changes.

## 4.2 Limitations

# 5. Performance and scaling impact
## 5.1 API and control plane
#### Scaling and performance for API and control plane

## 5.2 Forwarding performance
#### Scaling and performance for API and forwarding

# 6. Upgrade
None

# 7. Deprecations
None

# 8. Dependencies
None.

# 9. Testing
## 9.1 Unit tests
a) Configure bgpaas with primary/secondary controller and check it is peered with configued one
b) Confiugre bgpaas without controller and check it is working in the existing model.

## 9.2 System tests

# 10. Documentation Impact

# 11. References
