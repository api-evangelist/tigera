---
name: Capture pod traffic to troubleshoot a Calico policy
description: >-
  Start a scoped, time-bounded PacketCapture against the workloads a policy selects, list what is
  running cluster-wide, and clean up afterwards — the loop for proving whether a policy, not the
  application, is dropping traffic.
api: openapi/tigera-calico-api-openapi-original.json
operations:
  - listProjectcalicoOrgV3PacketCaptureForAllNamespaces
  - createProjectcalicoOrgV3NamespacedPacketCapture
  - readProjectcalicoOrgV3NamespacedPacketCapture
  - listProjectcalicoOrgV3NamespacedPacketCapture
  - deleteProjectcalicoOrgV3NamespacedPacketCapture
  - listProjectcalicoOrgV3NamespacedNetworkPolicy
---

# Capture pod traffic to troubleshoot a Calico policy

A PacketCapture is a **namespaced** resource whose `spec.selector` picks the workload endpoints
to capture from. It writes pcap files inside the cluster for retrieval — it is a privileged,
data-exposing operation, so treat it accordingly.

## Before you start

Packet capture reads real payloads from real workloads. Confirm you are permitted to capture in
the target namespace before you create anything, and prefer the narrowest selector that will
answer the question.

## Steps

1. **Identify the policy under suspicion.**
   `listProjectcalicoOrgV3NamespacedNetworkPolicy` for the namespace. Note the `spec.selector` of
   the policy you believe is dropping traffic — you want the capture selector to match the same
   workloads.

2. **Check nothing is already capturing.**
   `listProjectcalicoOrgV3PacketCaptureForAllNamespaces` shows every active capture in the
   cluster. Captures consume node disk; do not stack them.

3. **Dry-run the capture definition.**
   `createProjectcalicoOrgV3NamespacedPacketCapture` with `?dryRun=All` validates the selector
   syntax and admission without starting a capture. A malformed selector comes back as `422
   Invalid` with `details.causes[].field` = `spec.selector`.

4. **Start the capture.**
   `createProjectcalicoOrgV3NamespacedPacketCapture` in the target namespace. Scope
   `spec.selector` to the smallest label set that covers the failing workloads.

5. **Watch it come up.**
   `readProjectcalicoOrgV3NamespacedPacketCapture` by name, or list with `watch=true` to follow
   status changes rather than polling.

6. **Stop it.**
   `deleteProjectcalicoOrgV3NamespacedPacketCapture`. **Always delete the capture when the
   investigation ends** — an orphaned capture keeps writing pcap to node disk and keeps
   collecting payloads nobody is reading.

## Failure handling

| Status | `reason` | What to do |
|---|---|---|
| 403 | `Forbidden` | You lack `create` on packetcaptures in this namespace. |
| 422 | `Invalid` | `details.causes[].field` names the bad selector path. |
| 404 | `NotFound` | Namespace or capture name is wrong — captures are namespaced. |

## Never do this

- Never call `deleteProjectcalicoOrgV3CollectionNamespacedPacketCapture` to "clean up" without
  listing first; it removes every capture in the namespace, including one a colleague is
  actively using.
- Never widen `spec.selector` to `all()` to "catch everything". That captures every workload's
  traffic in the namespace.
