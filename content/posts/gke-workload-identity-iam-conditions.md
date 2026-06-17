+++
date = '2026-06-17T00:00:00Z'
draft = false
title = 'GKE Workload Identity Needs Better IAM Condition Support'
description = 'IAM Conditions cannot reference Kubernetes service account attributes when using GKE Workload Identity — here is why that matters and what I am proposing to fix it.'
tags = ["gke", "workload-identity", "iam", "gcp", "kubernetes", "devsecops", "least-privilege"]
categories = ["GCP", "Kubernetes", "Security"]
+++

I've opened a feature request with Google to address a gap in GKE Workload Identity that I keep running into: [Issue #475763043](https://issuetracker.google.com/issues/475763043).

## The Problem

When a pod authenticates via Workload Identity (`PROJECT_ID.svc.id.goog`), GKE already knows which Kubernetes service account is behind the request — it knows this at token minting time. The problem is that IAM Conditions can't see any of it.

At the project, folder, or org level there's no structured way to reference KSA attributes in an IAM Condition. The only thing available is `principal.subject`, an opaque string that isn't even supported for evaluation in project-level IAM Conditions.

So namespace and cluster level `principalSet` bindings work fine for broad access, but there's no clean way to distinguish between different service accounts within the same namespace when writing IAM policy higher up the resource hierarchy.

## Why the Workarounds Don't Cut It

- **String parsing on `principal.subject`** — brittle, unsupported at project level, breaks silently if the format changes
- **Enumerating KSAs in IAM policies** — doesn't scale, and you'll hit policy size limits on busy clusters
- **One Google service account per workload** — the most common workaround, but you're just adding identities to paper over a missing primitive

## What I'm Proposing

Expose a limited set of KSA-scoped attributes to IAM Conditions for Workload Identity principals:

- Kubernetes service account name
- Opt-in KSA labels and annotations (allowlisted, not everything)

These would be read-only, asserted by GKE at token minting time, and evaluated by IAM at request time. The syntax would look something like:

```
principal.kubernetes.serviceAccount == "redis-primary"
principal.kubernetes.serviceAccount.startsWith("redis-")
principal.kubernetes.serviceAccountLabels["app"] == "redis"
```

This is intentionally narrow — it doesn't change how `principalSet` works, it just adds the missing layer of expressiveness on top.

## Why It Matters

Without this, teams are forced to choose between operational simplicity and least-privilege — and simplicity usually wins. Workloads end up with broader access than they need, which increases blast radius if something is compromised.

The underlying data is already there. GKE knows the KSA at minting time. Surfacing it in IAM Conditions is the missing primitive that makes workload-level least-privilege practical at scale.

## How You Can Help

If this affects you, please star or comment on the issue: [issuetracker.google.com/issues/475763043](https://issuetracker.google.com/issues/475763043).

Google uses engagement on Issue Tracker to help prioritise work, and this is one of those foundational fixes that looks small on paper but has a meaningful impact for anyone running GKE at scale.
