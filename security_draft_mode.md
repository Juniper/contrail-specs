
# 1. Introduction

Allow to define if security modifications in a scope (global or project) should
be applied directly or have to be audited by a tier team/person before enforce
them.

# 2. Problem statement

Some organizations need to audit and validate any security policy modifications
before allowing them.

# 3. Proposed solution

Introduce a security policy draft mode at the scope level that determines if any
modifications on the five security resources should be directly enforce or not.
If security policies need to be modified, three cases can happen:

1. Modifications create new security resources

   In that case, Contrail creates that new security resources as a child of a
   scoped dedicated policy management resource with all defined properties and
   references. That new security resource is not enforced and does not change
   data path.

   Followed modification on that created security resource will be apply on that
   draft version and not enforced in the data path.

2. Modification update on exiting security resources

   Here, Contrail clones concerned security resources, as a child of the same
   scoped dedicated policy management resource and apply modifications on that
   cloned resource. That modified security resource is not enforced and does not
   change data path. The clone also copies references and parent link, that
   permits to not remove a resource (e.g. tag) if a pending security resource
   reference it.

   Then if the user modifies that security resource again, Contrail continues to
   append modifications on the same cloned resource and modification still not
   enforced on the data path.

3. Modifications remove security resources

   In that case, Contrail clones concerned security resources, as a child of the
   scoped dedicated policy management resource and it marks the security
   resource as *'pending delete'*.

   All modifications on that *'pending delete'* resource should be forbidden.

   Later, if the user decides to not remove the resource, the only way to do
   that is to revert all pending modifications.

After that, a team/person in charge of the security, can review pending security
modifications thanks to new REST API calls to list and show them and then have
two choices:

1. commit modifications:

   When security resource modifications were validated and ready to be enforced,
   the security team/person can commit them. In that first version, the commit
   granularity will be limited to the scope. When a commit is on going, no more
   security modifications are allowed.

2. abandon modifications:

   When security resource modifications were not validated, the security
   team/person can abandon them. In that first version, the abandon/revert
   granularity will be limited to the scope. When an abandon/revert is on going,
   no more security modifications are allowed.

The Contrail API will be enhance with a filter that permits to get the original
or the modified security resource version if the resource have some pending
modifications.

The five security resources are:

* Application Policy Set
* Firewall Policy
* Firewall Rule
* Service Group
* Address Group

## 3.1 Alternatives considered

None

## 3.2 API schema changes

There is two possible scopes:

* **global** level which is own by the global default `policy-management`
  resource,
* and at the **project** level which is own by a `project` resources.

All draft clones of a security resources will be a child of dedicated policy
management resource which belongs to the scope:

* for global scoped, its `fq_name` is `draft-policy-management`,
* and for project scope its `fq_name` is construct with the project `fq_name`
  with `draft` string appended (e.g. `domainX:projectY:draft`) as usual.

Draft clones will have the same name as the original resource but have a
different and unique UUID.

The Contrail model will be modify accordingly:

1. Modifying `policy-management` owners from `config-root` to `config-root` or
   `project`.

2. add `enable_security_policy_draft` in scope resources,

   For the global scope, that properties will be added to the
   `global-system-config` resource and to the `project` resource for the
   project scope. If the flag is true, all security policy modifications will
   not be enforce until an authorized person/team approve and commit it, if it's
   false, all modifications will be enforce directly.

3. add `draft_mode_state` property to the five security resources,

   That property determines which security resources are a draft resource and
   if that draft resource is a creation, modification or a deletion. If a draft
   security resource was deleted, that permits to not allow anymore
   modifications on it. Possible value are `created`, `updated` or `deleted`.
   If that property is not set, that means that security resource not is a
   draft version but an original version. This property is only writable by the
   system only and can only be read by users.

For users no much changes, they will continue to use API to work with enforced
security resources. New API calls will permit to identify modified security
resources and to commit or revert them:

1. List all draft security resources per scope and resource type:

  ```
  GET /<RESOURCE_TYPE>s?parent_id=<DRAFT_POLICY_MANAGEMENT_UUID>
  ```
  or with policy FQ-name
  ```
  GET /<RESOURCE_TYPE>s?parent_type=policy-management&parent_fq_name_str=<DRAFT_POLICY_MANAGEMENT_FQ_NAME_STRING>
  ```

