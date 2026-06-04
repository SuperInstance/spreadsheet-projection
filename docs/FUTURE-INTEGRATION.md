# Future Integration: spreadsheet-projection

## Current State
PLATO spreadsheet projection — renders PLATO rooms as 3rd-person spreadsheet views. Standalone Python with no dependencies. Loadable as a PLATO shell tool.

## Integration Opportunities

### With ternary-spreadsheet
spreadsheet-projection provides the 3rd-person rendering layer for ternary-spreadsheet. Instead of viewing the ternary grid from inside (as one cell), the projection shows the entire room from above — every cell, every connection, every energy flow. This is the "god's eye view" of a room.

### With room-as-codespace
When an agent enters a room, it can switch between 1st-person (inside one cell) and 3rd-person (spreadsheet-projection view). The projection shows the room's overall health: which cells are active, which are dying, where energy is flowing, what the ensign is doing. This is the room's dashboard.

### With superinstance-spreadsheet
The projection's PLATO integration maps to superinstance-spreadsheet's room visualization. Each PLATO room becomes a sheet tab; the projection renders each room's cell grid as a live, updatable spreadsheet. Multi-room views combine sheets from different PLATO rooms into one view.

## Dormant Ideas Now Unlockable
The PLATO shell bridge (`PlatoShell`) was an island. Now the room-as-codespace architecture provides the context: the projection IS the room's visualization layer. The shell bridge becomes the room's API — agents query the projection to understand room state.

## Potential in Mature Systems
Every room has a spreadsheet-projection view accessible from any frontend (terminal, browser, mobile). The projection renders real-time: cells pulse with energy, connections glow with transfer entropy, dying cells fade. It's a living map of the room.

## Cross-Pollination Ideas
- **Spreadsheet-moment**: Projection provides 3rd-person view; Moment provides interactive editing
- **open-minded**: Visualization of the fleet's vector models through projection
- **captains-log**: Projection snapshots in fleet history entries

## Dependencies for Next Steps
- Bridge from Python PLATO shell to Rust ternary-cell
- Real-time update mechanism (WebSocket from ternary grid)
- Multi-room projection composition
