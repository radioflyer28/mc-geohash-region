# MeshCore Geohash Region Spec

Geohash-based region names for MeshCore that let users pick how far their messages travel, from a town, to a state, to several states. Boundaries are predictable: any GPS fix or map lookup tells you exactly which region you're in, unlike airport-code or city-name schemes where the boundary can be ambiguous. This is meant to complement existing region schemes, not replace them. No firmware changes required, this is only a naming spec.

## Quick Start

This spec has two sides: repeater operators *publish* regions, channel users *subscribe* to them.

### Repeater operators

Look up your repeater's location on a [Geohash map](https://www.geohash.es/browse) and copy the 4-character cell. For a repeater in Roanoke, VA that's `dnxh`. Then publish three regions — full cell, drop the last char, drop the last two:

```text
set region gc-dnxh
add region gc-dnx
add region gc-dn
```

That's the minimum.

### Channel users

Pick the tier that matches the reach you want and set it as your channel region. Using the same Roanoke example:

```text
set channel region gc-dnxh   # neighborhood-of-towns scope
set channel region gc-dnx    # metro/sub-state scope
set channel region gc-dn     # multi-state scope
```

Pick a higher precision (more characters) for tighter, more local traffic; a lower precision for wider reach. You only need one — the repeaters covering your area publish all three tiers, so any of them will route to you.

## Goals

This spec exists to give MeshCore operators a small, predictable vocabulary for configuring regions on MeshCore repeaters and channels.

1. **Tier flood and propagation distance.** Offer a clean hierarchy from multi-state regions (2 digit precision) through state/sub-state areas (3 digits) down to town-sized areas (4 digits). Users choose the tier that matches the reach a channel/message should have.
2. **Not to replace but augment existing region schemas.** Meant to preserve existing region configs while offering option to use either. For example, when configuring channels, use `us-va` or `gc-dq`/`gc-dn`, you decide.
2. **Stop short of neighborhood scope.** 5 digit precision and finer are out of scope. The goal is regional control, not neighborhood-level addressing; finer cells fragment the mesh and offer little operational value.
3. **Avoid collisions with existing identifiers.** The `gc` and `gcd` prefixes keep regions distinct from ISO-3166-2 codes already in use on meshes.
4. **Keep the common case trivial.** A repeater that only needs basic regional membership publishes three short strings (`gc-dn`, `gc-dnx`, `gc-dnxh`) and is done.
5. **Allow shape and routing refinement when needed.** The neighbor selector allows selecting multple cells in a single region entry, aiding irregular region shapes and special cases. Using neighbor selectors allows defining additional `gc-` regions beyond the repeater's precision 2-4 single cells. Additionally, distant link (`gcd-`) extensions exist for tailored long-haul routing paths. Neighbor selectors and distant links are opt-in only.

## Rules

1. Use Geohash spec:
    - https://en.wikipedia.org/wiki/Geohash
    - https://www.geohash.es/browse
2. Prefix geohash with `gc-` to avoid collisions with ISO-3166-2 like codes currently in use.
3. Separators: use `-` between the prefix and the geohash. The `+` separator has two roles: in `gc-` entries it introduces the neighbor selector and joins selector tokens; in `gcd-` entries it joins multiple geohash anchors in a corridor entry (see Distant Link).
4. Must set repeater's region to precisions 2, 3, 4 for its current location.
5. May use neighbor selector extension to define additional region(s), however, not in leu of the mandatory 2-4 precision regions outlined above.
    - For example, geohash cell edges/verticies may adversely divide cities, so operators may wish to enlargen the region to correctly encompass the city/metro.
    - For example, mesh/channel operators may wish to enable segmenting large metro areas into N, E, S, W regions.

### Example for Minimum Compliance

The precision 4 geohash for a repeater in Roanoke, VA is `dnxh`, thus determining precision 2 & 3 regions is achieved by simply removing trailing chars.

