# Runbook: Deploying a Service

How to deploy and update a service on the platform. No cluster access required —
all changes go through Git.

## Prerequisites

- Write access to the service repository
- Service already onboarded (see onboarding-a-new-service.md)

## Deploy a change

1. Edit the manifest in the service repo under `k8s/`
   (image tag, replica count, environment variables, resource limits)
2. Commit with a descriptive message
3. Push to `main`
4. Argo CD detects the change within ~3 minutes and applies it automatically

There is no manual sync step and no kubectl command. The push IS the deployment.

## Verify a deployment

Check the Argo CD UI for the application — expect `Synced` and `Healthy`.

From the CLI:

    kubectl get pods -l app=<service-name>

New pods will show a low AGE value. Older pods that did not need replacing
keep their original age.

## Sync policy

All applications run with automated sync enabled:

- **prune** — resources deleted from Git are deleted from the cluster
- **selfHeal** — manual changes to live resources are reverted to match Git

This means imperative changes do not persist. Running `kubectl scale` or
`kubectl edit` against a managed resource will be undone automatically.
To make a durable change, change Git.

## Rollback

Revert the offending commit and push:

    git revert <commit-sha>
    git push

Argo CD applies the previous state on its next sync. Do not roll back by
editing live resources — selfHeal will undo it.

## Troubleshooting

**Application shows OutOfSync and does not resolve**
Check the Argo CD app for a sync error. Most common cause is invalid YAML.
Validate locally before pushing:

    kubectl apply --dry-run=client -f k8s/

**Application shows Degraded**
The manifest applied successfully but pods are not healthy. Check pod status
and logs:

    kubectl get pods -l app=<service-name>
    kubectl logs <pod-name>

Common causes: image pull failure, container port mismatch, or health check
probes pointing at endpoints the application does not serve.

**Changes pushed but nothing happened**
Confirm the push landed on the branch Argo CD tracks, and that the file lives
under the path configured in the application spec.
