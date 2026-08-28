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
and return short verdicts through the `task` result; only bulk evidence goes to
`evidence/<domain>.md`. The critic receives the hypothesis alone — no reasoning,
no attribution — and must confirm or refute it with its own tool calls. The
commander then files the report through a tool, so a converged investigation
cannot end with nothing delivered.

Read-only twice over: the credential grants only get/list/watch, and filesystem
permissions deny writes outside `evidence/`.

Nothing about the cluster is hardcoded. `discovery.py` measures the version,
per-node architecture, ingress controller (and derives that controller's 502 vs
503 semantics), storage provisioners, CSI workload names, which observability and
GitOps systems are *absent*, and the restart-cohort baseline — which the critic
is then told to reject as a non-explanation.

## Install

```bash
python3 -m venv .venv
.venv/bin/pip install git+https://github.com/azyphon/swarmr-lib
.venv/bin/pip install git+https://github.com/azyphon/swarmr-k8s-incident
```

**Core first, and in that order.** This package declares `swarmr>=0.1,<0.2`,
which pip resolves against an index; `swarmr` is not published to one, so
installing this package into an empty environment fails with `No matching
distribution found for swarmr`. Installing core from git first satisfies the
requirement, and pip leaves the already-installed copy alone. The constraint
stays a version rather than a direct git URL so that publishing to an index
later needs no change here.

Both must end up in the same environment: the `teams` CLI and `teams-mcp`
server come from `swarmr` and pick this team up from installed metadata.

```bash
teams --list                # k8s_incident
teams --target k8s_incident # profile the cluster, then exit
teams k8s_incident "payments.demo.local returns 502, namespace demo"
```

Over MCP the tool is `start_k8s_incident`.

## Credentials

This team needs a **read-only** credential. It refuses your ambient kubeconfig,
which is usually cluster-admin:

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
before it opens a connection, and mints a new one when it has less than five
minutes left — using your admin credentials for that context, exactly as
`incident-credentials` does, and announcing it on stderr. So a credential going
stale overnight is not something you handle; it used to surface as a 401 raised
sixty frames deep in the generated Kubernetes client.

Three limits keep that from becoming a surprise. Only files this team minted are
ever rewritten, so an `INCIDENT_KUBECONFIG` of your own is reported, never
replaced. A refreshed credential that can do more than read is refused rather
than used. And the refresh never settles an ambiguous cluster choice: with
several credentials and no selection, it is still an error, because refreshing
all of them would be picking a cluster to investigate. `INCIDENT_NO_REFRESH=1`
turns it off and restores the "run this command" error.

## Structure

```
src/swarmr_k8s_incident/
├── __init__.py        declares TEAM and its vocabulary, cheap to import
├── agent.py           commander + 5 subagents, filesystem permissions
├── prompts.py         prompt templates, no cluster facts inside
├── discovery.py       profiles the live cluster at build time
├── tools.py           broad reads: k_get, k_events, k_logs, k_top
├── describe.py        the deep single-object read, status + events
├── registry.py        image reference parsing, OCI manifest lookup
├── client.py          credential resolution and API clients
├── kinds.py           kind aliases -> API resource
├── output.py          pruning, byte cap, guard + cache
├── projection.py      object -> compact row, log cleaning
├── digest.py          one-line summaries, and what counts as an error
├── rbac.py            the read-only rule set and its manifest
├── credentials.py     mints the read-only kubeconfig
├── credentials_cli.py the incident-credentials entrypoint
├── report_tool.py     the report-filing tool
├── redaction.py       what never reaches a delivered report
├── tests/             this team only
└── demo/              staged faults
```

Everything this team is lives in this one package: code, tests, RBAC and demo
manifests. Its only dependency on the harness is `swarmr`'s ABI — `Team`,
`RunContext`, `TeamBuild`, `Lazy` and the `swarmr.teams` entry-point group.

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
python3 -m venv .venv
.venv/bin/pip install -e ../swarmr-lib -e ".[dev]"   # core from a sibling checkout
.venv/bin/python -m pytest                            # no cluster, no model calls
INCIDENT_E2E=1 .venv/bin/python -m pytest -s          # adds a live investigation
.venv/bin/ruff check . && .venv/bin/pyright
```

`tests/test_contract.py` is the important one: it pins the vocabulary `swarmr`
reads off this team, that the heavyweight fields stay behind `Lazy`, and that
publishing the MCP surface imports none of this package's agent stack.
