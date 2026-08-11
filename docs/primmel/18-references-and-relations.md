# Chapter 18, References and Relations

> *In this chapter:* the `ref` construct — one typed triple for every
> citation and every cross-model relationship — the predicate registry,
> the legacy spellings it unifies, and the edges that deliberately stay
> dedicated.

---

## 18.1 The problem: five spellings for one idea

Before v3's relation wave, a Primmel package said "this element is
related to that address" five different ways:

```prl
form … { references { report-format { "urn:…#clause-4.7" } } }   # role-grouped block
requirement … { source { doc "urn:…" clause "5.5.1" } }          # doc+clause pair
field … { references { reference { "urn:…#anx-E" } } }           # the same idea, other shape
field … { specification_reference "R 60-3, 2.1.2.4" }            # a bare free-text string
test … { targets { /req/… } }                                    # a bare id list
```

Each spelling had its own parser, its own dump, its own validator, and
its own drift class (the bare string never even became a URN). And the
vocabulary itself was closed: a modeller who needed to say *this
implementation model implements that reference model*, or *this edition
supersedes that one*, or *this requirement is equivalent to that ISO
provision*, had no keyword at all.

Chapter 9 (provenance) already established the doctrine: every element
is an interpretation of something published, and the link must be
mechanical. This chapter generalises the mechanism.

## 18.2 The construct: `ref <predicate> "<target>"`

One line, three parts — a typed triple with the enclosing element as
the subject:

```prl
ref report-format "urn:oiml:pub:r:60-3:2021#clause-4.7"
ref test-procedure "urn:oiml:pub:r:60-2:2021#clause-2.10.2"
ref derives-from  "urn:oiml:pub:r:60-1:2021#clause-5.5.1"
ref implements    "urn:oiml:pub:iso-iec:17025:2017#clause-6.4"
ref equivalent    "urn:oiml:pub:r:76:2006#clause-T.2.2.2"
ref supersedes    "urn:oiml:pub:r:60:2017"
```

- **The subject** is the element the `ref` line sits on (a form, a
  field, a requirement, a conformance test, a calculation, a symbol,
  a term, or the package itself).
- **The predicate** is an id from the predicate registry (§18.3) —
  data, not grammar; adding a predicate never touches the codec.
- **The target** is a URI: a document anchor (a clause URN, an annex
  anchor) or a model element id (which is itself a URI). One slot for
  both, because both are just addresses.

A `ref` line may carry a note:

```prl
ref equivalent "urn:oiml:pub:r:76:2006#clause-T.2.2.2" {
  note "The R 76 definition of the analogue data processing device is the one R 60-3, 4.6 cites."
}
```

## 18.3 The predicate registry

Predicates are declared in the metamodel layer's registry
(`oiml-smart-core`'s `specification/predicates.yaml`), so a program can
extend the vocabulary without a grammar change. Every entry:

```yaml
- id: derives-from
  kind: citation            # citation | semantic
  description: The element is the model's interpretation of the target clause.
  subject_kinds: [requirement, conformance_test, form, field, calculation, symbol, term, package]
  target_kinds: [document-anchor]
  inverse: interpreted-by
  transitive: false
  symmetric: false
  resolution: must-resolve  # the linker proves the anchor exists in sources-prd
```

The registry is where the semantics live: `kind: citation` edges feed
the coverage and reconstruction machinery (chapter 9 and 11);
`kind: semantic` edges feed the model graph (mappings, layering,
edition lineage). `resolution: must-resolve` is the linker's hook: a
citation anchor must exist in the `.prd` fragment registry, a semantic
target must exist as a model element.

### The core predicate set

| Predicate | Kind | Reads as | Replaces |
|---|---|---|---|
| `requirement` | citation | this form/test provides evidence for that requirement clause | `references { requirement {…} }` |
| `test-procedure` | citation | the procedure the test follows | `references { test-procedure {…} }` |
| `calculation` | citation | the computation procedure | `references { calculation {…} }` |
| `report-format` | citation | the report-format table this form renders | `references { report-format {…} }` |
| `derives-from` | citation | the clause this element interprets (provenance) | `source { doc clause }` |
| `cites` | citation | an informative citation | `reference`, `specification_reference` |
| `implements` | semantic | this (implementation-side) element fulfils that (reference-side) element | — (new) |
| `equivalent` | semantic | the two elements state the same constraint | — (new) |
| `supersedes` | semantic | this edition/element replaces that one | — (new) |
| `restates` | semantic | a normative restatement without change | — (new) |
| `specializes` | semantic | a narrowing (see `specialization`, §18.5) | — (new) |

A layer declares additional predicates in its own registry section;
the program's composed registry is the union, and the validation gate
(chapter 11) rejects an undeclared predicate.

## 18.4 What the legacy spellings become

The migration is one-to-one and the round-trip is provable:

| Legacy spelling | Canonical form |
|---|---|
| `references { <role> { "<urn>" … } }` | `ref <role> "<urn>"` per entry |
| `source { doc "<urn>" clause "<c>" }` | `ref derives-from "<urn>#clause-<c>"` (the anchor is computed, never hand-typed) |
| `specification_reference "R 60-3, 2.1.2.4"` | `ref cites "…"` with the real URN |
| `reference { "<urn>" }` | `ref cites "<urn>"` |

The codec reads both forms during the transition and dumps the
canonical form. The byte-clean guards (the round-trip gate and the
SSOT drift guard, chapter 11) are what make the mechanical re-dump of
every package safe.

## 18.5 What stays dedicated — and why

Not every edge becomes a `ref`. The rule: **if the runtime branches on
it, it keeps its own slot; if it is evidence or cross-model semantics,
it is a `ref`.** The dedicated set:

- `targets` — the conformance test *judges* the requirement; the
  verdict chain walks it.
- `dependencies`, `inherits_from`, `extends` — the requirement/test
  graph and class inheritance; the applicability engine and the
  class-specific test inheritance execute against them.
- `specialization { dimension, template_id }` — the class-instance
  mechanism (chapter 3); structured, not a bare triple.
- `maps_to` (package) — the model supply chain's abstract import
  (chapter 15); pinned with editions.
- `structure { predicate, propagation }` — designed composition
  (partOf/consists_of/connectsTo) with propagation rules; a `ref` has
  no propagation semantics.

These are the hot edges: execution reads them, and folding them into a
generic edge soup would make every runtime query a filter scan and
every dump ambiguous. The `ref` construct is for the long tail —
citations and declared semantic relations — where the consumers are
the provenance chain, the linker, the coverage gates, and the reader.

## 18.6 The validation discipline

`primmel check` gains the relation rules:

1. **declared-predicate** — every `ref` predicate resolves against the
   composed registry (a typo is an error, not a silent new predicate).
2. **kind-discipline** — the subject and object kinds satisfy the
   predicate's declared kinds (`implements` from a requirement to a
   clause anchor is rejected: that is a citation, use `derives-from`).
3. **resolution** — per the predicate's `resolution:` policy, the
   linker proves the target exists (a clause anchor in the fragment
   registry, a model element in the package graph).

## 18.7 The interop note

The triple shape is the RDF shape. A package's `ref` set exports to
RDF/OWL with the predicate registry as the property vocabulary
(chapter 12) — `derives-from` maps to `prov:wasDerivedFrom`,
`implements` to a program-scoped property, `equivalent` to
`owl:equivalentClass` where the subjects are classes. The registry's
metadata carries the mapping, so the export is generated, never
hand-written.