| Area                             | Precision | Geohash | Region String |
|----------------------------------|-----------|---------|---------------|
| VA, NC, TN, etc.                 | 2         | `dn`    | `gc-dn`       |
| Roanoke, Lynchburg, Danville     | 3         | `dnx`   | `gc-dnx`      |
| North Roanoke, Salem, Daleville  | 4         | `dnxh`  | `gc-dnxh`     |

## Terms/Definitions

- *Region:* a geohash-based identifier (with optional extensions) that a repeater publishes and users/clients subscribe to. The fundamental unit of this spec.
- *Entry / Advertisement:* a single complete region string published by a repeater (e.g., `gc-dnxh+W..N`). A repeater publishes one or more entries.
- *Prefix:* the leading `gc` or `gcd` token identifying the entry type.
- *Selector:* the optional `+`-introduced string after a geohash in a `gc-` entry that names which neighbor cells (and optionally the home cell itself) belong to the entry. Built from compass tokens (`N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW`), the open token `O`, the clockwise arc shorthand `A..B`, and the reserved token `ALL`. Example: the `+W..N+O` in `gc-dnxh+W..N+O`. Selectors are not used in `gcd-` entries.
- *Corridor entry:* a `gcd-` entry naming two or more geohashes joined by `+` (e.g., `gcd-dnh+dnq+dnw`). All listed cells are anchors of a shared long-haul path; every repeater along the corridor publishes the same string. See Distant Link.
- *Precision (p):* the number of geohash characters in the cell identifier. This spec supports p=2, 3, and 4. Higher precision = smaller cell.
- *Home repeater:* the repeater being evaluated for region configuration.
- *Home cell:* the geohash cell containing the home repeater's physical location, at the precision being expressed. Distinct from "home repeater" which is the device.
- *Neighbor cell:* any of the 8 geohash cells immediately surrounding (N, NE, E, SE, S, SW, W, NW) a given cell at the same precision.
- *Adjacent:* sharing an edge or vertex with the home cell at the precision under discussion (i.e., one of the 8 neighbor cells, or the home cell itself).
- *Non-adjacent:* at least one cell removed from the home cell in any direction at the precision under discussion.

## Optional Rule Extensions

### Neighbor Selector

Used to add neighboring cells to a region where shapes are irregular or the repeater sits near cell edges/vertices. The selector is human-readable: operators can write it directly without external tooling.

#### Encoding

- The selector is appended to the geohash with a leading `+`.
- Tokens are the 8 compass directions plus `O` (open):
  `N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW`, `O`.
- Multiple selections are joined with `+` (e.g., `+N+S`).
- A contiguous **clockwise arc** of 2 or more directions MAY be written as `A..B`, where `A` is the start and `B` is the end of the arc, inclusive. Example: `N..E` = `N+NE+E`; `W..N` = `W+NW+N` (wraps through `NW`).
- Arcs are always read clockwise, so direction matters: `N..S` (eastern half: N, NE, E, SE, S) and `S..N` (western half: S, SW, W, NW, N) select different cell sets. Pick the one matching the half you want.
- Wrap-around past N is allowed (e.g., `NE..NW` covers 7 cells, wrapping past S).
- `A..A` is not allowed; use the bare token (e.g., `N`, not `N..N`). Arcs MUST cover ≥ 2 cells.
- Reserved token `ALL` = the full ring of 8 neighbors. Use `ALL` instead of `N..NW`.
- Token `O` (open) excludes the home cell. Allows defining donuts or C/L-shapes. When present, `O` MUST be the last token.
- Canonical output order: directions and arcs listed in **clockwise order starting from N**, then `O` if present. Parsers SHOULD accept any order on input but emitters MUST produce canonical order.
- Tokens are case-insensitive on input; emit uppercase. Geohash characters are always lowercase. (Selectors are only used in `gc-` entries; `gcd-` entries never carry selectors, so this grammar is unambiguous within each entry type.)
- Empty selector (no neighbors, home cell only) → omit the `+` and the selector entirely.
- All tokens are ASCII: 1 char = 1 byte under UTF-8. Worst case at p=4 is using all tokens `gc-dnxh+N+NE+E+SE+S+SW+W+NW+O` = 30 bytes, but in this case +ALL+O should be used instead.

