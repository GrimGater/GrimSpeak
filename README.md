# GrimSpeak

A controlled grammar for AI agent containment.

GrimSpeak is a language specification — 10 verbs and 35 nouns that define everything an
AI agent can express to the system it operates in. An agent states intent through this
bounded vocabulary. A compiler transforms the statement into a structured tuple. That
tuple flows through a pipeline that gates, executes, and audits the operation.

Operations not in the vocabulary aren't blocked after the fact. They're structurally
impossible to compile. The agent never touches credentials. The agent never bypasses
the pipeline. The constraint is the security.

## Why This Exists

I was trying to get [OpenClaw](https://docs.composio.dev/introduction/intro/overview)
agents to actually function inside a production environment — git workflows, error logging,
credential management, the full stack. The agents were capable. The containment wasn't.

Every approach I found followed the same pattern: let the agent express whatever it wants,
then build a policy layer to evaluate what it said. OPA/Rego evaluates structured JSON
that the application constructs. AWS IAM checks API calls the SDK builds. Kubernetes
admission controllers inspect API objects. They all work. They all assume someone already
parsed the agent's intent into a structured request.

AI agents don't work that way. They produce natural language output, and *someone* has
to decide what that output means before policy can evaluate it. That gap — between what
the agent says and what the policy engine sees — is where things break.

Then [ClawHavoc](https://blog.huntr.com/clawhavoc-how-we-found-over-1000-malicious-mcp-tools)
happened. January 2026. Over 1,100 malicious skills discovered on OpenClaw
(CVE-2026-25253). Three attack vectors: `curl | bash` in install scripts, SOUL.md prompt
poisoning, and dependency confusion. The marketplace model — browse, install, run — was
fundamentally vulnerable. Not because of bad implementation, but because the architecture
let agents reach outside the boundary.

GrimSpeak was the answer I built. Instead of constraining what agents can *say* (which
fights how language models work), constrain what the system can *extract*. The agent talks
freely. The manifest — a JSON file listing every allowed verb and noun — limits what the
compiler can produce. Unauthorized operations aren't detected and blocked. They're
inexpressible in the output vocabulary.

OPA says "this request is denied." GrimSpeak says "this request was never formed."

## The Grammar

### The Tuple

Every GrimSpeak operation compiles into the same five-field structure:

```
{
  scope:        { domain_id, objective_id }
  verb:         one of 10 allowed verbs
  noun:         one of 35 defined nouns (or any string for exec)
  adjectives:   { key: value parameters }
  content:      human-readable payload
}
```

**scope** identifies where in the system this operation is happening — which project,
which work objective. Scope drives permissions: same-objective operations share context,
cross-objective operations require policy evaluation.

**verb** is one of 10 allowed actions. Each verb has a gate level (how much scrutiny
before it proceeds) and a `compilable` flag (whether it can be extracted from passive
terminal observation).

**noun** is either from a closed set (bound to a specific verb, 35 defined) or from an
open class (only `exec` — any command string). Every closed noun is bound to exactly one
verb. You can `snap error` but you can't `view error` — the grammar enforces the pairing.

**adjectives** carry parameters. `--to user` becomes `{ to: "user" }`. `--message "text"`
becomes `{ message: "text" }`. Adjectives handle what other grammars would use adverbs,
prepositions, or determiners for. One construct, multiple roles.

**content** is the human-readable payload. The commit message. The error description. The
observation text. Max 64KB, sanitized for secrets before and after compilation.

### Noun Classes

Two types:

- **CLOSED** — a fixed set per verb. `snap` has 11 nouns (error, lesson, idea, feedback,
  observation, decision, action, input, message, note, credential). `git` has 3 (start,
  submit, status). You cannot invent new nouns at runtime. The vocabulary is the vocabulary.

- **OPEN** — only `exec` has an open noun class. The noun IS the command string: `exec "npm
  test"` means the noun is `npm test`. This is the escape hatch — and it's the only verb
  that requires human approval (full-gate) before execution.

### Gate Levels

Four tiers of increasing restriction:

| Level | Value | Meaning |
|-------|-------|---------|
| `open` | 0 | No gate. Diagnostic and read-only operations. |
| `auth-only` | 1 | Identity verified. Agent is who it claims to be. Proceed. |
| `elevated` | 2 | Additional authorization. Administrative and policy operations. |
| `full-gate` | 3 | Human approval required. Nothing executes without a person saying yes. |

Plus `mixed` — the verb has different gate levels across its nouns. Look up the specific
noun to determine the gate. `policy verify` is open. `policy update` is elevated (and
constitutionally overridden to full-gate — the policy engine can never modify itself
without maximum authorization).

### The Compilable Boundary

One field in the manifest — `compilable` — creates the observation-versus-action boundary.

Only `snap` has `compilable: true`. This means the terminal observer (which watches agent
output passively) can **only** extract snap tuples: observations, notes, errors, messages.
It can record what an agent says. It cannot execute what an agent says.

All other verbs — `exec`, `git`, `manage`, `decide`, `policy` — require deliberate
structured invocation. They cannot be extracted from terminal output no matter what the
agent writes. An agent that outputs `exec rm -rf / [exec]` produces... nothing. The
terminal observer can't compile it. The verb isn't compilable.

One boolean per verb. Same grammar, same manifest, same compiler. But terminal observation
mode is read-only by vocabulary constraint. The security distinction between watching and
acting isn't a separate permission system. It's a property of the grammar itself.

### Four Input Paths

The compiler accepts input four ways, all producing the same tuple format:

1. **CLI** — `grimgate git submit -m "message"` → deterministic tag extraction
2. **Scratchpad** — tagged text: `"deployment failed [error] rollback succeeded [observation]"` → one tuple per tag (POSTFIX: text before a tag becomes that tag's content)
3. **Terminal observer** — passive extraction from agent output, restricted to `compilable: true` verbs only
4. **Batch** — array of operations: `[{verb, noun, content}, ...]` → compiled in sequence, one failure kills the batch

## The Insight

Every policy system in production — OPA, Sentinel, Cedar, IAM — evaluates structured
requests that the calling application explicitly constructs. The application decides what
to send to the policy engine.

GrimSpeak inverts this. The manifest decides what the system can *receive*. The attack
surface for vocabulary probing is the manifest itself — a static JSON file with 10 verbs
and 35 nouns. There's no denied request to retry, rephrase, or probe against. There's
no request at all.

The structure is the security. The constraint is the efficiency. They're coupled — one
design decision, two properties. A 10-verb vocabulary means agents need fewer tokens to
express intent. Fewer tokens means faster compilation. Faster compilation means lower
latency. The thing that makes the system secure is the same thing that makes it fast.

Everyone else compiles the rules. GrimSpeak compiles the speech.

## The Example: Git Submit

A single command through the GrimSpeak pipeline:

```bash
grimgate git submit -m "fix authentication timeout on /api/health"
```

What happens:
1. **Compile** — the command becomes a tuple: `{ verb: "git", noun: "submit", adjectives: { message: "fix authentication timeout on /api/health" } }`
2. **Dispatch** — the pipeline checks the gate level (`auth-only` for git), evaluates scope, approves
3. **Handle** — the git handler stages files, commits, pushes via SSH (key resolved internally), checks for an existing merge request, creates one if needed, assigns a reviewer
4. **Emit** — the result gets written to the audit log, workflow state updates, webhooks fire

The caller typed one command and provided a commit message. Everything else — the SSH key,
the API token for the merge request, the GitLab API URL, the project path encoding, the
reviewer assignment — was resolved by the handler. The caller never saw a credential.

### What that replaces

Doing the same thing manually:

```bash
# Step 1: Stage
git add -A

# Step 2: Commit
git commit -m "fix authentication timeout on /api/health"

# Step 3: Push (you need to know the SSH key path)
GIT_SSH_COMMAND="ssh -i /path/to/key -o StrictHostKeyChecking=no" \
  git push -u origin my-branch

# Step 4: Get CSRF token from credential store (you need the dashboard token)
CSRF=$(curl -s -H "Authorization: Bearer $TOKEN" \
  http://credential-store/api/csrf-token | jq -r '.token')

# Step 5: Retrieve GitLab PAT (you need the credential name, the API URL)
PAT=$(curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" -H "X-CSRF-Token: $CSRF" \
  http://credential-store/api/credentials/GITLAB_PAT/inject | jq -r '.value')

# Step 6: Check for existing MR (you need the project path encoding)
curl -s -H "PRIVATE-TOKEN: $PAT" \
  "http://gitlab/api/v4/projects/org%2Frepo/merge_requests?source_branch=my-branch&state=opened"

# Step 7: Create MR (you need target branch, reviewer IDs, API payload format)
curl -s -X POST -H "PRIVATE-TOKEN: $PAT" -H "Content-Type: application/json" \
  -d '{"source_branch":"my-branch","target_branch":"main","title":"my message","reviewer_ids":[1]}' \
  "http://gitlab/api/v4/projects/org%2Frepo/merge_requests"
```

| Metric | Manual | GrimSpeak | Reduction |
|--------|--------|-----------|-----------|
| Commands | 7 | 1 | 86% |
| Credentials exposed to caller | 3 | 0 | 100% |
| Infrastructure details required | 9+ | 2 | ~78% |
| Error handling | Manual | Built-in | — |
| Audit trail | None | Automatic | — |

Every operation through the pipeline — not just git — gets the same credential isolation,
the same error handling, the same audit trail. One tuple format, one pipeline, one audit
system. Adding a new operation means adding a JSON entry to the manifest and a handler.
No compiler changes. No dispatcher changes. No new audit plumbing.

## The 10 Verbs

| Verb | Gate Level | Compilable | What It Does |
|------|-----------|------------|--------------|
| **snap** | auth-only | yes | Record observations, notes, errors, messages, ideas |
| **exec** | full-gate | no | Run arbitrary commands (human approval required) |
| **view** | open | no | Read audit trails, pending requests, messages, tokens |
| **manage** | elevated | no | System administration — lockdown, init, revoke, garbage collection |
| **decide** | elevated | no | Approve or deny pending requests |
| **diagnose** | open | no | Health checks, audit inspection |
| **serve** | elevated | no | Start dashboard, approval interface |
| **git** | auth-only | no | Git workflows — branch, commit, push, merge request |
| **policy** | mixed | no | Verify or update security policy |
| **batch** | auth-only | no | Run a sequence of operations as a single payload |

Note the `compilable` column. Only `snap` is `true`. That single field is what separates
observation from action in terminal monitoring mode.

## Prior Art

**Attempto Controlled English (ACE)** is the academic ancestor. ACE is a Controlled
Natural Language — a subset of English with unambiguous, machine-parseable grammar.
GrimSpeak applies the CNL concept to agent containment rather than knowledge
representation. ACE parses natural language into formal logic. GrimSpeak compiles
structured input into gated execution tuples. Different application, same lineage.

**NVIDIA NeMo Guardrails / Colang** is the closest production system. Colang uses
"canonical forms" that are structurally similar to tuples, but resolves them through
semantic embedding — probabilistic matching against vector representations. GrimSpeak
resolves through manifest lookup — deterministic matching against a closed vocabulary.
Colang targets conversation safety (keeping chatbots on-topic). GrimSpeak targets
execution gating (controlling what agents can do). The resolution mechanism is the key
difference: probabilistic means you tune confidence thresholds and accept false positives.
Deterministic means it compiles or it doesn't.

**PCAS (arXiv 2602.16708, February 2026)** is the closest academic work. PCAS describes
a Datalog-derived policy system using the reference monitor pattern for AI agents. It
independently validates the core approach — structured policy enforcement between agent
intent and system execution. The difference: PCAS instruments existing agents with policy
checks at the application layer. GrimSpeak constrains at the grammar layer. PCAS adds a
gate to the door. GrimSpeak removes the door and leaves only the gate.

**OPA/Rego, Hashicorp Sentinel, AWS Cedar** are policy evaluation engines. They evaluate
structured requests that the application explicitly constructs. They all operate *after*
intent is expressed. None of them serve as the interface itself. You write Rego rules on
top of an existing API. You write Cedar policies over existing actions. GrimSpeak's
manifest IS the interface — the vocabulary defines what can be expressed, not what can be
allowed.

**OpenClaw / ClawHavoc** demonstrated the risk of the marketplace model. In January 2026,
1,184 malicious skills were discovered on OpenClaw (CVE-2026-25253), exploiting three
vectors: `curl | bash` in install scripts, SOUL.md prompt poisoning, and dependency
confusion. GrimSpeak's architecture prevents all three: agents can't install arbitrary
skills (vocabulary is bounded), there are no install scripts (operations compile through
the manifest), and prompt content never becomes execution (the readable plane and the
enforcement plane are separated). The grammar doesn't need to detect malicious intent. It
can't express it.

## Real-World Mapping — OpenClaw Skills

[OpenClaw](https://docs.composio.dev/introduction/intro/overview) is an open-source AI
agent skill framework. These are real skills (or components of skills) that agents use in
production. Here's how they map to GrimSpeak's 10-verb grammar — no new verbs needed.

| OpenClaw Skill | GrimSpeak Tuple | Gate Level | What Happens |
|----------------|----------------|------------|--------------|
| `file_read` | `view:file` | open | Compiles directly, no approval needed |
| `web_search` | `exec:web_search` | full-gate | Policy evaluates, human approves |
| `git_commit` | `git:submit` | auth-only | Full workflow (stage, commit, push, MR) in one tuple |
| `send_message` | `snap:message --to agent` | scope-checked | Chain position determines if agent can reach target |
| `run_code` | `exec:python` | full-gate | Most dangerous operation, highest gate |

Three of five route through safe verbs (`view`, `git`, `snap`). Two hit `exec` — the one
verb with an OPEN noun class and full-gate. The grammar didn't expand. The dangerous
operations automatically get the strongest containment.

This is why ClawHavoc (January 2026, 1,184 malicious skills, CVE-2026-25253) attacks are
structurally impossible in GrimSpeak. Malicious skills can't compile because the vocabulary
doesn't include the operations they need. A skill that tries to `curl | bash` an install
script? That's an `exec` — full-gate, human approval required. A skill that tries to
poison a prompt via SOUL.md? Prompt content never becomes execution. The readable plane
and the enforcement plane are separated. There's nothing to poison.

### Chained Skills — Batch Mapping

Multi-step OpenClaw skills map to GrimSpeak batches. Each step gets its own gate evaluation.

**OpenClaw skill: `deploy_app`** (chains 4 operations):

```json
{
  "source": "batch",
  "payload": {
    "ops": [
      { "verb": "git",  "noun": "submit",  "content": "deploy v2.1",     "on": "success" },
      { "verb": "exec", "noun": "docker build", "content": "-t app:v2.1 .", "on": "success" },
      { "verb": "exec", "noun": "docker push",  "content": "registry/app:v2.1", "on": "success" },
      { "verb": "snap", "noun": "message", "content": "deployed v2.1",   "adjectives": {"to": "ops-team"}, "on": "success" }
    ]
  }
}
```

One OpenClaw skill = one batch = N tuples = N independent gate evaluations. The `git:submit`
compiles clean (auth-only). The two `exec` steps hit full-gate — a human approves each one.
The `snap:message` sends notification on completion. The chain doesn't bypass anything.
Every tuple earns its own gate.

`on:success` and `on:failure` give branching without loops or variables. Turing-incomplete
by design. A Turing-complete policy language is a security liability. The grammar
intentionally limits expressiveness to limit attack surface.

## The Specification

### Formal Grammar (EBNF)

```ebnf
(* GrimSpeak Tuple Grammar — v0.1 *)

tuple           = scope , verb , noun , adjectives , content ;
scope           = "{" , "domain_id" , ":" , integer ,
                  "," , "objective_id" , ":" , integer ,
                  [ "," , "task_id" , ":" , integer ] , "}" ;

verb            = "snap" | "exec" | "decide" | "view" | "manage"
                | "diagnose" | "serve" | "git" | "policy" | "batch" ;

noun            = closed_noun | open_noun ;
closed_noun     = (* any noun defined in manifest.json bound to the given verb *) ;
open_noun       = string ;    (* only valid when verb = "exec" *)

adjectives      = "{" , { key , ":" , value } , "}" ;
content         = string ;    (* max 64KB, sanitized for secrets *)

(* Gate level assigned per verb in the manifest *)
gate_level      = "open" | "auth-only" | "elevated" | "full-gate" ;
```

### Compilation Properties

The compiler achieves three guarantees:

- **Deterministic noun-verb binding.** Every closed noun is bound to exactly one
  verb. Compilation enforces this binding — a noun cannot be used with a verb it is
  not registered to. Compilation produces a valid tuple or fails entirely. There is
  no partial compilation.

- **Input path isolation.** The compiler accepts four input formats, each with
  distinct extraction logic, all producing the same canonical tuple. The terminal
  observer path enforces the `compilable` boundary — only verbs marked compilable
  can produce tuples from passive observation.

- **Secret redaction.** Content fields are sanitized for credential patterns at
  multiple stages of compilation, preventing secret fragments from reconstituting
  in the output tuple. The specific sanitization strategy is part of the reference
  implementation.

Batch operations compile each tuple independently. One failure rejects the entire
batch. Compilation is deterministic, pure, and synchronous — no model inference, no
network calls, no side effects.

### Dispatch Properties

The dispatcher evaluates each compiled tuple against gate levels, scope boundaries,
and constitutional overrides:

- Gate level is resolved from the manifest per verb and noun
- Scope evaluation determines whether an operation crosses organizational boundaries
- A constitutional override ensures the policy engine can never modify itself without
  maximum authorization (full-gate, hardcoded, immutable)
- Gate enforcement routes tuples to immediate execution, additional authorization,
  or human approval queues based on the resolved gate level

### Four-Stage Pipeline

```
input → compile → dispatch → handle → emit

compile:   Input → validated tuple (or rejection)
dispatch:  Tuple → gate evaluation → approved / denied / pending
handle:    Approved tuple → capability execution (one handler per verb)
emit:      Result → fan-out writes (audit log + typed tables + webhooks + delivery)
```

Handlers are pure — they don't evaluate policy. They execute what was approved. The
compiler doesn't decide policy — it decides vocabulary. Each stage has one job. The
separation means you can swap handlers without affecting security, change policy without
touching handlers, and add audit targets without modifying either.

## File Reference

| File | What It Is |
|------|-----------|
| `src/manifest.json` | The vocabulary definition — 10 verbs, 35 nouns, gate levels, noun classes, fan-out targets. This is the grammar. |
| `src/types.ts` | TypeScript type definitions for the tuple structure, compiler inputs/outputs, dispatch results, and pipeline stages. |
| `examples/dummy-data.json` | 10 compilation examples showing inputs, expected outputs, and error cases. Reference data. |

## GrimGate

[GrimGate](https://github.com/thegrims/grimgate) is the reference implementation —
the system that speaks GrimSpeak. It implements the compiler, the four-stage pipeline,
the credential store, and nine verb handlers. It's coming.

This repo is the language definition. The grammar, the tuple format, the vocabulary, the
rules. GrimSpeak is the spec. GrimGate is the runtime.

## Contributing

The GrimSpeak grammar is strict by design. If you want to propose a new noun for the closed vocabulary, open a PR. If it expands the attack surface without a structural gate, it will be rejected.

## Acknowledgments

This work was created with the help of [Claude Code](https://claude.ai/claude-code)
(Claude Opus 4.6), [ChatGPT](https://chatgpt.com) (GPT-5.2), and
[Google Gemini](https://gemini.google.com) (Gemini 3.1 Pro).

## License

MIT License

Copyright (c) 2026 Chris Karley

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in the
Software without restriction, including without limitation the rights to use, copy,
modify, merge, publish, distribute, sublicense, and/or sell copies of the Software,
and to permit persons to whom the Software is furnished to do so, subject to the
following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED,
INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A
PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT
HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION
OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE
SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.

Product names mentioned in this document are trademarks of their respective owners.
GrimSpeak is not affiliated with or endorsed by any listed project.
