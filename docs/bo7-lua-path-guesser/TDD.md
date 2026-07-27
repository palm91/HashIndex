# Technical Design: BO7 Lua Internal-Hash Resolver

Status: ready for implementation
Prepared: 2026-07-25
Target repository: `HashIndex`

## 1. Objective

Build a deterministic command-line tool that resolves hashes used inside
decompiled BO7 Lua files, writes an annotated source copy, and emits a complete
resolution manifest.

This internal-hash resolver is the primary deliverable. Resolving
`Luafile_<hash>.lua` asset filenames is a separate second phase documented in
`PATH_GUESSER_TDD.md`.

Only exact hash matches are resolutions. Context may select the correct hash
namespace and rank candidates, but similarity alone is never a result.

## 2. Current evidence

Correct source roots:

```text
D:\Sort\fftools_cod25_workspace_20260627\postupdate_new_lua_20260724\source
D:\Sort\fftools_cod25_workspace_20260627\postupdate_corearchive_20260723\lua\source
```

Do not use this semantically invalid BO6-profile output:

```text
D:\Sort\fftools_cod25_workspace_20260627\postupdate_new_lua_20260724\hashed
```

Measured inventory:

| Source set | Lua files | Unique 16-hex constants | Existing HashIndex matches | Unresolved |
|---|---:|---:|---:|---:|
| Fresh Lua folder | 1,617 | 50,871 | 5,738 | 45,133 |
| Full post-update archive | 12,076 | 118,543 | 18,978 | 99,565 |

Fresh-folder matches split into:

| Match | Count |
|---|---:|
| Exact stored hash | 5,534 |
| Match after typed low-56 masking | 204 |

Full-archive matches split into:

| Match | Count |
|---|---:|
| Exact stored hash | 18,622 |
| Match after typed low-56 masking | 356 |

These counts are unique hash values, not occurrence counts.

## 3. Supplied files

### `internal_hash_targets.tsv`

One row per unique 16-hex value found in the fresh Lua source.

Columns:

```text
Hash64
Hash56
Occurrences
Contexts
SampleFile
SampleLine
```

### `existing_internal_matches.tsv`

Existing HashIndex candidates for fresh-source targets.

Columns:

```text
Hash64
Hash56
Candidate
IndexSource
MatchKind
```

`MatchKind` is `exact64` or `masked56`.

### Companion path-resolution inputs

The existing `targets.tsv`, `known_lua_paths_bo7.tsv`, and
`known_hash_vectors.tsv` belong to the later asset-filename phase described in
`PATH_GUESSER_TDD.md`.

## 4. Hash namespaces

Do not put every `0x...` value into one target set.

### 4.1 Typed Lua string/table-key hashes

CodLua resolves these using standard FNV-1a:

```python
FNV_OFFSET = 0xCBF29CE484222325
FNV_PRIME = 0x100000001B3
MASK64 = 0xFFFFFFFFFFFFFFFF
MASK56 = 0x00FFFFFFFFFFFFFF

def fnv1a64(value: str) -> int:
    result = FNV_OFFSET
    for byte in value.encode("utf-8"):
        result = ((result ^ byte) * FNV_PRIME) & MASK64
    return result
```

BO6/BO7 typed constants commonly store type information above the low 56 hash
bits. This is where CodLua's `--bits-in-hash 56` behavior is relevant.

### 4.2 Lua asset/module paths

Lua asset filenames and some resource references use IW Resource FNV:

```python
IW_OFFSET = 0x47F5817A5EF961BA
IW_PRIME = 0x100000001B3
MASK60 = 0x0FFFFFFFFFFFFFFF
```

