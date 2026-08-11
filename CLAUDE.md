# home-ops

Single-node Talos + Flux homelab cluster, generated from
[onedr0p/cluster-template](https://github.com/onedr0p/cluster-template).

**This repository is public.** Nothing unencrypted that matters may be committed.
Secrets live in `*.sops.yaml` (age-encrypted) and in gitignored local files.

## The one rule that matters

**`kubernetes/`, `talos/` and `bootstrap/` are generated output.** Never
hand-edit them, and never restructure the generated kustomizations.

Edit `cluster.toml` (or `template/`), then:

```sh
mise exec -- just configure
```

Hand-editing `kubernetes/apps/kustomization.yaml` away from the template's
`components:` pattern is what destroyed a previous attempt at this cluster: it
broke SOPS decryption inheritance to child Kustomizations, so every app looked
for a `sops-age` secret in its own namespace and never found one.

New applications are the exception — they are added as new directories under
`kubernetes/apps/<namespace>/<app>/`, following the shape of the existing ones.

## Toolchain

All tooling is pinned in `.mise/config.toml` and is not on `PATH` otherwise:

```sh
mise exec -- kubectl get pods -A
mise exec -- flux get ks -A
mise exec -- talosctl -n <node> dashboard
```

`KUBECONFIG`, `TALOSCONFIG` and `SOPS_AGE_KEY_FILE` are set by mise and only
resolve from inside this repo — `cd` here first.

**Git hooks need the mise environment too.** `lefthook` runs `oxfmt` and `just`
on commit, so use `mise exec -- git commit`; a bare `git commit` fails with
`oxfmt: not found`.

## Secrets

Never read or print these — reading a secret into an assistant's context is what
exposes it:

`cluster.toml` · `age.key` · `deploy.key` · `flux-webhook-token.txt` ·
`cloudflare-tunnel.json` · `1password-credentials.json`

Inspect config without the secret: `grep -v '^token' cluster.toml`.
Confirm the token is populated: `grep -q '^token = ".\+"' cluster.toml`.

Before every push, verify nothing sensitive became tracked:

```sh
for f in cluster.toml age.key deploy.key cloudflare-tunnel.json 1password-credentials.json; do
  printf '%-30s ' "$f"; git check-ignore -q "$f" && echo IGNORED || echo "*** TRACKED ***"
done
git diff --name-only | grep '\.sops\.yaml$' | xargs -r grep -L 'ENC\[AES256_GCM'   # must be empty
```

`.sops.yaml` itself is config, not a secret — it holds only the age public key.
Every `just configure` rewrites the `*.sops.yaml` files with fresh nonces, so
they always show as modified; that is expected.

## Workflow

```sh
cd /opt/k8s-migration/home-ops
# edit cluster.toml or add kubernetes/apps/<ns>/<app>/
mise exec -- just configure
# audit (above)
mise exec -- git add -A && mise exec -- git commit -m "feat(<scope>): ..."
mise exec -- git push
```

A push webhook notifies the cluster, so reconcile starts within a second or two
of the push — there is normally no need to run `flux reconcile source git
flux-system` by hand. Reach for it only when the webhook itself is suspect;
GitHub records every attempt under the repository's webhook deliveries, and a
non-200 there is the first thing to check.

Commit messages: conventional style, **no AI attribution**, and no
infrastructure detail that is not already in the committed files.

## Verification

```sh
mise exec -- flux get ks -A | awk 'NR==1 || $5!="True"'
mise exec -- flux get hr -A | awk 'NR==1 || $4!="True"'
mise exec -- kubectl get pods -A | grep -vE 'Running|Completed'
```

Do **not** wait on readiness with `flux get ks --status-selector ready=false` —
in-progress resources report `Unknown`, not `false`, so it reports success
immediately. Use `kubectl wait --for=condition=Ready` against the specific
object instead.

## Cluster facts

Single control-plane node, schedulable, all workloads on it. A NUC reboot or
Talos upgrade is a full outage — this is accepted, not an oversight. Adding
control planes later is a `cluster.toml` edit plus `just configure`.

|          |                                                               |
| -------- | ------------------------------------------------------------- |
| Node     | `talos-01` · `10.0.0.156` · disk `/dev/sda`                   |
| API VIP  | `10.0.0.50` (`k.x00.sh`, via a DNS override on the router)    |
| Gateways | internal `10.0.0.51` · external `10.0.0.52` · DNS `10.0.0.53` |
| Storage  | node-local NVMe for state; NFS for media and backups          |
| Ingress  | Cloudflare tunnel, created with the **CLI**                   |

**Tunnels must be created with `cloudflared tunnel create`, not the dashboard.**
The template needs `cloudflare-tunnel.json` in `AccountTag`/`TunnelID`/
`TunnelSecret` form and derives both the `TUNNEL_TOKEN` and
`kubernetes/apps/network/cloudflare-tunnel/app/dnsendpoint.yaml` from it.
Dashboard-created tunnels are remotely managed, so cloudflared would ignore the
in-repo ingress rules and take its config from Cloudflare — moving ingress out
of Git.

Internal name resolution depends on split-horizon DNS on the router forwarding
the cluster domain to `10.0.0.53`. Hosts under that domain which are _not_
cluster apps need explicit exceptions there.

## Migration context

This cluster is replacing a docker-compose stack that is still running. The
plan, phase gates and cutover procedure live outside this repo, in
`../README.md` — read it before migrating any service.
