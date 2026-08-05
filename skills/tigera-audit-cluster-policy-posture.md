---
name: Audit a cluster's Calico policy posture
description: >-
  Read-only sweep of a cluster's network-security posture — tiers and their evaluation order,
  every global and namespaced policy, the network sets and threat feeds policy rules depend on,
  and what the calling identity is actually allowed to do.
api: openapi/tigera-calico-api-openapi-original.json
operations:
  - getProjectcalicoOrgV3APIResources
  - listProjectcalicoOrgV3Tier
  - listProjectcalicoOrgV3GlobalNetworkPolicy
  - listProjectcalicoOrgV3NetworkPolicyForAllNamespaces
  - listProjectcalicoOrgV3StagedGlobalNetworkPolicy
  - listProjectcalicoOrgV3GlobalNetworkSet
  - listProjectcalicoOrgV3NetworkSetForAllNamespaces
  - listProjectcalicoOrgV3GlobalThreatFeed
  - listProjectcalicoOrgV3HostEndpoint
  - createProjectcalicoOrgV3AuthorizationReview
---

# Audit a cluster's Calico policy posture

Every operation here is a read. Nothing in this skill changes cluster state.

## Steps

1. **Confirm the API is served.**
   `getProjectcalicoOrgV3APIResources` returns the resource list the cluster actually serves. If
   this 404s or 503s, the Calico aggregated API server is not installed or its APIService is
   unhealthy — stop and report that rather than reporting "no policies".

2. **Establish what you are allowed to see.**
   `createProjectcalicoOrgV3AuthorizationReview` — POST the resource attributes you care about
   and the server returns `AuthorizedResourceVerbs`, the verbs your identity holds. Run this
   first so you can distinguish "there are no policies" from "I cannot see the policies".

3. **Read the tier order.**
   `listProjectcalicoOrgV3Tier`. Sort by `spec.order` ascending — this is the evaluation order,
   and it is the single most important fact about a cluster's posture. A permissive rule in an
   early tier overrides everything after it.

4. **Enumerate enforced policy.**
   - `listProjectcalicoOrgV3GlobalNetworkPolicy` — cluster-wide.
   - `listProjectcalicoOrgV3NetworkPolicyForAllNamespaces` — every namespace in one call.
   Group by `spec.tier`, then by `spec.order` within the tier.

5. **Enumerate staged (non-enforcing) policy.**
   `listProjectcalicoOrgV3StagedGlobalNetworkPolicy`. Staged policies are evaluated and reported
   but do **not** enforce. Reporting a staged policy as protection is the classic false positive
   in a posture audit — label them explicitly.

6. **Resolve what the rules point at.**
   - `listProjectcalicoOrgV3GlobalNetworkSet` and `listProjectcalicoOrgV3NetworkSetForAllNamespaces`
     for the named CIDR/domain sets that rules select.
   - `listProjectcalicoOrgV3GlobalThreatFeed` for pulled feeds; each feed materialises into a
     GlobalNetworkSet named in `spec.globalNetworkSet`, so a feed change silently changes policy.
   - `listProjectcalicoOrgV3HostEndpoint` for host-level (node) protection, which is easy to
     forget and is where node-port exposure gets decided.

7. **Page through everything.**
   Set `limit` and follow `metadata.continue` until it comes back empty. A truncated list read as
   complete is an audit that under-reports. If a `continue` token expires you get `410 Gone` —
   restart the list, do not fill the gap.

## Reading the result

- A workload matched by **no** policy in **any** tier falls through to the default behaviour of
  the last tier — verify that rather than assuming default-deny.
- `spec.selector`, `spec.namespaceSelector` and `spec.serviceAccountSelector` are evaluated at
  runtime against labels. A policy that looks strict can select nothing. Cross-check the selector
  against real workload labels before calling it coverage.

## Notes

- Use `labelSelector` / `fieldSelector` to scope a large cluster instead of listing everything.
- To keep an audit live rather than point-in-time, re-issue any of the list calls with
  `watch=true` and `allowWatchBookmarks=true`; see `conventions/tigera-conventions.yml`.
