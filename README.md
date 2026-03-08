# Argo CD Private Repo Starter

This repository is a small GitOps starter for installing, bootstrapping, and self-managing [Argo CD](https://argo-cd.readthedocs.io/) from a private GitHub repository with Helm.

The bootstrap sequence is:

1. Install Argo CD into the cluster with Helm.
2. Install Sealed Secrets so repository credentials can be stored encrypted in Git.
3. Register the private Git repository in Argo CD using a `SealedSecret`.
4. Apply one bootstrap `Application`.
5. Let Argo CD manage Argo CD and Sealed Secrets from Git.

## Quick Start

This is the shortest path to a working local bootstrap.

1. Create a cluster:

```bash
kind create cluster --name argocd-lab
```

2. Install Argo CD:

```bash
kubectl create namespace argo-cd
helm dependency build charts/argo-cd
helm upgrade --install argo-cd charts/argo-cd --namespace argo-cd
```

3. Install Sealed Secrets:

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets-controller \
  sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --create-namespace \
  --set-string fullnameOverride=sealed-secrets-controller
```

4. Create a read-only GitHub deploy key for this private repo, copy [bootstrap/repository-secret.example.yaml](/Users/mike/Documents/src/argo-cd/bootstrap/repository-secret.example.yaml) to `bootstrap/repository-secret.private.yaml`, insert the private key, then seal it:

```bash
kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  --format yaml \
  < bootstrap/repository-secret.private.yaml \
  > bootstrap/repository-secret.sealed.yaml
kubectl apply -f bootstrap/repository-secret.sealed.yaml
```

5. Bootstrap Argo CD from Git:

```bash
kubectl apply -f bootstrap/root-application.yaml
kubectl get applications -n argo-cd
```

6. Access the UI:

```bash
kubectl port-forward svc/argo-cd-server -n argo-cd 8080:80
kubectl get secret argocd-initial-admin-secret -n argo-cd -o jsonpath='{.data.password}' | base64 --decode
echo
```

## Repository layout

`charts/argo-cd`
: Helm wrapper chart for the upstream `argo-cd` chart.

`apps/`
: Child applications managed by the root app. This now includes Argo CD itself and the Sealed Secrets controller.

`bootstrap/root-application.yaml`
: One-time manifest you apply manually after the repository credential exists.

`bootstrap/repository-secret.example.yaml`
: Example unsealed repository credential for a private GitHub repo over SSH. Use it only as local input to `kubeseal`; do not commit a live private key.

`charts/root-app`
: Optional Helm chart that renders the same bootstrap `Application` if you want a parameterized bootstrap later.

## Prerequisites

- A Kubernetes cluster you can experiment with.
  - Good local options: `kind`, `minikube`, or Docker Desktop Kubernetes.
- `kubectl`
- `helm`
- `kubeseal`

## Bootstrap model

This starter assumes:

- the Git repository is private
- Argo CD reads it over SSH using a read-only deploy key
- repository credentials are committed only in encrypted form with Sealed Secrets
- the first bootstrap step is done from an operator workstation with `kubectl`

That is a reasonable default for a professional starter because it keeps repository access declarative and avoids embedding personal Git credentials into the platform.

This repository still keeps the Argo CD server in insecure mode for local port-forwarded access. That is acceptable for a lab or internal bootstrap phase, but not for an internet-facing environment.

## Step 1: Create a local cluster

Example with `kind`:

```bash
kind create cluster --name argocd-lab
kubectl cluster-info --context kind-argocd-lab
```

## Step 2: Install Argo CD with Helm

Create the namespace and install the chart:

```bash
kubectl create namespace argo-cd
helm dependency build charts/argo-cd
helm upgrade --install argo-cd charts/argo-cd --namespace argo-cd
```

Wait for the core components:

```bash
kubectl get pods -n argo-cd
kubectl rollout status deployment/argo-cd-server -n argo-cd
kubectl rollout status statefulset/argo-cd-application-controller -n argo-cd
```

## Step 3: Install Sealed Secrets

Install the Sealed Secrets controller before you generate the encrypted repository credential.

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm upgrade --install sealed-secrets-controller \
  sealed-secrets/sealed-secrets \
  --namespace kube-system \
  --create-namespace \
  --set-string fullnameOverride=sealed-secrets-controller
kubectl rollout status deployment/sealed-secrets-controller -n kube-system
```

The `fullnameOverride=sealed-secrets-controller` setting keeps the controller name aligned with the `kubeseal` CLI convention described in the Sealed Secrets project docs.

This manual install is only for first bootstrap. After the root app syncs, Argo CD will manage the Sealed Secrets release from [apps/sealed-secrets.yaml](/Users/mike/Documents/src/argo-cd/apps/sealed-secrets.yaml).

## Step 4: Log in to Argo CD

Port-forward the API server:

```bash
kubectl port-forward svc/argo-cd-server -n argo-cd 8080:80
```

Get the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argo-cd -o jsonpath='{.data.password}' | base64 --decode
echo
```

Then open <http://localhost:8080>.

- Username: `admin`
- Password: value from the secret above

The upstream getting-started flow documents the same login pattern and initial admin secret handling in the Argo CD docs.

## Step 5: Register the private Git repository with a Sealed Secret

Before Argo CD can sync the root application, it must be able to clone this repository.

### Recommended bootstrap path in this starter: SSH deploy key plus Sealed Secrets

1. Create a read-only deploy key in GitHub for this repository.
2. Copy [bootstrap/repository-secret.example.yaml](/Users/mike/Documents/src/argo-cd/bootstrap/repository-secret.example.yaml) to a local untracked file such as `bootstrap/repository-secret.private.yaml`.
3. Replace the placeholder private key with the deploy key contents.
4. Seal it for this cluster:

```bash
kubeseal \
  --controller-name sealed-secrets-controller \
  --controller-namespace kube-system \
  --format yaml \
  < bootstrap/repository-secret.private.yaml \
  > bootstrap/repository-secret.sealed.yaml
```

5. Commit the sealed manifest if you want the encrypted bootstrap credential in Git.
6. Apply the sealed secret:

```bash
kubectl apply -f bootstrap/repository-secret.sealed.yaml
```

7. Verify that Argo CD can see the unsealed repository secret:

```bash
kubectl get secret repo-argo-cd -n argo-cd
```

Important:

- do not commit a live private key to Git
- keep the deploy key read-only
- rotate the key if it is ever exposed
- the sealed manifest is safe to commit, but it is cluster-specific
- if you bootstrap a different cluster, re-seal the secret against that cluster's controller certificate

This repository includes `.gitignore` entries for common local secret manifest names, but you should still treat the working tree carefully.

## Step 6: Bootstrap GitOps

Apply the root application:

```bash
kubectl apply -f bootstrap/root-application.yaml
```

That root app points Argo CD at `apps/`, which currently contains:

- `argocd.yaml`: an `Application` that manages `charts/argo-cd`
- `sealed-secrets.yaml`: an `Application` that manages the Sealed Secrets Helm chart in `kube-system`

This means Argo CD becomes self-managed after the initial Helm install, and Sealed Secrets also moves under GitOps management after its one-time bootstrap install. That pattern keeps first bootstrap simple while moving ongoing configuration into Git.

## Step 7: Verify sync

Check the applications:

```bash
kubectl get applications -n argo-cd
```

Expected result:

- `root-app` appears after you apply the bootstrap manifest
- `sealed-secrets` appears after `root-app` syncs
- `argo-cd` appears after `root-app` syncs
- all three eventually move to `Synced` and `Healthy`

If something is not syncing, inspect:

```bash
kubectl describe application root-app -n argo-cd
kubectl describe application argo-cd -n argo-cd
```

## Repository authentication strategy

This starter uses:

- SSH repository URLs in [bootstrap/root-application.yaml](/Users/mike/Documents/src/argo-cd/bootstrap/root-application.yaml), [apps/argocd.yaml](/Users/mike/Documents/src/argo-cd/apps/argocd.yaml), and [charts/root-app/values.yaml](/Users/mike/Documents/src/argo-cd/charts/root-app/values.yaml)
- a declarative Argo CD repository `Secret`
- Sealed Secrets to store the repository credential in encrypted form
- an Argo CD-managed Sealed Secrets controller defined in [apps/sealed-secrets.yaml](/Users/mike/Documents/src/argo-cd/apps/sealed-secrets.yaml)
- a read-only deploy key as the bootstrap credential

That is a practical private-repo path that still keeps secrets in Git in encrypted form.

For a more mature production design, especially across many repositories, prefer a GitHub App over per-repository deploy keys. Argo CD supports declarative GitHub App credentials as well, and the official docs cover that model.

## What is configured today

The Helm values in [charts/argo-cd/values.yaml](/Users/mike/Documents/src/argo-cd/charts/argo-cd/values.yaml) keep the setup small and local-friendly:

- Argo CD CRDs are installed by Helm
- Dex is disabled
- Notifications are disabled
- ApplicationSet is enabled
- The API server runs in insecure mode for local port-forwarded access

This is a reasonable baseline for learning. For a more production-like setup, the next steps would usually be:

- configure ingress and TLS
- replace local admin usage with SSO
- split apps by environment
- add Argo CD projects with tighter RBAC and destination restrictions
- replace deploy keys with GitHub App authentication if you will manage multiple repositories
- commit sealed credentials and other sealed bootstrap artifacts once generated

## Optional: Render the bootstrap app with Helm

If you want the bootstrap `Application` to be templated instead of using the static manifest:

```bash
helm template root-app charts/root-app
```

That chart renders the same app-of-apps entrypoint and is mainly here as a stepping stone for future environment-specific bootstrapping.

## Notes on Git access

The bootstrap and child applications now use this repository's SSH URL:

```text
git@github.com:michaelbrister/argo-cd.git
```

That means Argo CD must have a repository credential before the root app will sync.

If you want to switch to HTTPS plus GitHub App later, change the `repoURL` fields to:

```text
https://github.com/michaelbrister/argo-cd.git
```

You would need to update:

- [bootstrap/root-application.yaml](/Users/mike/Documents/src/argo-cd/bootstrap/root-application.yaml)
- [apps/argocd.yaml](/Users/mike/Documents/src/argo-cd/apps/argocd.yaml)
- [charts/root-app/values.yaml](/Users/mike/Documents/src/argo-cd/charts/root-app/values.yaml)

## Professional hardening notes

For a DevOps-oriented next iteration, the biggest gaps are:

- replace the shared `default` Argo CD project with one or more explicit `AppProject` resources that restrict allowed repositories, destinations, and cluster-scoped resource access
- SSO and RBAC instead of ongoing admin-password use
- `AppProject` boundaries instead of the global `default` project
- ingress, TLS, and DNS instead of port-forward access
- audit-ready repository credential rotation
- environment separation and promotion strategy
- a deliberate multi-cluster secret strategy, since Sealed Secrets output is cluster-specific

## Next exercises

Once this is running, useful learning exercises are:

1. Add a second application under `apps/`.
2. Create an `AppProject` and bind apps to it instead of using `default`.
3. Add sealed application manifests for repo credentials and other bootstrap secrets.
4. Add a staging/production split with separate values files.
5. Replace port-forward access with an ingress and TLS.

## References

- [Argo CD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [Argo CD Declarative Setup](https://argo-cd.readthedocs.io/en/latest/operator-manual/declarative-setup/)
- [Argo CD Private Repositories](https://argo-cd.readthedocs.io/en/latest/user-guide/private-repositories/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