Normalize with lowercase and `\` to `/`. This namespace must not be truncated
to 56 bits.

### 4.3 Other engine asset hashes

Calls such as `RegisterMaterial`, image/model/sound registration, and some
engine APIs may contain asset hashes from an asset-specific namespace. Preserve
the call name and argument position as context. Test only algorithms validated
by known vectors or existing HashIndex rows.

### 4.4 Non-hash numeric constants

Some 16-hex values are IDs, packed flags, colors, handles, or ordinary numeric
data. The tool must support `classification=unknown_numeric` and must never
invent a name merely because a numeric value looks hash-shaped.

## 5. Context classification

Classify every occurrence before candidate search. Required context classes:

```text
table_key
table_value
member_id
require_arg0
require_arg1
registered_type
material
image
model
sound
animation
event
function_argument
unknown_numeric
```

Use syntax and call position, not the value's high byte alone.

Examples:

```lua
object[0x0123456789ABCDEF]              -- table_key
requireUnlessHotload(0x..., 0x...)     -- require_arg0 / require_arg1
MenuBuilder.BuildRegisteredType(0x...) -- registered_type
RegisterMaterial(0x...)                -- material
element.id = 0x...                     -- member_id
```

When one hash appears in multiple contexts, retain every context.

## 6. Existing-index resolution

Recursively load every CSV below:

```text
HashIndex\hashes
```

Build separate lookup tables:

```text
exact64[hash]
masked56[hash & MASK56]
masked60[hash & MASK60]
```

Resolution policy:

1. Prefer an exact 64-bit match.
2. Use low-56 matching only for typed Lua contexts.
3. Use low-60 matching only for validated IW Resource contexts.
4. Retain every candidate if an index contains multiple strings for one key.
5. Mark multi-candidate results ambiguous.

Never apply a low-56 candidate globally to a material/resource occurrence
without its context supporting that namespace.

## 7. Candidate corpus

Build one reusable value corpus from:

- Every HashIndex CSV value.
- Readable strings and identifiers from the target Lua file.
- Readable strings and identifiers from one-hop required modules.
- Known Lua paths and their directory/basename components.
- Localize, stringtable, rawfile, and configuration assets when explicitly
  supplied.
- GSC resolutions and shared engine vocabulary.

Every value must retain provenance. Normalize variants only when appropriate
for the selected namespace:

```text
original
lowercase
CamelCase joined lowercase
underscore-separated
slash-normalized path
```

Do not generate arbitrary case permutations.

## 8. Search tiers

### Tier 0: exact index replay

Apply existing exact64, masked56, and context-valid masked60 matches. This tier
must reproduce the supplied 5,738 fresh-source matches.

### Tier 1: target-local direct candidates

Hash readable names from the same file before using global vocabulary:

- Function names.
- Table/member names.
- Registration and state names.
- Literal strings.
- Require/import neighbors.

### Tier 2: cross-game direct candidates

Hash every unique HashIndex value under each context-valid algorithm. Hash a
candidate once per algorithm and compare it against all targets for that
namespace.

### Tier 3: bounded identifier composition

Generate deterministic combinations using observed joiners:

```text
left + right
left + "_" + right
prefix + stem
stem + suffix
prefix + stem + suffix
```

Prioritize target-local tokens, then dependency-local tokens, then the global
ranked vocabulary. Use a configured per-shard ceiling.

### Tier 4: resource/path templates

For module and asset contexts, use templates derived from observed HashIndex
paths. The reversible-FNV and meet-in-the-middle requirements in
`PATH_GUESSER_TDD.md` apply here.

### Tier 5: numeric mutations

Only mutate an observed numeric position. Do not append numbers to every word.
Try bounded values `0-32` and zero-padded equivalents.

## 9. Performance and durability

The implementation should be one standard-library Python entry point:

```text
scripts/resolve_bo7_lua_hashes.py
```

It may use multiprocessing after single-process correctness is verified.

Required behavior:

- Hash each unique candidate once per algorithm.
- Compare against in-memory target sets.
- Split large searches into deterministic shards.
- Store JSON shard manifests with input/rules checksums.
- Resume completed shards without repeating them.
- Write outputs atomically.
- Never edit the input source tree.

New vocabulary and path fragments belong in one versioned data file:

```text
data/bo7_lua_hash_rules.json
```

Future updates should normally change corpora or rule data, not resolver code.

## 10. CLI

```powershell
python scripts\resolve_bo7_lua_hashes.py `
  --source D:\Sort\fftools_cod25_workspace_20260627\postupdate_new_lua_20260724\source `
  --hash-root C:\Users\palm\Documents\Github\HashIndex\hashes `
  --rules data\bo7_lua_hash_rules.json `
  --state D:\Sort\fftools_cod25_workspace_20260627\lua\resolver_state `
  --output D:\Sort\fftools_cod25_workspace_20260627\lua\resolved_next
```

