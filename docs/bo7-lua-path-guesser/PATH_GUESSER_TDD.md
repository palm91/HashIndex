# Companion Technical Design: BO7 Lua Asset-Path Guesser

Status: ready for max-coverage implementation
Prepared: 2026-07-24; max-coverage revision 2026-07-25
Target repository: `HashIndex`

## 1. Objective

Build a small, deterministic command-line tool that generates plausible BO7 Lua
asset paths and reports candidates whose **60-bit IW Resource hash exactly
matches** an unresolved `Luafile_<hash>.lua` asset ID.

The tool is a targeted dictionary guesser. It is not an unrestricted preimage
brute-forcer, a fuzzy-name resolver, or a decompiler.

Example target:

```text
Luafile_ef31607dc6156c8.lua
```

Desired result:

```text
ef31607dc6156c8    ui/generated/widgets/.../somewidget.lua
```

Only exact hash matches are results. Candidate similarity alone is never a
resolution.

## 2. Current evidence

The attached data was generated from:

```text
D:\Sort\fftools_cod25_workspace_20260627\
```

Post-update archive:

```text
postupdate_corearchive_20260723
```

Correct decompiled source:

```text
postupdate_corearchive_20260723\lua\source
postupdate_new_lua_20260724\source
```

Do **not** use this directory as semantic input:

```text
postupdate_new_lua_20260724\hashed
```

That directory was produced by running BO7 bytecode through CodLua's BO6
profile. The command completed, but the instruction/constant decoding is
incorrect.

Verified inventory:

| Item | Count |
|---|---:|
| `Luafile_*` occurrences in the full post archive | 289 |
| Unique target IDs | 286 |
| Known Lua paths in local `hashes/global/bo7.csv` | 4,819 |
| Current exact target matches in that corpus | 0 |
| Verified hash vectors supplied with this design | 10 |

Additional repository-wide evidence gathered on 2026-07-25:

| Item | Count |
|---|---:|
| Lua rows across every HashIndex CSV | 23,663 |
| Unique cross-game Lua paths | 8,423 |
| Unique observed Lua directories | 435 |
| Unique observed Lua basenames | 8,371 |
| Exact target hits from all cross-game paths | 0 |
| Directory/basename recombinations tested | 3,633,014 |
| Hits from that recombination | 0 |
| Target-local readable function-name/directory candidates tested | 216,566 |
| Hits from that source-guided baseline | 0 |

These zero-hit baselines are important: an implementation limited to corpus
replay and a small directory/basename Cartesian product is not the requested
max-coverage tool.

There are 288 newly appearing `Luafile_*` paths and one path already present
before the update. Some IDs occur in more than one fastfile and may point to
different bytecode payloads. The target loader must retain every occurrence
while hashing against a unique target-ID set.

## 3. Supplied files

All paths below are relative to this document.

### `targets.tsv`

One row per extracted `Luafile_*` occurrence.

Columns:

| Column | Meaning |
|---|---|
| `Hash60` | Target asset ID normalized to 60 bits |
| `OccurrencesForHash` | Number of extracted occurrences for this ID |
| `FastFile` | Extraction bucket/fastfile name |
| `RelativePath` | Path below the post archive's `lua/bytecode` directory |
| `BytecodeSHA256` | Extracted payload identity |
| `BytecodeBytes` | Payload size |
| `UpdateClassification` | `new_content`, `renamed_existing_content`, or `unchanged_path` |
| `SourceRelativePath` | Corresponding path below the post archive |
| `KnownBO7PathMatches` | Existing exact matches; currently blank for all targets |

### `known_lua_paths_bo7.tsv`

Portable snapshot of all `.lua` rows currently present in
`hashes/global/bo7.csv`.

Columns:

```text
Hash60    Path
```

### `known_hash_vectors.tsv`

Ten known Lua paths with:

```text
Path    FullHash64    Hash60    StoredHash60    Matches
```

Every row must pass before target guessing starts.

### `SHA256SUMS.tsv`

Integrity hashes for this handoff packet.

## 4. Hash namespace

Lua asset paths in this corpus use the **IW Resources** FNV-1a variant:

