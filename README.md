<!--
Copyright (c) 2026 Chris Karley
SPDX-License-Identifier: MIT
-->

# GrimSpeak
- not turtles. verbs.

A controlled natural language that compiles to typed, auditable operations.

GrimSpeak is a coding language whose surface syntax is English. Agents, humans,
and automation write sentences. A compiler produces typed tuples. Handlers
execute them.

---

## The Grammar

Every operation is a sentence:

```
VERB NOUN "CONTENT" .
```

- **VERB** — the action. Determines the effect type and minimum authorization.
- **NOUN** — the target. Determines which handler runs and what clearance applies.
- **"CONTENT"** — everything else, in double quotes. Optional. Opaque to the grammar.
  The quotes are punctuation — they delimit the payload the way the period
  delimits the sentence.
- **.** — sentence terminator.

A verb and a noun are sufficient. Content is present when the handler needs
it and absent when the verb-noun pair is self-describing.

```
diagnose health.
read file "/tmp/config.json".
record error "connection refused after 3 retries".
```

That is the entire grammar.

---

## Parts of Speech

GrimSpeak maps to English grammar. The terms are used precisely, not as metaphor.

The language is written in **imperative mood** — every sentence is a command.
The implied subject is the system.

**Verbs** are a **closed class**. The set is finite, fixed, and enumerable.
Adding a verb changes the compiler. Each verb declares an effect type — what
kind of side effect it produces (observation, mutation, delivery, unbounded
execution, etc.).

**Nouns** are an **open class**. The vocabulary grows through registration.
Each noun binds to one or more verbs. The binding is declared in a manifest.
One verb class accepts any string as its noun — this is the explicit boundary
where the compiler can no longer decide what the program does.

**Content** is the **object complement**. It completes the meaning of the
noun. The grammar never interprets it — handlers define what content means
for each verb-noun pair.

**Scope** (the subject) comes from the runtime context, not the sentence.

---

## The Type System

The type system is a **manifest** — a registry of all legal verbs and nouns.
In linguistic terms, the manifest is the **lexicon**.

Authorization levels function like sociolinguistic **register** — different
words carry different usage constraints. The effective authorization for any
sentence is the maximum of the verb's level and the noun's clearance. It is
monotonic — a noun can never lower the requirement of its verb.

---

## Sentence Composition

GrimSpeak sentences compose the way English sentences do.

**Simple sentence** — one independent clause. One verb, one noun, one action.
Compiles to one tuple.

**Compound sentence** — multiple independent clauses joined by coordination.
A named operation (a **skill**) expands to multiple simple sentences. Each
constituent compiles independently.

**Complex sentence** — skills calling skills. Dependent clauses — one operation
cannot execute without another resolving first. No recursion. The expansion
tree is always finite.

```
L0  Tuple              machine instruction
L1  Simple Sentence    source code
L2  Compound (Skill)   function
L3  Complex (Comp.)    program
    ─── DECIDABILITY BOUNDARY ───
L4  Constrained NL     model-interpreted
L5  Freeform           observation only
```

Everything below the boundary is deterministic. Same input, same expansion,
same tuples. Above it, a language model proposes sentences — the compiler
validates them.

---

## Emergent Properties

These are structural consequences of a grammar-bounded coding language, not
separate features.

- **Debugging** — every sentence has an enumerable execution tree that can be
  expanded or collapsed across layers.
- **Scoped security** — authorization resolved at compile time from the grammar.
  If an operation is not expressible, it cannot execute.
- **Unified instruction and action set** — the execution log, the training
  corpus, and the audit trail are the same artifact in the same format.
- **Formal verification** — everything below the decidability boundary has
  provable properties: termination, effect safety, type correctness.
- **Code analysis** — any codebase can be described in the grammar. Coverage
  gaps are precisely the operations that require human review.
- **Composable security** — the authorization level of a composition is the
  maximum of its parts. Security is structural.
- **Discoverability** — the manifest drives completion at every level.
  The grammar tells you what is valid before you type it.
- **Portability** — the language is model-independent. Any system that
  produces English text can produce valid sentences.
- **Reproducibility** — deterministic below the boundary. Action sequences
  can be replayed, diffed, and regression-tested.

---

## License

The GrimSpeak language specification is licensed under the [MIT License](LICENSE).

Reference implementations may be licensed separately.
