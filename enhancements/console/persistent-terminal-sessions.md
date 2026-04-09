---
title: persistent-terminal-sessions-in-web-console
authors:
  - "@bbabbar"
reviewers:
  - "@logonoff, for terminal component and webterminal-plugin expertise, please review the terminal session lifecycle and WebSocket management changes"
  - "@khatchou, for UX design review, please verify alignment with HPUX-742 Figma designs"
  - "Console team lead, for overall console architecture impact"
approvers:
  - "@amobrem"
api-approvers:
  - None
creation-date: 2026-04-09
last-updated: 2026-04-09
status: provisional
tracking-link:
  - https://redhat.atlassian.net/browse/RFE-6966
see-also:
  - https://redhat.atlassian.net/browse/HPUX-742
---

# Persistent Terminal Sessions in OpenShift Web Console

## Summary

Today, when a user opens a terminal session for a pod or node in the OpenShift
web console and navigates to another page (e.g., switching tabs, changing
namespaces, or visiting a different resource), the terminal session is
immediately destroyed. This enhancement introduces a persistent terminal drawer
that keeps pod exec/attach WebSocket sessions alive across navigation, allowing
users to multitask in the console without losing their terminal context.

## Motivation

The OpenShift web console provides terminal access to pods and nodes via the
Terminal tab on the pod details page. However, the current implementation
couples the terminal session lifecycle to the React component lifecycle: when
the user navigates away from the Terminal tab, `PodConnect` unmounts, its
`useEffect` cleanup sends `exit\r` over the WebSocket and calls
`wsRef.current.destroy()`, and the session is gone. The user must reconnect
from scratch, losing any running commands, shell history, and environment
state.

This is a significant productivity issue for developers, SREs, and cluster
administrators who routinely need to:

- Monitor a running command in one pod while inspecting another resource
- Switch between namespaces to compare configurations while keeping a debug
  session open
- Run long-running diagnostic commands (e.g., `tcpdump`, `strace`) while
  navigating the console for context

The existing Cloud Shell (Web Terminal Operator) already demonstrates a
persistent terminal pattern in the same console codebase. This enhancement
extends that pattern to ad-hoc pod and node terminal sessions.

### User Stories

- As a developer, I want to keep my pod terminal session running when I switch
  to the Logs tab or another namespace so that I can cross-reference information
  without losing my debug session.

- As an SRE, I want to have multiple persistent terminal sessions to different
  pods docked at the bottom of the console so that I can triage an incident
  across several components simultaneously.

- As a cluster administrator debugging a node, I want my node debug terminal to
  remain connected when I navigate to check cluster settings so that I do not
  have to re-establish the debug session each time.

### Goals

1. Terminal sessions for pods and nodes persist across page navigation in the
   web console.
2. A docked terminal drawer (bottom of screen) provides access to active
   sessions via labeled tabs.
3. Multiple concurrent terminal sessions are supported (up to a configurable
   limit, e.g., 8).
4. Users can minimize, restore, and close individual sessions from the drawer.
5. Users can optionally disable this feature via user preferences.

### Non-Goals

1. Persisting terminal sessions across browser page reloads or browser tab
   closures (the WebSocket connection is inherently lost).
2. Persisting sessions across different browser tabs or windows.
3. Replacing or modifying the Cloud Shell / Web Terminal Operator feature.
4. Server-side session persistence (e.g., `tmux`/`screen` integration).
5. Changes to the Kubernetes exec/attach API itself.

## Proposal

Integrate persistent pod terminal sessions into the **existing Cloud Shell
drawer** (`CloudShellDrawer` / `MultiTabbedTerminal`) as additional tabs. When
a user opens a pod terminal and clicks "Detach to Cloud Shell," the session
metadata is stored in the webterminal-plugin Redux store, and a new tab appears
in the Cloud Shell drawer at the bottom of the screen. This tab creates a fresh
WebSocket connection to the same pod/container and renders an xterm.js terminal,
persisting across navigation because the drawer is mounted above the router.

