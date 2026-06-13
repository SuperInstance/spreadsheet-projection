# Spreadsheet Projection — PLATO Rooms as Living Spreadsheets

A **spreadsheet projection** is a 3rd-person rendering of a PLATO room: every tile, every agent response, every knowledge artifact becomes a cell in a 2D grid with formulas, dependencies, and confidence scores. This library implements the full bridge — PLATO rooms are first-class citizens; the spreadsheet is the human's view of them. It provides dependency graph analysis, bottleneck detection, break-point identification, cascade simulation, and a "Deckboss mode" that renders the same data as an interactive flowchart.

## Why It Matters

Spreadsheets are the most successful end-user programming environment ever built. A billion people use them daily to model finances, track projects, and analyze data. The reason is simple: a cell is a universal primitive. It can hold a value, a formula, a reference to another cell, or an entire sub-sheet. You can zoom into a cell and find another spreadsheet inside.

PLATO rooms have the same fractal structure: a room contains tiles, tiles contain answers, answers reference other tiles, and rooms reference other rooms. Rendering this as a spreadsheet gives humans something they already know how to use — rows, columns, formulas, cell references — while exposing the full complexity of the agent knowledge graph underneath.

The projection is **bidirectional**: changing a cell's value propagates to the underlying room. Changing a room's tiles updates the spreadsheet. This makes the spreadsheet a live interface, not a static export.

## How It Works

### Cell Model

Every entity in a PLATO room maps to a `SpreadsheetCell`:

```python
@dataclass
class SpreadsheetCell:
    cell_id: str           # unique identifier (e.g., "forge:tile:0")
    cell_type: str         # tile, room, value, array, instance, ...
    value: Any             # the cell's current value
    formula: str           # how the value was derived (e.g., "=TILE(forge, 0)")
    dependencies: List[str]  # cells this cell depends on
    dependents: List[str]    # cells that depend on this cell
    confidence: float      # trust score [0.0, 1.0]
    children: List[str]    # sub-cells (for arrays/rooms)
```

The **dependency graph** is the key data structure. It is a directed acyclic graph (DAG) where an edge from cell A to cell B means "A depends on B." When B changes, every downstream cell (transitively dependent on B) must recalculate.

### Cascade Analysis

When a cell changes, the projection computes the **transitive closure** of dependents — all cells that need recalculation:

```python
def cascade_from(self, cell_id: str) -> List[str]:
    affected = []
    to_process = [cell_id]
    processed = set()
    while to_process:
        current = to_process.pop(0)
        if current in processed:
            continue
        processed.add(current)
        cell = self.cells.get(current)
        if cell:
            affected.append(current)
            for dep_id in cell.dependents:
                if dep_id not in processed:
                    to_process.append(dep_id)
    return affected
```

This is **BFS traversal** of the dependents subgraph. Time complexity: O(V + E) where V = affected cells and E = dependency edges among them. Space: O(V).

**Worst case:** A single root cell that all others depend on triggers O(N) recalculation. Average case for sparse dependency graphs is much better — typically O(log N) affected cells per change.

### Bottleneck Detection

A **bottleneck** is a cell with many dependents — many cells depend on its value. If this cell is slow to update or has low confidence, it degrades the entire downstream chain:

```python
def find_bottlenecks(self) -> List[Tuple[str, int]]:
    bottleneck_scores = [
        (cid, len(cell.dependents))
        for cid, cell in self.cells.items()
        if len(cell.dependents) > 0
    ]
    bottleneck_scores.sort(key=lambda x: -x[1])
    return bottleneck_scores[:10]
```

This is O(N log N) for the sort (N = total cells, returning top-10). In practice, the sort is on a small list because most cells have zero or one dependent.

### Break-Point Detection

A **break point** is a cell with many dependencies but low confidence — it is fragile. If any of its upstream dependencies change, the cell's low-confidence value becomes unreliable, potentially cascading errors downstream:

```python
def find_break_points(self) -> List[Tuple[str, float]]:
    break_points = [
        (cid, cell.confidence)
        for cid, cell in self.cells.items()
        if len(cell.dependencies) > 3 and cell.confidence < 0.5
    ]
    break_points.sort(key=lambda x: x[1])  # lowest confidence first
    return break_points[:10]
```

Threshold: >3 dependencies AND confidence < 0.5. These are the cells most likely to propagate errors.

### Deckboss Flowchart Mode

The spreadsheet and the flowchart are **dual views** of the same dependency graph:

