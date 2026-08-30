# pipeline-triage-agent

A retrieval-augmented service that reads a failed data-pipeline log and returns a
diagnosed failure mode, the evidence it relied on, and a suggested remediation.

Built to answer a narrow question: when a Databricks job, an ADF activity, or a
Spark executor fails at 3am, can an LLM grounded in a team's own runbook triage
the failure well enough to be worth acting on?

Independent project. Synthetic data, self-authored runbook, no employer data.
Source repository is private. Available on request.

---

## What it does

A POST to `/classify` takes raw error text and the emitting system. It returns a
closed-set failure mode, a confidence score, a line of evidence quoted from the
input, and a remediation step drawn from the runbook.

Given a Spark executor killed by YARN for exceeding memory limits, it returns
`OUT_OF_MEMORY` at 0.92 confidence, quotes the line it relied on, and suggests
raising the executor memory overhead — the remediation the runbook specifies,
not a generic one.

Eleven failure modes, plus `UNKNOWN`. The label set is closed, and anything the
model cannot place returns `UNKNOWN` rather than being forced into the nearest
plausible bucket.

Deployed on Azure Container Apps. `/health` is liveness; `/ready` reports corpus
size and the embedding model actually in use.

---

## Architecture

Medallion layering, because the transformations are genuinely staged. **Bronze**
preserves what arrived, append-only, with lineage. **Silver** normalises three
dissimilar log formats — Databricks JSONL, ADF JSONL, Spark text — into one
incident schema with an extracted error class. **Gold** is the retrieval corpus:
the runbook, chunked and embedded.

At request time the service retrieves candidate runbook entries from a Chroma
vector store, builds a prompt around them, calls Azure OpenAI, and validates the
response against the permitted label set before returning it.

**Stack:** Python, FastAPI, Chroma, Azure OpenAI, Docker, Terraform,
Azure Container Apps.

---

## What the evaluation actually showed

Two evaluation sets. The clean set is generated from the same templates the
runbook and the parser regexes were written against, so it verifies the pipeline
runs end to end and nothing more. The hard set is generated separately: an LLM
writes unfamiliar wordings seeded only with raw signatures — never the runbook —
and perturbations are applied afterwards (truncation, wrapper exceptions,
interleaved noise, stripped error codes). It also contains out-of-taxonomy
failures where the correct answer is abstention.

Both sets classify at 100%. That is a finding about the task, not a claim about
the system, and the ablation is what makes it interpretable.

**Ablation, hard set, n=48:**

| Condition | Accuracy | Abstained |
|---|---|---|
| Full system | 1.000 | 0.083 |
| Without extracted error class | 1.000 | 0.083 |
| Without retrieval | 0.979 | 0.104 |
| Retrieval only, no LLM | 0.792 | 0.000 |
| Keyword rules only | 0.146 | 0.938 |

Three things worth stating plainly.

**Retrieval does not drive accuracy here.** Removing it costs one record out of
48 — well inside noise. These failure signatures are public, canonical, and
heavily represented in any frontier model's training data. Retrieval still
supplies the remediation text and the grounding, but it is not what makes the
classification correct. A runbook containing genuinely internal procedure is
where I would expect that to change, and it is the experiment I would run next.

**The deterministic path collapses without a symbolic error code.** The keyword
baseline scores 0.800 on records where an error class was extracted, and 0.070
where none was — and only 16% of hard-set records carry one, against 67% of the
clean set. That gap is the argument for an LLM being in the system at all.

**Retrieval alone cannot abstain.** It forces every out-of-taxonomy record into a
known class. The LLM path gets all of them right, at confidence below 0.50.

---

## Design decisions

### No API keys anywhere

The service authenticates to Azure OpenAI with a managed identity and a
short-lived Entra token, fetched per call.

Storing a secret well has real operational cost — rotation, vaulting, leak
response. Not having one is cheaper. A vault remains right for credentials that
must exist; an Azure service calling another Azure service is not that case.