```python
OFFSET = 0x47F5817A5EF961BA
PRIME = 0x100000001B3
MASK64 = 0xFFFFFFFFFFFFFFFF
MASK60 = 0x0FFFFFFFFFFFFFFF

def lua_asset_hash(path: str) -> int:
    value = OFFSET
    normalized = path.lower().replace("\\", "/")
    for byte in normalized.encode("utf-8"):
        value = ((value ^ byte) * PRIME) & MASK64
    return value & MASK60
```

Known example:

```text
ui/generated/widgets/ingame/mp/rewardnotificationicons.lua
```

Full IW Resource hash:

```text
8000bdb560e69b49
```

Stored 60-bit value:

```text
000bdb560e69b49
```

The stored CSV representation may omit leading zeroes.

### Why CodLua's 56-bit option is not used

CodLua's BO6 `--bits-in-hash 56` setting exists for typed hash constants inside
BO6 Lua bytecode. In that representation, high bits carry type information and
the low 56 bits carry the lookup value.

Lua **asset filenames** are a separate IW Resource/XAsset namespace. HashIndex
normalizes this namespace to 60 bits. Applying a 56-bit mask to:

```text
ef31607dc6156c8
```

would incorrectly turn it into:

```text
0f31607dc6156c8
```

The implementation must not share a bytecode-constant mask with the Lua
asset-name mask.

## 5. Feasibility

An unrestricted search is not feasible:

| Space | Candidates | Worst case at 1 billion hashes/second |
|---|---:|---:|
| 56-bit | 72,057,594,037,927,936 | 2.28 years |
| 60-bit | 1,152,921,504,606,846,976 | 36.5 years |

Python would be substantially slower than one billion full path hashes per
second. The useful strategy is to reduce the input space to plausible path
strings derived from known Call of Duty naming conventions and local content.

## 6. Scope

### In scope

- Load unresolved 60-bit target IDs.
- Validate the IW Resource hashing implementation.
- Load known Lua paths.
- Generate bounded path candidates in deterministic tiers.
- Load Lua paths from every HashIndex game corpus, not only BO7.
- Mine target-local bytecode/source evidence and preserve which target supplied
  each token.
- Reverse IW Resource FNV states and perform meet-in-the-middle joins.
- Run deterministic shards with checkpoints and resume support.
- Hash each unique candidate once.
- Emit exact matches with provenance.
- Preserve multiple candidates that collide on the same 60-bit target.
- Print per-tier candidate and hit totals.

### Out of scope

- Decompiling Lua.
- Repairing CodLua's BO7 instruction profile.
- Distributed brute force.
- Fuzzy matches presented as resolved paths.
- Automatically modifying `hashes/global/bo7.csv`.
- Automatically creating `hashes/lua/bo7.csv`.
- General alphanumeric brute force.

An optional GPU backend is allowed only after the reference implementation,
reverse-hash self-tests, and CPU search plan are correct. GPU support is not an
acceptance requirement because reversible FNV joins remove larger search spaces
than a faster flat Cartesian loop.

## 7. Proposed implementation

Add one implementation file:

```text
scripts/guess_lua_paths.py
```

Use the Python standard library only. Do not add a project dependency or a new
framework.

The implementation may add one data file:

```text
data/bo7_lua_path_rules.json
```

It contains vocabulary aliases, fixed directory fragments, basename affixes,
join characters, and enabled search shapes. Search logic must remain generic:
new evidence should normally change the data file or input corpora, not Python
code.

The script should expose its hash and normalization functions at module level
so they can be checked independently. A `--self-test` option should run the
provided hash vectors and a synthetic end-to-end candidate match.

### Proposed CLI

```powershell
python scripts\guess_lua_paths.py `
  --targets docs\bo7-lua-path-guesser\targets.tsv `
  --known docs\bo7-lua-path-guesser\known_lua_paths_bo7.tsv `
  --hash-root hashes `
  --source D:\Sort\fftools_cod25_workspace_20260627\postupdate_corearchive_20260723\lua\source `
  --bytecode D:\Sort\fftools_cod25_workspace_20260627\postupdate_corearchive_20260723\lua\bytecode `
  --rules data\bo7_lua_path_rules.json `
  --state D:\Sort\bo7_lua_guesser_state `
  --output D:\Sort\bo7_lua_name_hits.tsv
```

Required arguments:

