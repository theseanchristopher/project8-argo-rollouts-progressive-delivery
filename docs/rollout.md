## Blue/Green Example (Documentation Only)

In addition to the canary Rollout used in dev/pre/prod, this repository
includes a standalone Blue/Green example under:

`docs/examples/rollout-bluegreen-example.yaml`

This example demonstrates how Argo Rollouts would be configured for a
Blue/Green deployment strategy, including:

- **activeService**: the live production service
- **previewService**: an isolated preview environment
- **autoPromotionEnabled: false** for manual approval before traffic cutover

This example is **not wired into Kustomize or Argo CD** and is **not applied to
the cluster by default**. It exists only to show understanding of Blue/Green
patterns alongside the fully implemented canary rollout strategy.

