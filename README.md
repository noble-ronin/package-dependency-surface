# package-dependency-surface

A cheatsheet for one question: **how many packages does installing something actually pull
in, and can you find out without installing it?** Every registry answers this differently —
npm's `dependencies` field is exact-enough-to-recurse (names, no version resolution needed to
walk the graph shape), PyPI's `requires_dist` and crates.io's sparse-index `deps[]` carry
version *ranges* that would need real resolution to follow precisely, so this repo recurses
npm fully and goes one hop on the other two, disclosed explicitly rather than faked.

Companion piece (the conversational version — lodash and chalk have zero dependencies, the
`is-even` meme is down to 3 packages, express is 63): [dev.to/ronin13 — lodash Installs With
Zero Dependencies. express Installs With
63.](https://dev.to/ronin13/lodash-installs-with-zero-dependencies-express-installs-with-63-i-counted-without-running-npm-2jhk)

![banner](assets/banner-1.png)

## Endpoints used

| registry | endpoint | field | recursion |
|---|---|---|---|
| **npm** | `GET registry.npmjs.org/{name}/latest` | `.dependencies` (name → semver range) | ✅ full BFS, names only, ranges ignored for traversal |
| **PyPI** | `GET pypi.org/pypi/{name}/json` | `.info.requires_dist[]` (PEP 508 strings) | ⛔ one hop — see "Why not recurse PyPI/crates" |
| **crates.io** | sparse index `GET index.crates.io/{2-char}/{2-char}/{name}` (1–3 char names use a different path shape) | last line (latest version) → `.deps[]` filtered to `kind=="normal"` | ⛔ one hop |

All three are keyless. No npm/pip/cargo CLI involved anywhere — every number below came from a
raw HTTP GET against the registry's own metadata API.

## npm: full recursive crawl, zero installs

```bash
#!/usr/bin/env bash
# usage: ./crawl-npm-deps.sh <package-name>
# Recursively walks registry.npmjs.org's own `dependencies` field. No npm install.
seed="${1:-express}"
python3 - "$seed" <<'PYEOF'
import json, sys, urllib.request

seed = sys.argv[1]
visited = {seed}
frontier = [seed]
depth = 0
edges = []

while frontier:
    nxt = []
    for name in frontier:
        url = f"https://registry.npmjs.org/{name}/latest"
        try:
            with urllib.request.urlopen(url, timeout=8) as r:
                data = json.load(r)
        except Exception as e:
            print(f"  ! {name}: {e}", file=sys.stderr)
            continue
        for dep in (data.get("dependencies") or {}):
            edges.append((name, dep))
            if dep not in visited:
                visited.add(dep)
                nxt.append(dep)
    if nxt:
        depth += 1
    frontier = nxt

print(f"{seed}: {len(visited)} total unique package(s), {depth} level(s) deep")
PYEOF
```

Live results, this run (all `latest` at time of writing):

| seed | version | direct deps | total unique (recursive) | depth |
|---|---|---|---|---|
| `left-pad` | 1.3.0 | 0 | 1 | 0 |
| `lodash` | 4.18.1 | 0 | 1 | 0 |
| `chalk` | 6.0.1 | 0 | 1 | 0 |
| `debug` | 4.4.3 | 1 (`ms`) | 2 | 1 |
| `is-even` | 1.0.0 | 1 (`is-odd`) | 3 (`is-even` → `is-odd` → `is-number`, then nothing) | 2 |
| `express` | 5.3.0 | 28 | **63** | 6 |

`chalk` and `lodash` both shipped dependency-free at the time of writing — neither has ever
been a "drags in half the registry" package, whatever its reputation. `is-even`'s chain used
to run several packages deeper (older write-ups cite a longer `kind-of`/`is-buffer`-style
tail); `is-number` rewrote itself as a single zero-dependency file at v7, and that quietly
shortened the whole meme to 3 packages.

## PyPI and crates.io: one hop, mandatory-only

```bash
# PyPI — mandatory (non-extra) direct dependencies only
curl -s "https://pypi.org/pypi/${1:-requests}/json" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)['info']
rd = d.get('requires_dist') or []
mandatory = [x for x in rd if 'extra ==' not in x]
print(f\"{d['name']} {d['version']}: {len(mandatory)} mandatory / {len(rd)} total listed\")
for x in mandatory: print(' -', x)
"
```

```bash
# crates.io — normal-kind direct deps from the sparse index, deduped by name
name="${1:-tokio}"; c1="${name:0:2}"; c2="${name:2:2}"
curl -s "https://index.crates.io/${c1}/${c2}/${name}" | tail -1 | python3 -c "
import json, sys
d = json.load(sys.stdin)
deps = [x['name'] for x in d.get('deps', []) if x.get('kind') == 'normal']
print(f\"{d['name']} {d['vers']}: {len(set(deps))} unique / {len(deps)} rows (dupes = per-target cfg variants)\")
"
```

| package | registry | mandatory direct deps | note |
|---|---|---|---|
| `requests` | PyPI | 4 | 2 more (`PySocks`, `chardet`) are optional extras, excluded |
| `flask` | PyPI | 7 | — |
| `black` | PyPI | 8 | — |
| `django` | PyPI | 3 | smallest of the four checked |
| `serde` | crates.io | 2 | `serde_core`, `serde_derive` |
| `regex` | crates.io | 4 | `aho-corasick`, `memchr`, `regex-automata`, `regex-syntax` |
| `clap` | crates.io | 2 | `clap_builder`, `clap_derive` |
| `tokio` | crates.io | **13 unique** (16 raw rows) | `libc` and `mio` each appear more than once — different `cfg(target_os = ...)` variants, not actually duplicate dependencies. The sparse index does not collapse these; a naive `len(deps)` overcounts by 3. |

## Why not recurse PyPI and crates.io the same way as npm

npm's `dependencies` field names a package and a semver *range*; walking the graph by name
alone, ignoring the range, gives you the correct *shape* of the tree (which packages are
reachable at all) even though it can't tell you which exact version would be installed when
ranges conflict. PyPI (`requires_dist`, PEP 508 markers and specifiers) and crates.io
(`deps[].req`) carry the same kind of range information, but recursing them "by name only" the
same naive way would produce numbers that *look* equally precise while quietly skipping real
resolver behavior PyPI and Cargo both do (environment markers, feature unification, picking a
single compatible version across the whole graph). Rather than publish numbers that imply a
rigor they don't have, this repo stops at direct dependencies for those two and says so.

## Caveats (same list as the companion article, kept here too)

- **A floor, not an exact match.** Two packages that depend on genuinely incompatible version
  ranges of the same package can both end up installed side-by-side in a real `node_modules` —
  this crawl counts that package once.
- **`latest` only.** An older major version pinned in your actual lockfile can have a
  completely different dependency set (express 3.x looked nothing like 5.x here).
- **Dev/peer/optional dependencies excluded on all three registries** — only what installs by
  default for a normal consumer.

## The actor behind the npm crawl

[`package-registry-scraper`](https://apify.com/ponderable_hydrometer/package-registry-scraper)
on Apify pulls structured npm/PyPI/crates.io metadata (including `dependencies` /
`requires_dist` / sparse-index `deps[]`) as a dataset — useful if you want this as a pipeline
step instead of a one-off shell script.