| Argument | Purpose |
|---|---|
| `--targets` | `targets.tsv` or a directory containing `Luafile_*.lua` |
| `--known` | Known `Hash60, Path` TSV/CSV corpus |
| `--output` | Exact-match TSV |

Optional arguments:

| Argument | Purpose |
|---|---|
| `--source` | Decompiled source directory used to mine path words |
| `--bytecode` | Bytecode directory used for payload identity and printable-string mining |
| `--hash-root` | Recursively load `.lua` paths from every HashIndex CSV |
| `--rules` | Data-driven search vocabulary and shapes |
| `--state` | Checkpoint, shard manifests, and completed-tier state |
| `--max-candidates` | Per-shard hard ceiling; default 100,000,000 |
| `--tier` | Highest candidate tier to run |
| `--workers` | CPU worker count; default logical CPU count |
| `--resume` | Skip completed deterministic shards and continue an interrupted run |
| `--self-test` | Validate hashing and a synthetic end-to-end match, then exit |

Do not add a generic configuration framework. The one rules JSON file is a
versioned search dataset, not application configuration.

## 8. Candidate generation

Candidates must be emitted as a stream and deduplicated before hashing.
Generation order must be deterministic.

Normalize every candidate before hashing:

1. Trim whitespace.
2. Convert `\` to `/`.
3. Remove a leading `/`.
4. Collapse repeated `/`.
5. Lowercase.
6. Require a `ui/` prefix.
7. Require a `.lua` suffix.
8. Reject `.` and `..` path components.

### Tier 0: known corpus replay

Rehash every known path from BO7 and every other HashIndex corpus.

Purpose:

- Validate the corpus and algorithm.
- Detect malformed index rows.
- Confirm target-set lookup.

This tier is expected to find zero current targets based on the supplied
snapshot. A different result after the corpus changes is valid.

### Tier 1: known directory/basename recombination

Split the known BO7 Lua corpus into:

- Directory paths.
- Basenames without `.lua`.

Recombine only directory and basename groups that share an observed path shape.
Examples:

```text
ui/generated/widgets/frontend/<name>.lua
ui/generated/widgets/ingame/<name>.lua
ui/generated/menus/frontend/<name>.lua
ui/utils/<name>.lua
```

Avoid an unrestricted Cartesian product across every directory and basename.
Group by broad family (`widgets`, `menus`, `utils`, `ingame`, `frontend`) and
stop at `--max-candidates`.

### Tier 2: source-guided names

Mine correct CoDLuaDecompiler output for likely identifiers:

- `MenuBuilder.registerType` arguments when readable.
- Top-level uppercase subsystem names.
- Widget class names.
- Data-source names.
- Literal strings that already resemble paths.
- Literal strings ending in known UI suffixes.

Convert CamelCase and underscore identifiers to lowercase path basenames.
Preserve the original joined lowercase form first because generated Lua paths
usually omit separators inside basenames.

Example:

```text
SeasonalEventLeaderboard
```

Candidate:

```text
seasonaleventleaderboard.lua
```

Use only suffixes observed in the known corpus, including:

```text
button
container
data
grid
item
menu
panel
utils
widget
```

Do not invent arbitrary English words.

### Tier 3: local content vocabulary

Optionally mine additional words from:

```text
postupdate_corearchive_20260723\localize
postupdate_corearchive_20260723\stringtable
postupdate_corearchive_20260723\rawfile
```

Accept only identifier-like tokens:

```text
[A-Za-z][A-Za-z0-9_]{2,63}
```

Prioritize tokens already adjacent to `ui`, `widget`, `menu`, `lua`, mode
names, or known subsystem names.

This tier must remain disabled unless a source directory is explicitly passed.

### Tier 4: bounded numeric mutations

Only mutate basenames that already contain digits.

Try:

```text
0-20
00-20
```

at the existing numeric position. Do not append arbitrary numbers to every
candidate.

### Tier 5: reversible-FNV meet-in-the-middle

IW Resource FNV is reversible because its prime is odd and therefore has a
multiplicative inverse modulo `2^64`.

Implement:

```python
INV_PRIME = pow(PRIME, -1, 1 << 64)

def reverse_byte(value: int, byte: int) -> int:
    return ((value * INV_PRIME) & MASK64) ^ byte
