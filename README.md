# pipeline-triage-agent

A retrieval-augmented service that reads a failed data-pipeline log and returns a
diagnosed failure mode, the evidence it relied on, and a suggested remediation.

Built to answer a narrow question: when a Databricks job, an ADF activity, or a
Spark executor fails at 3am, can an LLM grounded in a team's own runbook triage
the failure well enough to be worth acting on?

Independent project. Synthetic data, self-authored runbook, no employer data.

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

A POST to `/triage` takes a whole failed run — several errors with their timing —
and returns which one is the root cause, which are consequences of it, and which,
occasionally, are genuinely unrelated. `/triage/stream` returns the same thing as
Server-Sent Events, one per reasoning step, so a caller sees the agent form a
hypothesis and search rather than fifteen seconds of silence.

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

Single-log classification is deterministic: one retrieval, one call, one label.
Multi-error runs go through an agent that chooses what to search for, implemented
as a LangGraph state machine.

**Stack:** Python, FastAPI, LangGraph, LangChain, Chroma, Azure OpenAI, Docker,
Terraform, Azure Container Apps.

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

## Multi-error runs

A failed run rarely emits one error. It emits an executor dying, then a stage
aborting, then the job being killed — and only the first of those is a cause. The
single-log classifier labels each one in isolation, which is the wrong shape for
the question an on-call engineer is actually asking.

The agentic path takes the whole run. It forms a hypothesis about what failed,
searches the runbook for that hypothesis in its own words, compares the returned
symptoms against the run, and searches again if they do not match. It is capped at
four searches, after which it is forced to commit.

The taxonomy decision that makes this tractable: **consequences get no failure
mode.** They are attributed to a root cause and given a descriptive event type.
The eleven modes stay closed over causes. Without that rule, a stage abort has to
be labelled as something, and the label set fills up with effects.

Evaluated on 47 generated multi-error runs — 39 with a single cause, 8 with two —
on four separate metrics rather than one blended number: did it find the root
cause, did it label that cause correctly, did it correctly identify genuinely
independent failures, and did it attribute the consequences to the right cause.

### Is the agent worth it?

The same 47 runs through the deterministic single-log classifier, one call per
run, measured in the same session:

| | deterministic | agentic |
|---|---|---|
| Failure mode correct | 95.7% | 100% |
| — single-cause runs | 100% | 100% |
| — two-cause runs | 75.0% | 100% |
| Tokens per run | 2,229 | 5,120 |
| Latency per run | 6.8s | 17.0s |

The accuracy difference is two runs out of the eight two-cause cases, which is
not a margin this corpus can distinguish from variance.

The difference that does justify the cost is structural. The deterministic
classifier returns a label. It cannot identify which error is the root cause,
cannot attribute consequences to it, and cannot flag an independent second
failure — those outputs do not exist in its response shape at any price.

So the boundary is not about accuracy. Use the deterministic path when the
question is "what kind of failure is this": it is 2.3x cheaper and 2.5x faster
and right about the mode almost always. Use the agent when the question is
"which of these errors actually caused the others", because the cheap path
cannot answer it.

---

## Porting to LangGraph, and what it bought

The agentic path was first written as a hand-rolled tool-calling loop against the
Azure OpenAI API: a while loop, a four-iteration cap, and a forced final call with
tools disabled. It worked. It was then ported to a LangGraph state machine, and
both implementations were kept so the difference could be measured rather than
argued about.

### Parity first

A port that changes behaviour is a rewrite. Before anything was added, the two
implementations were run against the same 47 runs, with the same retriever, the
same prompt, the same parser and the same scorer, in the same session.

| | hand-rolled loop | LangGraph |
|---|---|---|
| Root cause found | 100% | 100% |
| Root mode correct | 100% | 100% |
| Independent failures correct | 93.6% | 91.5% |
| Consequence attribution correct | 93.6% | 91.5% |
| Mean reasoning length (characters) | 257 | 254 |
| Tokens per run | 5,120 | 4,863 |
| Latency per run | 17.0s | 15.6s |

The two disagree on three runs out of 47, **in both directions** — the loop wins
two, the graph wins one. All three are two-cause independence calls, the metric
this corpus cannot establish (see limitations). A port defect would produce
failures in one direction; churn in both is what sampling looks like.

The honest statement is that the LangGraph implementation reproduces the loop's
behaviour within observed run-to-run variance. Not "identical". The token and
latency differences are sampling, not an architectural result, and are reported
here only so nobody has to wonder whether they were hidden.

### What the framework actually provided

