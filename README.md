# César Zubillaga Beraza — Platform & DevOps Engineer

Pamplona / Iruña, Spain · working fully remote · willing to travel · open to EU / US
[info@zbc.space](mailto:info@zbc.space) ·
[LinkedIn](https://www.linkedin.com/in/cesar-zubillaga-beraza) ·
[zubillagaberazacesar.com](https://zubillagaberazacesar.com)

Twelve years building software, the last six keeping it running. I came into platform
engineering from application development and never put the code down, so I debug the
service and the cluster underneath it with the same hands: Kubernetes, GitOps, Terraform
and observability on one side; Python, TypeScript and PHP on the other.

**The CV is at [zubillagaberazacesar.com](https://zubillagaberazacesar.com)** — or open [`index.html`](index.html) locally, and print it to PDF.

## Incident write-ups

Three failures from the Kubernetes cluster I run for a personal trading platform,
written up the way I would write them up for a team — including what I thought was
broken before I found what actually was.

| | |
|---|---|
| [422 restarts from a single character](postmortems/k3s-eviction-loop.md) | A `=` where the kubelet wanted a `<` put k3s in a restart loop for six hours, while the workloads kept serving traffic. |
| [The API that its own liveness probe killed](postmortems/api-killed-by-its-own-liveness-probe.md) | Thousands of "no free DB connection" lines, every one of them a symptom. The trigger was a burst from the frontend against a synchronous handler. |
| [An SSO login page served as a container image](postmortems/cloudflare-access-vs-oci-registry.md) | Cloudflare Access answering `/v2/` with a redirect broke Flux and Trivy at once. |

## Current stack

| | |
|---|---|
| Orchestration | Kubernetes, k3s, Helm, Helmfile, Docker, Traefik |
| Infrastructure as code | Terraform, Ansible, Kustomize |
| GitOps & CI/CD | ArgoCD, Flux, Jenkins, Bitbucket Pipelines, GitHub Actions, Gitea Actions |
| Observability | Prometheus, Grafana, Alertmanager, Datadog, EFK/Kibana, Jaeger |
| Security | Trivy, Kyverno, Sealed Secrets, AWS Secrets Manager, Cloudflare Tunnel + Access |
| Languages | Python, TypeScript, PHP, SQL, Bash |
| Data | PostgreSQL, MySQL, MongoDB, Redis, Elasticsearch |
| AI in production | LLM agents with quota and audit controls, Anthropic API, agent review loops |