Required arguments:

| Argument | Purpose |
|---|---|
| `--source` | Decompiled Lua source root |
| `--hash-root` | Recursive HashIndex CSV root |
| `--output` | Annotated-copy and manifest output root |

Optional arguments:

| Argument | Purpose |
|---|---|
| `--rules` | Versioned vocabulary/search rules |
| `--state` | Resume and shard manifests |
| `--tier` | Highest enabled tier |
| `--workers` | Worker count; default logical CPU count |
| `--max-candidates` | Per-shard limit; default 100,000,000 |
| `--resume` | Continue matching completed input checksums |
| `--self-test` | Run hash, parser, and synthetic replacement tests |

## 11. Output

### Resolution manifest

Write UTF-8 TSV:

```text
Hash64
MaskedHash
Namespace
Candidate
MatchKind
Confidence
Ambiguous
Contexts
Occurrences
Files
Provenance
```

Confidence values:

```text
exact64
typed56
resource60
generated_exact
ambiguous
```

`generated_exact` means an exact hash preimage produced by a search tier. It
does not mean semantic similarity.

### Annotated source copy

Preserve the original source layout below:

```text
<output>\source_annotated
```

Use a reversible annotation that keeps the numeric value:

```lua
object[0x0123456789ABCDEF --[[ resolved: candidate_name ]]]
```

If inserting a comment would make the Lua invalid, leave the line unchanged
and rely on the sidecar manifest. Never blindly replace numeric constants with
quoted strings.

### Unresolved inventory

Write:

```text
unresolved.tsv
```

with contexts and occurrence locations so later rules can target the remaining
families without rescanning source.

## 12. Validation

`--self-test` must cover:

1. Standard FNV-1a known vectors.
2. IW Resource known vectors from `known_hash_vectors.tsv`.
3. Low-56 typed-bit masking.
4. Low-60 resource masking.
5. Context parser examples for every required class.
6. Exact versus ambiguous lookup behavior.
7. Synthetic candidate generation and exact match.
8. Annotated-copy behavior without input mutation.
9. Interrupted-shard resume behavior.

Before unknown searching:

- Tier 0 must reproduce all 5,738 supplied fresh matches.
- Rehash every proposed candidate under its recorded namespace.
- Report exact64, typed56, resource60, ambiguous, and unresolved totals
  separately.

## 13. Acceptance criteria

Implementation is complete when:

- All 50,871 supplied fresh targets load.
- Tier 0 reproduces 5,738 existing matches.
- Exact and masked matches remain distinguishable.
- Context classification is retained per occurrence.
- Every emitted candidate rehashes to its target.
- Ambiguous candidates are preserved rather than silently selected.
- The source tree is never modified.
- An annotated copy and complete manifest are produced.
- Large tiers are deterministic, bounded, and resumable.
- Adding new corpora or vocabulary normally requires no code change.
- A valid zero-new-hit search returns success.

## 14. Implementation order

1. Implement both hash functions and self-tests.
2. Implement source inventory and context classification.
3. Implement recursive HashIndex loading.
4. Reproduce the supplied Tier 0 match counts.
5. Emit manifest, unresolved inventory, and annotated copy.
6. Implement target-local and dependency-local direct candidates.
7. Implement cross-game direct candidates.
8. Implement deterministic composition shards and resume.
9. Add resource/path templates using the companion path TDD.
10. Run against the fresh set, then the full archive.

Do not begin with GPU code or unrestricted brute force. Correct namespace
classification and target-local evidence provide more coverage per candidate.
