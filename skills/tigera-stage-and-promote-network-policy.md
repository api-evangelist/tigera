---
name: Stage and promote a Calico network policy
description: >-
  Introduce a new Calico network policy without risking a production outage: place it in a tier,
  ship it first as a StagedNetworkPolicy so it is evaluated but not enforced, verify it with a
  dry-run write, then promote it to an enforced policy.
api: openapi/tigera-calico-api-openapi-original.json
operations:
  - listProjectcalicoOrgV3Tier
  - readProjectcalicoOrgV3Tier
  - createProjectcalicoOrgV3NamespacedStagedNetworkPolicy
  - listProjectcalicoOrgV3NamespacedStagedNetworkPolicy
  - deleteProjectcalicoOrgV3NamespacedStagedNetworkPolicy
  - createProjectcalicoOrgV3NamespacedNetworkPolicy
  - readProjectcalicoOrgV3NamespacedNetworkPolicy
  - replaceProjectcalicoOrgV3NamespacedNetworkPolicy
---

# Stage and promote a Calico network policy

Enforcing a wrong network policy takes traffic down. Calico's staged-policy mechanism exists so
you never have to find that out in production. Always stage first.

## Before you start

- **Auth.** The Calico API is a Kubernetes aggregated API server. Send a cluster bearer token or
  client certificate; there is no Calico-specific credential. See
  `authentication/tigera-authentication.yml`.
- **Permission.** Writing a policy inside a tier needs `get` on the `Tier` **and** the write verb
  on the policy resource. A 403 with `reason: Forbidden` usually means the tier grant is missing,
  not the policy grant.
- **Base path.** `/apis/projectcalico.org/v3` on the cluster's own API server.

## Steps

1. **Find the tier you are allowed to write into.**
   Call `listProjectcalicoOrgV3Tier`. Tiers are evaluated in ascending `spec.order`; a policy in
   an earlier tier can deny traffic a later tier would have allowed. Confirm the target tier with
   `readProjectcalicoOrgV3Tier` and note its `spec.order` before choosing where the policy goes.

2. **Validate the policy body without writing it.**
   Call `createProjectcalicoOrgV3NamespacedStagedNetworkPolicy` with `?dryRun=All`. Admission and
   schema validation run, nothing is persisted. A 422 comes back with
   `details.causes[].field` naming the exact offending path (for example
   `spec.ingress[0].source.selector`). Fix and repeat until the dry run succeeds.

3. **Create the staged policy for real.**
   Call `createProjectcalicoOrgV3NamespacedStagedNetworkPolicy` without `dryRun`. Set
   `spec.stagedAction: Set`, `spec.tier` to the tier from step 1, and `spec.selector` to the
   workloads in scope. The policy is now evaluated and reported on, but **not enforced** — no
   traffic changes.

4. **Observe before promoting.**
   Leave the staged policy in place long enough to cover a normal traffic cycle, and read the
   flow logs / Service Graph in the Calico console to see which connections the policy *would*
   have denied. This step is deliberately outside the API; do not skip it.

5. **Promote to an enforced policy.**
   Call `createProjectcalicoOrgV3NamespacedNetworkPolicy` with the same `spec` minus
   `stagedAction`, again with `?dryRun=All` first. Keep the same `spec.tier` and `spec.order` so
   evaluation position does not shift between staged and enforced.

6. **Remove the staged twin.**
   Call `deleteProjectcalicoOrgV3NamespacedStagedNetworkPolicy`. Leaving both in place is
   confusing but harmless; leaving the staged one is not a substitute for the enforced one.

## Updating an existing policy safely

Read with `readProjectcalicoOrgV3NamespacedNetworkPolicy`, mutate the returned object, and send
it back with `replaceProjectcalicoOrgV3NamespacedNetworkPolicy` **carrying the
`metadata.resourceVersion` you read**. If someone else changed the policy in between you get a
`409 Conflict` — re-read, re-apply your change, retry. Do not strip `resourceVersion` to make the
409 go away; that is how you silently overwrite a colleague's fix.

Prefer server-side apply (`PATCH` with `Content-Type: application/apply-patch+yaml` and a stable
`fieldManager`) when the policy is managed by automation. Applying the same manifest repeatedly
converges on the same state — that is this API's idempotency contract. See
`conventions/tigera-conventions.yml`.

## Failure handling

| Status | `reason` | What to do |
|---|---|---|
| 403 | `Forbidden` | Check for `get` on the Tier as well as the policy verb. |
| 409 | `Conflict` | Re-read, re-apply, retry. Never blind-retry a write. |
| 422 | `Invalid` | Read `details.causes[].field`; it names the bad path. |
| 429 | `TooManyRequests` | Honour `details.retryAfterSeconds`. |

Full envelope in `errors/tigera-problem-types.yml`.

## Never do this

- Never call `deleteProjectcalicoOrgV3CollectionNamespacedNetworkPolicy` unattended. It deletes
  every policy matching the selector and can strip a namespace's protection in one request.