```

Only 60 target bits are stored. For each target, enumerate all 16 possible full
states:

```python
full_target = target60 | (high_nibble << 60)
```

For a search shape such as:

```text
<directory>/<left><right>.lua
```

1. Hash `<directory>/<left>` forward and index the resulting state.
2. Reverse each of the 16 full target states through `<right>.lua`.
3. Join equal intermediate states.
4. Reconstruct and fully rehash every joined candidate.
5. Emit only candidates whose final low 60 bits equal the target.

Support the observed join characters:

```text
""
"_"
"/"
```

Required search shapes:

```text
directory / basename
directory / stem + suffix
directory / prefix + stem
directory / left + right
directory / left + middle + right
directory / family / basename
directory / mode / basename
```

The rules data controls which fragment sets feed these generic shapes.
Meet-in-the-middle tables must be built once per shared prefix or suffix and
reused across targets.

### Tier 6: target-local semantic expansion

Mine each target's own source and bytecode for:

- Readable non-decompiler function names.
- Module and registration identifiers.
- `requireUnlessHotload` relationships.
- Widget, menu, material, mode, event, scenario, REX, SAW, and map vocabulary.
- Printable bytecode strings even when instruction decompilation is incomplete.
- Identifiers shared with required modules.

Keep target-local tokens separate from global vocabulary and search them first.
Generate joined lowercase, underscore, and observed-numeric variants. A token
from target A must not automatically multiply every directory for every other
target.

### Tier 7: dependency-graph propagation

Build a graph from readable `require`/`requireUnlessHotload` relationships and
known Lua modules. When a target depends on a known module:

- Prioritize the known module's directory and sibling directories.
- Reuse its basename affixes and family/mode components.
- Propagate only one graph hop by default; allow two hops as a later shard.

This tier changes candidate ordering, never match correctness.

### Search-plan calibration

Before unknown-target search, automatically run deterministic holdouts from
each known game corpus. Record recovery by rule and search shape. Disable rules
that produce no holdout recovery after their configured budget unless the rule
is explicitly marked as BO7 evidence.

The generated report must show candidates-per-recovered-holdout so expensive,
unproductive shapes are visible without changing engine code.

## 9. Matching and confidence

Build an in-memory set:

```python
targets: set[int]
```

For each unique normalized candidate:

```python
candidate_hash = lua_asset_hash(candidate)
if candidate_hash in targets:
    emit(candidate_hash, candidate, tier, provenance)
