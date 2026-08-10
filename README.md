# nscale-calc — deploy

Kubernetes manifests for **nscale-calc**, a static scale-conversion
calculator for N-scale CAD work. Served internally at
`https://nscale-calc.sirddail.net`.

| | |
|---|---|
| Build repo | [dvystrcil/nscale-calc-docker](https://github.com/dvystrcil/nscale-calc-docker) |
| Image | `harbor.sirddail.net/ai/nscale-calc` |
| Namespace | `nscale-calc` |
| Reachability | internal only — PiHole → istio ingress, no Cloudflare record |

## Layout

```
base/        namespace, deployment, service, harbor pull secret,
             image-updater RBAC, VPA
overlays/    kustomize entry point — image tag lives here
image-updater/  ImageUpdater CR (semver, writes back to overlays/)
```

`overlays/` is the ArgoCD source path and the only place the image tag is
set. Read it first — it overrides anything in `base/`.

## How a change reaches the cluster

The page itself lives in the **build** repo, not here:

1. Edit `site/index.html` in `nscale-calc-docker`, merge to `main`
2. CI builds and pushes `:dev` to Harbor, then cuts a patch release
3. `docker-release` retags `:dev` as `:X.Y.Z`
4. argocd-image-updater writes that tag into `overlays/kustomization.yaml`
5. ArgoCD syncs

Nothing in this repo needs touching for a content change.

## Probes

All three probes target `/healthz`, which the nginx config returns from
memory with no redirect. This is deliberate: kubelet HTTP probes follow
redirects, so probing a path that 302s tests wherever the redirect lands
rather than the path named — the failure behind
[homelab#820](https://github.com/dvystrcil/homelab/issues/820).

## Routing

The HTTPRoute is **not** defined here. External reachability comes from
the `gateway-services` chart as a second source on the ArgoCD Application
in `argocd-projects/nscale-calc/`, per the routing convention.
