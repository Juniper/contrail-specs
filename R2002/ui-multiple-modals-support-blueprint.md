# Multiple modals support blueprint

## 1. Introduction

This is the Product Requirements Document for supporting multiple modals in Contrail Command.

### Terminology

- modal - a panel that opens in the browser on top of the existing UI.
- mask - a semi-transparent overlay between the modal and the UI in the background. It prevents the
  user from interacting with the UI in the background.
- modal with mask - a modal that has a mask behind it.
- modal without mask - a modal that does not have a mask behind it. When such a modal is open, the
  user can interact with the rest of the UI in the background.

## 2. Problem statement

Story link: [CEM-10459](https://contrail-jws.atlassian.net/browse/CEM-10459)

The user of Contrail Command can try have more than one modal open at the same time.

Such a situation may occur for example when the user:

1. Navigates to a table view that opens a settings/filters modal (e.g. Overlay -> Logical Routers)
2. Opens the _Filters_ modal. The modal opens, but allows interacting with the UI in the background.
3. Clicks on the _Remove_ button in some row in the background. This opens a new _Remove_ modal.

Since Contrail Command currently supports having only one modal open at the same time. When the
_Remove_ modal opens, the first _Filters_ modal will be closed. There may be more unwanted
side-effects in such scenarios.

The situation may occur more often with the introduction of the Guidance Design modal. It can be
shown on every view in Contrail Command. As a result, every view that previously has shown a single
modal now can show at least 2.

Thus, having support for opening multiple modals at the same time will benefit the user experience.

## 3. Proposed solution

The solution is to add support for showing multiple modals at the same time.

The UX style guide is available at
https://projects.invisionapp.com/d/main/default/#/console/13051452/280675693/preview (see the
_Opening multiple modals_ section).

There are 2 types of modals:

1. Modals that require the user's immediate attention. Such modals are _create_, _edit_, _remove_.
   Those modals should have a mask.
2. Modals that enhance the UI. Such modals are _filters_, _Guidance Design_, _settings_. Those
   modals should not have a mask.

The UI should allow for having multiple modals without mask open at the same time. They should
behave like windows in an operating system:

- when a new modal is open, it is on top of the other modals.
- when focusing a modal that is not on top (e.g. the 2nd modal), it becomes the topmost modal.

When a modal with mask is opened, the mask and the modal appear on top of the rest of the modals.
Thus, the user can only complete the action in the modal with mask.

## 4. API schema changes

None.

## 5. UI changes

Described in the [_Proposed solution_](#3.-proposed-solution) section.

There are 2 user-facing UI changes:

1. Upon opening a new modal, the previous modal will stay on the screen instead of closing.
2. The modals will have a shadow behind them, as on the UX designs. The shadow's size will depend on
   the order of the modals.

## 6. Implementation

Refactor the existing components used for handling modals. Those include:

1. `ModalManager`
2. `withModalHOC`
3. `ModalContainer`

The modification are described in more detail in [the tech spec](ui-multiple-modals-support-tech-spec.md).

## 7. Testing

Unit and end-to-end tests will be added to ensure that the feature is working correctly.

The tests are described in more detail in [the tech spec](ui-multiple-modals-support-tech-spec.md).

## 8. References

- [CEM-10459 story](https://contrail-jws.atlassian.net/browse/CEM-10459)
- [UX styleguide for modals](https://projects.invisionapp.com/d/main/default/#/console/13051452/280675693/preview)
