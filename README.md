# idp-platform

Internal Developer Platform - GitOps infrastructure and platform components



\# idp-platform



A minimal but working \*\*Internal Developer Platform (IDP)\*\* built on Kubernetes,

GitOps, and policy-as-code. Developers deploy by committing to Git; the platform

handles delivery and enforces security standards automatically.



This repository holds the platform layer — the GitOps configuration, security

policies, and documentation. Application code lives in separate repositories

(\[idp-service-template](https://github.com/FujimotoFC/idp-service-template),

\[idp-sample-service](https://github.com/FujimotoFC/idp-sample-service)).



\## What it does



\- \*\*Git is the source of truth.\*\* Argo CD watches this repo and the service repos,

&#x20; and reconciles the cluster to match. A `git push` is the deployment; there is no

&#x20; manual `kubectl apply` step in the normal workflow.

\- \*\*Self-healing.\*\* Manual changes to live resources are automatically reverted to

&#x20; match Git, preventing configuration drift.

\- \*\*Security enforced at admission.\*\* A Kyverno policy blocks any workload that does

&#x20; not declare CPU and memory limits — the resource is rejected before it is created,

&#x20; not flagged after the fact. The policy is scoped to application namespaces;

&#x20; platform/system namespaces are excluded.



\## Architecture



```

Developer

&#x20;  │  git push

&#x20;  ▼

GitHub repo ──────watched by──────▶ Argo CD ──────applies to──────▶ Kubernetes

&#x20;                                                                       │

&#x20;                                                         every resource passes through

&#x20;                                                                       ▼

&#x20;                                                               Kyverno admission

&#x20;                                                         (reject if no resource limits)

```



\## Repository layout



```

idp-platform/

├── apps/

│   └── test-app/            # sample app deployed via GitOps

├── platform/

│   └── kyverno/

│       ├── require-resource-limits.yaml   # the enforced ClusterPolicy

│       └── bad-deploy.yaml                # intentionally non-compliant, for testing

├── clusters/                # dev/stage/prod placeholders

├── charts/

└── docs/

&#x20;   └── deploying-a-service.md   # developer runbook

```



\## Tech stack



| Concern        | Tool                          |

|----------------|-------------------------------|

| Kubernetes     | kind (local), Kubernetes v1.32 |

| GitOps         | Argo CD                       |

| Policy engine  | Kyverno                       |

| Packaging      | Helm / raw manifests          |



\## How to run it locally



Requires Docker, kind, kubectl, and Helm.



```bash

\# 1. Create the cluster

kind create cluster --name idp-dev \\

&#x20; --image kindest/node:v1.32.2@sha256:f226345927d7e348497136874b6d207e0b32cc52154ad8323129352923a3142f



\# 2. Install Kyverno and apply the policy

helm repo add kyverno https://kyverno.github.io/kyverno/

helm repo update

helm install kyverno kyverno/kyverno -n kyverno --create-namespace

kubectl apply -f platform/kyverno/require-resource-limits.yaml



\# 3. Install Argo CD

kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml



\# 4. Point Argo CD at the app repos (via UI or CLI), with automated sync enabled.

\#    Argo CD then deploys the apps and keeps them in sync.

```



\## Demonstrating the security guardrail



```bash

\# This deployment has no resource limits — the policy rejects it at admission:

kubectl apply -f platform/kyverno/bad-deploy.yaml

\# Error: admission webhook "validate.kyverno.svc-fail" denied the request ...

```



Compliant workloads (with resource limits declared) deploy normally. This was

validated against the platform's own apps: a non-compliant manifest was blocked,

and the fix was to bring the manifest up to standard — not to weaken the policy.



\## Status



Built through core platform capabilities: GitOps delivery, self-healing, and

policy enforcement. Observability (Prometheus/Grafana) and multi-environment

promotion are planned extensions.

