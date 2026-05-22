---
name: bolt-eck
description: >
  ECK (Elastic Cloud on Kubernetes) deployment variant for bolt-bootstrap.
  BACKLOG — ECK deployment is not yet implemented. bolt-bootstrap halts when
  DEPLOYMENT_TYPE=eck is set. Use ECH or Serverless for current deployments.
---

# Demo Bootstrap — ECK Variant

**Status: BACKLOG — not yet implemented.**  
**Supported deployment types today: `ech`, `serverless`**  
**ECK support is on the roadmap — see `docs/todo.md`.**

---

## Halt on invocation

When `bolt-bootstrap` routes here because `DEPLOYMENT_TYPE=eck` is set, stop immediately:

```
⛔ ECK deployment type is not yet implemented.

   DEPLOYMENT_TYPE=eck was detected in your .env.

   Supported deployment types today:
     • ech        → Elastic Cloud Hosted (ECH)
     • serverless → Elastic Cloud Serverless

   ECK support is on the roadmap. It requires research and validation across
   cloud providers (GKE, EKS, AKS) before a clean Terraform-based deploy
   pattern can be documented. See docs/todo.md.

   Action required:
     1. If you intended ECH or Serverless, update DEPLOYMENT_TYPE in .env and re-run.
     2. If you need ECK, this is a known gap — track it in the engagement backlog
        and check docs/todo.md for ECK roadmap status.
```

Do not proceed to ECH as a fallback. Do not generate any deployment artifacts.

---

## What ECK will require (for when this is implemented)

Tracked in `docs/todo.md`. Key differences from ECH that need validated patterns before implementation:

- **Cluster endpoint format** — ingress/LoadBalancer URL vs Elastic Cloud URL; TLS cert handling
- **API key provisioning** — Kubernetes secret creation vs Cloud API key management
- **Terraform provider** — `elastic/eck-operator` Terraform resources vs `elastic/elasticstack` provider
- **ILM tier availability** — depends on node roles defined in Kubernetes StatefulSets
- **Feature flag availability** — Agent Builder, Workflows: depends on ECK version and Kibana CR config
- **Cross-cloud-provider differences** — GKE, EKS, AKS each have provider-specific networking and storage patterns

## Roadmap

File or link a ticket in `docs/todo.md` to track ECK implementation progress.