```

An exact hash match is a **hash hit**, not necessarily a unique semantic
resolution. Sixty-bit collisions are unlikely in the bounded candidate set but
are possible.

If more than one candidate matches the same target:

- Retain every candidate.
- Mark the target as ambiguous.
- Do not pick a winner automatically.

Source semantics may be recorded as review evidence, but must never replace the
exact hash check.

## 10. Output

Write UTF-8 TSV:

```text
TargetHash60	CandidatePath	Tier	Provenance	Ambiguous
```

Example:

```text
ef31607dc6156c8	ui/generated/widgets/frontend/example.lua	2	source:PRESTIGE	false
```

Sort final results by:

1. `TargetHash60`
2. `Tier`
3. `CandidatePath`

Write each shard atomically. Store a manifest containing the input checksums,
rules checksum, tier, shard coordinates, candidate count, elapsed time, and
completion flag. `--resume` must reuse only manifests whose checksums still
match.

Print a concise run summary:

```text
targets: 286
known paths: 4819
candidates hashed: N
exact hits: N
ambiguous targets: N
limit reached: true|false
```

Do not use a database. JSON shard manifests and append-only exact-hit TSV parts
are sufficient.

## 11. Validation

### Hash-vector self-test

`--self-test` must:

1. Load `known_hash_vectors.tsv`.
2. Recompute `FullHash64` and `Hash60`.
3. Assert every `Matches` row remains true.
4. Hash a synthetic candidate.
5. Confirm the synthetic target is emitted exactly once.
6. Confirm a nonmatching candidate is not emitted.

Exit nonzero on failure.

### Holdout check

Before targeting unknown BO7 IDs:

1. Remove a small deterministic sample of known paths from the seed set.
2. Run the enabled candidate tiers.
3. Report which holdout paths are regenerated.

This measures candidate-generation usefulness. It does not change correctness:
zero false hash hits is still mandatory.

### Target parsing checks

The loader must demonstrate that:

```text
ef31607dc6156c8
```

remains:

```text
ef31607dc6156c8
```

It must not become:

```text
0f31607dc6156c8
```

Duplicate target occurrences must result in one hash lookup target while
remaining visible in input statistics.

## 12. Acceptance criteria

Implementation is complete when:

- The ten supplied vectors pass.
- All known corpus rows rehash to their stored 60-bit values, or malformed rows
  are reported with their exact line numbers.
- All 289 occurrences and 286 unique targets load successfully.
- Tier 0 through Tier 2 run deterministically.
- Reverse-byte tests round-trip every byte value and multiple random hash states.
- Meet-in-the-middle recovers synthetic two- and three-fragment paths for all
  16 possible high nibbles.
- Cross-game corpus replay loads all 8,423 current unique Lua paths.
- Target-local source and printable-bytecode mining are deterministic.
- Interrupted shards resume without repeating completed work.
- Rules and corpora can be extended without modifying generator code.
- The candidate ceiling is enforced before exceeding it.
- Only exact 60-bit matches appear in the output.
- Multiple matching candidate strings are retained.
- No HashIndex source CSV is modified.
- The script has no non-stdlib dependency.
- The command returns zero for a valid zero-hit run.
- The command returns nonzero for invalid inputs or failed hash vectors.

A zero-hit result is not an implementation failure. It means the generated
dictionary did not contain a preimage for the supplied targets.

## 13. Integration after reviewed hits

Reviewed, unambiguous matches may be copied into a new file:

```text
hashes/lua/bo7.csv
```

Format:

```text
<lowercase hash without 0x>,<normalized lowercase path>
```

Before committing:

1. Rehash every proposed path.
2. Confirm the result equals the stored hash.
3. Run `scripts\format.ps1`.
4. Run `scripts\validate.ps1`.
5. Keep ambiguous collisions out of the committed index.

The guesser itself must not perform this integration.

## 14. Known risks

| Risk | Handling |
|---|---|
| `luaAssetHash` is not the expected IW Resource namespace | Hash vectors validate the algorithm; exact target hits remain the final proof |
| Candidate vocabulary never generates the real path | Report zero hits and expand evidence-driven tiers only |
| 56-bit truncation causes false collisions | Keep asset-name hashing fixed at 60 bits |
| A real 60-bit collision produces multiple paths | Retain all and mark ambiguous |
| Source was generated by the broken BO6 CodLua profile | Use only the documented CoDLuaDecompiler source directories |
| Candidate explosion | Family grouping and `--max-candidates` hard limit |
| Local `bo7.csv` changes during development | Treat the supplied TSV as the reproducible baseline |

## 15. Implementation order

1. Implement normalization and IW Resource hashing.
2. Implement `--self-test` using `known_hash_vectors.tsv`.
3. Implement target and known-corpus loaders.
4. Implement Tier 0 and exact-match output.
5. Implement Tier 1.
6. Implement reversible FNV and meet-in-the-middle synthetic tests.
7. Implement target-local source/bytecode mining.
8. Implement deterministic shards and resume manifests.
9. Run cross-game holdouts and write the calibration report.
10. Run against all 286 targets.
11. Review exact hits manually.

Use multiprocessing only after the single-process reference path passes every
test. Do not start with GPU code, plugins, or a generic wordlist.

## 16. Coverage ceiling and maintenance contract

No finite guesser can guarantee recovery of an arbitrary unknown 60-bit
preimage. The goal is therefore not “never add evidence”; it is “do not rewrite
the engine when evidence changes.”

The implementation is considered durable when:

- New game/update corpora are discovered automatically below `--hash-root`.
- New source and bytecode trees are accepted through existing CLI arguments.
- New vocabulary, aliases, directories, and search shapes are enabled through
  the versioned rules data.
- All runs are resumable and reproducible from manifests.
- Exact matching remains the only resolution criterion.

Future work should normally consist of adding corpus rows or rule data. Engine
changes are justified only by a genuinely new path grammar that cannot be
expressed by the required generic shapes.
