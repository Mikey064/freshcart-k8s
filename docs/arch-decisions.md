# FreshCart on Kubernetes — Architecture

## 1. Overview

This document describes how FreshCart's `checkout-api` and `storefront` services are deployed and operate on Kubernetes. It builds directly on two earlier pieces of this capstone series: the GCP infrastructure provisioned with Terraform, and the container images built and pushed by the GitHub Actions CI/CD pipeline. This phase replaces "one VM running everything" with a cluster that can route traffic, scale a service under load, recover from pod failure on its own, and roll out new versions without downtime.

The system was built and fully verified on a local **Kind** cluster first, then deployed unchanged (apart from two environment-specific values) to **GKE**. Every manifest in this repo is plain YAML — no Helm, no Kustomize, no service mesh — by design: the goal was to demonstrate the core Kubernetes primitives clearly, not to add tooling the project didn't need.

## 2. High-level architecture

```
                              Internet
                                 │
                                 ▼
                             Ingress
                 (ingress-nginx locally / GCE on GKE)
                 ┌───────────────┴───────────────┐
            /api/*                              everything else
                 │                                   │
                 ▼                                   ▼
      checkout-api Service                   storefront Service
          (ClusterIP)                            (ClusterIP)
                 │                                   │
                 ▼                                   ▼
    checkout-api Deployment                 storefront Deployment
     2 pods, HPA-managed 2–5                      2 pods
                 │
                 ▼
         postgres Service
          (ClusterIP)
                 │
                 ▼
       postgres Deployment
           (1 pod)
                 │
                 ▼
              PVC (2Gi)
```

Everything in the diagram lives in a single namespace, `freshcart`.

## 3. Components

### 3.1 Namespace
All resources live in the `freshcart` namespace. This keeps the project self-contained and is what the RBAC scoping in §6 is defined against.

### 3.2 checkout-api
- **Deployment**, 2 replicas, autoscaled 2–5 by the HPA described in §5.
- **Liveness and readiness probes** both hit `GET /healthz`. The readiness probe is what makes the rolling-update guarantee in §7.2 possible — a pod is never sent traffic until it reports healthy, and the liveness probe restarts a pod that's gone unresponsive without needing a human to notice.
- **Database connection string** is injected as an environment variable from a Kubernetes Secret (`checkout-db-secret`), never hardcoded in the manifest or image.
- **Resource requests/limits** are set (`100m`/`250m` CPU, `128Mi`/`256Mi` memory) so the scheduler can place pods sensibly and the HPA has a CPU baseline to measure utilization against.

### 3.3 storefront
- **Deployment**, 2 replicas, serving the Week 4 nginx-based static image.
- No database dependency, no probes beyond the default TCP check — it's a static file server, so there's less that can go wrong, and keeping it simple here was a deliberate choice rather than an oversight.

### 3.4 Services
Both `checkout-api` and `storefront` are exposed internally via `ClusterIP` Services. Neither is reachable from outside the cluster directly — all external traffic comes through the single Ingress, which is the only point where this system is exposed to the internet. That's a smaller attack surface and a simpler mental model than giving each Deployment its own LoadBalancer.

### 3.5 Ingress
One Ingress resource routes by path:
- `/api/*` → `checkout-api`
- everything else → `storefront`

Locally this runs on `ingress-nginx` (installed manually into Kind, since Kind ships no ingress controller by default). On GKE it runs on the built-in GCE ingress controller. The routing rules are the same; only the ingress-class-specific annotations differ between the two manifest variants in this repo.

### 3.6 Postgres and persistence
Postgres runs as a single-replica in-cluster Deployment backed by a 2Gi PersistentVolumeClaim, fronted by its own ClusterIP Service so `checkout-api` reaches it at a stable DNS name (`postgres`) regardless of which node the pod lands on.

**Why in-cluster instead of Cloud SQL, and why that wouldn't hold at production scale:**

This choice kept the project focused on Kubernetes primitives (PVC, Secret, Service) that the rubric specifically asked to demonstrate, avoided a second piece of cloud infrastructure to provision and pay for, and is genuinely sufficient for a single-developer demo workload with no concurrent users and no uptime requirement.

It would not be the right call for a real checkout system, for reasons that are about what a lone stateful pod can't provide, not about Kubernetes being incapable:

- **No high availability.** One replica means one point of failure. A plain Deployment also can't be scaled to multiple Postgres replicas for HA — there's no leader election, so three replicas would just be three independently-diverging databases, not a cluster. Real HA needs either a managed service with a standby (Cloud SQL) or a dedicated operator (e.g. CloudNativePG) that handles replication and failover correctly.
- **No automated backups or tested restore path.** The PVC survives pod restarts, but there's no backup schedule and no point-in-time recovery. Losing the disk, or a bad migration, means losing the data. Cloud SQL provides this by default.
- **Shared failure domain.** A cluster-level incident can take down the database and the application together, where a managed or externally-operated database decouples that blast radius.
- **Operational cost shifts to the team.** Version upgrades, tuning, replication monitoring, and credential rotation all become this team's responsibility instead of the cloud provider's — a reasonable trade for a capstone, not for a product with paying customers.
- **Secret handling is minimal.** `DATABASE_URL` sits in a plain Kubernetes Secret, which is base64-encoded, not encrypted, unless etcd encryption is separately enabled. Production would route this through Secret Manager with rotation.

A production version of this same system would keep `checkout-api`'s Deployment, Service, Ingress, HPA, and RBAC exactly as they are here, and move only the data layer out to Cloud SQL with private (VPC-peered or Auth Proxy) connectivity and credentials sourced from Secret Manager. The stateless and stateful halves of the system are allowed to have different lifecycles — this demo collapses them into one cluster for simplicity; production wouldn't.

## 4. Secrets

`checkout-db-secret` holds `DATABASE_URL` and is consumed by `checkout-api` via `secretKeyRef`, keeping the credential out of the Deployment manifest and out of the container image. The real secret is created imperatively (`kubectl create secret`) and never committed; the repo includes a redacted `01-secret.example.yaml` so the shape is documented without exposing the value.

## 5. Autoscaling

A HorizontalPodAutoscaler targets `checkout-api`, scaling between 2 and 5 replicas on a 70% average CPU utilization target. This was load-tested, not just declared: a `busybox` pod running a tight request loop against the `checkout-api` Service drove CPU usage up and the HPA scaled the Deployment from 2 to 5 replicas; deleting the load-generating pod let CPU usage fall back below target and the HPA scaled back down to 2. `storefront` has no HPA — it's a lightweight static server with a fixed, predictable load, so autoscaling it would have added a resource without a real need.

## 6. Security and access scoping

A namespace-scoped `Role` and `RoleBinding` (`freshcart-deployer`, bound to a dedicated `ci-deployer` ServiceAccount) grant only the verbs needed to manage Deployments and read Pods/Services inside the `freshcart` namespace — not a `ClusterRole`, and not access to any other namespace. This is the piece that connects conceptually to the Week 6 CI/CD pipeline: whatever identity a deploy pipeline uses should be able to update this application and nothing beyond it. Actually swapping the pipeline's GCP Workload Identity Federation token for one bound to this ServiceAccount is a reasonable next step, not something this phase needed to complete.

## 7. Verified resilience

Two specific behaviors were demonstrated against the running cluster, with timestamped evidence captured for the blog post:

### 7.1 Self-healing
A `checkout-api` pod was deleted by hand (`kubectl delete pod`). Kubernetes' Deployment controller detected the replica count had dropped below the desired 2 and scheduled a replacement automatically — no human intervention, no alert needing a response. The timestamp gap between the delete event and the new pod's `Scheduled`/`Started` events is the evidence for this.

### 7.2 Safe rolling updates
`checkout-api`'s image tag was changed and `kubectl rollout status` was run while a continuous curl loop hit `/api/healthz`. The curl loop showed an unbroken stream of `200` responses throughout the rollout — the readiness probe is what makes this possible, since Kubernetes never routes traffic to a new pod until it reports ready, and never kills the old pod until the new one has taken over.

## 8. Local vs. cloud environment differences

The system is intentionally the same manifest set in both environments. The only differences:

| | Local (Kind) | Cloud (GKE) |
|---|---|---|
| Ingress controller | ingress-nginx, installed manually | GCE ingress, built in |
| Ingress manifest | `nginx` class, rewrite annotation | no class/annotation needed |
| Container images | loaded into the Kind node via `kind load docker-image` | pulled from Artifact Registry |
| Node management | single Kind node, Docker-backed | GKE Autopilot — no node pool sizing to manage |

Running locally first meant every failure mode (probe misconfiguration, RBAC typos, HPA metrics-server quirks) was found and fixed for free, before touching billed cloud resources.

## 9. Known limitations / what a production version would add

- Managed, highly-available Postgres (Cloud SQL) instead of in-cluster, as detailed in §3.6.
- Secret Manager + an External Secrets Operator instead of a static Kubernetes Secret.
- TLS termination at the Ingress (cert-manager + a real domain) — this project runs over plain HTTP, which is fine for a local/demo cluster but not for anything public-facing.
- NetworkPolicies restricting which pods can talk to Postgres, rather than relying on namespace isolation alone.
- The CI/CD pipeline's deploy step actually assuming the `ci-deployer` ServiceAccount's scoped permissions, rather than those permissions existing but being unused.

None of these were needed to satisfy this capstone's requirements; they're listed here to be explicit about where "built to demonstrate the primitives" ends and "built for production" would begin.
