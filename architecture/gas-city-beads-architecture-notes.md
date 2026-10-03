# Gas City / Beads Architecture Notes

> Consolidated from our discussion of Gas City’s October 2026 Memory Beads / BDP direction and how it fits into the wider Gas City orchestration model.

---

## 1. The shortest mental model

Gas City is evolving Beads from an **issue/dependency tracker** into a more generic **typed, versioned graph of persistent work and knowledge**.

The important abstraction shift is:

```text
Before

Issue ──Dependency──> Issue
```

becoming:

```text
Now

                Bead
              /      \
          Issue      Memory
             \        /
               Link
```

More precisely:

- **Bead** = generic persistent, typed node.
- **Issue** = a Bead type representing work.
- **Memory** = a Bead type representing durable knowledge.
- **Link** = generic persistent, typed, directed edge between Beads.
- **Dependency** = one specific kind of Link.
- **Dolt** = the storage implementation used by the current Beads graph preview.
- **BDP (Beads Protocol)** = the HTTP/data-model contract for interacting with Beads independently of the underlying database schema.
- **`bd` CLI** = the local/user-facing tool for operating Beads.
- **Gas City Pack / Agent / Formula / Skill / MCP configuration** = the layer that defines how agents behave and how work is orchestrated.
- **Beads** = runtime persistent state moving through that orchestration.

The most important conceptual separation is:

> **Gas City orchestration defines how work is performed. Beads persist what the work is and what knowledge should survive it.**

---

# 2. What changed in Beads

## 2.1 The old model was strongly Issue-centric

Historically, Beads treated information primarily as:

```text
Issue
  ├── status
  ├── priority
  ├── assignee
  └── dependencies
```

This is a good fit for work because Issues naturally have lifecycle semantics such as:

- ready
- blocked
- claimed
- completed / closed

But not everything an agent needs is “work”.

For example:

```text
"PRs must target the integration branch."
"Service A owns attendee identity."
"Approach X was rejected because of constraint Y."
"Webhook delivery can be duplicated."
```

These are not Issues.

If they were forced into an Issue model, basic lifecycle questions stop making sense:

```text
When is the architecture policy "done"?
Should a service ownership rule become "closed"?
Does another Issue have to wait for a policy to close?
```

That is the limitation Memory Beads are designed to fix.

---

# 3. The generic Bead model

The new conceptual model is:

```text
Bead = identified + typed + properties
Link = identified + typed + directed + properties
```

So the graph is not specifically:

```text
Memory ──> Issue
```

It is more generally:

```text
Bead ──Link──> Bead
```

With the currently demonstrated concrete Bead types:

```text
Bead
├── Issue
└── Memory
```

This means the graph can contain any combination of currently supported nodes:

```text
Issue  ──> Issue
Issue  ──> Memory
Memory ──> Issue
Memory ──> Memory
```

The **Type** supplies the semantic meaning of the node or edge.

That distinction matters because a graph is not useful merely because things are connected. The types tell tools and agents what those connected things *mean*.

---

# 4. What is an Issue Bead?

An Issue Bead is best understood as:

> **the persistent representation of work**

It is not literally the execution of the work itself.

For example:

```text
Actual work

Agent:
- opens source files
- edits code
- runs tests
- creates a PR

        ↓ tracked by

Issue Bead:
"Fix checkout retry bug"
```

The Issue persists the work state even if the current agent exits, loses context, or another agent takes over.

Typical Issue semantics include things such as:

```text
open
ready
blocked
assigned
closed
```

So when we say “Issue = work”, the exact wording should really be:

> **Issue = durable record representing a unit of work.**

---

# 5. What is a Memory Bead?

A Memory Bead is:

> **durable knowledge that should survive beyond the current conversation or work session**

Examples:

```text
Memory:
"PRs target Jim's integration branch."

Memory:
"Webhook X can deliver duplicate events."

Memory:
"We rejected approach A because it violates constraint B."

Memory:
"Service Foo owns attendee identity."
```

A Memory has different lifecycle semantics from an Issue.

It does **not** naturally become:

```text
ready
blocked
done
```

