# discofetch-model-lib

The pure domain model discofetch is written against: the label alphabet, the
kind registry, the four dimensions, config validation, the secret window, and
the row shapes. Extracted verbatim from `api/supervisor.lua`.

It is the floor. It opens no database, calls no `host.*`, declares no queue,
never reads the clock, and depends on **none** of the other eight libraries in
this set. Everything it cannot compute from its arguments — the deployment
zone and a JSON decoder — arrives through `M.new`.

## Surface

### 1. Entry points

Pure, plain fields on `M`, no constructor:

| | |
|---|---|
| `M.valid_ip(s)` | an IP literal, v4 strictly and v6 loosely |
| `M.base32_from_hex(hex)` | 8-bit bytes in, 5-bit symbols out |
| `M.valid_label(s)` | a drawn label: 16 symbols of `[a-z2-7]` |
| `M.valid_name_label(s)` | a reserved/system name: 3–63, starts on a letter, never `--` |
| `M.validate_config(kind, config, dims)` | the registry's validator; `config` or `nil, sentence` |
| `M.secret_fields(kind)` | ordered list **and** set of the kind's credential fields |
| `M.config_view(impersonating, kind, config)` | the read half of the window |
| `M.same_config(a, b)` | field-by-field, never encoded |
| `M.DIM.validate(body)` | the matrix; `d` or `nil, code, sentence` |
| `M.DIM.kind_of(d)` | TRANSITIONAL: which kind serves these dimensions |

Bound, because they need the zone or a decoder — `local model = M.new(deps)`:

| | |
|---|---|
| `model.DIM.of(r)` | `d, cfg` — the dimensions of a row, and its decoded config |
| `model.fp_row(r, d)` | the JSON shape of a fetchpoint |
| `model.fp_row_status(r)` | `fp_row` plus `mode` and `advertised` |
| `model.fetchpoint_host(h)` | `label, verb` for a Host under the zone, or `nil` |

The instance also carries the pure half (`model.valid_label`, `model.KINDS`,
`model.REDACTED`, …), so a consumer holds one handle — `deps.model` — rather
than a module and an instance.

### 2. Configurable values

`M.B32`, `M.LABEL_BYTES` (tied to each other — see below), `M.REDACTED`,
`M.DIM.STATUSES`. The zone is not among them: it is deployment config and
arrives at `M.new`.

### 3. Fan-out points

`M.KINDS` — the kind registry. A new kind is an entry here, not a branch
anywhere else: `validate_config` enforces it, `GET /v1/kinds` renders it, the
verb gate and the stats rollup read it.

`M.DIM.choices` — the vocabulary. `M.DIM.set` is derived from it **at module
load** by a `for` loop, which is a statement and not a declaration: if it does
not run, every membership test in `DIM.validate` silently passes nothing.

`M.DIM.FIELDS` — every key a dimension-shaped body may carry.
`M.DIM.RETIRED` — the config keys the dimensions replaced, and by what.

The remaining dispatch is `validate_config`'s `f.type` switch (string, ip,
port, url, choice) and its per-kind cross-field rules; both are in depth and
marked there.

## Injected deps

```lua
local model = M.new({
  zone = 'discofetch.link',  -- required, non-empty string; the deployment's zone
  json = json,               -- required; a table with a `decode` function
})
```

Both are asserted **at `new`**, with a sentence that names the dep, rather than
surfacing as a nil call inside `fp_row` on the first list request. Two
instances coexist — a test's zone and the deployment's — because the only
state is in those closures. There is no module-level mutable state.

`json` is needed for **decode only**: a stored config arrives from the database
as text. Nothing here encodes.

There is no `deps.now`: nothing in this library reads a clock.

## Usage

```lua
local model = M.new({ zone = ZONE, json = json })

-- the dispatcher, deciding whether a Host names a FetchPoint
local label, verb = model.fetchpoint_host(req.host)

-- create: validate the shape, then the bag
local d, code, msg = M.DIM.validate(body)
local kind = M.DIM.kind_of(d)
local config, cerr = M.validate_config(kind, body.config or {}, d ~= nil)

-- read: the dimensions of a row, then its public shape
local dims, cfg = model.DIM.of(row)
local out = model.fp_row(row, dims)
out.config = M.config_view(req.impersonating, row.kind, cfg)
```

## Who depends on this

Edges point **downward into** this library and never out of it:

- `discofetch-db-lib` — `valid_label` and the label draw, **by injection**
  (`draw_label` + `valid_label` through its own `M.new`), so `claim_label`
  holds no `host.crypto` and db-lib holds no upward edge.
- `discofetch-accounts-lib` — `base32_from_hex`, for the token mint and the
  invite-code draw.
- `discofetch-fetchpoint-lib` — `KINDS`, `DIM.of`, `valid_ip`, `valid_label`,
  `valid_name_label`, `fetchpoint_host`, `fp_row`, `fp_row_status`,
  `config_view`, `secret_fields`, `REDACTED`, `same_config`.
- `supervisor.lua` — `fetchpoint_host` in the dispatcher, and `AUDIT_MASK`
  should import `M.REDACTED` rather than spelling `'redacted'` inline.