Accuracy was not the point, and a framework swap could not have changed it —
retrieval, prompt, taxonomy and model are what decide accuracy, and none of them
moved. What the framework provided is behaviour the loop did not have:

**Control flow as data.** The graph is declared as nodes and edges, so it can draw
itself. The architecture diagram cannot drift from the implementation because it
is generated from it.

**Durable execution.** A run that dies partway through resumes from its last
completed step rather than restarting. Demonstrated by terminating the process
outright — no exception, no cleanup, no flushed buffers — then resuming in a new
process and confirming the recovered tokens and messages.

**A human approval gate.** The run stops before its remediation is released, with
its state persisted and nothing held in memory. A reviewer approves, edits or
rejects it later, from a different process. A rejection preserves what the model
proposed, because an audit wants to see what was suggested, not only what was
allowed.

**Parallel fan-out.** One branch per incident, concurrent, then a join that makes
every causal call. Branch isolation is enforced by the input schema rather than by
convention: a branch physically cannot see the other incidents.

**Streaming.** One event per completed reasoning step, so a caller can show the
agent searching rather than waiting.

### Fan-out, measured

| | single conversation | fan-out |
|---|---|---|
| Root cause / mode | 100% / 100% | 100% / 100% |
| Independent failures correct | 91.5% | 97.9% |
| Two-cause subset | 50.0% (4/8) | 87.5% (7/8) |
| Model calls per run | ~2 | incidents + 1 |
| Tokens per run | 4,863 | 11,988 |
| Latency per run | 15.6s | 20.7s |

Fan-out cost 2.34× the tokens and was **slower**, not faster. Branches run
concurrently, but the join waits for the slowest and there is simply more work.
The prediction that parallelism would pay for the extra calls was wrong.

It scored highest on the metric both other implementations struggled with, and
there is a plausible mechanism — the join receives explicit per-incident
assessments rather than holding everything in one conversation. But the gap is two
runs on a subset of eight, and this same corpus moved by three runs between two
measurements of the *unchanged* implementation. **Suggestive, not established.**
Establishing it would need repeated runs, and is not claimed here.

### What splitting the work nearly broke

The single-conversation agent grounds its remediation because the retrieved
runbook text sits in its conversation history. Fan-out discards that: a branch
reads the runbook, writes an assessment, and the runbook text dies with the
branch.

The first version therefore had a join step with no runbook at all, writing
remediation from the model's own knowledge. It suggested a configuration flag that
suppresses an error, where the runbook deliberately recommends fixing the query
semantics instead. **It scored 100% on all four metrics**, because none of them
look at the remediation field.

The fix was to carry the runbook text forward explicitly, and to record the
remediation so grounding could be measured rather than spot-checked. Across 47
fan-out runs, n-gram overlap against the runbook averages 0.87, and every
low-scoring case traces back to a real runbook section as paraphrase rather than
invention.

The general lesson, which is the reason this is written up at all: when a pipeline
is split into parallel parts, anything previously shared implicitly through
conversation history has to become explicit data. That, rather than the token
count, is the real cost of fan-out.

---

## Observability

The evaluation harness answers "was it right". The structured per-request log
answers "what happened in this request". Neither answers "where did the 15
seconds go", which is the question asked when something is slow at 3am.

Spans are emitted for the run, each graph node, each model call and each
retrieval, using OpenTelemetry. The instrumentation says nothing about where
traces end up: `TRACING_EXPORTER` selects one JSON line per span through the
standard logger (the default), Application Insights, or nothing at all.

The default matters. Container Apps already ships stdout to the Log Analytics
workspace the deployment pays for, so spans become queryable with KQL with no
new resource, no new credential, and nothing leaving the Azure boundary.
Application Insights is opt-in and its exporter is deliberately not a
dependency, because it bills on ingestion and installing it should be a
decision rather than a side effect of a deploy.

### Why not LangSmith

It is already installed. `langsmith` arrives as a hard dependency of
`langchain-core`, and it is disabled.

It is a good product. It is also a hosted service outside the Azure boundary
that requires an API key, and this system was built specifically to have no
API keys. Pipeline error text in a real deployment carries table names, file
paths and occasionally credentials inside stack traces. Sending that to a
third party is a data-residency decision rather than a tooling preference,
and nothing here needs it.

### What one trace showed

A single two-iteration run, broken down:

| | ms | share |
|---|---|---|
| Model calls (2) | 14,288 | 96.1% |
| Retrieval, including query embedding | 569 | 3.8% |
| Everything else — graph, nodes, parsing | 8.5 | 0.06% |