Instead, it may be:

```text
updated
corrected
superseded
retired
versioned
```

A Memory may be produced by work:

```text
[Issue]
Investigate attendee duplication
        |
        | produces / discovers
        v
[Memory]
Event delivery is at-least-once,
so consumer X must be idempotent.
```

But a Memory does not have to originate from an Issue. A human or agent may directly record durable project knowledge.

Therefore:

```text
Issue  ≈ representation of work
Memory ≈ representation of knowledge
```

Neither term should be confused with arbitrary external artifacts such as source files, binaries, PRs, screenshots, or documents.

---

# 6. Is a Memory “the output” of an Issue?

Sometimes, but not necessarily.

A useful distinction is:

```text
Work
│
├── external artifacts
│   ├── code
│   ├── PR
│   ├── test result
│   └── generated file
│
└── durable knowledge
    └── Memory Bead
```

An Issue can produce both.

Example:

```text
Issue:
"Investigate sync duplication"
        |
        +------------------+
        |                  |
        v                  v
   Code/PR artifact     Memory
                       "Consumer X must use
                        idempotency key Y"
```

So the safest definition is:

> A Memory is **knowledge worth retaining**, not simply “the output field of an Issue”.

---

# 7. Links: what actually creates the graph

A Link is a first-class, typed, directed relationship between Beads.

Conceptually:

```text
[Bead A] ── LINK_TYPE ──> [Bead B]
```

Examples could include:

```text
Issue  ── depends-on ───────> Issue
Issue  ── references ───────> Memory
Memory ── related-to ───────> Issue
Memory ── explains ─────────> Memory
```

The important part of the Gas City direction is that a Link is not merely an anonymous pair of node IDs.

It has:

```text
identity
type
direction
properties
```

This allows tools to distinguish relationships intentionally rather than guessing what a connection means.

---

# 8. Today: two important Bead types; architecture: more than two

In the October 2026 preview we discussed, the two concrete Bead types being demonstrated are:

```text
Issue
Memory
```

So, practically, saying:

> “Right now there are Issue Beads and Memory Beads”

is a useful model.

But the architecture deliberately generalizes to:

```text
Bead
├── Issue
├── Memory
├── future custom type
├── future custom type
└── ...
```

The larger design explicitly leaves room for user-installed Types.

Therefore the distinction is:

```text
Current preview:
Issue + Memory

Underlying model:
arbitrary typed Beads and typed Links
```

This is important because otherwise there would be little reason to introduce the generic `Bead` abstraction at all.

---

# 9. Versioning is one of the more important additions

Suppose a project policy changes:

```text
Version A:
"Branch from integration."

Version B:
"Branch from Jim's integration branch."
```

If an old PR followed Version A, reading only the latest Memory later could incorrectly make that PR look noncompliant.

The new model therefore wants both:

```text
reference current Bead
```

and:

```text
reference exact historical version
```

Conceptually:

```text
code-flow-policy
   ├── v1
   ├── v2
   └── v3
```

Then historical work can cite:

```text
Issue/PR ── followed-policy ──> code-flow-policy@v1
```

while new work uses:

```text
code-flow-policy@current
```

This makes persistent agent knowledge far more useful because a citation can preserve what something meant **at the time**.

---

# 10. Versioning also helps concurrent agents

Imagine two agents read the same Memory revision:

```text
Agent A reads R1
Agent B reads R1
```

Then:

```text
Agent B writes R2
Agent A later attempts to write based on R1
```

A guarded write can reject Agent A's stale update instead of silently overwriting Agent B's newer state.

The article demonstrates revision/token-based guards such as:

```text
--if-revision TOKEN
--if-source-revision TOKEN
```

This is effectively optimistic concurrency control applied to durable agent knowledge.

That becomes increasingly important as multiple agents operate on the same project state.

---

# 11. Where are Issues, Memories, and Links stored?

In the implementation discussed in the article:

> **Dolt is the underlying storage layer.**

Conceptually:

```text
            Beads engine
                 |
        +--------+--------+
        |                 |
      Issue             Memory
        \                 /
             Links
               |
               v
             Dolt
```

