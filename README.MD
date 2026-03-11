# ci-cd-platform-k8s

Infrastructure configuration for a fully automated CI/CD platform running on bare metal Kubernetes. This repo is the infrastructure side of the [ci-dashboard](https://github.com/sagi-l/ci-dashboard) project.

## Overview

Everything here is version-controlled and declarative — no manual cluster state, no clicking through UIs. Lose a pod, run Helm, get back to identical state.

```
GitHub Push
    → Jenkins (build, test, scan, push image)
        → opens PR with updated manifest
            → ArgoCD (detects merge, syncs cluster)
                → new version live
```

## What's in Here

| File | Purpose |
|------|---------|
| `values.yaml` | Jenkins Helm chart values — full controller config, JCasC, plugins, agents |
| `argocd/values.yaml` | ArgoCD Application manifest for GitOps sync |
| `jenkins-agent-rbac.yaml` | RBAC Role + RoleBinding scoped to `jenkins-agents` namespace |

## Jenkins

Deployed via the [jenkins/jenkins](https://github.com/jenkinsci/helm-charts) Helm chart with `defaultConfig: false` — this is intentional and critical. Disabling the default config prevents Helm from injecting a duplicate Kubernetes cloud that conflicts with the JCasC-managed cloud config, which caused non-deterministic agent failures.

All Jenkins configuration is managed through **JCasC (Configuration as Code)**:

- **Kubernetes cloud** — ephemeral BuildKit agents, spun up per build and auto-terminated
- **Security** — local auth, no anonymous access, no legacy API tokens
- **Credentials** — DockerHub and GitHub credentials injected from Kubernetes Secrets
- **Prometheus** — metrics endpoint enabled for monitoring integration
- **Jobs** — `ci-dashboard` pipeline job defined as code via Job DSL, with `[skip ci]` exclusion to prevent webhook loops

Agents use **WebSocket** (not TCP) for controller communication — this was a hard-won fix after debugging connection failures caused by protocol mismatch.

## ArgoCD

ArgoCD watches the `k8s/` directory in the [ci-dashboard](https://github.com/sagi-l/ci-dashboard) repo and automatically syncs any changes to the cluster. `selfHeal: true` ensures drift is corrected automatically. `prune: true` removes resources that are no longer in git.

## RBAC

The Jenkins ServiceAccount (in the `jenkins` namespace) is granted a scoped Role in the `jenkins-agents` namespace — not cluster-wide. Permissions are minimal:

- **Pods** — create, delete, get, list, watch (for spinning up ephemeral agents)
- **Pods/log** — get, list (for build log streaming)
- **Pods/exec** — create (required for pipeline `sh` steps)
- **Secrets** — get only (read credentials, no write)
- **Events** — create, patch, update

## Bootstrap

Prerequisites: a running Kubernetes cluster with Helm installed, and the required secrets created in the `jenkins` namespace.

```bash
# Create required secrets first
kubectl create secret generic jenkins-admin-credentials \
  --from-literal=jenkins-admin-user=admin \
  --from-literal=jenkins-admin-password=<password> \
  -n jenkins

kubectl create secret generic dockerhub-creds \
  --from-literal=username=<user> \
  --from-literal=password=<token> \
  -n jenkins

kubectl create secret generic github-creds \
  --from-literal=username=<user> \
  --from-literal=password=<token> \
  -n jenkins

# Install Jenkins via Helm
helm repo add jenkins https://charts.jenkins.io
helm install jenkins jenkins/jenkins -f values.yaml -n jenkins --create-namespace

# Apply agent RBAC
kubectl apply -f jenkins-agent-rbac.yaml

# Deploy ArgoCD Application
kubectl apply -f argocd/values.yaml
```

ArgoCD will take it from there — once the Application is created it syncs the `ci-dashboard` manifests automatically.

## Hard Problems Solved

**Duplicate Kubernetes clouds** — Helm's default config injects its own Kubernetes cloud entry. Combined with a JCasC-defined cloud, this caused Jenkins to non-deterministically pick the wrong (misconfigured) cloud for agent scheduling. Fixed by setting `defaultConfig: false`.

**WebSocket vs TCP agent mismatch** — Jenkins agents were failing to connect to the controller. Root cause was a protocol mismatch: the controller expected WebSocket but agents were attempting TCP (JNLP). Fixed by explicitly setting `webSocket: true` in the cloud config and disabling the TCP agent listener (`agentListenerEnabled: false`).

**Pod template inheritance** — The Kubernetes plugin silently falls back to an empty template when `inheritFrom` points to a non-existent parent, with no warning. Debugged by removing inheritance and defining all containers explicitly.

## Related

- **[ci-dashboard](https://github.com/sagi-l/ci-dashboard)** — the application, pipeline definition, and K8s manifests
- **[cicd-lab.dev](https://cicd-lab.dev)** — live dashboard
