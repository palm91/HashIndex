# BO7 IW-dvar promotion (2026-07-27)

The public Pastebin `BFigK79r` (`vanguard dvar dump`) supplied 4,475 candidate
names. Its Vanguard hash values were discarded. Each name was rehashed with
ACTS `HashIWDVar` behavior for BO7 and matched only against unresolved BO7 Lua
targets.

- 95 new exact 64-bit BO7 IW-dvar mappings came from the name campaign.
- 55 previously external-only exact IW-dvar mappings were also reconciled.
- All 150 hashes are unique, occur in BO7 Lua, have no competing stored name,
  and rehash exactly after ASCII case and slash normalization.
- No masked, ambiguous, or original Vanguard hash was promoted.

The mappings are stored in `hashes/global/bo7_lua_generated.csv`; occurrence
context and source provenance are retained in
`bo7_lua_generated_provenance.tsv`.
