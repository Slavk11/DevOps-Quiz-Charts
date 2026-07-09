# DevOps-Quiz-Charts

Helm charts for the [DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz)
platform: an **umbrella chart** that deploys all 12 application services —
9 Go microservices and 3 frontend apps — with one command.

```bash
helm install devops-quiz ./umbrella -f values/prod.yaml
```

> Cluster and cloud resources are provisioned by
> [DevOps-Quiz-Terraform](https://github.com/Slavk11/DevOps-Quiz-Terraform);
> in-cluster dependencies (CockroachDB, Redis, NATS, Consul, ingress) come from
> [DevOps-Quiz-Infra](https://github.com/Slavk11/DevOps-Quiz-Infra).
> This repo owns the **application layer** only.

## Structure

```
.
├── umbrella/                 # top-level chart: the whole platform
│   ├── Chart.yaml            # lists all services as dependencies
│   └── values.yaml           # cross-service defaults
├── charts/
│   ├── microapi/             # API gateway
│   ├── quiz/                 # quiz domain service
│   ├── leads/                # leads service
│   ├── users/                # users service
│   ├── notif/                # notifications
│   ├── jobber/               # scheduled & background tasks
│   ├── chrome/               # headless-chrome screenshots
│   ├── show/                 # traffic routing + widget serving
│   ├── uploader/             # file uploads
│   ├── app/                  # React user account
│   └── land/                 # Hugo promo site
└── values/
    ├── prod.yaml             # production overrides
    ├── dev.yaml              # dev environment
    └── review.yaml           # template for dynamic review envs
```

Each service chart follows the same conventions (image, resources, probes,
service, ingress where applicable) — learning one chart means understanding
all of them.

## How deployment works

- **One release per environment.** `helm upgrade --install` of the umbrella
  chart is the only deploy command; CI calls it with the right values file
- **Per-environment values.** `prod` / `dev` / `review-*` differ only in
  values: replicas, resources, domains, image tags — never in templates
- **Review environments.** CI deploys the same umbrella chart into a
  per-branch namespace with `review.yaml` + a generated domain
  (`<branch>.devops-quiz.com`), then `helm uninstall` on merge
- **Rollback.** `helm rollback` returns the previous application version;
  DB migrations are written to be backward-compatible, so rolling back the
  app never requires rolling back the schema

## Conventions every chart follows

| Concern | Convention |
|---|---|
| Probes | liveness + readiness for every service; startup probe for slow starters |
| Resources | requests & limits set explicitly — no unbounded pods |
| Config | env-specific config via values; secrets referenced, never stored |
| Ingress | only `microapi`, `show`, `app`, `land` are exposed; the rest are cluster-internal |
| Naming | one chart = one service = one image, named after the service |

## Usage

```bash
# render manifests locally without installing
helm template devops-quiz ./umbrella -f values/dev.yaml

# dry-run against the cluster
helm upgrade --install devops-quiz ./umbrella -f values/prod.yaml --dry-run

# real deploy (what CI runs)
helm upgrade --install devops-quiz ./umbrella -f values/prod.yaml --atomic --timeout 10m
```

`--atomic` rolls the release back automatically if any service fails to become
ready — a half-deployed platform is never left behind.

## Design decisions

- **Umbrella over 12 separate releases** — one versioned unit for the whole
  platform: environments stay consistent, and "what's deployed" has a single
  answer
- **Values-only environment differences** — templates are identical across
  environments, which makes review envs cheap and prod predictable
- **Explicit exposure** — internal services aren't reachable from outside by
  design; the attack surface is 4 endpoints, not 12
- **Backward-compatible migrations as a contract** — this is what makes
  `helm rollback` safe and zero-touch releases possible

---

Part of the **[DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz)** platform ·
[Terraform](https://github.com/Slavk11/DevOps-Quiz-Terraform) ·
[Infra](https://github.com/Slavk11/DevOps-Quiz-Infra) ·
[CI library](https://github.com/Slavk11/DevOps-Quiz-CI-Lib) ·
[GitLab Runner](https://github.com/Slavk11/DevOps-Quiz-Gitlab-Runner) ·
[Frontend](https://github.com/Slavk11/DevOps-Quiz-Frontend)