#### Token reference

| Token | Meaning                                                          |
|-------|------------------------------------------------------------------|
| `N`, `NE`, `E`, `SE`, `S`, `SW`, `W`, `NW` | Single compass neighbor             |
| `A..B` | Clockwise arc from `A` through `B`, inclusive (≥ 2 cells)       |
| `ALL`  | All 8 neighbors (full ring)                                     |
| `O`    | Open: exclude home cell. MUST appear last when present.         |

#### Examples

| Neighbors | Geohash | Selector  | Region String        |
| --------- | ------- | --------- | -------------------- |
| None      | `dnxh`  | —         | `gc-dnxh`            |
| All       | `dnxh`  | `ALL`     | `gc-dnxh+ALL`        |
| N         | `dnxh`  | `N`       | `gc-dnxh+N`          |
| E         | `dnxh`  | `E`       | `gc-dnxh+E`          |
| S         | `dnxh`  | `S`       | `gc-dnxh+S`          |
| W         | `dnxh`  | `W`       | `gc-dnxh+W`          |
| N,NE,E    | `dnxh`  | `N..E`    | `gc-dnxh+N..E`       |
| E,SE,S    | `dnxh`  | `E..S`    | `gc-dnxh+E..S`       |
| S,SW,W    | `dnxh`  | `S..W`    | `gc-dnxh+S..W`       |
| W,NW,N    | `dnxh`  | `W..N`    | `gc-dnxh+W..N`       |
| N + S (non-contiguous) | `dnxh` | `N+S` | `gc-dnxh+N+S`     |
| Donut (all + open)     | `dnxh` | `ALL+O` | `gc-dnxh+ALL+O` |
| C-shape (N..S, home open) | `dnxh` | `N..S+O` | `gc-dnxh+N..S+O` |

#### Visual Examples

Each grid is oriented with north up. `■` = cell included, `□` = cell excluded. The center cell is the home geohash, marked with `*`. The center cell is `dnxh`.

**All** — `gc-dnxh+ALL`

| ■   | ■   | ■   |
| --- | --- | --- |
| ■   | ■\* | ■   |
| ■   | ■   | ■   |

**N,NE,E** — `gc-dnxh+N..E`

| □   | ■   | ■   |
| --- | --- | --- |
| □   | ■\* | ■   |
| □   | □   | □   |

**E,SE,S** — `gc-dnxh+E..S`

| □   | □   | □   |
| --- | --- | --- |
| □   | ■\* | ■   |
| □   | ■   | ■   |

**S,SW,W** — `gc-dnxh+S..W`

| □   | □   | □   |
| --- | --- | --- |
| ■   | ■\* | □   |
| ■   | ■   | □   |

**W,NW,N** — `gc-dnxh+W..N`

| ■ | ■ | □ |
|---|---|---|
| ■ | ■\* | □ |
| □ | □ | □ |

**Donut** (Open token set, home cell excluded) — `gc-dnxh+ALL+O`

| ■ | ■ | ■ |
|---|---|---|
| ■ | □\* | ■ |
| ■ | ■ | ■ |

**Eastern half** — `gc-dnxh+N..S`

Clockwise from N to S includes N, NE, E, SE, S.

| □ | ■ | ■ |
|---|---|---|
| □ | ■\* | ■ |
| □ | ■ | ■ |

**Western half** — `gc-dnxh+S..N`

Clockwise from S to N (wrapping past W) includes S, SW, W, NW, N.

| ■ | ■ | □ |
|---|---|---|
| ■ | ■\* | □ |
| ■ | ■ | □ |

### Coverage mapping

