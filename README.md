# MeshCore Geohash Region Spec

[Geohash](https://www.geohash.es/browse) based region definitions for MeshCore.

## Goals

This spec exists to give MeshCore operators a small, predictable vocabulary for configuring regions on MeshCore repeaters and channels. In priority order:

1. **Tier flood and propagation distance.** Offer a clean hierarchy from multi-state regions (2 digit precision) through state/sub-state areas (3 digits) down to town-sized areas (4 digits). Operators choose the tier that matches the reach a message should have.
2. **Not to replace but augment existing region schemas.** Meant to preserve existing region configs while offering option to use either. For example, when configuring channels, use `us-va` or `gc-dq`/`gc-dn`, you decide.
2. **Stop short of neighborhood scope.** 5 digit precision and finer are out of scope. The goal is regional control, not neighborhood-level addressing; finer cells fragment the mesh and offer little operational value.
3. **Avoid collisions with existing identifiers.** The `gc`/`gcc`/`gcd` prefixes keep regions distinct from ISO-3166-2 codes already in use on meshes.
4. **Keep the common case trivial.** A repeater that only needs basic regional membership publishes two or three short strings (`gc-dn`, `gc-dnx`, `gc-dnxh`) and is done.
5. **Allow shape and routing refinement when needed.** The neighbor selector, coverage mapping (`gcc-`), and distant link (`gcd-`) extensions exist for special cases, irregular coverage areas, and deterministic long-haul paths — opt-in, not required.

## Rules

1. Use Geohash spec:
    - https://en.wikipedia.org/wiki/Geohash
    - https://www.geohash.es/browse
2. Prefix geohash with `gc` to avoid collisions with ISO-3166-2 and IATA codes currently in use.
3. Use `-` hyphen separator between fields
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
- *Entry / Advertisement:* a single complete region string published by a repeater (e.g., `gcc-dnxh-61`). A repeater publishes one or more entries.
- *Prefix:* the leading `gc`, `gcc`, or `gcd` token identifying the entry type.
- *Suffix:* the trailing 1- or 2-character Base32 string after a geohash that encodes neighbor selector bits (e.g., the `7z` in `gc-dnxh-7z`).
- *Precision (p):* the number of geohash characters in the cell identifier. This spec supports p=2, 3, and 4. Higher precision = smaller cell.
- *Home repeater:* the repeater being evaluated for region configuration.
- *Home cell:* the geohash cell containing the home repeater's physical location, at the precision being expressed. Distinct from "home repeater" which is the device.
- *Target cell:* in a `gcd-` entry, the geohash cell that a neighbor selector's bits apply to. (For `gc-` and `gcc-` entries the neighbor selector applies to the home cell, so this term is unnecessary there.)
- *Neighbor cell:* any of the 8 geohash cells immediately surrounding (N, NE, E, SE, S, SW, W, NW) a given cell at the same precision.
- *Adjacent:* sharing an edge or vertex with the home cell at the precision under discussion (i.e., one of the 8 neighbor cells, or the home cell itself).
- *Non-adjacent:* at least one cell removed from the home cell in any direction at the precision under discussion.

## Optional Rule Extensions

### Neighbor Selector

Used to add neighboring cells to region where region shapes are irregular or repeater is near cell edges/verticies.

- Encoded with Geohash's modified Base32
- Bit *i* in the table contributes $2^i$ to a 9-bit unsigned integer (bit 0 is the LSB).
- The integer is left-padded to 10 bits with a leading `0` and emitted as two Base32 characters, most-significant 5 bits first.
- If the value fits in 5 bits (< 32), the leading `0` character is omitted, producing a single-char suffix.
- Do not drop trailing zero `0`.
- *Open* bit means to exclude the home cell (or target cell, for `gcd-` entries). Allows defining donuts or C/L-shapes.

| Bit | Direction |
|-----|-----------|
| 0   | N         |
| 1   | NE        |
| 2   | E         |
| 3   | SE        |
| 4   | S         |
| 5   | SW        |
| 6   | W         |
| 7   | NW        |
| 8   | Open      |

#### Examples

| Neighbors | Bits  | Geohash | Suffix | Region String |
| --------- | ----- | ------- | ------ | ------------- |
| None      | -     | `dnxh`  |        | `gc-dnxh`     |
| All       | 0-7   | `dnxh`  | `7z`   | `gc-dnxh-7z`  |
| N         | 0     | `dnxh`  | `1`    | `gc-dnxh-1`   |
| E         | 2     | `dnxh`  | `4`    | `gc-dnxh-4`   |
| S         | 4     | `dnxh`  | `h`    | `gc-dnxh-h`   |
| W         | 6     | `dnxh`  | `20`   | `gc-dnxh-20`  |
| N,NE,E    | 0,1,2 | `dnxh`  | `7`    | `gc-dnxh-7`   |
| E,SE,S    | 2,3,4 | `dnxh`  | `w`    | `gc-dnxh-w`   |
| S,SW,W    | 4,5,6 | `dnxh`  | `3h`   | `gc-dnxh-3h`  |
| W,NW,N    | 6,7,0 | `dnxh`  | `61`   | `gc-dnxh-61`  |
| Donut     | 0-8   | `dnxh`  | `gz`   | `gc-dnxh-gz`  |

#### Visual Examples

Each grid is oriented with north up. `■` = cell included, `□` = cell excluded. The center cell is the home geohash, marked with `*`. The center cell is `dnxh`.

**All** — `gc-dnxh-7z`

| ■   | ■   | ■   |
| --- | --- | --- |
| ■   | ■\* | ■   |
| ■   | ■   | ■   |

**N,NE,E** — `gc-dnxh-7`

| □   | ■   | ■   |
| --- | --- | --- |
| □   | ■\* | ■   |
| □   | □   | □   |

**E,SE,S** — `gc-dnxh-w`

| □   | □   | □   |
| --- | --- | --- |
| □   | ■\* | ■   |
| □   | ■   | ■   |

**S,SW,W** — `gc-dnxh-3h`

| □   | □   | □   |
| --- | --- | --- |
| ■   | ■\* | □   |
| ■   | ■   | □   |

**W,NW,N** — `gc-dnxh-61`

| ■ | ■ | □ |
|---|---|---|
| ■ | ■\* | □ |
| □ | □ | □ |

**Donut** (Open bit set, home cell excluded) — `gc-dnxh-gz`

| ■ | ■ | ■ |
|---|---|---|
| ■ | □\* | ■ |
| ■ | ■ | ■ |

### Coverage mapping

For advertising repeater coverage by defining areas of receiving radios (repeaters, room servers, etc) that directly link with your repeater. Allows users to discover repeater's coverage. And allows channel users to limit flooding from an un-usually high elevation repeater to its coverage area.

- Only use precision 2-4.
- Use `gcc` prefix to denote coverage beyond home cell.
- Always include home cell.
- Coverage regions SHALL contain areas where the home repeater has direct contact with other stationary radios (repeaters, room servers, sensors, etc).
- Coverage regions SHOULD contain areas where the home repeater has direct contact with companion radios. This is difficult to do without extensive analytics, so it's only a goal, not a requirement. 
- Coverage regions SHALL include adjacent coverage by using the neighbor selector extension. This aids in reducing usage of repeater's 32 region slots.
- Coverage SHOULD be defined with a single region entry with the neighbor selector where possible to conserve repeater region slots. Multiple coverage entries are meant to be used in highly irregular coverage areas. Splitting up coverage into sub-regions for more localized traffic SHOULD use `gc-....` instead.
- Coverage regions SHOULD use highest precision that results in coverage being only neighboring cells. The goal is to avoid too high a resolution that creates un-necessary chains or gaps between your repeater a nearby repeater.
- Do not use lower resolution when a higher resolution can be used while meeting the other rules.

#### Example

Home repeater is at `dnxh` (Roanoke, VA). Each row below describes a separate coverage advertisement.

| Scenario | Linked repeater(s) | Bits | Region Str |
|----------|--------------------|------|-------|
| Single link NE to repeater in `dnxm` | NE | 1 | `gcc-dnxh-2` |
| Two links: E (`dnxk`) and S (`dnx5`) | E, S | 2,4 | `gcc-dnxh-n` |
| Arc of links W through N | W,NW,N | 6,7,0 | `gcc-dnxh-61` |
| Full ring of neighbors at p=4 | All | 0-7 | `gcc-dnxh-7z` |
| Drop to p=3 because closest peer is 2+ p=4 cells away | All | 0-7 | `gcc-dnx-7z` |
| Distant long-range link to `dq` (no neighbor selector) | — | — | `gcc-dq` |

#### Visual Examples

Same conventions as Neighbor Selector grids: `■` = cell included in coverage, `□` = excluded, `*` marks the home cell. Letters in cells label the linked repeater geohash (precision 4 unless noted).

**Single link W** — `gcc-dnx-20`

Home links to one repeater in `dnw`.

| □ | □ |  |
|---|---|---|
| ■ dnw | ■\* dnx | □ |
| □ | □ | □ |

**Two links: E and S** — `gcc-dnxh-n`

Home links to repeaters in `dnxk` (E) and `dnx5` (S).

| □ | □ | □ |
|---|---|---|
| □ | ■\* dnxh | ■ dnxk |
| □ | ■ dnx5 | □ |

**Arc W,NW,N** — `gcc-dnxh-61`

Home links to repeaters across the northwestern arc.

| ■ dnwv | ■ dnxj | □ |
|---|---|---|
| ■ dnwu | ■\* dnxh | □ |
| □ | □ | □ |

**Full ring at p=4** — `gcc-dnxh-7z`

Coverage in every direction at the home precision.

| ■ | ■ | ■ |
|---|---|---|
| ■ | ■\* dnxh | ■ |
| ■ | ■ | ■ |

**Drop to p=3** — `gcc-dnx-7z`

Same shape, but home cell is widened to `dnx` because the closest peer is more than one p=4 cell away. This avoids gaps between home and the linked repeaters.

| ■ | ■ | ■ |
|---|---|---|
| ■ | ■\* dnx | ■ |
| ■ | ■ | ■ |

### Distant Link

Used to advertise a directed path to a far-away region, forcing traffic to flow toward that region via this repeater. Unlike Coverage mapping (which describes *who I can reach*), Distant Link describes *where traffic should be routed through me to reach*. This is the mechanism for pinning down deterministic long-haul paths through the mesh and concentrate traffic to specific repeaters for the purpose.

- Use `gcd` prefix to denote a distant link region.
- Only use precision 2-4 for the link target.
- Non-adjacent target cells are strongly preferred — adjacent targets are the Coverage mapping case and SHOULD be expressed there instead.
- Adjacent targets ARE permitted when a directional preference must be enforced over an otherwise ambiguous coverage relationship (e.g., two repeaters in adjacent cells where one is the explicit upstream).
- MAY use the neighbor selector extension on the target cell to widen the destination region (e.g., to support multiple alternative paths for nearby repeaters).
- Do NOT include the home cell in a `gcd` entry. Distant Link entries describe the *target* region only; the home cell is implied by the publishing repeater.
- Each distant link is published as its own `gcd-` entry. A repeater may publish multiple.

> [!IMPORTANT] Distant Link is bidirectional by configuration, not by design.  
> A `gcd-` entry only steers traffic in one direction — toward the target. For the path to actually carry traffic both ways, **every repeater along the route must publish matching entries**:
> - The far-end repeater must publish a `gcd-` entry pointing back at the near-end region.
> - Every intermediate repeater along the path must publish `gcd-` entries that include the next hop in the chain.
>
> *Example:* For an Atlanta ⇄ Washington DC long-haul path, Atlanta-area repeaters publish `gcd-` entries pointing at the DC region, DC-area repeaters publish `gcd-` entries pointing back at Atlanta, and any relay repeaters in between (e.g., Charlotte, Richmond) publish `gcd-` entries pointing at *both* endpoints. Miss any one of those and the path becomes lossy or one-way.

> [!TIP] Convention: shared anchor region  
> To avoid every repeater on a corridor maintaining a full set of pairwise `gcd-` entries, participating repeaters MAY agree by convention to publish a single shared anchor region — e.g., every repeater on the Atlanta ⇄ DC corridor publishes `gcd-dq` (the DC-area cell) regardless of which end they sit on. Traffic destined for the anchor flows naturally toward DC, and traffic in the reverse direction rides the same advertised path back because every hop along the route already carries the same entry. This trades per-hop directional precision for drastically simpler configuration and is the recommended default for well-known long-haul corridors.

#### Example

Home repeater is at `dnxh` (Roanoke). Each row is a separate `gcd-` advertisement.

| Scenario | Target | Region String |
|----------|--------|-------|
| Long-haul link to a repeater in p=2 cell `dq` (offshore example) | `dq` (p=2) | `gcd-dq` |
| Link to a p=3 cell `dqb` to the NE | `dqb` | `gcd-dqb` |
| Link to a repeater near a cell vertex in `dnz`, also serving `dqb` (E) and `dq8` (SE) | `dnz` + neighbors E,SE | `gcd-dnz-d` |
| Forced directional preference to adjacent cell `dnw` (W) | `dnw` | `gcd-dnw` |
| Pin upstream toward a southern p=3 cell | `dnj` (p=3) | `gcd-dnj` |

#### Visual Examples

Conventions: each grid is **anchored on the target cell**, marked with `*`, since the home cell is not part of a `gcd` entry. `■` = cell(s) included in the entry, `□` = not part of this entry. The home repeater (`dnxh`) is somewhere off-grid unless explicitly noted.

**Long-haul link to `dq`** — `gcd-dq`

Single p=2 target cell, no neighbor selector. The target stands alone; home (`dnxh` in `dn`) is many cells away to the SW.

| ■\* dq |
|---|

**Adjacent directional pin W** — `gcd-dnw`

Forces westbound traffic through home toward the repeater in `dnw`. Single p=3 target cell, no neighbor selector. Adjacent target is allowed here because it expresses *direction of preference*, not coverage. (Home `dnx` is the cell immediately E of the target and is not shown.)

| ■\* dnw |
|---|

**Target with neighbor widening** — `gcd-dnz-d`

The target repeater sits near the SE vertex of `dnz` and effectively serves `dnz` plus its E neighbor (`dqb`) and SE neighbor (`dq8`). Neighbor selector `d` = bits 2,3 = E,SE applied to the *target* cell.

| □ | □ | □ |
|---|---|---|
| □ | ■\* dnz | ■ dqb |
| □ | □ | ■ dq8 |

## `meshcore_geohash.py`

Example library for computing neighbor selectors

```python
"""MeshCore Geohash Region — neighbor selector encoding/decoding library.

Implements the Neighbor Selector extension defined in
``MeshCore Geohash Region Spec.md``:

- Bit i (0..8) contributes 2**i to a 9-bit unsigned integer (LSB-first).
- Bits 0..7 select the 8 compass neighbors clockwise from N.
- Bit 8 ("Open") excludes the home/target cell from the region.
- The integer is padded to 10 bits and emitted as two geohash-base32
  characters, most-significant 5 bits first.
- If the value fits in 5 bits (< 32), the leading character is dropped,
  producing a single-char suffix. A trailing '0' is never dropped.

Geohash neighbor lookup is delegated to ``pygeohash``.
"""

from __future__ import annotations

from dataclasses import dataclass
from enum import IntEnum

import pygeohash as pgh


GEOHASH_BASE32 = "0123456789bcdefghjkmnpqrstuvwxyz"
_BASE32_INDEX = {c: i for i, c in enumerate(GEOHASH_BASE32)}


class Direction(IntEnum):
    """Bit positions for the neighbor selector. Clockwise from N."""

    N = 0
    NE = 1
    E = 2
    SE = 3
    S = 4
    SW = 5
    W = 6
    NW = 7
    OPEN = 8  # excludes the home/target cell

    @classmethod
    def compass(cls) -> tuple["Direction", ...]:
        """The 8 compass directions, clockwise from N (excludes OPEN)."""
        return (cls.N, cls.NE, cls.E, cls.SE, cls.S, cls.SW, cls.W, cls.NW)


# Map Direction -> sequence of pygeohash adjacency steps.
# pygeohash uses 'top' (N), 'bottom' (S), 'left' (W), 'right' (E).
_ADJACENT_STEPS: dict[Direction, tuple[str, ...]] = {
    Direction.N: ("top",),
    Direction.NE: ("top", "right"),
    Direction.E: ("right",),
    Direction.SE: ("bottom", "right"),
    Direction.S: ("bottom",),
    Direction.SW: ("bottom", "left"),
    Direction.W: ("left",),
    Direction.NW: ("top", "left"),
}


def neighbor(geohash: str, direction: Direction) -> str:
    """Return the geohash of the neighbor cell in the given compass direction."""
    if direction == Direction.OPEN:
        raise ValueError("OPEN is not a compass direction")
    cell = geohash
    for step in _ADJACENT_STEPS[direction]:
        cell = pgh.get_adjacent(cell, step)
    return cell


def neighbors(geohash: str) -> dict[Direction, str]:
    """Return all 8 compass neighbors of ``geohash``."""
    return {d: neighbor(geohash, d) for d in Direction.compass()}


@dataclass(frozen=True)
class NeighborSelector:
    """A neighbor selector bitmask.

    Use ``NeighborSelector.from_directions(...)`` to construct from
    high-level directions, or ``NeighborSelector(value)`` directly with
    a 9-bit integer.
    """

    value: int

    def __post_init__(self) -> None:
        if not 0 <= self.value <= 0x1FF:
            raise ValueError(
                f"value must fit in 9 bits (0..511), got {self.value}"
            )

    # -- constructors -------------------------------------------------

    @classmethod
    def from_directions(cls, *directions: Direction) -> "NeighborSelector":
        """Build a selector from one or more ``Direction`` values."""
        value = 0
        for d in directions:
            value |= 1 << int(d)
        return cls(value)

    @classmethod
    def from_suffix(cls, suffix: str) -> "NeighborSelector":
        """Decode a 1- or 2-char base32 suffix into a selector."""
        if len(suffix) not in (1, 2):
            raise ValueError(f"suffix must be 1 or 2 chars, got {suffix!r}")
        try:
            if len(suffix) == 1:
                value = _BASE32_INDEX[suffix]
            else:
                high = _BASE32_INDEX[suffix[0]]
                low = _BASE32_INDEX[suffix[1]]
                value = (high << 5) | low
        except KeyError as exc:
            raise ValueError(f"invalid base32 char in {suffix!r}") from exc
        return cls(value)

    # -- queries ------------------------------------------------------

    def has(self, direction: Direction) -> bool:
        """Return True if the given direction's bit is set."""
        return bool(self.value & (1 << int(direction)))

    def directions(self) -> tuple[Direction, ...]:
        """Return all set directions in canonical (bit-order) order."""
        return tuple(d for d in Direction if self.has(d))

    @property
    def excludes_home(self) -> bool:
        """True if the OPEN bit is set (home/target cell excluded)."""
        return self.has(Direction.OPEN)

    # -- encoding -----------------------------------------------------

    def to_suffix(self) -> str:
        """Encode as 1- or 2-char base32 suffix per the spec."""
        high = self.value >> 5
        low = self.value & 0x1F
        if high == 0:
            return GEOHASH_BASE32[low]
        return GEOHASH_BASE32[high] + GEOHASH_BASE32[low]

    def __str__(self) -> str:
        return self.to_suffix()


# -- top-level region builders ----------------------------------------


def format_region(prefix: str, geohash: str, selector: NeighborSelector | None = None) -> str:
    """Compose a complete region entry like ``gc-dnxh-7z``.

    ``prefix`` is one of ``"gc"``, ``"gcc"``, ``"gcd"``.
    ``selector`` is optional; if ``None`` or empty (value 0), no suffix is appended.
    """
    parts = [prefix, geohash]
    if selector is not None and selector.value != 0:
        parts.append(selector.to_suffix())
    return "-".join(parts)


def expand_region(geohash: str, selector: NeighborSelector) -> list[str]:
    """Return all geohash cells covered by ``geohash`` + ``selector``.

    The home/target cell itself is included unless the OPEN bit is set.
    Compass neighbors are included for each set direction bit.
    """
    cells: list[str] = []
    if not selector.excludes_home:
        cells.append(geohash)
    for d in Direction.compass():
        if selector.has(d):
            cells.append(neighbor(geohash, d))
    return cells
```