The preview supports both:

```text
embedded Dolt
```

and:

```text
shared Dolt SQL server
```

The article walkthrough explicitly starts an ordinary Dolt SQL server and initializes Beads against it.

So:

```text
Issue records
Memory records
Links
version/history state
```

are persisted through the Beads engine into Dolt.

---

# 12. BDP is not the storage layer

BDP stands for **Beads Protocol**.

It should be thought of as:

> **the stable network/data-model contract for working with Beads**

not:

```text
the database
```

and not merely:

```text
the server binary
```

A useful stack is:

```text
Agent / App / UI / MCP Server
            |
            | BDP / HTTP
            v
       Beads server
            |
       Beads engine
            |
            v
           Dolt
```

Therefore:

```text
BDP      = protocol/specification
bd serve = server implementation exposing that protocol
Dolt     = persistence
```

---

# 13. Why have BDP if `bd` CLI already exists?

Because the CLI and the protocol solve different integration problems.

## `bd` CLI

Good for:

```text
human
local coding agent
shell-based workflow
local automation
```

Example:

```bash
bd show ...
bd remember ...
bd links ...
```

Conceptually:

```text
Human / local Agent
        |
        | shell
        v
      bd CLI
        |
        v
   Beads engine
        |
        v
       Dolt
```

## BDP

Good for:

```text
web UI
remote service
MCP server
other applications
remote/federated Beads stores
third-party implementations
```

Conceptually:

```text
App / MCP / remote Agent
          |
          | HTTP / BDP
          v
      Beads server
          |
          v
      Beads engine
          |
          v
         Dolt
```

Without BDP, an external service might have to:

```text
spawn `bd`
parse stdout
interpret exit codes
depend on local workspace configuration
```

With BDP, it can speak a stable machine-to-machine HTTP contract.

The shortest distinction is:

> **`bd` is a tool that knows how to operate Beads.  
> BDP is the contract that allows arbitrary software to operate Beads.**

---

# 14. Is BDP “like MCP”?

Only partially.

BDP really is a protocol, but its scope is much narrower.

```text
MCP
= general protocol for exposing tools/resources/capabilities
  to model clients

BDP
= domain-specific protocol for operating the Beads graph
```

They can be layered:

```text
LLM
 |
 | MCP
 v
MCP Server
 |
 | BDP
 v
Beads Server
 |
 v
Dolt
```

So BDP is not “a replacement for MCP”.

It is closer to:

```text
a domain-specific graph API contract for Beads
```

---

# 15. Scope: why BDP matters beyond one local repository

BDP introduces the idea of a **Scope**.

A Scope is essentially an ownership / identity boundary with a canonical URL.

Conceptually:

```text
https://beads.example/team/
https://beads.example/project-a/
```

This allows something in one scope to reference something in another:

```text
Project A Issue
      |
      | follows-policy
      v
Team-level Memory
```

without copying the entire team knowledge store into Project A.

That starts to look like a federated graph:

```text
Team Scope
  └── Memory: code-flow-policy

Project A Scope
  └── Issue: feature-A
       |
       +──────────> team/code-flow-policy

Project B Scope
  └── Issue: feature-B
       |
       +──────────> team/code-flow-policy
```

This is one reason a URL-based protocol is valuable in addition to a local CLI.

---

# 16. Why the Memory/Link move is architecturally strong

The value is not simply:

> “They added graph edges.”

A trivial graph schema is easy to build.

The stronger architectural decision is:

> **work state and durable knowledge share one persistence and identity model without pretending they have the same semantics**

That gives Gas City:

```text
shared identity
shared linking
shared history
shared querying
shared protocol
shared persistence
```

while still allowing:

```text
Issue semantics != Memory semantics
```

This means agent work can form a lifecycle like:

```text
work
  ↓
knowledge
  ↓
future work
  ↓
more knowledge
```

instead of only:

```text
work
  ↓
work
  ↓
work
```

For a long-running multi-agent software factory, this is a natural direction.

---

# 17. The biggest risk: graph-shaped garbage

Persisting memory is not the hardest problem.

The harder questions become:

```text
What deserves to become durable Memory?
Who decides?
When is a Memory stale?
How do we resolve contradictions?
How do we avoid duplicates?
What should be linked?
How do we retrieve the right 5 memories out of 50,000?
```

A naive implementation can easily become:

```text
10,000 Memory Beads
├── duplicate observations
├── stale policies
├── contradictory statements
├── trivial details
├── poor links
└── low-value agent chatter
```

So the architectural challenge shifts from:

```text
"Can agents remember?"
```

to:

```text
"Can we maintain high-quality organizational knowledge?"
```

This is where ontology, schema, provenance, retrieval, and semantic modeling become much more important.

---

# 18. Graph != Ontology

This distinction is essential.

This:

```text
Memory A ──related-to──> Issue B
```

gives you a graph.

It does **not automatically give you an ontology**.

An ontology becomes more interesting when relationships have meaningful domain semantics:

```text
Service
  ├── OWNED_BY ──────> Team
  ├── PUBLISHES ─────> Event
  └── EXPOSES ───────> Endpoint

Decision
  ├── APPLIES_TO ────> Service
  ├── SUPERSEDES ────> Decision
  └── JUSTIFIED_BY ──> Incident
```

The difference is:

```text
Graph
= things connected

Ontology
= things connected under an explicit semantic model
```

Gas City's generic Type/Link direction creates a foundation that *could* become much richer semantically, but the existence of Bead + Link by itself does not make the system an ontology.

---

# 19. How this relates to Gas City orchestration

A very important distinction from our discussion:

> **The Bead itself does not define the entire agent workflow.**

The Bead is persistent state.

Gas City's orchestration layer defines:

```text
who performs work
how work is sequenced
what instructions agents receive
what capabilities they have
what outputs are expected
```

A useful conceptual breakdown is:

```text
Agent   = WHO does the work
Bead    = WHAT persistent work/knowledge exists
Formula = HOW work is orchestrated
Rig     = WHERE work happens
Pack    = configuration/package defining the setup
Skill   = behavioral/domain instructions
MCP     = external capabilities/data/tool access
```

So saying:

> “I customize a Bead to tell it which MCP server to call”

would be misleading.

A better model is:

> “I configure an agent/workflow that receives Beads and has certain Skills and MCP capabilities.”

---

# 20. A practical custom Gas City setup

Conceptually, you might have:

```text
my-city/
│
├── pack.toml
│
├── agents/
│   └── researcher/
│       ├── agent.toml
│       ├── prompt.md
│       ├── skills/
│       │   └── ontology/
│       │       └── SKILL.md
│       └── mcp/
│           └── ontology.toml
│
├── formulas/
│   └── investigate.toml
│
├── skills/
│   └── shared-skill/
│       └── SKILL.md
│
└── mcp/
    └── sourcegraph.toml
```

The exact file layout can evolve, but the architectural separation is the key part.

---

# 21. Markdown vs TOML vs executable capability

One question we discussed was whether Gas City orchestration is “really just Markdown instructions”.

The answer is:

> **The behavioral layer is heavily Markdown-driven, but the entire orchestration system is not just Markdown.**

A useful approximation is:

```text
Markdown
= instructions / judgment / agent behavior

TOML
= configuration / topology / workflow wiring

MCP / scripts / underlying tools
= actual capabilities

Beads
= persistent runtime work and knowledge state
```

For example, an agent prompt or Skill might contain:

```markdown
You are an architecture researcher.

1. Query Ontology first.
2. Validate findings against source.
3. Do not speculate.
4. Persist important architectural discoveries.
```

That is behavioral logic expressed in Markdown.

But workflow structure might look conceptually like:

```toml
[[steps]]
id = "investigate"
agent = "researcher"

[[steps]]
id = "implement"
agent = "engineer"
needs = ["investigate"]

[[steps]]
id = "review"
agent = "reviewer"
needs = ["implement"]
```

And MCP configuration is separate again.

---

# 22. “Judgment out of Go”

One of the architectural ideas we discussed is that Gas City tries to keep domain judgment out of the orchestration engine.

The engine should perform mechanical orchestration such as:

```text
dependency satisfied?
    ↓
dispatch next step

agent crashed?
    ↓
restart / recover

multiple steps ready?
    ↓
fan out
```

While configuration/instructions define things like:

```text
What should a researcher investigate?
What counts as sufficient evidence?
Which MCP should be consulted?
When should something be remembered?
What output structure should a reviewer produce?
```

This creates a useful separation:

```text
Gas City runtime / SDK
= generic orchestration machinery

Your Pack
= your actual agent organization
```

---

# 23. Output structure: who defines it?

The Bead does not inherently define every output schema.

A workflow can require structured output such as:

```json
{
  "finding": "...",
  "evidence": [],
  "confidence": 0.94,
  "affectedServices": [],
  "followUpWork": []
}
```

But that contract belongs more naturally to:

```text
Formula
Agent prompt
Skill
tool schema
workflow conventions
```

rather than to the generic concept of `Bead`.

So:

```text
Bead
"What persistent state exists?"

Agent
"Who acts on it?"

Skill / Prompt
"How should the agent behave?"

MCP
"What external capabilities are available?"

Formula
"How do steps compose?"

Output contract
"What must this workflow step produce?"
```

This is a cleaner mental model than making the Bead responsible for everything.

---

# 24. Example: Ontology-enabled research workflow

A concrete architecture could look like:

```text
[Issue Bead]
Investigate why attendee sync duplicates records
        |
        v
Researcher Agent
        |
        +----------------------+
        |                      |
        v                      v
Ontology MCP             Sourcegraph MCP
        |                      |
        +----------+-----------+
                   |
                   v
               Research
                   |
          +--------+---------+
          |                  |
          v                  v
      code / PR          [Memory Bead]
                         "Event delivery is
                          at-least-once..."
```

Then future work can reference that Memory:

```text
[Memory]
Event delivery is at-least-once
        |
        | informs
        v
[Issue]
Refactor consumer idempotency
```

This is the type of workflow where the new Memory/Link model becomes genuinely useful.

---

# 25. Where EDN would fit

From our earlier discussion, EDN should not be confused with the canonical Beads persistence model.

If Gas City stores Issues/Memories in Dolt, then:

```text
Dolt
= canonical durable Beads state
```

EDN could still be useful as:

```text
snapshot
exchange representation
agent-readable context artifact
serialized orchestration state
portable knowledge bundle
```

but it would not automatically replace:

```text
Issue Bead
Memory Bead
Link
version history
```

inside Dolt.

A useful distinction is:

```text
Beads / Dolt
= canonical persistent operational graph

EDN
= possible serialized representation / snapshot / context artifact
```

---

# 26. Current implementation vs intended direction

This distinction is critical.

## Demonstrated in the integration preview

The article describes working support for:

```text
Memory creation and editing
Issue creation
Issue/Memory graph links
generic graph-oriented CLI operations
local version listing
version comparison
embedded Dolt
shared Dolt server
read-only BDP HTTP access
```

## Still incomplete / future direction

The article also calls out remaining work including:

```text
BDP HTTP writes
HTTP history behavior
full Memory lifecycle/history behavior
user-installed Types
cross-Scope References
full compatibility with existing Issue workflows
```

So the architectural destination is broader than the current implementation.

Do not confuse:

```text
"the model allows this"
```

with:

```text
"the current production implementation already fully supports this"
```

---

# 27. End-to-end architecture

The cleanest overall picture from our discussion is:

```text
                    GAS CITY
                       |
              +--------+--------+
              |                 |
            Pack              Formula
              |                 |
         configuration       workflow
              |
            Agent
              |
       +------+------+
       |             |
     Skills         MCP
       |             |
       +------+------+
              |
              v
        performs work
              |
              v
            Beads
      persistent runtime state
          /         \
       Issue       Memory
          \         /
            Links
              |
              v
             Dolt
              ^
              |
        Beads engine
              ^
              |
      +-------+--------+
      |                |
    bd CLI            BDP
 local interface    network protocol
```

That diagram captures most of the architectural distinctions we discussed.

---

