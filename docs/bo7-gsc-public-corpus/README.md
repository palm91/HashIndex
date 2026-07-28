# BO7 public GSC corpus resolutions

Source corpora:

- `SyndiShanX/COD-GSC-Source`, directory `BO6-GSC`, commit
  `477bfba91d91eb3c7d2603b5775b9160cde9f4bb`.
- `timing1337/BlackOps6Data`.
- `bradselph/COD-Settings-Changer`.
- `Tustin/super-awesome-bo6-tool`.

The 480 public BO6 GSC/CSC files produced 71,113 distinct identifiers and
script-path candidates. Every emitted mapping was matched against an
unresolved BO7 dump target and then rehashed before export.

- `hashes/scr/bo7_generated.csv`: 2,305 safe T10 script-hash mappings.
- `hashes/global/bo7_gsc_generated.csv`: 290 exact x64/FNV mappings.
- 9 script-path mappings were already present elsewhere in HashIndex and were
  not duplicated.

The BO6 data/config pass tested 1,961,869 additional exact candidates. It
added 30 T10 mappings and 1 x64 mapping after removing mappings already
present in HashIndex. The exact additions are retained beside this report as
`blackops6data_bo7scr_new.csv` and `blackops6data_x64_new.csv`.

The wider public dump pass tested more than 31 million additional exact
candidates from BO6, MWIII, BO4 Lua, H2 and local Cold War dumps. It recovered
2,100 T10 and 84 x64 mappings that were not yet saved in HashIndex. Most had
already been found by the earlier BO7 brute-force run; 89 safe mappings were
genuinely new to the readable dump and replaced 391 remaining occurrences.

The applied readable dump is:

`D:\Sort\fftools_cod25_workspace_20260627\gsc_full_rescan_sat1a_20260725\gsc\source_named_resolved_corpus_final_20260726`

Its `gsc_symbol_resolution.tsv` records every applied mapping, replacement
count, and affected file. The cumulative manifest contains 17,348 mappings.
The remaining placeholders are 10,352 `function_*`, 14,344 `var_*`, and
3,766 `hash_*` occurrences.

The closest-family BO4/Cold War exact pass added another 156 T10 mappings and
22 x64 mappings to HashIndex. Of those, 36 T10 and 15 x64 values occurred in
the BO7 source and replaced 167 placeholders. The final readable dump is:

`D:\Sort\fftools_cod25_workspace_20260627\gsc_full_rescan_sat1a_20260725\gsc\source_named_resolved_treyarch_final_20260726`

Its cumulative manifest contains 17,399 mappings. Remaining placeholder
occurrences are 10,311 `function_*`, 14,242 `var_*`, and 3,742 `hash_*`.

The IW8/IW9/JUP Lua corpus added 10 T10 and 7 x64 mappings that occur in the
BO7 GSC dump, replacing another 31 placeholders. The latest readable dump is:

`D:\Sort\fftools_cod25_workspace_20260627\gsc_full_rescan_sat1a_20260725\gsc\source_named_resolved_iw8plus_final_20260726`

Its cumulative manifest contains 17,416 mappings. Remaining placeholder
occurrences are 10,311 `function_*`, 14,218 `var_*`, and 3,735 `hash_*`.

The supporting IW8 GSC/rawfile and T7 Lua pass added 7 more T10 mappings.
Six occur as BO7 placeholders and replaced 17 occurrences. The latest
readable dump is:

`D:\Sort\fftools_cod25_workspace_20260627\gsc_full_rescan_sat1a_20260725\gsc\source_named_resolved_online_final_20260726`

Its cumulative manifest contains 17,422 mappings. Remaining placeholder
occurrences are 10,304 `function_*`, 14,208 `var_*`, and 3,735 `hash_*`.

The Linux positional campaign later produced 384 exact T10 script-hash hits.
Twenty-six were already known and 358 were new. The new, rehash-verified pairs
are retained in `linux_overnight_new.csv`. After all campaigns,
`hashes/scr/bo7_generated.csv` contains 2,683 unique exact mappings and
`hashes/global/bo7_gsc_generated.csv` contains 290 unique exact x64 mappings.

The disjoint Linux depth-2 campaign searched 50,000 ranked first-position
tokens by 50,000 ranked last-position tokens (2.5 billion candidates). It
produced 72 exact T10 hits; 5 were already indexed and 67 were new. The new,
rehash-verified pairs are retained in
`linux_depth2_p50000x50000_20260727_new.csv`, bringing
`hashes/scr/bo7_generated.csv` to 2,750 unique exact mappings.

Follow-up depth-2 campaigns covered the remaining underscore-position blocks
and the complete no-separator first-token cross-product without rerunning
recorded pair domains. The no-separator family was the productive one: it
produced 248 campaign-new exact hits across 5,012,700,000 candidates. After
rehash verification, canonical-casing checks, and deduplication against this
index, all follow-up runs contributed 228 new mappings. They are retained in
`linux_depth2_followup_20260727_new.csv`, bringing
`hashes/scr/bo7_generated.csv` to 2,978 unique exact mappings.

The fixed-verb three-part concatenation campaign then searched 24 GSC verb
prefixes by the top 5,000 middle-position tokens and 50,000 endings
(6.0 billion candidates across two disjoint ranked bands). It produced 168
unique campaign hits; 2 were already indexed and 166 were new. The
rehash-verified pairs are retained in
`linux_concat3_verbs24_middle_top5000_20260727_new.csv`, bringing
`hashes/scr/bo7_generated.csv` to 3,144 unique exact mappings.

The next fixed-verb band searched middle-token ranks 5,001 through 10,000
(6.0 billion candidates). It produced 36 exact hits; 33 were new to the
VM campaign set and 32 were new to this index. The rehash-verified pairs are
retained in
`linux_concat3_verbs24_middle_rank5001_10000_20260727_new.csv`, bringing
`hashes/scr/bo7_generated.csv` to 3,176 unique exact mappings.

A second fixed-verb set searched 24 additional GSC prefixes by the top
5,000 middle tokens and 50,000 endings (6.0 billion candidates). It
produced 68 exact hits and 63 VM-new rows. Two rows were already indexed and
one case-only alias retained the existing canonical spelling, leaving 60 new
mappings in `linux_concat3_verbs24b_middle_top5000_20260728_new.csv`.
`hashes/scr/bo7_generated.csv` now contains 3,236 unique exact mappings.
