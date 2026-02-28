<!--
Copyright (c) 2026 Chris Karley
SPDX-License-Identifier: MIT
-->

# GrimSpeak

A controlled natural language for AI agent operations.

---

## The Sentence

Every operation is a sentence. Every sentence has five parts:

**verb** — what you are doing (`snap`, `exec`, `read`, ...)

**noun** — what you are doing it to (`error`, `file`, `directory`, ...)

**adjectives** — key-value parameters that modify the operation (`severity:critical`, `format:json`)

**content** — free text payload. The message body, error description, command string, or note.

**scope** — where the operation happens. A domain and objective identifier that locates the agent in its work hierarchy.

A minimal sentence needs only a verb and noun. Everything else is optional unless the gate level requires it.

```
snap error
snap error "connection timeout on /health"
snap error "connection timeout on /health" because debugging-incident-2847
```

---

## Parts of Speech

### Verbs (13)

Gate levels: open < auth-only < elevated < full-gate

Each verb carries a minimum gate level. The effective gate for any sentence is the maximum of the verb's gate and the noun's clearance.

See Reference Tables for the full verb and noun listings.

### Nouns (43 CLOSED + OPEN)

**Noun class** determines what values are valid. CLOSED means the noun is a fixed word from a registered set. OPEN means the noun is any string — used only with `exec`.

Every CLOSED noun binds to exactly one verb. The pairing is fixed in the registry.

See Reference Tables for the full verb and noun listings.

### Adjectives

Key-value parameters. Written as `key:value` pairs after the noun.

```
snap error severity:critical component:auth
read file format:json
search tasks status:active limit:20
```

Adjectives modify how the operation runs. They do not change the verb or noun. A sentence with unknown adjectives for its noun is a grammar error.

### Content

The free text payload. Appears after adjectives, delimited by quotes or a `because` clause marker.

```
snap error "database connection refused after 3 retries"
exec "git log --oneline -10"
```

Content is opaque to the grammar. The verb-noun pair determines what content means. For `exec`, content is the command string. For `snap error`, content is the error description.

### Scope

Domain and objective identifiers. Locates the agent in its work hierarchy.

```
snap error "timeout" scope:domain-3/objective-1182
```

Scope identifies the agent's position in the work hierarchy.

---

## Sentence Rules

1. **Every noun binds to exactly one verb.** `file` is a `read` noun. You cannot write `snap file` or `exec file`. The pairing is fixed in the registry.

2. **Effective gate = max(verb gate, noun clearance).** `diagnose audit` — verb is open, noun is elevated — resolves to elevated. The stricter requirement wins.

3. **OPEN noun class is exec-only.** Every other verb has a fixed noun set. Only `exec` accepts an arbitrary string as its noun.

4. **Snap is the only compilable verb.** Only `snap` can be extracted from unstructured agent output. All other verbs require explicit input.

5. **Unknown words are hard errors.** An unregistered verb or noun is a hard error. No fallback, no fuzzy match.

---

## Input Formats

GrimSpeak sentences can be provided via command line, tagged text blocks, terminal observation, or batch payload. All formats produce the same sentence structure.

---

## Reference Tables

### Verb Table

| Verb | Gate | Compilable | Noun class |
|------|------|-----------|------------|
| view | open | no | CLOSED |
| diagnose | open | no | CLOSED |
| govern | open* | no | CLOSED |
| snap | auth-only | yes | CLOSED |
| git | auth-only | no | CLOSED |
| batch | auth-only | no | CLOSED |
| search | auth-only | no | CLOSED |
| read | auth-only | no | CLOSED |
| decide | elevated | no | CLOSED |
| manage | elevated | no | CLOSED |
| serve | elevated | no | CLOSED |
| write | elevated | no | CLOSED |
| exec | full-gate | no | OPEN |

*Effective gate raises when noun clearance exceeds verb gate. `govern update` resolves to elevated.

### Noun Table

| Noun | Verb | Clearance | Class |
|------|------|-----------|-------|
| error | snap | open | CLOSED |
| lesson | snap | open | CLOSED |
| idea | snap | open | CLOSED |
| feedback | snap | open | CLOSED |
| observation | snap | open | CLOSED |
| decision | snap | open | CLOSED |
| action | snap | open | CLOSED |
| input | snap | open | CLOSED |
| message | snap | open | CLOSED |
| note | snap | open | CLOSED |
| credential | snap | open | CLOSED |
| approve | decide | open | CLOSED |
| deny | decide | open | CLOSED |
| request | view | open | CLOSED |
| requests | view | open | CLOSED |
| trail | view | open | CLOSED |
| tokens | view | open | CLOSED |
| init | manage | open | CLOSED |
| lockdown | manage | open | CLOSED |
| unlock | manage | open | CLOSED |
| revoke | manage | open | CLOSED |
| gc | manage | open | CLOSED |
| health | diagnose | open | CLOSED |
| audit | diagnose | elevated | CLOSED |
| dashboard | serve | open | CLOSED |
| approver | serve | open | CLOSED |
| start | git | open | CLOSED |
| submit | git | open | CLOSED |
| status | git | open | CLOSED |
| history | git | open | CLOSED |
| messages | read | open | CLOSED |
| pickup | read | open | CLOSED |
| unread | read | open | CLOSED |
| file | read | open | CLOSED |
| listing | read | open | CLOSED |
| verify | govern | open | CLOSED |
| compile | govern | open | CLOSED |
| update | govern | elevated | CLOSED |
| all | search | elevated | CLOSED |
| chunks | search | open | CLOSED |
| tasks | search | open | CLOSED |
| handbooks | search | open | CLOSED |
| directory | write | open | CLOSED |
| *any string* | exec | — | OPEN |

---

## Reference Implementation

GrimGate is the reference implementation of GrimSpeak, licensed separately under BSL 1.1.