For advertising repeater coverage by defining areas of receiving radios (repeaters, room servers, etc) that directly link with your repeater. Allows users to discover repeater's coverage. And allows channel users to limit flooding from an un-usually high elevation repeater to its coverage area.

Coverage entries are simply additional `gc-` regions beyond the mandatory home-cell entries. There is no separate coverage prefix — a `gc-` entry is a membership claim, and the home cell is just the smallest such claim.

- Only use precision 2-4.
- Use the `gc` prefix (same as home-cell entries).
- Coverage regions SHALL contain areas where the home repeater has direct contact with other stationary radios (repeaters, room servers, sensors, etc).
- Coverage regions SHOULD contain areas where the home repeater has direct contact with companion radios. This is difficult to do without extensive analytics, so it's only a goal, not a requirement.
- Coverage regions SHALL include adjacent coverage by using the neighbor selector extension. This aids in reducing usage of repeater's 32 region slots.
- Coverage SHOULD be defined with a single region entry with the neighbor selector where possible to conserve repeater region slots. Multiple coverage entries are meant to be used in highly irregular coverage areas, or when the operator wants users to be able to subscribe to specific sub-regions of the overall coverage area instead of all of it.
- Coverage regions SHOULD use highest precision that results in coverage being only neighboring cells. The goal is to avoid too high a resolution that creates un-necessary chains or gaps between your repeater a nearby repeater.
- Do not use lower resolution when a higher resolution can be used while meeting the other rules.

#### Example

Home repeater is at `dnxh` (Roanoke, VA). Each row below describes a separate coverage advertisement.

| Scenario | Linked repeater(s) | Selector | Region Str |
|----------|--------------------|----------|------------|
| Single link NE to repeater in `dnxm` | NE | `NE` | `gc-dnxh+NE` |
| Two links: E (`dnxk`) and S (`dnx5`) | E, S | `E+S` | `gc-dnxh+E+S` |
| Arc of links W through N | W,NW,N | `W..N` | `gc-dnxh+W..N` |
| Full ring of neighbors at p=4 | All | `ALL` | `gc-dnxh+ALL` |
| Drop to p=3 because closest peer is 2+ p=4 cells away | All | `ALL` | `gc-dnx+ALL` |
| Distant long-range link to `dq` (no neighbor selector) | — | — | `gc-dq` |

#### Visual Examples

Same conventions as Neighbor Selector grids: `■` = cell included in coverage, `□` = excluded, `*` marks the home cell. Letters in cells label the linked repeater geohash (precision 4 unless noted).

**Single link W** — `gc-dnx+W`

Home links to one repeater in `dnw`.

| □ | □ |  |
|---|---|---|
| ■ dnw | ■\* dnx | □ |
| □ | □ | □ |

**Two links: E and S** — `gc-dnxh+E+S`

Home links to repeaters in `dnxk` (E) and `dnx5` (S).

| □ | □ | □ |
|---|---|---|
| □ | ■\* dnxh | ■ dnxk |
| □ | ■ dnx5 | □ |

**Arc W,NW,N** — `gc-dnxh+W..N`

Home links to repeaters across the northwestern arc.

| ■ dnwv | ■ dnxj | □ |
|---|---|---|
| ■ dnwu | ■\* dnxh | □ |
| □ | □ | □ |

**Full ring at p=4** — `gc-dnxh+ALL`

Coverage in every direction at the home precision.

| ■ | ■ | ■ |
|---|---|---|
| ■ | ■\* dnxh | ■ |
| ■ | ■ | ■ |

**Drop to p=3** — `gc-dnx+ALL`

Same shape, but home cell is widened to `dnx` because the closest peer is more than one p=4 cell away. This avoids gaps between home and the linked repeaters.

| ■ | ■ | ■ |
|---|---|---|
| ■ | ■\* dnx | ■ |
| ■ | ■ | ■ |

### Distant Link

Used to advertise long-haul routing paths through the mesh, concentrating traffic onto specific repeaters. Unlike Coverage mapping (which describes *who I can reach*), Distant Link describes *what far-away path I am part of*.