`drt-http-api-lib` deliberately does **not** depend on this: it dropped
`valid_ip` (nothing in it calls one — `public_forwarded` classifies
private/CGNAT/loopback/link-local inline), which is what keeps this library off
the wire library's import list and avoids a cycle.

## What was deliberately left out

- **`claim_label`** (supervisor 517–533). It is a retry loop around
  `db.try_exec` and a `host.crypto.random` draw; both belong to
  `discofetch-db-lib`, which takes the draw as an injection. The codec and the
  predicate it uses live here.
- **`valid_room`** (2486–2489). A room is not a modelled row — it exists only
  in `discofetch-fetchpoint-lib`'s memory store — and its `1-64 [%w%-_]` shape
  is the join/read/ice/outcome contract, not a schema fact.
- **`ZONE` itself**, `ICE`, `TIERS`, `TRAFFIC`, `BOOT_AT`. Deployment config
  and other libraries' subjects.
- **Encoding.** Nothing here calls `json.encode`; the shapes are Lua tables and
  the caller encodes.

## One signature changed

`config_view(req, kind, config)` became **`config_view(impersonating, kind,
config)`**. It used to read `req.impersonating`, a flag
`discofetch-accounts-lib`'s `act_as_apply` writes — a parameter-level
dependency on the accounts request shape that no injected-deps list would ever
show, and a cycle between the model and accounts. The caller reads the boolean
off `req`. This is the only restructure in this pass; everything else is
arithmetic, branch order and refusal text carried across unchanged.

## Known, carried over

Behaviour that looks wrong or sharp. It is preserved exactly; none of it was
"fixed" during extraction.

1. **`valid_ip` accepts leading zeros** — `01.2.3.4` passes, because the octet
   test is `#o > 3 or tonumber(o) > 255`. That arm is a LENGTH test, not an
   octal test, so it refuses padding only once it overflows three characters
   (`0001.2.3.4` and `1.2.3.0255` are refused; `01.2.3.4` is not). Its v6 arm
   is looser still: lowercase hex and colons in any arrangement, up to 45
   characters, so `FE80::1` is refused and `::::` is accepted. This is on
   purpose — "the nameserver will validate for real; this check keeps prose
   and URLs out of address columns" — but a reader expecting a strict parser
   should know. Both arms are pinned by cases in `test/cases.dlua`, so the
   behaviour cannot drift without the suite saying so.
2. **`validate_config` indexes `KINDS[kind]` without a guard.** An unknown kind
   is a nil-index error, not a refusal. Every call site checks `KINDS[kind]`
   first. `secret_fields` *does* guard (`k and k.config_fields or {}`), so the
   two disagree about an unknown kind.
3. **`fp_row_status` decodes without `pcall`** where `DIM.of` decodes with one.
   `json.decode` raises on malformed input (`json.decode: a bare 'n' that is
   not 'null'`), so a corrupt stored config makes a list row throw while the
   same config makes `DIM.of` fall back to `{}`.
4. **`DIM.of`'s old-row branch overwrites the `published_locked` column.** For
   a row with no `behavior` (written before migration 9), the rendezvous derive
   sets the lock from `config.address_override` even if the row carried a
   `published_locked` column. It converges as rows are rewritten.
5. **`KINDS` descriptions hardcode `discofetch.link`** (two of them, both in
   `system:reflect`, plus the `verbs` comment above the table) rather
   than interpolating the zone, so a self-hoster's `GET /v1/kinds` prose names
   the wrong domain even though every computed `fqdn` is right. That coupling
   travels with the table; fixing it is a change to customer-visible copy.
6. **`fp_row_status` reads `r.adv_address` and `r.adv_at`** — aliases invented
   by the advertisement `LEFT JOIN` in `discofetch-db-lib`. It is the one place
   this library knows a query shape, and it is the first thing to move out if
   the model is to stay query-agnostic.
7. **`same_config` is shallow**, which is not a shortcut: `validate_config`
   admits strings, numbers and booleans and nothing that nests. It stops being
   true the day a config field nests.

## `DIM` is one table on purpose

`DIM` is a namespace table rather than seven loose locals because a Lua main
chunk holds at most 200 active locals and `supervisor.lua` was within a
handful of the limit — the host refuses to load past it ("too many local
variables (limit is 200)"), a deploy-time failure with a compile-time cause.
That constraint **relaxes** here: a module has its own chunk. It is not
restructured in this pass, deliberately — the shape is what every caller is
written against, and moving it is a separate change from moving the code.

## Tests

```
sh test/run.sh          # DRT=/path/to/drt to point at another binary
```

259 assertions, and the run is green only if the last line is exactly `PASS`.
There is no standalone Lua on the box, so `test/run.sh` wraps `model.dlua` in
an IIFE, concatenates `test/cases.dlua`, and runs the result under `drt`. The
tests use drt's real `json`, not a stub.

## Consumption waits on `require`

Guests in DRT have no `require` and no `dofile` today; the load-time modules
slice is designed but unshipped. Nothing imports this module yet, and that is
expected. When `require` lands, `test/run.sh` becomes two `require` lines and
`supervisor.lua` stops carrying these 500 lines.
