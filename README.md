# swarmr-k8s-incident

Kubernetes incident response team for
[swarmr](https://github.com/azyphon/swarmr-lib). Diagnoses a live cluster and
proves the root cause.

```
commander (no cluster tools at all)
├── workload    is the container itself failing?
├── network     can traffic reach a serving backend?
├── storage     is the pod blocked before it ever started?
├── platform    is placement, capacity or node architecture the problem?
└── critic      independently tries to disprove the hypothesis
```

The commander holds no cluster tools by design: its context stays clean and it
cannot fabricate an observation it never received. Investigators run in parallel
and return short verdicts; only bulk evidence goes to `evidence/<domain>.md`.
The critic receives the hypothesis alone — no reasoning, no attribution — and
must confirm or refute it with its own tool calls. The commander then files the
report through a tool, so a converged investigation cannot end with nothing
delivered.

Read-only twice over: the credential grants only get/list/watch, and filesystem
permissions deny writes outside `evidence/`.

Nothing about the cluster is hardcoded. The team profiles the live cluster at
startup — version, per-node architecture, ingress controller (and that
controller's 502 vs 503 semantics), storage provisioners, CSI workload names,
which observability and GitOps systems are *absent*, and the restart-cohort
baseline, which the critic is then told to reject as a non-explanation.

## Install

```bash
uv tool install git+https://github.com/azyphon/swarmr-lib \
  --with git+https://github.com/azyphon/swarmr-k8s-incident \
  --with-executables-from swarmr-k8s-incident
```

That exposes `teams`, `teams-mcp` and `incident-credentials` on your PATH.
`--with-executables-from` is required for the third: `--with` alone installs
this package into the tool environment but only links the *core* package's
commands, leaving `incident-credentials` unreachable. A plain venv works too:

```bash
python3 -m venv .venv
.venv/bin/pip install git+https://github.com/azyphon/swarmr-lib
.venv/bin/pip install git+https://github.com/azyphon/swarmr-k8s-incident
```

**Core first, and in that order.** This package declares `swarmr>=1.0,<2`,
which pip resolves against an index; `swarmr` is distributed from git, not an
index, so installing this package into an empty environment fails with `No
matching distribution found for swarmr`. Installing core first satisfies the
requirement. The constraint stays a version rather than a git URL so that
moving to an index later changes nothing here.

Both must end up in the same environment: the `teams` CLI and `teams-mcp` server
come from `swarmr` and pick this team up from installed metadata.

```bash
teams --target k8s_incident   # profile the cluster, then exit
teams k8s_incident "payments.demo.local returns 502, namespace demo"
```

Over MCP the tool is `start_k8s_incident`.

## Credentials

This team needs a **read-only** credential. It refuses your ambient kubeconfig,
which is usually cluster-admin.

```bash
incident-credentials --list             # contexts, and what is minted
incident-credentials --context archdev  # create RBAC + mint a token
incident-credentials --print-manifest   # the same RBAC as YAML, for GitOps
```

That creates the `incident-reader` ServiceAccount and ClusterRole through the
API, mints an 8h token, writes `.incident-reader.<context>.kubeconfig` at mode
600, then asks the API server to confirm the credential can list pods and cannot
delete them.

**Multiple clusters** get one credential file each. Selection is explicit:
`INCIDENT_KUBECONFIG` (exact path), else `INCIDENT_CONTEXT` (context name), else
the single minted credential. Several minted credentials with no choice
expressed is an error — an investigation reported against the wrong cluster is
worse than one that refuses to start.

**The 8h token refreshes itself.** Every run reads the expiry out of the token
before it opens a connection and mints a new one when under five minutes remain,
announcing it on stderr. A credential going stale overnight used to surface as a
401 raised sixty frames deep in the generated Kubernetes client.

Three limits keep that from becoming a surprise. Only files this team minted are
ever rewritten, so an `INCIDENT_KUBECONFIG` of your own is reported, never
replaced. A refreshed credential that can do more than read is refused rather
than used. And the refresh never settles an ambiguous cluster choice, because
refreshing every credential would be picking a cluster to investigate.
`INCIDENT_NO_REFRESH=1` turns it off.

## Demo faults

`demo/` stages faults with distinct mechanisms, so routing cannot be memorised:

| Manifest | Symptom | Real cause |
| --- | --- | --- |
| `01-arch-mismatch.yaml` | ingress 503 | amd64-only image pinned to arm64 nodes |
| `02-targetport-drift.yaml` | ingress 502 | `targetPort` at a port nothing listens on; pods healthy |
| `03-nfs-mount-stall.yaml` | pod Pending | unreachable NFS export; provisioning never completes |
| `04-image-pull-failure.yaml` | ingress 503 | image tag does not exist in the registry |

```bash
kubectl apply -f src/swarmr_k8s_incident/demo/02-targetport-drift.yaml
teams k8s_incident "payments.demo.local returns 502, namespace demo"
kubectl delete ns demo
```

Two are deliberately near-twins. 02 is the one where `kubectl get pods` shows
`1/1 Running` and tells you nothing. 01 and 04 both show `ImagePullBackOff` and
503, and the tell is whether `platform` distinguishes "no matching platform" from
"tag does not exist" by reading the registry rather than pattern-matching the
mixed-arch cluster.

Run it against a healthy cluster too. The correct answer is "no incident found";
an agent that cannot say that is not usable.

## Development

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -e ../swarmr-lib -e ".[dev]"   # core from a sibling checkout
pytest                                     # no cluster, no model calls
INCIDENT_E2E=1 pytest -s                   # adds a live investigation
ruff check . && pyright
```

Activate the venv rather than calling `.venv/bin/…` directly: pyright resolves
the interpreter from PATH, so unactivated it type-checks against a Python that
has none of the dependencies and reports every import as unresolved. Without
activating, pass `--pythonpath .venv/bin/python`.

Everything this team is lives in one package: code, tests, RBAC and demo
manifests. Its only coupling to the harness is `swarmr`'s ABI — `Team`,
`RunContext`, `TeamBuild`, `Lazy` and the `swarmr.teams` entry-point group.

`tests/test_contract.py` is the important one: it pins the vocabulary `swarmr`
reads off this team, that the heavyweight fields stay behind `Lazy`, and that
publishing the MCP surface imports none of this package's agent stack.
