# gitops-supply-chain

The application layer for the [terraform-azure-platform](https://github.com/bertrandmbanwi/terraform-azure-platform)
ephemeral cluster. Terraform owns the infrastructure; Argo CD owns everything
that runs inside Kubernetes, and this repository is what Argo watches.

The cluster is ephemeral (it exists for minutes per CI run), so Argo is
installed fresh on every lifecycle, applies the root application below, syncs
whatever this repo declares, gets verified, and is destroyed with the cluster.
Nothing here assumes a long-lived control plane.

## Layout (app-of-apps)

```
bootstrap/
  root-app.yaml          Argo Application that watches apps/ (the "app of apps")
apps/
  hello.yaml             Argo Application that deploys workloads/hello
workloads/
  hello/
    namespace.yaml       hello namespace, Pod Security Standard: restricted
    deployment.yaml      nginx-unprivileged, non-root, read-only root fs
    service.yaml         ClusterIP on port 80 to container port 8080
```

The flow is two levels. CI applies `bootstrap/root-app.yaml`. That root
application points at `apps/` and creates one child Application per file it
finds there. Each child application points at a directory under `workloads/`
and syncs the plain Kubernetes manifests in it. Adding a new app is therefore
a matter of dropping a manifest set under `workloads/` and an Application that
references it under `apps/`; the root app picks it up with no pipeline change.

All applications use automated sync with prune and self-heal, so the cluster
state always converges to what is committed here.

## How it is consumed

This repo is not deployed by hand. The platform repo's
`apply-verify-destroy` workflow installs Argo CD via Helm, applies
`bootstrap/root-app.yaml`, waits for each application to report Synced and
Healthy, and runs an in-cluster HTTP check before capturing evidence and
destroying everything.

This repository must remain public for now: Argo pulls it anonymously during
the lifecycle. Private-repo access with scoped credentials is a planned
refinement.

## Conventions

- Workloads target the restricted Pod Security Standard. Pods run as non-root
  with a read-only root filesystem, all Linux capabilities dropped, and the
  RuntimeDefault seccomp profile. Scratch paths are backed by emptyDir.
- Container images are pinned. Tags are reused upstream, so production-grade
  pins should resolve to a digest.
- Resource requests and limits are set on every workload.

## Roadmap

This repo starts as a sync-and-serve proof and grows one layer at a time:

1. (current) Argo bootstraps and syncs a trivial workload inside the
   ephemeral window
2. Kyverno admission policies governing what is allowed to run
3. A self-built application image, signed keyless with cosign, with Kyverno
   enforcing the signature and rejecting unsigned images
4. External Secrets Operator pulling from Azure Key Vault via workload
   identity, with no secrets stored in the cluster or CI
5. SLSA provenance attestation on the built image
