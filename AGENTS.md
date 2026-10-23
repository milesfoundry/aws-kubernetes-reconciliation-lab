# Agent instructions — aws-kubernetes-reconciliation-lab

A Kubernetes reconciliation **lab**: Git-driven controller reconciliation,
observable failure/recovery, and eventual audited deletion of a temporary AWS
environment. Companion to the GCP Platform Delivery Lab. This repository is
mostly contracts, validators and local experiments; **no AWS resource is
provisioned by current code**, and nothing here is live lifecycle proof.

## Estate engineering foundation

The [estate constitution](https://github.com/milesfoundry/iac-platform/blob/main/CONSTITUTION.md) applies to this lab as well as the infrastructure
and agent repositories. Use its ownership/boundary, reproducibility/recovery,
evidence and bounded-review principles. Local provider contracts, commands and
runbooks below supply the implementation details. The canonical document is an
internal contributor reference and may require organization access.

Applicability does not grant CI admission, cloud credentials, spending or provider
mutation authority. The TAL-221 CI-control decision is complete (23 October 2026);
TAL-194 provider work remains open, so preserve its existing gates.

## Repository map

| Path | What it is | Notes |
| --- | --- | --- |
| `workload/source.json` | Shared workload lock: GCP source repo, revision, directory, published image digest | Single source of truth; validated by `scripts/check-source.py` |
| `lifecycle/*.py` | Offline contracts: run-intent preflight, EKS render, budget alerts, notification, delivery/observation validators | Unit tests live beside them |
| `scripts/local-reconcile.sh` | Disposable local kind + Flux reconcile/drift test | Creates and deletes cluster `portfolio-reconcile` only |
| `scripts/install-local-tools.sh` | Checksum-verified tool bootstrap (kind, kubectl, flux) into a caller-supplied absolute directory | Linux only; never pipes a remote script to a shell |
| `scripts/qualify-workload-gitops.py` | Hosted qualification driver for promote/rollback/deny paths | Produces `output/workload-gitops.json` |
| `scripts/check-eks-schemas.py` | Pinned CAPI/CAPA CRD schema check for rendered desired state | Downloads public artifacts, cached in `.cache/` |
| `management/` | Flux kustomizations and rendered resources (local, compatibility pins, workload-test) | Manifest data, not generated output |
| `bootstrap/` | kind cluster definitions | `kind.yaml` for local contract, `workload-kind.yaml` for workload tests |
| `docs/` | Architecture, runbook, EKS compatibility, workload delivery, observations | `docs/observations/<sha>/` retains per-run evidence |
| `.github/workflows/` | `validate.yaml` (contracts + local reconcile; disabled), `qualify-workload.yaml` (manual dispatch; paused) | YAML still declares GitHub-hosted `ubuntu-24.04` labels; the disabled Validate workflow does not run |

## Setup and offline checks

Run from the repository root. These checks need no AWS credentials, provider
mutation or cluster. Unit and shell checks can run offline; dependency and CRD
schema downloads require network access.

```sh
python3 -m unittest discover -s lifecycle -v
bash -n scripts/*.sh
shellcheck scripts/*.sh
python3 -m pip install PyYAML==6.0.2 jsonschema==4.23.0   # versions pinned by CI
python3 scripts/check-eks-schemas.py
python3 scripts/check-source.py
```

Contract validators (`run-intent.json` is gitignored; format in `docs/runbook.md`):

```sh
python3 lifecycle/preflight.py run-intent.json
python3 lifecycle/render_eks.py run-intent.json \
  --kubernetes-version <minor> --operator-cidr <cidr> --instance-type <type> \
  --control-plane-role <role> --node-role <role> --identity-name <name>
python3 scripts/validate-workload-observation.py output/workload-gitops.json
```

Heavy local experiment (needs Docker, network and spare capacity; it creates a
disposable kind cluster and deletes it in an EXIT trap):

```sh
bash scripts/install-local-tools.sh "$PWD/.tools"
export PATH="$PWD/.tools:$PATH"
bash scripts/local-reconcile.sh "$(git rev-parse HEAD)"
```

Flux fetches that exact commit from the public repository, so it must already be
pushed. Never point this at an existing cluster or reuse `portfolio-reconcile`.

## Architectural constraints

- Contracts stay separate from effects: validators and renderers take inputs and
  return structured output; only scripts that say so create clusters or touch
  controllers.
- The shared workload lock (`workload/source.json`) pins one external source
  revision and one published image digest. Do not fork the application into this
  repository; change the lock or leave it alone.
- Rendered EKS output is **pinned-CRD-schema-checked desired state only**.
  Admission, CAPI/CAPA controllers, IAM, ACK S3/SQS, quotas and audit coverage
  remain unqualified.
- Pinned tools, from source: kind `v0.33.0`, kubectl `v1.35.8`, Flux `v2.9.6`,
  kindest node `v1.35.8`, CAPI `v1.13.4`, CAPA `v2.13.1`, PyYAML `6.0.2`,
  jsonschema `4.23.0`; do not recommend an unpinned upgrade.
- The budget preflight, alert payloads and run-intent expiry are validators only:
  they are not spending approval, identity verification, a hard cap or an
  operational janitor.
- Hosted qualification evidence is single-machine, simulated-zone evidence. It
  never proves cloud-zone HA, AWS identity or cost enforcement.

## Deployment and runner authority

- No deployment path exists: nothing applies the cloud render; the local Flux
  source is `management/local` only. Live create, workload, export, delete and
  independent teardown audit remain open milestones — never claim them from
  source contracts. The full provider lifecycle remains blocked by **TAL-194**.
- `validate.yaml` declares `push` and `pull_request` triggers on GitHub-hosted
  `ubuntu-24.04`; its `local-reconciliation` job is gated to pushes to `main`.
  On 23 October 2026 the owner approved disabling Validate, and a Codex API
  readback confirmed `disabled_manually`. The YAML still declares hosted labels,
  but the disabled workflow is not running.
- `qualify-workload.yaml` remains manual dispatch and requires immutable,
  previously validated baseline/candidate revisions. Qualification is paused
  pending the isolated self-hosted replacement and its successful push/`main`
  Validate evidence dependency; do not remove or weaken those gates. No
  replacement workflow exists yet, and disabled checks are not passed checks.
- The estate self-hosted pool is not configured for this repository, and
  public-runner policy is unresolved. Do not change repository visibility or
  trust settings and do not introduce a hosted fallback or ephemeral per-change
  requirement.
- **The TAL-221 CI-control decision is complete** (23 October 2026): the owner
  approved disabling the Validate workflows in both labs, noting they are not production.
  Retained `docs/observations/<sha>/` hosted evidence remains historical and
  does not show current workflow triggers. TAL-194 provider work remains a
  separate concern.

## Secrets and preservation

No credentials are needed for any check here; the workflow YAML uses a read-only token with
`persist-credentials: false`. `.env`, `.tools/`, `output/`, `.cache/`,
`shared-source/`, `run-intent.json` and Terraform state/vars are gitignored.
Preserve other contributors' dirty checkouts and the retained
`docs/observations/<sha>/` evidence; do not rewrite accepted observations.
