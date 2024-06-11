# Source Hydrator

Tools like Helm and Kustomize allow users to express their Kubernetes manifests in a more concise and reusable way
(keeping it DRY - Don't Repeat Yourself). However, these tools can obscure the actual Kubernetes manifests that are
applied to the cluster.

The "rendered manifest pattern" is a way to push the hydrated manifests to git before syncing them to the cluster. This
allows users to see the actual Kubernetes manifests that are applied to the cluster.

The source hydrator is a feature of Argo CD that allows users to push the hydrated manifests to git before syncing them 
to the cluster.

To use the source hydrator, set the `spec.sourceHydrator` field of the Application. For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  sourceHydrator:
    drySource:
      repoURL: https://github.com/argoproj/argocd-example-apps
      path: helm-guestbook
      targetRevision: HEAD
    syncSource:
      targetBranch: environments/dev
      path: helm-guestbook
```

In this example, the hydrated manifests will be pushed to the `environments/dev` branch of the `argocd-example-apps` 
repository.

## Pushing to a "Staging" Branch

The source hydrator can be used to push hydrated manifests to a "staging" branch instead of the `syncSource` branch.
This provides a way to prevent the hydrated manifests from being applied to the cluster until some prerequisite
conditions are met (in effect providing a way to handle environment promotion via Pull Requests).

To use the source hydrator to push to a "staging" branch, set the `spec.sourceHydrator.hydrateTo` field of the 
Application. For example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
spec:
  project: my-project
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  sourceHydrator:
    drySource:
      repoURL: https://github.com/argoproj/argocd-example-apps
      path: helm-guestbook
      targetRevision: HEAD
    syncSource:
      targetBranch: environments/dev
      path: helm-guestbook
    hydrateTo:
      targetBranch: environments/dev-next
```

In this example, the hydrated manifests will be pushed to the `environments/dev-next` branch, and Argo CD will not sync
the changes until something moves them to the `environments/dev` branch.

You could use a CI action to move the hydrated manifests from the `hydrateTo` branch to the `syncSource` branch. To
introduce a gating mechanism, you could require a Pull Request to be opened to merge the changes from the `hydrateTo`
branch to the `syncSource` branch.

## Prerequisites

### Handle Secrets on the Destination Cluster

Do not use the source hydrator with any tool that injects secrets into your manifests as part of the hydration process
(for example, Helm with SOPS or the Argo CD Vault Plugin). These secrets would be committed to git. Instead, use a
secrets operator that populates the secret values on the destination cluster.

## Best Practices

### Make Hydration Deterministic

### Enable Branch Protection

Argo CD should be the only thing pushing hydrated manifests to the hydrated branches. To prevent other tools or users
from pushing to the hydrated branches, enable branch protection in your SCM.

It is best practice to prefix the hydrated branches with a common prefix, such as `environment/`. This makes it easier 
to configure branch protection rules on the destination repository.