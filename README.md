# DevOps-Quiz-Charts

Helm charts for the [DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz)
platform — built around **one universal service chart** instead of a dozen
copy-pasted ones.

Two charts deploy the entire platform:

| Chart | Deploys |
|---|---|
| **`service-chart`** | Any platform service — the same templates, parameterized per service via values |
| **`deps-chart`** | Platform dependencies |

Every microservice differs only by its values file. Adding a new service to
the platform means **adding one YAML file** — not writing a new chart.

## Structure

```
.
├── service-chart/            # universal application chart
│   ├── templates/            # deployment, service, ingress, probes — one set for all
│   ├── Chart.yaml
│   └── values.yaml           # sane defaults every service inherits
├── deps-chart/                # platform dependencies chart
│   ├── templates/
│   ├── Chart.yaml
│   └── values.yaml
└── values/
    ├── deps/
    │   └── values.yaml       # dependency configuration
    └── services/             # one file = one deployed service
        ├── app.yaml          # React user account
        ├── land.yaml         # Hugo promo site
        ├── quiz.yaml
        ├── leads.yaml
        ├── users.yaml
        ├── notif.yaml
        ├── jobber.yaml
        ├── uploader.yaml
        ├── show.yaml         # traffic routing + widget serving
        ├── show-v1.yaml      #   ├─ stable version
        ├── show-v2.yaml      #   ├─ next version
        └── show-canary.yaml  #   └─ canary — takes a slice of live traffic
```

## Deployment model

Each service is its own Helm release from the shared chart:

```bash
# any service — same chart, different values
helm upgrade --install quiz  ./service-chart -f values/services/quiz.yaml
helm upgrade --install leads ./service-chart -f values/services/leads.yaml

# dependencies
helm upgrade --install deps  ./deps-chart    -f values/deps/values.yaml
```

In practice these commands are executed by the
[CI library](https://github.com/Slavk11/DevOps-Quiz-CI-Lib) — each service's
pipeline deploys only its own release, so a release of `quiz` can never
break the release state of `leads`.

## Canary releases

The `show` service handles live visitor traffic (it serves the embedded quiz
widget), so it releases through a **canary flow** instead of a plain rollout:

```
show-v1 (stable) ──┐
                   ├── traffic split ── users
show-canary  ──────┘      │
     ▲                    │ metrics look good?
     └── show-v2 promoted ┘
```

- `show-v1` / `show-v2` — versioned releases running side by side
- `show-canary` — receives a controlled slice of real traffic first
- Promotion = shifting traffic, not redeploying; rollback = shifting it back,
  which takes seconds and loses nothing

The riskiest service in the platform is released in the safest way.

## Conventions baked into service-chart

| Concern | Convention |
|---|---|
| Probes | liveness + readiness for every service |
| Resources | requests & limits are explicit — no unbounded pods |
| Config | per-service env via values; secrets referenced, never stored |
| Exposure | only public-facing services get ingress; the rest stay cluster-internal |

Because conventions live in the chart, no service can "forget" its probes or
resource limits — the template simply enforces them.

## Design decisions

- **One chart, many values** — 12 services with zero template duplication;
  a probe fix or a securityContext change lands in every service at once
- **Release per service** — independent deploy and rollback per service,
  matching the microservice ownership model
- **Canary where it matters** — full canary machinery only for the
  traffic-facing service; internal services use plain rolling updates.
  Complexity is spent where the risk is
- **Values as the single source of truth** — "how is `notif` configured?"
  is answered by one file under `values/services/`

---

Part of the **[DevOps-Quiz](https://github.com/Slavk11/DevOps-Quiz)** platform ·
[Terraform](https://github.com/Slavk11/DevOps-Quiz-Terraform) ·
[Infra](https://github.com/Slavk11/DevOps-Quiz-Infra) ·
[CI library](https://github.com/Slavk11/DevOps-Quiz-CI-Lib) ·
[GitLab Runner](https://github.com/Slavk11/DevOps-Quiz-Gitlab-Runner) ·
[Frontend](https://github.com/Slavk11/DevOps-Quiz-Frontend)
