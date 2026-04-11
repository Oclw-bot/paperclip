# 2026-04-11 Faster Issue Navigation

Status: Proposed
Date: 2026-04-11
Audience: Product and engineering
Related:
- `doc/plans/2026-04-07-issue-detail-speed-and-optimistic-inventory.md`
- `ui/src/lib/router.tsx`
- `ui/src/components/IssueLinkQuicklook.tsx`
- `ui/src/components/IssueRow.tsx`
- `ui/src/components/EntityRow.tsx`
- `ui/src/components/MarkdownBody.tsx`
- `ui/src/pages/IssueDetail.tsx`
- `ui/src/pages/Inbox.tsx`
- `ui/src/pages/Issues.tsx`
- `ui/src/pages/Dashboard.tsx`
- `ui/src/pages/ApprovalDetail.tsx`
- `ui/src/pages/AgentDetail.tsx`
- `ui/src/pages/ProjectDetail.tsx`
- [PAP-1346](/PAP/issues/PAP-1346)

## 1. Purpose

This note narrows the issue-detail speed work to one specific user-facing question:

- how do we make issue-to-issue navigation feel instant from inbox, issue lists, dashboards, and issue references inside other issues

The existing 2026-04-07 plan covers the broader issue-detail overfetch problem. This plan focuses on the navigation path itself: what happens before the click, during the route switch, and in the first paint after navigation.

## 2. Findings

## 2.1 This is not a true full-page reload

The main issue links already use React Router through `@/lib/router`.

- `ui/src/lib/router.tsx` wraps `react-router-dom` `Link`
- inbox/list/detail issue links mostly route through that wrapper
- the "heavy reload" feeling is therefore client-side navigation plus a cold issue-detail bootstrap, not the browser leaving the SPA

## 2.2 The hot paths do very little intentional prefetching

Today, the app only has one issue-specific link enhancement: `IssueLinkQuicklook`.

- `IssueLinkQuicklook` fetches `issuesApi.get(issuePathId)` only when its popover opens
- many high-frequency links explicitly bypass it with `disableIssueQuicklook`
- `IssueRow`, which powers the inbox and issue list rows, disables quicklook entirely
- many other issue links are plain `Link` usages with no cache seeding and no prefetch step:
  - dashboard recent issues
  - approval linked issues
  - agent detail recent issues
  - my issues
  - project detail issue chips

Result: the places users click most often do not warm the detail cache before navigation.

## 2.3 Navigation state seeding is inconsistent

Some surfaces preserve breadcrumb/header state, but only a subset.

- inbox keyboard navigation seeds header state and breadcrumb state
- `IssueRow` stores lightweight issue header state before navigation
- many other links only pass a URL, so the destination page starts colder than it needs to

That makes issue switches feel worse outside the inbox even when the target issue is already partially known in the current screen.

## 2.4 Issue detail still has a broad cold-start shape

`IssueDetail` still mounts a large query fan-out immediately after route change:

- issue detail
- comments
- activity
- linked runs
- linked approvals
- attachments
- live runs
- active run
- child issues
- feedback votes
- instance settings
- plus shared company/session data

Local API timings for the current dev instance were fast:

- `GET /api/issues/:id` about `29ms`
- `GET /api/issues/:id/comments` about `19ms`
- `GET /api/issues/:id/activity` about `16ms`
- `GET /api/issues/:id/runs` about `22ms`

That points to aggregate client bootstrap and paint timing as the main problem, not one obviously slow endpoint.

## 2.5 Cache identity is ref-based instead of issue-based

`queryKeys.issues.detail(id)` keys the cache by the raw route ref.

- `PAP-1346` and the issue UUID are different cache keys
- prefetching or seeding one ref does not automatically satisfy the other
- live-update invalidation already has to resolve multiple refs because of this

That makes issue-detail warming less reliable than it should be.

## 2.6 Markdown issue references already prove part of the model

`MarkdownBody` already resolves issue refs and fetches issue detail for inline status icons.