A Distant Link entry is a **corridor entry**: `gcd-A+B[+C…]`, two or more geohashes joined with `+`. It means "I am on the long-haul path connecting all listed cells." The entry is symmetric and bidirectional — every repeater participating on the corridor publishes the **same string**, so traffic flows naturally between any pair of anchors via any participating hop.

#### Rules

- Use `gcd` prefix to denote a distant link region.
- Two or more geohashes joined by `+`. All anchors are peers; the entry is symmetric.
- Only use precision 2-4 for any geohash in the entry.
- Each distant link is published as its own `gcd-` entry. A repeater may publish multiple corridors.
- Every repeater participating on the corridor publishes the **identical string**. This is the mechanism that replaces per-hop pairwise pinning.
- **Canonical order:** geohashes MUST be emitted in ASCII-lexicographic order (e.g., `dnh+dnq+dnw`, not `dnw+dnh+dnq`). Parsers SHOULD accept any order on input but MUST reject duplicates.
- Geohash characters MUST be lowercase. All `+`-separated parts after the prefix are geohash anchors — neighbor selectors are not used in `gcd-` entries.
- Mixed precisions are allowed (e.g., `gcd-dn+dq` pairs two p=2 anchors; `gcd-dnh+dq` pairs a p=3 hop with a p=2 anchor). Use the lowest precision that unambiguously identifies the anchor.
- Each anchor SHOULD be non-adjacent to the others. Adjacent anchors are pointless on a corridor entry — use Coverage mapping instead.
- A publishing repeater’s own home cell SHOULD appear as one of the anchors when it is itself a hop on the corridor. This makes the corridor discoverable from the publisher’s end and avoids implicit asymmetry.
- To express coverage shape of a remote cell, the repeater in that cell advertises its own `gc-` entry with a selector; the long-haul publisher does not assert it.

> [!TIP] How a corridor entry replaces pairwise pinning  
> A corridor entry lets every repeater on a long-haul path publish the **one identical string** — e.g., `gcd-dnh+dq` for an Atlanta⇄DC corridor, or `gcd-dnh+dnq+dnw+dq` for a multi-hop route through Charlotte and Richmond. Subscribers see traffic destined for any anchor flow naturally along the path, in either direction, with no asymmetric configuration.

#### Example

Home repeater is at `dnxh` (Roanoke). Each row is a separate `gcd-` advertisement.

| Scenario | Anchors | Region String |
|----------|---------|---------------|
| Roanoke⇄DC corridor at p=2 | `dn`, `dq` | `gcd-dn+dq` |
| Roanoke⇄DC corridor with finer Roanoke anchor | `dnx`, `dq` | `gcd-dnx+dq` |
| Atlanta⇄Roanoke corridor | `dnh`, `dnx` | `gcd-dnh+dnx` |
| Atlanta⇄Charlotte⇄Richmond⇄DC multi-hop | `dnh`, `dnq`, `dnw`, `dq` | `gcd-dnh+dnq+dnw+dq` |

#### Visual Examples

Conventions: all listed anchors are marked with `+` and shown in their relative positions if co-locatable; otherwise listed beside the grid. `■` = anchor cell, `□` = not part of this entry. Geographic positions are illustrative (not to scale).

**Two-anchor corridor** — `gcd-dn+dq`

Roanoke⇄DC long-haul. Both p=2 cells are anchors; every repeater on the route between them publishes this exact string.

| ■+ dn | ■+ dq |
|-------|-------|

**Four-anchor corridor** — `gcd-dnh+dnq+dnw+dq`

Atlanta⇄Charlotte⇄Richmond⇄DC. Four anchors in lex order; every participating repeater — at any hop — publishes the same string.

| ■+ dnw | □   | ■+ dq  |
|--------|------|--------|
| ■+ dnq | □   | □     |
| ■+ dnh | □   | □     |