Two things fell out of that immediately.

**Retrieval is not cheap, and not for the reason expected.** 569ms is almost
entirely the query embedding, which is its own API round trip. The Chroma
lookup itself is negligible. Any latency work on retrieval would need to
target the embedding call, not the vector search.

**The framework costs 8.5ms.** "Does LangGraph add overhead" now has a measured
answer rather than an assurance.

**And the instrumentation found a defect in its first run.** The model
deployment rejects the `temperature` parameter, so every process was sending
it once, having it refused, and retrying without it — a wasted request on
every cold start and every evaluation worker. `AZURE_OPENAI_SEND_TEMPERATURE`
now lets a deployment skip that discovery, and both engines read it so they
stay comparable.

That fix removes one API request per process. It is **not** a latency
improvement: across three runs of the same input, total time varied by 1.7
seconds in both directions, dominated by how many completion tokens the model
happened to generate. The measurements do not support a performance claim and
none is made.

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

### One source of truth across implementations

The iteration cap, the tool description the model reads, the output contract, and
the function that renders a run into a prompt are each defined once and imported
by every implementation that needs them.

This is not tidiness. Two hand-maintained copies of a constant that is meant to be
identical will eventually stop being identical, and at that point a measured
difference between two implementations is no longer measuring what it claims to.
The prompt-rendering function in particular had to move out of the evaluation
scripts, because the container image excludes them — had the service rendered runs
its own way, every measured number would have described an input production never
sends.

### Framework defaults turned off deliberately

The LangGraph implementation disables the client library's automatic retries. The
hand-rolled loop retries once, and only for one specific parameter rejection.
Leaving the framework's retries on would mean a throttled call silently retried
underneath the graph, and the token and latency figures the two implementations
are compared on would include work that was never observed.

When porting for a measured comparison, the framework's helpful behaviours that
the original did not have are a source of difference, not a free improvement.

### Durability is opt-in, and the default is not durable

Attaching a checkpointer does not by itself give durable execution. The default
persistence is asynchronous, so a hard crash loses the step in flight — and the
resume then silently re-runs that work and reports success. The first version of
the crash-recovery demonstration passed for exactly that reason, and the re-run
was only visible by inspecting how many tokens and messages actually survived.

Synchronous persistence costs a write per step. That is what durability costs, and
the default trades it away for speed.

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

**A shared checkpoint store.** Durable execution and the approval gate are
demonstrated against SQLite, which is correct for a demonstration and wrong for
this deployment: the app scales to zero on an ephemeral filesystem, so the file
would vanish with the container. Serving them would mean a managed Postgres —
a one-line change in the application, and a standing cost this project has
deliberately avoided. They are therefore demonstrable locally and not exposed by
the deployed service, rather than exposed in a form that silently loses paused
reviews.

**Cost.** Roughly 1,570 prompt tokens per single incident, and around 5,000 per
multi-error run through the agentic path. At enterprise failure volume you would
cache repeated signatures and route obvious cases through the cheap deterministic
path first.

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

**Independent-failure detection is not measurable with this corpus.** The
two-cause generator pairs incidents whose failure modes differ, and the ground
truth then calls them independent. But two modes being different is not the same
as two failures being causally unrelated, and the model frequently makes a
defensible argument that the second is downstream — an argument the label does not
permit. A corpus that could measure this would need hand-curated pairs that are
genuinely causally independent. Every number reported on that metric should be
read with this in mind.

**Results are not stable across sessions.** The unchanged hand-rolled
implementation scored 100% on all four metrics in one session and 93.6% on two of
them weeks later, on the same corpus. Any single run reported as a fact is
overstating. The comparisons here were all measured within one session for that
reason, and differences of two or three runs out of 47 are not treated as results.

Part of that instability is not fixable here. The model deployment rejects the
`temperature` parameter, so decoding cannot be pinned to greedy. The same
prompt produced 112 completion tokens on one call and 303 on another during a
single evening. Tighter measurement would need a deployment that accepts
temperature, or many more repeats per condition.

**Grounding is measured for one implementation only.** The remediation field was
not recorded until the fan-out work, so the 0.87 figure covers that path and not
the others. Comparing grounding across implementations would need the earlier runs
repeated.

**The grounding check measures quotation, not grounding.** N-gram overlap scores a
faithful paraphrase of the correct source the same as an invention. It is useful
for finding cases worth reading by hand, and it does not decide them. The 0.5
threshold it reports against is a presentation choice, not a pass mark.