2. List pending modified security resources per scope

  ```
  GET /policy-management/<DRAFT_POLICY_MANAGEMENT_UUID>?fields=application_policy_sets,firewall_policys,firewall_rules,service_groups,address_groups
  ```
  That returns, the scoped and dedicated policy management resource with
  references to all pending security resources. The security resource types
  filter can be adjusted with the `fields` filter.

3. List pending created security resources per scope and resource type:

  ```
  GET /<RESOURCE_TYPE>s?parent_id=<DRAFT_POLICY_MANAGEMENT_UUID>&filters=draft_mode_state=="created"
  ```

4. List pending updated security resources per scope and resource type:

  ```
  GET /<RESOURCE_TYPE>s?parent_id=<DRAFT_POLICY_MANAGEMENT_UUID>&filters=draft_mode_state=="updated"
  ```

5. List pending deleted security resources per scope and resource type:

  ```
  GET /<RESOURCE_TYPE>s?parent_id=<DRAFT_POLICY_MANAGEMENT_UUID>&filters=draft_mode_state=="deleted"
  ```

7. Commit or revert modified security resources per scope

  ```
  POST /security-policy-draft
  {
    'scope_uuid': <SCOPE_UUID>,
    'action': [commit|revert]
  }
  ```

On the API server side, few tasks are added:

* Commit modifications:

  * For added security resources, Contrail calls the
    `vnc_cfg_api_server.vnc_db.VncDbClient.dbe_create` method with all draft
    resource version attributes but with the scope as
    owner<sup>[1](#footnote1)</sup>:
      * for the global scope, the owner is the global
        `default-policy-management` resource,
      * for the project scope, the owner is the project resource itself.


  * For updated security resources, Contrail calls the
    `vnc_cfg_api_server.vnc_db.VncDbClient.dbe_update` method with all draft
    resource version attributes on the original resource version.

  * For delete security resource, Contrail calls the
  `vnc_cfg_api_server.vnc_db.VncDbClient.dbe_delete` on it.

  Finally, when all modifications where applied on the committed resources and
  enforced on the data path, the scoped and dedicated policy management resource
  that owns all draft security resources can be purge as well as its children.

* Abandon/revert modifications

  The scoped and dedicated policy management resource that owns all draft
  security resources can be purge as well as its children.

* Lock security modifications when commit or revert is in progress

  A zookeeper lock will permits to know if a commit or a revert is in progress
  and return 409 errors to any security modifications. If a commit or a revert
  is in progress, the new draft API to list security modifications will return a
  empty list and the API call to do a commit or a revert will be block (returns
  409). Also, list and get API calls with `draft` filter set to true, return the
  draft version if a commit is in progress or the original version is a revert
  is in progress.

## 3.3 User workflow impact

See introduction in 3.

## 3.4 UI changes

* Add configuration knob for enable_security_policy_draft
    * For global scope, add a new tab  ‘Security Policy Options’ under
      Configure—>Infrastructure—>Global Config
    * For project scope, add checkbox ‘Security Policy Draft’ in
      Configure—>Infrastructure—>Project Settings—>Other Settings
* Changes in screens for authorized security persons to see and review modified security resource
    * In Existing  Security section (config—>security)
        * Add Firewall Rules tab to landing page
        * Add Commit/Draft toggle to list committed or drafted security resources
          in both global and project views
        * In Committed Security Resource view, highlight the drafted Security resources
        * Add Review button to see JSON diff of modified security resources in
          both global and project views
        * Add Commit/Revert options in JSON diff popup

## 3.5 Notification impact
N/A


# 4. Implementation
## 4.1 Work items
#### Describe changes needed for different components such as Controller, Analytics, Agent, UI.
#### Add subsections as needed.

# 5. Performance and scaling impact
## 5.1 API and control plane
Resource duplication in the config but nothing change on the controller side as
all duplicated security resources owned by scoped and dedicated policy management resource are ignored.

## 5.2 Forwarding performance
N/A

# 6. Upgrade
#### Describe upgrade impact of the feature
#### Schema migration/transition

# 7. Deprecations
#### If this feature deprecates any older feature or API then list it here.

# 8. Dependencies
#### Describe dependent features or components.

# 9. Testing
## 9.1 Unit tests
## 9.2 Dev tests
## 9.3 System tests

# 10. Documentation Impact

# 11. References

<a name="footnote1">1</a>: Another best solution will be just update that
new resource owner from the dedicated policy management to the scope but that
needs to validate there is no side effect to change owner.