# 28. The most useful terminology to keep straight

## Bead

Generic typed persistent node.

Not inherently “work” or “output”.

---

## Issue

A Bead type representing a durable unit of work / work state.

---

## Memory

A Bead type representing durable knowledge.

---

## Link

Typed, directed, persistent relationship between Beads.

---

## Dependency

A specific Link type associated with work dependency semantics.

---

## Dolt

The persistence technology used by the current Beads graph implementation.

---

## `bd`

CLI for interacting with Beads.

---

## BDP

Protocol/API contract for machine-to-machine Beads access.

---

## `bd serve`

A server exposing Beads through BDP.

---

## Agent

The worker/persona/model configuration that acts on work.

---

## Skill

Reusable instructions/domain knowledge that influence agent behavior.

---

## MCP

External tools/data/capabilities made available to an agent.

---

## Formula

Declarative workflow/orchestration describing how work steps compose.

---

## Pack

The configuration/package describing a Gas City setup: agents, workflows, prompts, Skills, MCP definitions, etc.

---

# 29. The most important conceptual conclusions

### 1. Beads are becoming generic graph entities

Do not equate:

```text
Bead == Issue
```

anymore.

Think:

```text
Bead
├── Issue
└── Memory
```

with room for additional Types later.

---

### 2. Issue and Memory deliberately have different semantics

```text
Issue = work lifecycle
Memory = knowledge lifecycle
```

Trying to make both behave identically would defeat the purpose of the new model.

---

### 3. The graph is between Beads

It is not specifically a “Memory-Issue graph”.

Any supported Bead can be linked to another supported Bead.

---

### 4. Bead does not mean “the actual execution”

The actual work happens in the agent/runtime/environment.

The Bead is the persistent representation.

---

### 5. Memory does not simply mean “output”

It means durable knowledge.

Some work outputs become Memories; many outputs do not.

---

### 6. Dolt stores the graph state

The Beads engine persists Issue, Memory, Link, and version-related state through Dolt in the current design.

---

### 7. `bd` and BDP are complementary

```text
bd
= local CLI interface

BDP
= machine/network protocol
```

---

### 8. BDP is a real protocol, but much narrower than MCP

```text
MCP
= general LLM capability integration

BDP
= Beads-specific graph protocol
```

---

### 9. Gas City customization lives above Beads

Agents, Skills, MCP access, workflow sequencing, and output contracts belong to Gas City's orchestration/configuration layer.

Beads persist the state those workflows operate on.

---

### 10. Markdown is important, but Gas City is not “just Markdown”

A practical approximation:

```text
Markdown = behavioral instructions
TOML     = configuration / orchestration
MCP      = capabilities
Beads    = persistent runtime state
Dolt     = persistence
BDP      = network contract
```

---

### 11. The Memory move is good, but memory quality becomes the hard problem

Once persistence is solved, the difficult questions become:

```text
what to remember
how to type it
how to link it
how to retire it
how to resolve contradictions
how to retrieve it efficiently
```

---

### 12. A Beads graph is not automatically an ontology

The ontology layer begins when the graph develops explicit, useful domain semantics rather than merely generic relationships.

This is where typed entities and relations such as:

```text
Service ── OWNED_BY ──> Team
Decision ── APPLIES_TO ──> Service
Decision ── SUPERSEDES ──> Decision
```

become much more meaningful than:

```text
Memory ── related-to ──> Issue
```

---

# 30. One-sentence architecture summary

> **Gas City is a configurable agent-orchestration runtime whose Packs define agents, instructions, workflows, Skills, and MCP capabilities; Beads provide the persistent typed graph of work and knowledge that survives those agents; Dolt stores that graph; `bd` operates it locally; and BDP exposes it as a machine-to-machine protocol.**

---

## Source basis

The Beads/Memory/Link/BDP/versioning/storage portions of this note are based on the Gas City article **“Extending Beads: Memories, Versions and the Wire Protocol”** (October 1, 2026), which we analyzed in the conversation.

The higher-level diagrams and terminology mappings in this document are architectural synthesis from our discussion and are intended as explanatory models rather than literal source-code schemas.