This approach reuses the existing proven infrastructure:

- **Mount above the router**: `CloudShellDrawer` already wraps routed content
  in `app.tsx` and survives navigation.
- **Global state management**: The webterminal-plugin Redux slice
  (`plugins.webterminal.cloudShell`) is extended with a `detachedSessions`
  array.
- **WebSocket lifecycle**: `DetachedPodExec` (new component) creates its own
  `WSFactory` connection, avoiding complex xterm.js instance transfer.

Key changes to existing components:

| Component | Change |
|-----------|--------|
| `cloud-shell-actions.ts` | Add `addDetachedSession`, `removeDetachedSession`, `clearDetachedSessions` actions |
| `cloud-shell-reducer.ts` | Add `detachedSessions: DetachedSession[]` to state |
| `cloud-shell-selectors.ts` | Add `getDetachedSessions` selector and `useDetachedSessions` hook |
| `MultiTabbedTerminal.tsx` | Render detached session tabs alongside Cloud Shell tabs |
| `CloudShell.tsx` | Show drawer when detached sessions exist (even without DevWorkspace) |
| `PodConnect` | Replace "Pin to drawer" with "Detach to Cloud Shell" button that dispatches Redux action |
| New: `DetachedPodExec.tsx` | Renders a single detached pod terminal (WebSocket + xterm.js) inside a Cloud Shell tab |

### Workflow Description

**Actors**: Developer, SRE, or Cluster Administrator using the OCP web console.

**Precondition**: User has access to a running pod with exec permissions.

**Flow**:

1. User navigates to a pod's Terminal tab. `PodConnect` renders and connects as
   it does today.
2. User clicks a **"Detach to Cloud Shell"** button (new UI element on the
   terminal toolbar).
3. `PodConnect` dispatches `addDetachedSession` (with pod name, namespace,
   container) and `setCloudShellExpanded(true)` to the Redux store.
4. The Cloud Shell drawer opens (or is already open) at the bottom of the
   screen. `MultiTabbedTerminal` reads `detachedSessions` from Redux and
   renders a new tab labeled `<pod-name>/<container>`.
5. The `DetachedPodExec` component in the new tab creates a fresh `WSFactory`
   WebSocket connection to the same pod/container and renders an xterm.js
   terminal.
6. The user can navigate freely; the Cloud Shell drawer persists because it is
   mounted above the router in `app.tsx`.
7. The user can open additional detached sessions from other pods; each gets
   its own tab alongside existing Cloud Shell tabs.
8. The user can close individual detached tabs (which destroys the WebSocket)
   or close the entire drawer.

**Error handling**:

- If the pod is deleted or the WebSocket drops, the `DetachedPodExec` tab
  shows a disconnected state with a "Reconnect" button.
- If the maximum number of tabs (8 total across Cloud Shell + detached) is
  reached, the "+" button and "Detach" button are disabled.

### API Extensions

None. This enhancement is purely a frontend (console UI) change. No new CRDs,
webhooks, aggregated API servers, or API modifications are required. The
existing Kubernetes pod exec/attach WebSocket API is used unchanged.

### Topology Considerations

#### Hypershift / Hosted Control Planes

No unique considerations. The console connects to the Kubernetes API server for
exec; the transport is the same regardless of control plane topology.

#### Standalone Clusters

Standard deployment target. No special considerations.

#### Single-node Deployments or MicroShift

No additional resource impact beyond what a normal terminal session already
requires. The persistent drawer holds the same WebSocket connections that would
otherwise exist on the Terminal tab.

#### OpenShift Kubernetes Engine

This feature is part of the web console UI and does not depend on OCP-only APIs.
It works on OKE identically.

### Implementation Details/Notes/Constraints

#### Architecture

The implementation integrates directly into the existing Cloud Shell stack:

```
app.tsx
  └── QuickStartDrawer
        └── CloudShellDrawer (existing - enhanced)
              └── MultiTabbedTerminal (existing - enhanced)
                    ├── Tab: CloudShellTerminal (existing Cloud Shell tabs)
                    └── Tab: DetachedPodExec (NEW - detached pod terminals)
```

There is no separate drawer. Detached pod terminal sessions appear as
additional tabs in the same `MultiTabbedTerminal` component that Cloud Shell
uses, sharing the drawer, tab management, minimize/restore, and close
behavior.

#### Detach Mechanism

When the user clicks "Detach to Cloud Shell" on a pod's terminal:

1. `PodConnect` dispatches `addDetachedSession` with session metadata (pod
   name, namespace, container name) to the Redux store.
2. `PodConnect` dispatches `setCloudShellExpanded(true)` to open the drawer.
3. `MultiTabbedTerminal` reads the `detachedSessions` array from Redux and
   renders a new `<Tab>` containing `<DetachedPodExec>`.
4. `DetachedPodExec` creates a **fresh** `WSFactory` WebSocket connection to
   the same pod/container exec endpoint. The original `PodConnect` WebSocket
   is not transferred -- it continues running until the user navigates away,
   at which point it is destroyed normally.
5. The user now has a persistent terminal in the drawer and may navigate
   freely.

This "fresh connection" approach avoids the complexity of transferring
xterm.js instances and WebSocket connections between React component trees.
The trade-off is a brief reconnection delay when the detached tab first
opens, but this is typically sub-second for local exec connections.

#### Cloud Shell Coexistence

`CloudShell.tsx` is updated to show the drawer when there are detached
sessions, even if the Web Terminal Operator (DevWorkspace) is not installed.
This ensures the feature works on clusters without the operator.

#### Key Source Files Affected

| File | Path | Change |
|------|------|--------|
| Redux actions | `webterminal-plugin/src/redux/actions/cloud-shell-actions.ts` | Add detached session actions |
| Redux reducer | `webterminal-plugin/src/redux/reducers/cloud-shell-reducer.ts` | Add `detachedSessions` state |
| Redux selectors | `webterminal-plugin/src/redux/reducers/cloud-shell-selectors.ts` | Add `useDetachedSessions` hook |
| Multi-tab terminal | `webterminal-plugin/src/components/cloud-shell/MultiTabbedTerminal.tsx` | Render detached tabs |
| Cloud Shell wrapper | `webterminal-plugin/src/components/cloud-shell/CloudShell.tsx` | Show drawer for detached sessions |
| Pod terminal | `frontend/public/components/pod-connect.tsx` | "Detach to Cloud Shell" button |
| New component | `webterminal-plugin/src/components/cloud-shell/DetachedPodExec.tsx` | Detached pod terminal renderer |

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Memory growth from many open xterm instances | Total tab cap of 8 (Cloud Shell + detached combined) |
| WebSocket resource exhaustion on the API server | Same connections that would exist on the Terminal tab; no net increase during active use; idle sessions close on pod deletion |
| Detached session loses data from original terminal | Documented trade-off: detach creates a fresh session, not a transfer. Users are informed via tooltip. |
| User confusion about Cloud Shell vs detached tabs | Detached tabs are labeled with pod/container name to distinguish them from numbered Cloud Shell tabs |

### Drawbacks

- Adds complexity to the webterminal-plugin Redux store and
  `MultiTabbedTerminal` component.
- Users who forget to close detached sessions may consume API server WebSocket
  connections unnecessarily, though this is bounded by the session cap and
  natural pod lifecycle.
- The "fresh connection" approach means the detached terminal starts with a
  new shell session rather than preserving the exact state from `PodConnect`.
  This is a deliberate simplicity trade-off.

## Alternatives (Not Implemented)

### Alternative 1: Separate PersistentTerminalDrawer

Instead of integrating into the Cloud Shell drawer, introduce a completely
separate `PersistentTerminalDrawer` and `PersistentTerminalProvider` that
wraps the app content independently. This was considered initially but
rejected because:

- Two separate drawer components would conflict visually and create a
  confusing UX with overlapping bottom panels.
- Duplicates existing infrastructure (tab management, minimize/restore,
  drawer resizing) that `CloudShellDrawer` already provides.
- The UX design (HPUX-742) explicitly specifies integration into the
  existing Cloud Shell drawer.

### Alternative 2: Keep terminal mounted but hidden

Instead of transferring sessions, keep the `PodConnect` component mounted but
visually hidden when the user navigates away (e.g., `display: none`). This was
rejected because:

- React Router unmounts route components on navigation; preventing this would
  require significant changes to the routing architecture.
- Hidden but mounted components still consume layout space and may interfere
  with other UI elements.

### Alternative 3: Server-side session persistence (tmux/screen)

Run a `tmux` or `screen` session inside the pod so the terminal can be
reconnected after a WebSocket drop. This was rejected because:

- Requires `tmux`/`screen` to be available in the container image (many
  minimal images do not include them).
- Adds operational complexity and security surface.
- Does not address the UX goal of a seamless in-console experience.

## Open Questions

1. **Resolved**: Detached sessions share the same Cloud Shell drawer and tab
   strip (per UX design HPUX-742).
2. **Resolved**: No xterm.js transfer is needed. `DetachedPodExec` creates a
   fresh WebSocket connection, avoiding transfer complexity.
3. Should there be an idle timeout for persistent detached sessions (beyond
   the natural Kubernetes exec idle timeout)?
4. Should the detach button be hidden when the total tab count (Cloud Shell +
   detached) reaches the maximum of 8?

## Test Plan

_Section to be completed when targeted at a release._

High-level strategy:

- **Unit tests**: Redux actions/reducer for detached sessions (add/remove/clear);
  `MultiTabbedTerminal` renders detached tabs correctly.
- **Integration tests**: Verify "Detach to Cloud Shell" button dispatches
  correct actions; verify drawer opens with new tab; verify tab close removes
  session from Redux store.
- **E2E tests**: Full flow from pod terminal to detach to navigation and back;
  verify detached terminal connects successfully; verify Cloud Shell tabs and
  detached tabs coexist; verify drawer shows for detached sessions even without
  Web Terminal Operator.

## Graduation Criteria

_Section to be completed when targeted at a release._

### Dev Preview -> Tech Preview

- Feature is behind a feature flag or user preference toggle (default off).
- Basic single-session persistence works across navigation.
- Unit and integration tests pass.

### Tech Preview -> GA

- Multi-session tabbed drawer is complete.
- User preference toggle works (default on).
- E2E tests pass on all supported topologies.
- Documentation in openshift-docs.
- Performance testing confirms no regression from persistent sessions.

### Removing a deprecated feature

N/A -- this is a new feature.

## Upgrade / Downgrade Strategy

This is a purely frontend feature with no persistent backend state. On upgrade,
the feature becomes available. On downgrade, the feature disappears and terminal
sessions revert to the current non-persistent behavior. No migration is needed.

## Version Skew Strategy

No version skew concerns. The feature is entirely within the console frontend
and uses the standard Kubernetes exec/attach WebSocket API, which is stable.

## Operational Aspects of API Extensions

N/A -- no API extensions are introduced.

## Support Procedures

### How to detect failures

- **Symptom**: Terminal sessions disconnect unexpectedly when pinned to the
  drawer.
- **Events/Logs**: Browser console logs from `WSFactory` (connection errors);
  console pod logs for proxy errors.
- **Metrics**: Existing WebSocket connection metrics on the API server.

### How to disable the feature

The feature can be disabled via:

1. User preference toggle in the console (per-user).
2. If a feature flag is used during tech preview, the flag can be unset via
   console operator configuration.

### Impact of disabling

- No impact on cluster health or running workloads.
- Terminal sessions revert to current behavior (destroyed on navigation).
- Any sessions in the persistent drawer are closed.

## Infrastructure Needed

None. All changes are within the existing `openshift/console` repository.