```
Spreadsheet View (human-friendly)     Flowchart View (maintenance-focused)
┌─────┬─────┬─────┐                  [Room A] ──────► [Room B]
│ A1  │ B1  │ C1  │                        │                │
│ =42 │ =A1 │ =B1 │                        ▼                ▼
└─────┴─────┴─────┘                   [Tile 0]         [Tile 3]
```

Deckboss mode renders the graph topology, making bottlenecks and break points visually obvious. It is the "maintenance mode" — when the spreadsheet view is too granular and you need to see the interconnection structure.

### Group Mathematics

The projection supports spreadsheet-style aggregate statistics over cell populations:

```python
stats = {
    "count": 42,
    "sum": 1234.5,
    "mean": 29.4,
    "min": 0.1,
    "max": 98.7,
    "by_type": {"tile": 30, "room": 5, "value": 7},
    "total_cells": 42,
    "dependency_edges": 87
}
```

This gives fleet operators "at a glance" health metrics: total knowledge volume, connectivity density, type distribution.

## Quick Start

```bash
# Run the demo (no dependencies — pure Python stdlib)
python3 spreadsheet_projection.py
```

```python
from spreadsheet_projection import (
    PLATOSpreadsheet, SpreadsheetCell, CELL_TYPE_TILE
)

# Create the bridge
bridge = PLATOSpreadsheet()

# Sync rooms
rooms = {
    "forge": [
        {"answer": "constraint α=0.05 converges", "confidence": 0.9},
        {"answer": "drift detected at cycle 847", "confidence": 0.85},
    ],
    "flux": [
        {"answer": "β₁ attractor at 666", "confidence": 0.8},
    ],
}
bridge.sync_all_rooms(rooms)

# Analyze
master = bridge.master_spreadsheet()
print(master.group_statistics())
print(master.find_bottlenecks())
print(master.find_break_points())

# Deckboss flowchart mode
flowchart = bridge.to_deckboss_flowchart()
print(f"Nodes: {len(flowchart['nodes'])}, Edges: {len(flowchart['edges'])}")
```

## API

### Core Classes

| Class | Purpose |
|---|---|
| `SpreadsheetCell` | A single cell: value, formula, dependencies, confidence |
| `SpreadsheetProjection` | A 2D grid of cells (one per room or view) |
| `PLATOSpreadsheet` | Bridge between PLATO rooms and spreadsheet projections |

### Key Methods

| Method | Returns | Description |
|---|---|---|
| `sync_room(name, tiles)` | `SpreadsheetProjection` | Convert room tiles to cells |
| `sync_all_rooms(rooms)` | — | Batch sync multiple rooms |
| `master_spreadsheet()` | `SpreadsheetProjection` | All rooms in one sheet |
| `to_deckboss_flowchart()` | `dict` | Graph view with nodes, edges, bottlenecks |
| `cross_room_dependencies()` | `dict` | Find cells referencing other rooms |
| `cascade_from(cell_id)` | `List[str]` | All cells affected by changing `cell_id` |
| `find_bottlenecks()` | `List[(str, int)]` | Top-10 cells by dependent count |
| `find_break_points()` | `List[(str, float)]` | Top-10 fragile cells (low confidence, high deps) |
| `group_statistics()` | `dict` | Aggregate stats (count, sum, mean, by_type) |

## Architecture Notes

Spreadsheet Projection is the **human interface layer** of the SuperInstance constellation. In the conservation law **γ + η = C**, the projection makes γ and η **visible**: cell confidence reflects γ quality (high-confidence = well-supported generation), dependency density reflects η connectivity (many edges = rich pulse exchange). A healthy fleet produces a spreadsheet with high average confidence and moderate connectivity. A degrading fleet shows break points — low-confidence cells with many dependencies — before any human-visible symptoms appear.

The projection integrates with the PLATO room architecture as the primary rendering surface for fleet operators. See the [SuperInstance Architecture](https://github.com/SuperInstance/SuperInstance/blob/main/ARCHITECTURE.md).

## References

1. Nardi, B. & Miller, J. "The Spreadsheet Interface: A View Through Different Research Paradigms" — why spreadsheets work as a universal computing primitive, [interactions, 1995](https://dl.acm.org/doi/10.1145/199664.199667)
2. Sajaniemi, J. "Modeling Spreadsheet Audit: A Declarative Approach" — dependency graph analysis in spreadsheets, [JFLP 2000](http://www.cs.tufts.edu/~nr/cs257/archive/jorma-sajaniemi/audit.pdf)
3. Knuth, D. E. *The Art of Computer Programming* Vol. 1 — topological sorting and DAG traversal (§2.2.3)

## License

MIT
