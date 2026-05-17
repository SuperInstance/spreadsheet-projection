# spreadsheet-projection

PLATOSpreadsheet + SpreadsheetProjection + SpreadsheetCell — PLATO rooms as 3rd-person rendering

## Dependencies

none (standalone)

## Usage

```python
from core.spreadsheet_projection import ...
```

## Shell Loading

This tool can be loaded into any PLATO shell environment:

```python
# Neo loads this tool from the weapon rack
from plato_shell_bridge import PlatoShell
shell = PlatoShell("agent-shell")
shell.load_tool("spreadsheet-projection")
```

## Tests

```bash
python3 -m pytest tests/test_spreadsheet_projection.py -v
```

## License

MIT — Part of the Cocapn Fleet Intelligence System