*Cost:* the app needs an Entra identity present. A key-based mode is retained as
an explicit opt-in escape hatch.

### User-assigned identity, not system-assigned

A system-assigned identity is created as part of the app, so role assignments can
only reference it after the app exists — while the app needs those roles to pull
its own image. Terraform rejects the resulting cycle.

A standalone identity makes the ordering linear: identity → roles → app. It also
survives the app being rebuilt, so grants are not re-issued on every deploy.

### Registry admin account disabled

The admin account is a shared username and password that cannot be scoped per
consumer or rotated independently. The app pulls using a role assignment on its
managed identity instead.

### Immutable image tags

Deployments reference a commit SHA. A `latest` tag is rejected at plan time by a
Terraform variable validation, not by convention — because "what is running in
production" is usually asked during an incident.

### Closed label set with explicit abstention

A response outside the permitted set is recorded as a formatting failure,
distinctly from a misclassification. Without that split, a plausible-but-unlisted
label scores as wrong against every class and looks like a model quality problem
when it is a prompt compliance problem.

### Batch-safe failure handling

Every LLM call has three failure modes: the API errors, the response is not valid
JSON, or the JSON is valid but the label is out of set. All three return a result
carrying the error rather than raising. A classifier that throws cannot be run
over a batch.

### A bug the ablation found

The classifier caches whether its deployment accepts a temperature parameter.
That flag was shared mutable state, and the eval harness runs across a thread
pool — four workers probing and retrying at once hung the first ablation run
indefinitely. It is now guarded by a lock, with each call reading a snapshot
rather than re-reading shared state. Single-threaded and four-worker runs now
produce identical results in a quarter of the time.

---

## In progress

The system is reactive: it needs a failure that already surfaced as an error.
The harder and more useful problem is detection — noticing that something is
wrong when nothing threw an exception at all. Two extensions are underway.

**Freshness and volume monitoring.** Track row counts and arrival times per run,
and flag deviation from recent history. A table that normally lands at 6am
arriving at 9am is an incident that produces no log line.

**Distribution monitoring.** Track null rates and cardinality per column over
time, and alert on drift. Silent data quality degradation never errors, so it is
invisible to everything this repo currently does.

Both are statistical rather than linguistic, which makes them a different problem
from the classification work here — and the reason detection, not triage, is
where the real engineering in this domain sits.

---

## What changes for production

Out of scope here, and named rather than left as a gap.

**Triggering.** Nothing calls this API by hand in production. Databricks and ADF
failure events would route through an event bus, so classification lands on the
incident before anyone opens it.

**Feedback.** The on-call engineer's correction is the most valuable output the
system can collect — it replaces synthetic ground truth with real labels and
shows which failure modes the runbook handles badly.

**Re-indexing.** The runbook is embedded at startup. A living runbook needs a
scheduled re-embed on change.

**Regression gating.** The eval harness should run in CI on every prompt change
and model version bump, and block the deploy on a drop. Model deployments get
deprecated and replaced; without a gate you find out in production.

**Network.** Public ingress today. Production runs an internal environment inside
a VNet, with the registry and Azure OpenAI reached over private endpoints.

**State and delivery.** Terraform state is local and applies run from a laptop.
Production uses a remote backend with locking, and gates apply on a reviewed plan
in CI authenticating via federated credentials rather than a stored secret.

**Cost.** Roughly 1,570 prompt tokens per incident. At enterprise failure volume
you would cache repeated signatures and route obvious cases through the cheap
deterministic path first.

---

## Known limitations

The hard-set variants are written by the same model family that classifies them,
which may make them systematically easier. Perturbation partly offsets this,
since the model cannot tidy what is corrupted after it writes it. A genuinely
independent set would need real production logs.

The runbook was authored alongside the failure taxonomy, so it documents what the
model already knows. A real runbook would not be like that, and that is precisely
why retrieval contributes so little here.

n=48 on the ablation. The gap between the full system and the no-retrieval
condition is one record. No strong claim should be read into it in either
direction.
