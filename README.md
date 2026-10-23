# AWS Kubernetes Reconciliation Lab

**Engineering foundation:** the shared [estate constitution](https://github.com/milesfoundry/iac-platform/blob/main/CONSTITUTION.md) applies here; local checks and provider limits are documented below. The constitution is an internal contributor reference and may require organization access.

[![Validate](https://github.com/milesfoundry/aws-kubernetes-reconciliation-lab/actions/workflows/validate.yaml/badge.svg)](https://github.com/milesfoundry/aws-kubernetes-reconciliation-lab/actions/workflows/validate.yaml)

A Kubernetes reconciliation lab: Git-driven controller reconciliation, observable
failure/recovery, and eventual **audited deletion** of a temporary AWS environment.
Companion: [GCP Platform Delivery Lab](https://github.com/milesfoundry/gcp-platform-delivery-lab).

**Status: historical hosted application/GitOps qualification, not an EKS implementation.**
The EKS rendering spike checks a bounded desired-state set against pinned
CAPI/CAPA CRD schemas; controller and cloud qualification are still pending.
The retained hosted kind/Flux run demonstrated application delivery, configuration
promotion/rollback, drift repair, direct permission/controller denial and one
network allow/deny path. The [qualified twelve-job record](docs/observations/d418d3f/qualification.md)
preserves exact inputs, failed attempts and limits; it is one-machine,
local-path evidence. CAPI/CAPA, ACK S3/SQS, AWS identity, independent teardown
audit and TTL janitor remain explicit next milestones. No AWS resources are
provisioned by current code. The TAL-221 CI-control decision is complete: on
23 October 2026 the owner approved disabling Validate, and a Codex API readback
confirmed `disabled_manually`. The YAML still declares hosted `ubuntu-24.04`
labels, but the disabled workflow does not run and disabled checks are not
passed checks. Qualification stays manually triggered but is paused pending the
isolated self-hosted replacement and its successful push/`main` Validate
evidence dependency, with its gates unchanged. The full cloud lifecycle (apply,
workload, teardown, independent audit) remains blocked by TAL-194.

## Credential-free checks

Requirements: Python 3, `shellcheck` and `bash` for the static checks; the schema
check downloads public release artifacts. Run from the repository root. None of
this needs AWS credentials, a cloud call or a cluster.

```sh
python3 -m unittest discover -s lifecycle -v
bash -n scripts/*.sh
shellcheck scripts/*.sh
python3 -m pip install PyYAML==6.0.2 jsonschema==4.23.0   # CI-pinned versions
python3 scripts/check-eks-schemas.py
python3 scripts/check-source.py
```

The validator contracts take a local intent or observation file. The preflight
also validates expiry and a positive per-run gross estimate — it is **not** AWS
identity verification, spending approval, an operational janitor or a hard cap:

```sh
# run-intent.json is gitignored; format in docs/runbook.md (expiry within 60 minutes)
python3 lifecycle/preflight.py run-intent.json
python3 lifecycle/render_eks.py run-intent.json \
  --kubernetes-version <minor> --operator-cidr <cidr> --instance-type <type> \
  --control-plane-role <role> --node-role <role> --identity-name <name>
```

The render prints a Kubernetes List that still has to be reviewed and retained;
see [EKS compatibility](docs/eks-compatibility.md) and the
[runbook](docs/runbook.md).

## Run the local reconciliation test

Requirements: Linux amd64/arm64, Docker, curl, tar, sha256sum, network access and
spare capacity for a disposable Kubernetes cluster. The `validate.yaml` YAML
declares this job on GitHub-hosted `ubuntu-24.04` for pushes to `main`, but the
workflow is disabled, so no hosted run occurs; run it locally only when you have
the capacity. Never point this experiment at an existing cluster.

```sh
git clone https://github.com/milesfoundry/aws-kubernetes-reconciliation-lab.git
cd aws-kubernetes-reconciliation-lab
bash scripts/install-local-tools.sh "$PWD/.tools"
export PATH="$PWD/.tools:$PATH"
bash scripts/local-reconcile.sh "$(git rev-parse HEAD)"
```

Use a commit already pushed to the public repository: Flux fetches that exact SHA.
The script creates only `portfolio-reconcile`, refuses to overwrite an existing
cluster with that name, installs pinned Flux controllers, reconciles a ConfigMap,
changes it out-of-band, then asserts Flux repaired it and applied the correct
source revision. An EXIT trap removes that disposable cluster and its kubeconfig.
If the process is forcibly killed, inspect `kind get clusters` and deliberately
remove only `portfolio-reconcile` after confirming it is this test's cluster.

## Run the shared application

The GCP repo's [workload quickstart](https://github.com/milesfoundry/gcp-platform-delivery-lab#try-the-application-locally)
is the single source of Report Workshop. [workload/source.json](workload/source.json)
is the lock: it pins the source revision and the published image digest instead of
maintaining a divergent copy. To reproduce that exact application version:

```sh
git clone https://github.com/milesfoundry/gcp-platform-delivery-lab.git
cd gcp-platform-delivery-lab
git checkout --detach 7e695d6791de82b9230e7ce3ab3621f6e4a56b72   # lock revision
cd workload
docker compose up --build -d
python3 smoke.py
docker compose down
```

The lock records that this image index passed Validate plus publish/native-ARM/
hosted-kind HA qualification in
[release run 38659839420](https://github.com/milesfoundry/gcp-platform-delivery-lab/actions/runs/38659839420),
including anonymous AMD64/ARM64 runtime-content verification. Those are real
process tests in simulated zones on one hosted machine — not an AWS deployment,
cloud storage/IAM qualification or physical-zone HA claim.

Earlier pinned sources document the application's local metrics and evidence
recording, durable API/worker tracing, bounded profiling and the dormant
three-instance database profile ([local observability](https://github.com/milesfoundry/gcp-platform-delivery-lab/blob/0e0a6133a18e4cfb10aae4defabd5b8f52ec69b5/docs/local-observability.md),
[application replica and database contracts](https://github.com/milesfoundry/gcp-platform-delivery-lab/blob/0e0a6133a18e4cfb10aae4defabd5b8f52ec69b5/docs/application-replicas.md)),
and the replica/SIGKILL recovery experiment passed
[hosted validation at de55de7](https://github.com/milesfoundry/gcp-platform-delivery-lab/actions/runs/37385769512).
The Validate workflow configuration checks both application and database renders
against pinned schemas; those renders are not connected to the current Flux
source and deploy nothing. The
opt-in hosted experiment uses a derived local fixture with explicit network and
rollout-health extensions and does not activate the cloud source root. See
[opt-in workload delivery](docs/workload-delivery.md) for the release-record
contract; it prepares a blocked candidate, not a running AWS workload.

## Read the design

- [Ephemeral London HA architecture, instance comparison and cost gates](docs/architecture-cost.md)
- [Architecture, ownership and compatibility](docs/implementation.md)
- [Recorded local verification](docs/verification.md)
- [Safety, lifecycle and next cloud gates](docs/runbook.md)
- [EKS rendering and provider compatibility spike](docs/eks-compatibility.md)
- [Shared cloud adapters and opt-in Flux workload delivery](docs/workload-delivery.md)
- [Hosted application GitOps promotion, rollback and permission experiment](docs/application-gitops.md)
- [Agent-facing instructions and pinned tool versions](AGENTS.md)
- [Third-party notices](THIRD_PARTY_NOTICES.md)

The architecture profile describes target capabilities, not completed cloud work.

## Known limits

- Cloud coverage is absent: no CAPI/CAPA controllers, AWS identity, ACK S3/SQS,
  EKS cluster or live lifecycle has been qualified, and nothing here creates them.
- Evidence is single-machine and simulated-zone. Public-runner policy is
  unresolved; the workflows declare GitHub-hosted `ubuntu-24.04` labels and do
  not use the estate self-hosted pool, while the disabled Validate workflow does
  not run. No isolated self-hosted replacement workflow exists yet.
- The render/schema check proves desired-state shape only — not admission,
  conversion behaviour, IAM, quotas or tag/audit coverage.
- The intent preflight and budget alerts validate and notify; they do not enforce
  a spend cap or delete anything by themselves.
- `docs/workload-delivery.md` still calls `2202e139` the current source pin;
  `workload/source.json` is authoritative and pins `7e695d6`.
