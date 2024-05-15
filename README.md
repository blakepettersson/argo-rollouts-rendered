# argo-rollouts-rendered

This is an example repo which demonstrates usage of the [Rendered Manifests](https://akuity.io/blog/the-rendered-manifests-pattern/) 
pattern. For this we use [Kargo Render](https://kargo-render.akuity.io/).

How this works is that on every merge to `main`, a Github Action runs which renders the output of 
`charts/argo-rollouts` into an `env/test`, an `env/staging` and an `env/prod` branch. For `env/staging` and `env/prod`, it
does not immediately merge into those branches, rather it creates PRs going into those branches so that they can be 
reviewed before the actual merge.

This is configured mainly in `kargo-render.yaml`. Here's an example with the `env/test branch`:

```yaml
# kargo-render.yaml
branchConfigs:
  # The branch to render to. There's a `branchConfig` for each branch we want to render to
  - name: env/test
    appConfigs:
      # Rollouts app, multiple apps can be configured and rendered to the same branch - useful for e.g. mono-repos.
      rollouts:
        configManagement:
          path: charts/argo-rollouts
          helm:
            releaseName: argo-rollouts
            valueFiles:
              - values-test.yaml
        # The path the rendered output of `charts/argo-rollouts` should be rendered to on `env/test`
        outputPath: rollouts-rendered
    prs:
      enabled: false # Whether PRs should be created for this branch; for env/test we instead merge directly
```

The other part of the configuration is in the Github Action itself:

```yaml
# Required permissions for Kargo Render
permissions:
  contents: write
  pull-requests: write

on:
  push:
    branches: ['main']
  workflow_dispatch:

jobs:
  render-manifests:
    name: Render and Commit Manifests (or submit a PR)
    runs-on: ubuntu-latest
    strategy:
      matrix:
        targetBranch:
          - env/test
          - env/staging
          - env/prod
    steps:
      - uses: blakepettersson/kargo-render-action@blake-test # This has the most current version of Kargo Render
        with:
          personalAccessToken: ${{ secrets.MY_PAT }}
          targetBranch: ${{ matrix.targetBranch }}
```

To try this out, an ApplicationSet can be used which refers to the rendered manifest branches:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: rollouts
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          # Cluster names to install Argo Rollouts into, replace list with your own cluster names
          - name: blake-dev
            stage: test
          - name: blake-staging
            stage: staging
          - name: blake-prod
            stage: prod            
  template:
    metadata:
      name: '{{ .name }}-rollouts'
    spec:
      project: default
      source:
        repoURL: 'https://github.com/blakepettersson/argo-rollouts-rendered'
        targetRevision: 'env/{{ .stage }}'
        path: rollouts-rendered
      destination:
        namespace: argo-rollouts
        name: '{{ .name }}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        retry:
          limit: 5
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 3m0s
        syncOptions:
          - CreateNamespace=true
```