- that means the app already accepts the idea that issue references can warm detail data ahead of navigation
- but it happens ad hoc inside markdown instead of through one shared issue-navigation primitive

## 3. Plan

## 3.1 Phase 1: Introduce one shared issue-navigation primitive

Create a dedicated helper or component for issue links instead of relying on generic `Link` plus scattered ad hoc state.

Responsibilities:

- accept a known `Issue` object when available
- build the canonical issue detail path
- seed breadcrumb/header state consistently
- prefetch issue detail on user intent:
  - `pointerenter`
  - `focus`
  - `touchstart`
  - keyboard selection in list views

Suggested first touchpoints:

- `IssueRow`
- `EntityRow` when `to` targets an issue
- dashboard recent issues
- approval linked issues
- issue ancestors and child issue links
- project detail issue chips

Expected outcome:

- the issue header can paint immediately on click from the main navigation surfaces
- the detail query is usually already in cache by the time the route changes

## 3.2 Phase 2: Make issue detail use prefetched and seeded data aggressively

Once links warm the cache, `IssueDetail` should consume that data without falling back to a cold-looking transition.

Changes:

- use `placeholderData` or explicit cache seeding for the core issue detail query
- treat route-state header seed as a UI fallback, not the only fast path
- avoid blocking the main shell on secondary queries when the core issue object is already known

Expected outcome:

- route switches show a stable title, status, priority, and project label immediately
- the page feels like a view transition, not a reload into skeletons

## 3.3 Phase 3: Split navigation-critical data from background data

Not everything on `IssueDetail` needs to block the first usable paint.

Navigation-critical:

- issue core detail
- enough comment/thread state to show the active tab shell
- breadcrumb context

Background:

- activity
- attachments
- approvals
- run history
- lower-priority side panels

This phase should align with the broader issue-detail speed work in the 2026-04-07 plan, especially around bounded comments and narrower child-issue loading.

## 3.4 Phase 4: Canonicalize issue cache identity

Make the cache treat identifier and UUID references as the same logical issue.

Options:

- normalize all issue detail access to one canonical cache key after fetch
- alias both `id` and `identifier` into the cache whenever issue detail is fetched or seeded

Expected outcome:

- prefetch from any source remains useful after route normalization
- live updates and invalidation logic get simpler

## 3.5 Phase 5: Add instrumentation and enforce it in tests

Add lightweight instrumentation for issue navigation so this does not regress silently.

Measure:

- click to issue-header paint
- click to first interactive issue shell
- cache-hit vs cold-load navigation timings

Test coverage:

- unit tests for the shared issue-link primitive
- tests that state seeding happens from list rows and dashboard cards
- tests that issue navigation keeps working with both identifier and UUID refs

## 4. Suggested Implementation Order

1. Build the shared issue-link/prefetch helper.
2. Convert inbox and issues list first, since they are the hottest paths.
3. Convert dashboard, approval, agent, project, and issue-detail child/ancestor links.
4. Update `IssueDetail` to consume cache-seeded core detail without a cold shell.
5. Canonicalize detail cache refs.
6. Add timing instrumentation and regression tests.

## 5. Success Criteria

- Clicking an issue from inbox or issues list shows the destination header immediately, without a blank-feeling transition.
- Hover/focus on a likely next issue warms the core detail query before click.
- Issue-detail navigation from dashboard, approvals, and issue-to-issue links uses the same fast path as inbox rows.
- Identifier-based and UUID-based issue refs share useful cached detail instead of duplicating work.
- The remaining slow-feeling work is clearly attributable to deeper issue-detail loading, not the route switch itself.

## 6. First PR Scope

The safest first PR for `PAP-1346` is:

- add the shared issue-link prefetch helper
- convert inbox rows and issues list rows to use it
- seed the core issue detail cache from the already-loaded `Issue` object on click or hover
- update `IssueDetail` to prefer cached core detail immediately

That should deliver a visible navigation improvement without needing to solve every issue-detail overfetch problem in the same change.
