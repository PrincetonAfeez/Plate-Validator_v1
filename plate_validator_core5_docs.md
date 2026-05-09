# Architecture Decision Record

## App 40 — Plate Validator
**DataGuard Group | Document 1 of 5**

### Title
Use a JSON-backed provider registry, regex validation engine, and security middleware for license plate validation.

### Status
Accepted

### Date
2026-05-09

### Context
Plate Validator began as a focused regex exercise: validate California license plates in the standard `1ABC234` pattern and return a clear pass/fail answer. The project then grew into a broader validation tool that supports all 50 United States, United Kingdom and France examples, content filtering, audit history, Rich terminal presentation, CSV batch validation, and simple correction suggestions. The codebase is intentionally small and educational, but it now has enough responsibilities that the design needs a clear separation between data, validation logic, safety checks, persistence, and terminal interaction.

The core constraint is that plate validation rules should be editable without rewriting validation logic. Another important constraint is honesty: the patterns are simplified teaching examples, not DMV-authoritative rules. The README explicitly states that `data/patterns.json` is for learning and demos and that real plate rules vary by plate type, era, and jurisdiction. The design therefore needs to communicate that the app validates against a local training registry, not against live government systems.

### Decision Drivers
- Keep regex pattern data outside application logic.
- Preserve a small beginner-friendly CLI shape.
- Make California validation the original anchor while allowing more regions.
- Provide specific feedback instead of only returning `False`.
- Add content filtering without mixing safety logic into regex matching.
- Support audit history and CSV bulk validation without expanding into a database-backed system.
- Use readable Python and common standard-library modules where possible.
- Accept Rich as a UI dependency because the terminal menu and history output are part of the learning objective.

### Options Considered

#### Option 1 — Hard-code California-only validation in one function
The simplest approach would be one function using a California regex such as `^[0-9][A-Z]{3}[0-9]{3}$`. This would satisfy the earliest version of the project, but it would not demonstrate a scalable validation design. Adding Texas, Florida, other states, or international formats would require editing code repeatedly. Failure messages would also become tangled with pattern matching.

#### Option 2 — Put all region regexes in Python dictionaries
A Python dictionary would be easier than external JSON and still avoid dozens of `if` statements. However, the region catalog would remain coupled to source code. That makes data updates look like logic changes and makes it harder to show the Provider Pattern that the project set out to practice.

#### Option 3 — Use a JSON pattern registry and a separate validation engine
This approach keeps region pattern data in `data/patterns.json` and loads it through `PatternRegistry`. `ValidatorEngine` receives a plate string and region data, cleans the input, runs `re.fullmatch`, and returns a Boolean plus the cleaned plate. This supports region expansion without code changes, and it keeps the engine focused on execution rather than data ownership.

#### Option 4 — Use a full validation framework or external DMV data
A larger production design could pull rules from DMV sources, model plate categories, or use a validation DSL. That would exceed the learning scope. It would also introduce legal and freshness concerns because plate rules change and vary by class. For this academic project, a documented simplified registry is more appropriate.

#### Option 5 — Block inappropriate plates after successful format validation only
The security filter could have been run only after regex validation passed. That would make the final result simpler but would miss a key learning point: format and content safety are separate concerns. The app instead checks both format validity and appropriateness, logs both, and rejects unsafe content independently.

### Decision
Use a compact Provider Pattern architecture:

- `PatternRegistry` loads regional patterns and examples from `data/patterns.json`.
- `ValidatorEngine` performs regex matching, cleaned input creation, failure explanation, and simple correction suggestions.
- `SecurityValidator` performs leetspeak-aware content filtering.
- `AuditManager` stores the last ten validation attempts in JSON.
- `UIManager` renders Rich menus and history tables.
- `PlateValidatorApp` orchestrates interactive commands and CSV bulk validation.

The project remains a single-module application (`plate_validator.py`) with an external data file and tests, rather than a fully packaged multi-module system.

### Rationale
The JSON-backed registry is the strongest architectural choice because it separates policy-like data from code. The validation engine can remain generic: it does not need to know whether a region is California, Texas, the United Kingdom, or France. It only needs a regex, an example, and optionally a description.

The security validator is separate because inappropriate content is not the same problem as plate shape. A plate can match a state pattern but still be rejected for content. A plate can also be malformed but still contain restricted text. Separating these checks makes the result easier to reason about and test.

Using `re.fullmatch` instead of partial matching is important. It prevents strings with extra leading or trailing characters from passing simply because they contain a valid-looking substring. The test suite includes a regression test proving that `17GHT429` does not pass a California pattern even though part of it resembles a valid plate.

The failure explainer improves the user experience while staying simple. It compares cleaned input to the region example by length, character kind, exact character position, and finally pattern-only mismatch. This is not a full regex parser, but it gives useful beginner-friendly feedback.

### Trade-offs Accepted
- The registry patterns are simplified and educational, not legal or authoritative.
- The project uses a single source file, so it is easier to run but less clean than a package with separate modules.
- The audit log stores only the last ten entries and rewrites a JSON array instead of appending JSONL records.
- The content blacklist is intentionally small and must be manually maintained.
- Leetspeak normalization can produce false positives or false negatives in edge cases.
- The fuzzy correction feature only checks one-character swaps from a small map.
- The interactive loop is blocking and does not provide a noninteractive command parser for single validation from shell arguments.
- Rich improves UI but adds a runtime dependency.

### Consequences
The design supports adding new region codes by editing JSON instead of application logic. It also makes unit testing straightforward: tests can instantiate the registry, engine, security validator, audit manager, and app separately.

The single-module structure is acceptable for a learning project, but future maintenance would benefit from splitting it into `registry.py`, `engine.py`, `security.py`, `audit.py`, `ui.py`, `bulk.py`, and `cli.py`. That would reduce scroll distance and make the architecture match the conceptual boundaries already present in the code.

Because the app persists data in the repository-local `data/` directory by default, users must be aware that audit and CSV output files can change during normal operation. This is acceptable for a small CLI, but a production design should use an application data directory or a user-configurable path.

### Superseded By
Not superseded.

### Constitution Alignment
This decision satisfies the Constitution by showing architectural thinking appropriate to scope. The app demonstrates regex fundamentals, JSON data loading, error explanation, security filtering, persistence, CSV processing, and tests. It also acknowledges a key limitation: the plate data is simplified and not an official DMV reference.

-e

---

# Technical Design Document

## App 40 — Plate Validator
**DataGuard Group | Document 2 of 5**

### Purpose & Scope
Plate Validator is a terminal application for validating license plate strings against simplified regional license plate patterns. The original learning goal was California-specific regex validation, especially the `1ABC234` pattern. The app evolved into a broader validation exercise that includes:

- a JSON registry of simplified regional plate patterns;
- format validation using regular expressions;
- cleaned uppercase alphanumeric comparison;
- leetspeak-aware restricted-word filtering;
- detailed validation failure messages;
- fuzzy one-character correction suggestions;
- audit logging of recent validation attempts;
- Rich-based menu and history display;
- CSV batch validation;
- regression tests for engine, registry, security, audit, and bulk behavior.

The app is not a legal compliance validator. Its region patterns are teaching examples. The purpose is to practice data validation, regex matching, error handling, data cleansing, and modular CLI design.

### System Context
The system runs locally from the repository root. It uses `plate_validator.py` as the main executable script and `data/patterns.json` as the region pattern database. Users interact through a Rich-rendered terminal loop. Tests import the same classes directly.

External dependencies are minimal:

- `rich` for terminal panels, tables, and column layouts;
- `pytest` for tests;
- standard library modules: `re`, `json`, `sys`, `os`, and `csv`.

The app reads plate pattern data from JSON, optionally writes audit history to `data/audit_log.json`, and writes CSV batch results to `data/results.csv` unless another output path is provided through the tested method.

### Component Breakdown

#### `PatternRegistry`
Responsibility: Data provider for region formats.

`PatternRegistry` loads a JSON dictionary from `data/patterns.json` by default. It stores only keys that are strings and do not start with `_`, allowing metadata such as `_meta` to live in the JSON file without being treated as a region. `get_format(region_code)` uppercases the requested code and returns the matching region dictionary or `None`.

Important behavior:
- Missing files and invalid JSON do not crash initialization; the registry falls back to an empty pattern dictionary.
- Region lookup is case-insensitive because input is uppercased before lookup.
- Data ownership remains outside `ValidatorEngine`.

#### `SecurityValidator`
Responsibility: Content filter middleware.

`SecurityValidator` holds a restricted-word list and a leetspeak substitution map. It converts input to uppercase, substitutes common leet characters such as `4 -> A`, `0 -> O`, `5 -> S`, and then searches letter runs for restricted content.

Important behavior:
- Long words of four or more letters use letter-boundary logic to reduce false positives.
- Short words of three or fewer letters use substring matching because short offensive terms may be embedded in other strings.
- A small allowlist handles known false positives such as allowing `BUMBLE` for the short word `BUM`.
- The method returns `(True, None)` when safe and `(False, word)` when a restricted word is found.

#### `ValidatorEngine`
Responsibility: Regex execution and validation feedback.

`ValidatorEngine.validate(plate_text, region_data)` extracts the regex from the region dictionary, cleans user input to uppercase alphanumeric characters, and uses `re.fullmatch` to decide validity. It returns a tuple `(valid, clean_plate)`.

Additional methods:
- `_char_kind(ch)` classifies a character as `digit`, `letter`, or `character`.
- `get_failure_reason(plate, region_data)` explains why a cleaned plate failed.
- `suggest_correction(plate, region_data)` tries one-character swaps such as `0/O`, `1/I`, and `5/S` and returns the first corrected candidate that matches.

#### `AuditManager`
Responsibility: JSON persistence for recent validation attempts.

`AuditManager` stores validation attempts in a JSON list at `data/audit_log.json` by default. `log_attempt(entry)` reads existing history, appends the new entry, ensures the `data` directory exists, and writes only the last ten entries. `get_all_logs()` returns an empty list if the file is missing or cannot be parsed.

#### `UIManager`
Responsibility: Terminal presentation.

`UIManager.display_menu(patterns)` renders supported regions as Rich panels in a column layout. `UIManager.display_history(logs)` renders recent audit entries as a Rich table or displays a no-history message when there are no logs.

This component has no validation authority. It is only responsible for presentation.

#### `PlateValidatorApp`
Responsibility: Application orchestration.

`PlateValidatorApp` wires together `PatternRegistry`, `ValidatorEngine`, `SecurityValidator`, and `AuditManager`. Its `run()` method handles interactive commands:

- `Q`: quit;
- `L`: list supported regions;
- `H`: show audit history;
- `B`: start CSV bulk processing;
- any other input: treat as a region code and ask for a plate.

It also contains CSV helpers:
- `_csv_cell(row, name)` performs case-insensitive CSV column lookup.
- `bulk_validate_csv(input_path, output_path)` validates rows from CSV and returns `(results, warnings)`.
- `process_bulk()` prompts for a path and writes default results to `data/results.csv`.

#### `data/patterns.json`
Responsibility: simplified validation data.

The file contains `_meta`, all 50 U.S. state codes, and `UK` and `FR`. Each region has at least:

- `name`;
- `pattern`;
- `example`.

Some regions also include `desc` for clearer pattern mismatch messages.

#### `test_plate_validator.py`
Responsibility: regression verification.

The tests cover California validation, leetspeak filtering, false-positive cases, failure explanations, full-match behavior, registry metadata skipping, audit logging, and CSV bulk validation branches.

### Module Dependency Graph

```text
plate_validator.py
├── re
│   ├── ValidatorEngine regex matching
│   ├── SecurityValidator leetspeak/content scan
│   └── PatternRegistry-independent input cleanup
├── json
│   ├── PatternRegistry loads data/patterns.json
│   └── AuditManager reads/writes audit_log.json
├── os
│   └── AuditManager and CSV output directory checks
├── sys
│   └── interactive quit path
├── csv
│   └── PlateValidatorApp.bulk_validate_csv
└── rich
    ├── Console
    ├── Panel
    ├── Table
    └── Columns

data/patterns.json
└── consumed by PatternRegistry

test_plate_validator.py
└── imports PatternRegistry, ValidatorEngine, SecurityValidator, AuditManager, PlateValidatorApp
```

The conceptual dependency direction is:

```text
Pattern Data → PatternRegistry → PlateValidatorApp
SecurityValidator ─────────────→ PlateValidatorApp
ValidatorEngine ───────────────→ PlateValidatorApp
AuditManager ──────────────────→ PlateValidatorApp
UIManager ─────────────────────→ PlateValidatorApp
```

`ValidatorEngine` does not import or own the registry. This is the central Provider Pattern decision.

### Core Algorithms & Logic

#### Region loading algorithm
1. Open `data/patterns.json`.
2. Parse JSON into `raw`.
3. If `raw` is a dictionary, keep every key-value pair where the key is a string and does not begin with `_`.
4. If loading or parsing fails, initialize an empty registry.
5. `get_format()` uppercases the requested region code and returns the matching entry.

This supports metadata while avoiding fake regions caused by documentation keys.

#### Plate validation algorithm
1. Extract `pattern` from `region_data`.
2. Convert user input to uppercase.
3. Remove all non-`A-Z0-9` characters using `re.sub(r'[^A-Z0-9]', '', ...)`.
4. Use `re.fullmatch(pattern, clean_plate)`.
5. Return `(True, clean_plate)` if the full cleaned string matches.
6. Return `(False, clean_plate)` otherwise.

This means spaces and dashes are ignored for format validation, while lowercase input is normalized rather than rejected. That differs from the earliest tutorial goal that mentioned rejecting lowercase; the current implementation favors cleansing and normalization.

#### Security filtering algorithm
1. Convert input to uppercase.
2. Replace leetspeak symbols and digits using `leet_map`.
3. Remove spaces and dashes.
4. Split normalized text around digits to focus on alphabetic runs.
5. For each letter run and restricted word:
   - if the restricted word is four or more letters, apply a letter-boundary regex;
   - if exact match or suspicious embedded match remains, reject;
   - if the restricted word is three or fewer letters, use substring matching with allowlist exceptions.
6. Return the offending word on failure.

This middleware prevents simple obfuscations such as `B4D` from bypassing the filter.

#### Failure explanation algorithm
1. Clean the example plate from `region_data['example']` to uppercase alphanumeric text.
2. Compare cleaned user plate length to cleaned example length.
3. If lengths differ, return a length mismatch.
4. Compare character kind at each position: digit versus letter.
5. If a kind differs, return the first mismatched position and expected kind.
6. Compare exact characters at each position.
7. If a character differs, return the first mismatched position and expected/found values.
8. If all positions visually match the example but regex still fails, return a pattern-only mismatch.
9. Otherwise return a generic pattern mismatch.

This is intentionally example-driven. It works best when the region example reflects the regex shape.

#### Correction suggestion algorithm
1. Define a swap map for common mistakes: `0/O`, `1/I`, `5/S` and reverse.
2. Iterate over each character in the cleaned plate.
3. If the character has a swap candidate, replace it in a copy.
4. Run `re.fullmatch` against the region pattern.
5. Return the first candidate that passes.
6. Return `None` if no single swap repairs the plate.

The algorithm is intentionally bounded and deterministic. It is not a fuzzy edit-distance search.

#### CSV bulk validation algorithm
1. Open the input CSV using `csv.DictReader`.
2. If the file has no header, emit a warning.
3. Validate that case-insensitive `region` and `plate` headers exist.
4. For each row:
   - read region and plate using case-insensitive lookup;
   - if region is blank, append a `FAIL` result;
   - if region is unknown, append a `FAIL` result;
   - otherwise validate format and security;
   - append `PASS` only when both format and security pass.
5. If there are no data rows after a valid header, emit a warning.
6. Ensure the output directory exists.
7. Write `Region,Plate,Status` rows to the output CSV.
8. Return `(results, warnings)` for tests and caller feedback.

### Data Structures

#### Region data
```python
{
    "name": "California",
    "pattern": "^[0-9][A-Z]{3}[0-9]{3}$",
    "example": "1ABC234"
}
```

Some entries also include:

```python
{"desc": "2 Letters, 2 Digits, 3 Letters"}
```

#### Registry map
```python
{
    "CA": {...},
    "TX": {...},
    "UK": {...},
    "FR": {...}
}
```

Metadata keys beginning with `_` are skipped.

#### Audit entry
```python
{
    "region": "CA",
    "plate": "1ABC234",
    "valid": True,
    "safe": True
}
```

Only the last ten entries are persisted.

#### CSV result row
```python
{
    "Region": "CA",
    "Plate": "7GHT429",
    "Status": "PASS"
}
```

#### Security state
```python
blacklist = ["BAD", "HELL", "UGLY", "CRAP", "BUM", "SHIT", "FUCK"]
leet_map = {"4": "A", "0": "O", "5": "S", ...}
_short_word_allowed_parts = {"BUM": frozenset({"BUMBLE"})}
```

### State Management
The app does not maintain complex in-memory domain state. Each validation attempt is mostly stateless:

- registry data is loaded at app startup;
- security validator contains fixed filter rules;
- engine methods are stateless;
- audit state lives in `data/audit_log.json`;
- CSV results are built in local lists and written to disk;
- the interactive loop constructs one `PlateValidatorApp` and repeatedly calls `run()`.

The biggest mutable state is the audit history file. It is read, appended in memory, truncated to ten entries, and rewritten.

### Error Handling Strategy
The app uses a mix of defensive fallback, warning lists, and direct terminal messages.

- Missing or invalid pattern JSON creates an empty registry instead of crashing.
- Missing audit file returns an empty history.
- Invalid audit JSON returns an empty history.
- Unknown region codes print an error in the interactive UI.
- Missing bulk CSV file prints `File not found` in `process_bulk()`.
- Missing CSV headers return warnings from `bulk_validate_csv()`.
- CSV rows with blank or unknown regions become explicit `FAIL` rows.
- The tests verify warnings for missing columns, no data rows, and no header rows.

The design is forgiving, but one improvement would be a typed exception hierarchy so noninteractive callers can distinguish file, data, validation, and output errors.

### External Dependencies

#### Runtime
- Python 3.x.
- `rich>=13.0.0` for terminal UI.

#### Development/test
- `pytest>=7.0.0`.

#### Standard library
- `re` for regex validation and content filtering.
- `json` for pattern and audit files.
- `csv` for batch processing.
- `os` for path and directory management.
- `sys` for exiting the interactive loop.

There is no `pyproject.toml` in the inspected repository. Dependency installation is described through `requirements.txt`.

### Concurrency Model
The app is single-threaded. It performs blocking terminal input and synchronous file I/O. No concurrency, async, multiprocessing, or background workers are used.

### Known Limitations
- Patterns are simplified and not official DMV references.
- Lowercase is normalized to uppercase, not rejected, even though the original tutorial goal listed lowercase rejection.
- Personalized and commercial California plates are mentioned in the learning goal but are not separately modeled as pattern categories.
- All region formats are treated as one regex per region.
- Audit logging rewrites a small JSON array instead of using append-only JSONL.
- Audit entries do not include timestamps.
- CSV validation records only `PASS` or `FAIL`; it does not include failure reason or correction suggestion.
- Fuzzy suggestions only handle a single one-character swap from a small map.
- Security filtering uses a hardcoded blacklist and allowlist.
- Rich output is useful for humans but not ideal for scripting.
- There is no installable console entry point; users run the Python script directly.

### Design Patterns Used

#### Provider Pattern
`PatternRegistry` provides data to `ValidatorEngine`, allowing validation logic to remain region-agnostic.

#### Middleware
`SecurityValidator` acts as a safety layer separate from regex validation.

#### Controller/Orchestrator
`PlateValidatorApp` coordinates registry, engine, security, audit, and UI.

#### Repository-like persistence helper
`AuditManager` abstracts the audit file.

#### Strategy-like validation data
Each region’s regex pattern acts as a data-driven validation strategy.

#### Functional helper methods
Methods such as `_csv_cell`, `normalize_slot`-style region normalization, and `suggest_correction` keep small pieces of logic isolated.

### Constitution Alignment
The technical design demonstrates appropriate growth from a beginner regex exercise into a small but meaningful validation system. It remains within scope because it avoids network services, databases, and authoritative DMV claims. The tests provide verification evidence, and the README clearly documents the simplified nature of the rules.

-e

---

# Interface Design Specification

## App 40 — Plate Validator
**DataGuard Group | Document 3 of 5**

### Invocation Syntax
The repository exposes a script-style interface rather than a packaged console command.

Canonical interactive invocation:

```bash
python plate_validator.py
```

Typical test invocation:

```bash
pytest
```

Dependency installation:

```bash
pip install -r requirements.txt
```

There is no documented `plate-validator` console script or `python -m plate_validator` package entry point.

### Interactive Command Syntax
After startup, the app displays:

```text
Validator v1.5 (L: List | H: History | B: Bulk | Q: Quit)
Select Option or State Code:
```

Accepted interactive inputs:

```text
L
H
B
Q
CA
TX
FL
UK
FR
<any supported region code>
```

When a region code is entered, the app prompts:

```text
Enter <Region Name> Plate:
```

### Argument Reference Table
Because the app is interactive, command-line arguments are not used.

| Interface | Name | Type | Required | Default | Valid Values | Description |
|---|---:|---|---:|---|---|---|
| Process | script path | path | yes | none | `plate_validator.py` | Python script to run. |
| Interactive command | option | string | yes | none | `L`, `H`, `B`, `Q`, region code | Selects list, history, bulk, quit, or validation flow. |
| Region validation prompt | plate | string | yes | none | free text | Plate input cleaned to uppercase alphanumeric text for regex matching. |
| Bulk prompt | CSV path | path | yes | none | readable CSV path | Input CSV for batch validation. |
| Programmatic method | `bulk_validate_csv(input_path)` | path/string | yes | none | CSV file | Reads rows with `region` and `plate` headers. |
| Programmatic method | `bulk_validate_csv(output_path)` | path/string | no | `data/results.csv` | writable CSV path | Writes results CSV. |

### Input Contract

#### Region code input
Region codes are looked up case-insensitively after `strip().upper()`. Supported codes come from `data/patterns.json`. The inspected registry includes all 50 U.S. states plus `UK` and `FR`.

Examples:

```text
CA
TX
UK
FR
```

Unknown codes are rejected with an invalid-region message.

#### Plate input
Plate input is free-form text. For format validation, it is cleaned as follows:

```python
clean_plate = re.sub(r'[^A-Z0-9]', '', plate_text.upper())
```

This means:
- lowercase letters are converted to uppercase;
- spaces and punctuation are removed before regex validation;
- dashes do not affect format matching;
- symbols do not directly cause format failure unless the cleaned result fails.

Security filtering sees a normalized version of the original input after leetspeak conversion and removal of spaces/dashes.

#### Pattern JSON contract
Each region entry should contain:

| Key | Type | Required | Description |
|---|---|---:|---|
| `name` | string | yes | Human-readable region name. |
| `pattern` | string | yes | Regex used with `re.fullmatch`. |
| `example` | string | yes | Example plate used for failure feedback. |
| `desc` | string | no | Human-readable pattern description. |

Top-level keys beginning with `_` are ignored by `PatternRegistry`.

#### Bulk CSV input contract
The CSV must include headers equivalent to:

```csv
region,plate
CA,7GHT429
TX,ABC1234
```

Header matching is case-insensitive, so `Region,Plate` is valid. Blank region values and unknown region codes produce `FAIL` rows instead of being silently dropped.

### Output Contract

#### Successful validation output
A valid and safe plate is displayed in a Rich panel similar to:

```text
VALID: 1ABC234
California
```

The exact appearance depends on Rich rendering and terminal capabilities.

#### Format failure output
A format failure appears in a Rich panel titled `Format Error`. The message may include:

```text
Length mismatch: Expected 7 chars, got 6.
```

or:

```text
Position 1: expected a digit (like '1'), found 'A'.
```

or:

```text
Pattern mismatch: same length as example but does not match the rule (...).
Did you mean: <candidate>?
```

#### Security failure output
Unsafe content is displayed in a Rich panel titled `Security`:

```text
REJECTED: Found restricted word: 'BAD'
```

#### Region menu output
`L` renders supported region codes and names in Rich panels arranged as columns.

#### History output
`H` renders recent audits in a Rich table with columns:

```text
Region | Plate | Status
```

Status is `PASS` only when both `valid` and `safe` are truthy.

#### Bulk output
Interactive bulk processing prints warnings if applicable and ends with:

```text
Bulk processing complete. Results saved to data/results.csv
```

The CSV output schema is:

```csv
Region,Plate,Status
CA,7GHT429,PASS
XX,1,FAIL
```

### Exit Code Reference
The script does not define a formal exit code table.

| Scenario | Expected behavior |
|---|---|
| User enters `Q` | Calls `sys.exit()`; process exits normally unless interrupted. |
| Normal validation | Continues loop. |
| Invalid region | Prints message and continues loop. |
| Missing bulk file | Prints message and continues loop. |
| Unhandled exception | Python default traceback and nonzero process exit. |
| Tests pass | `pytest` exits 0. |
| Tests fail | `pytest` exits nonzero. |

A future CLI wrapper should define explicit `0`, `1`, and `2` exit codes for scripting.

### Error Output Behavior
Errors are primarily human-readable Rich or terminal messages. The app does not consistently separate stdout and stderr.

Examples:

```text
Invalid Region Code.
File not found.
Warning: Missing required column 'plate'. Use headers 'region' and 'plate' (case-insensitive).
```

Bulk validation returns warning strings programmatically from `bulk_validate_csv()`, which is useful for tests and future noninteractive CLI support.

### Environment Variables
No environment variables are read by the app.

### Configuration Files
The primary configuration/data file is:

```text
data/patterns.json
```

Lookup behavior:

1. `PatternRegistry()` defaults to `data/patterns.json`.
2. A custom file path can be passed to `PatternRegistry(filepath)` in tests or programmatic use.
3. Top-level keys beginning with `_` are ignored.
4. Missing or invalid JSON produces an empty registry.

There is no TOML or environment-based configuration.

### Side Effects
The app can create or modify:

```text
data/audit_log.json
data/results.csv
```

Additional side effects:
- Rich output to terminal;
- blocking interactive prompts;
- CSV output path creation when a directory is specified;
- `data/` directory creation for audit logging and default bulk output.

### Usage Examples

#### Start the app
```bash
python plate_validator.py
```

#### List supported regions
```text
Select Option or State Code: L
```

#### Validate California standard plate
```text
Select Option or State Code: CA
Enter California Plate: 1ABC234
```

Expected result:

```text
VALID: 1ABC234
```

#### Validate with punctuation that gets cleaned
```text
Select Option or State Code: CA
Enter California Plate: 1-ABC-234
```

Cleaned value:

```text
1ABC234
```

Expected result: valid, assuming the cleaned value matches the California regex.

#### Trigger format failure
```text
Select Option or State Code: CA
Enter California Plate: ABC1234
```

Likely feedback:

```text
Position 1: expected a digit (like '1'), found 'A'.
```

#### Trigger content filter
```text
Select Option or State Code: CA
Enter California Plate: B4D-PLT
```

Expected result:

```text
REJECTED: Found restricted word: 'BAD'
```

#### Show recent history
```text
Select Option or State Code: H
```

#### Run bulk validation
```text
Select Option or State Code: B
Enter CSV path (e.g., data/input.csv): data/input.csv
```

Expected CSV input:

```csv
region,plate
CA,7GHT429
XX,1
```

Expected output file:

```csv
Region,Plate,Status
CA,7GHT429,PASS
XX,1,FAIL
```

#### Run tests
```bash
pytest
```

### Constitution Alignment
The interface is appropriate for a learning-stage CLI. It is human-first and intentionally simple, but it still supports a testable programmatic seam through `bulk_validate_csv()`. The biggest interface limitation is the absence of a noninteractive command-line mode for single validation.

-e

---

# Runbook

## App 40 — Plate Validator
**DataGuard Group | Document 4 of 5**

### Prerequisites
- Python 3.x.
- `pip`.
- Terminal capable of interactive input.
- Repository files present, especially:
  - `plate_validator.py`
  - `data/patterns.json`
  - `requirements.txt`
  - `test_plate_validator.py`

Runtime dependency:

```text
rich>=13.0.0
```

Test dependency:

```text
pytest>=7.0.0
```

### Installation Procedure
From a clean checkout:

```bash
cd Plate-Validator_v1
python -m venv .venv
```

Activate the environment.

Windows PowerShell:

```powershell
.\.venv\Scripts\Activate.ps1
```

macOS/Linux:

```bash
source .venv/bin/activate
```

Install dependencies:

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

There is no inspected `pyproject.toml`, so editable package installation is not required.

### Configuration Steps
Verify pattern data exists:

```bash
python - <<'PY'
from plate_validator import PatternRegistry
reg = PatternRegistry()
print(len(reg.patterns))
print(reg.get_format('CA'))
PY
```

Expected:
- nonzero pattern count;
- California data with pattern `^[0-9][A-Z]{3}[0-9]{3}$`.

No additional configuration is required.

### Standard Operating Procedures

#### Run the interactive app
```bash
python plate_validator.py
```

Use commands:

```text
L   list regions
H   show recent audit history
B   bulk CSV processing
Q   quit
CA  validate California plate
TX  validate Texas plate
UK  validate United Kingdom plate
FR  validate France plate
```

#### Validate one plate interactively
1. Start app.
2. Enter a supported region code such as `CA`.
3. Enter the plate string.
4. Read the Rich panel result.
5. Use `H` to confirm it was logged.

#### Process a CSV batch
Create `data/input.csv`:

```csv
region,plate
CA,7GHT429
CA,ABC1234
XX,1
```

Run the app:

```bash
python plate_validator.py
```

Choose:

```text
B
```

Enter:

```text
data/input.csv
```

Review:

```text
data/results.csv
```

#### Run tests
```bash
pytest
```

### Health Checks

#### Health check 1 — Registry loads California
```bash
python - <<'PY'
from plate_validator import PatternRegistry
reg = PatternRegistry()
ca = reg.get_format('CA')
assert ca is not None
assert ca['example'] == '1ABC234'
print('registry ok')
PY
```

#### Health check 2 — California valid plate passes
```bash
python - <<'PY'
from plate_validator import ValidatorEngine
engine = ValidatorEngine()
ca = {'pattern': '^[0-9][A-Z]{3}[0-9]{3}$'}
assert engine.validate('7GHT429', ca)[0] is True
print('engine ok')
PY
```

#### Health check 3 — Partial match fails
```bash
python - <<'PY'
from plate_validator import ValidatorEngine
engine = ValidatorEngine()
ca = {'pattern': '^[0-9][A-Z]{3}[0-9]{3}$'}
assert engine.validate('17GHT429', ca)[0] is False
print('fullmatch ok')
PY
```

#### Health check 4 — Leetspeak filter catches restricted term
```bash
python - <<'PY'
from plate_validator import SecurityValidator
security = SecurityValidator()
assert security.is_appropriate('B4D-PLT')[0] is False
print('security ok')
PY
```

#### Health check 5 — Test suite
```bash
pytest
```

### Expected Output Samples

#### Valid plate
```text
VALID: 7GHT429
```

#### Invalid region
```text
Invalid Region Code.
```

#### Format failure
```text
Length mismatch: Expected 7 chars, got 6.
```

#### Security failure
```text
REJECTED: Found restricted word: 'BAD'
```

#### Empty history
```text
No history found.
```

#### Bulk success
```text
Bulk processing complete. Results saved to data/results.csv
```

### Known Failure Modes

| Failure Mode | Trigger | Expected Output/Behavior | Resolution |
|---|---|---|---|
| Missing dependency | Rich not installed | `ModuleNotFoundError: rich` | Run `pip install -r requirements.txt`. |
| Empty registry | Missing/invalid `data/patterns.json` | Region lookups fail | Restore `data/patterns.json`. |
| Unknown region | User enters unsupported code | `Invalid Region Code.` | Use `L` to list supported codes. |
| Bad CSV path | Bulk path does not exist | `File not found.` | Provide correct CSV path. |
| CSV missing `plate` | Header lacks plate column | Warning string from bulk validator | Use `region,plate` headers. |
| CSV no rows | Header exists but no data | Warning: no data rows | Add data rows. |
| Unsafe plate | Restricted word after leet normalization | Security rejection | Choose a different plate. |
| Format mismatch | Cleaned plate does not match regex | Failure reason panel | Adjust to region example/pattern. |
| Audit JSON corrupted | Invalid `audit_log.json` | History resets to empty | Delete or repair audit file. |

### Troubleshooting Decision Tree

```text
App will not start
├── ModuleNotFoundError: rich?
│   └── Install requirements.txt.
├── Syntax/import issue?
│   └── Confirm you are running from the repo root with Python 3.
└── Terminal display odd?
    └── Try a standard terminal that supports Rich output.

Region code fails
├── Did you type a supported code?
│   └── Use L to list regions.
├── Is data/patterns.json present?
│   └── Restore the data file.
└── Is the JSON valid?
    └── Parse it with python -m json.tool data/patterns.json.

Plate fails unexpectedly
├── Is length different from the example?
│   └── Match the region example length after removing separators.
├── Is a digit where a letter is expected, or reverse?
│   └── Follow the failure position feedback.
├── Is a restricted word present after leetspeak normalization?
│   └── Choose a safe plate string.
└── Could O/0 or I/1 be swapped?
    └── Check the suggestion line if shown.

Bulk CSV fails
├── File path wrong?
│   └── Provide a readable CSV path.
├── Missing headers?
│   └── Use region,plate.
├── Unknown region rows?
│   └── They are intentionally marked FAIL.
└── Need diagnostics per row?
    └── Current CSV only outputs PASS/FAIL; inspect manually or extend the output schema.
```

### Dependency Failure Handling
If Rich is missing, the app cannot import because Rich imports occur at module import time. Install dependencies:

```bash
python -m pip install rich pytest
```

or:

```bash
python -m pip install -r requirements.txt
```

If `pytest` is missing, runtime app usage still works, but tests cannot run.

### Recovery Procedures

#### Restore patterns
If `data/patterns.json` is damaged, restore it from version control:

```bash
git checkout -- data/patterns.json
```

#### Reset audit history
```bash
rm data/audit_log.json
```

Windows PowerShell:

```powershell
Remove-Item data\audit_log.json
```

The next validation attempt recreates it.

#### Reset bulk results
```bash
rm data/results.csv
```

Windows PowerShell:

```powershell
Remove-Item data\results.csv
```

#### Verify JSON syntax
```bash
python -m json.tool data/patterns.json > /tmp/patterns.validated.json
```

### Logging Reference
Audit logs are written to:

```text
data/audit_log.json
```

Schema:

```json
[
  {
    "region": "CA",
    "plate": "1ABC234",
    "valid": true,
    "safe": true
  }
]
```

Only the last ten entries are stored. The log has no timestamps. A future improvement would use JSONL with timestamps and failure reasons.

### Maintenance Notes
- Keep `patterns.json` honest by preserving the `_meta` warning that patterns are simplified teaching examples.
- Add tests whenever a new region pattern or blacklist rule is changed.
- Avoid adding complex validation categories into one regex if it makes feedback unreadable.
- Consider splitting the single file into modules before adding more features.
- Consider adding a noninteractive command mode for CI/script usage.
- Consider making audit and output paths configurable.
- Keep offensive-word filtering conservative to avoid excessive false positives.

### Constitution Alignment
The runbook provides repeatable setup, test, recovery, and operating procedures. It acknowledges dependency, data, and persistence failure modes and keeps support guidance proportional to the app’s small CLI scope.

-e

---

# Lessons Learned

## App 40 — Plate Validator
**DataGuard Group | Document 5 of 5**

### Project Summary
Plate Validator is a data validation CLI centered on license plate pattern matching. It started with California regex validation and expanded into a broader Provider Pattern application. It now loads simplified regional patterns from JSON, validates cleaned plate strings with `re.fullmatch`, filters inappropriate content with leetspeak normalization, explains failures, suggests simple corrections, logs recent attempts, renders Rich terminal menus, and validates CSV batches.

This is a strong closing app for the early DataGuard-style section of the portfolio because it combines many validation concerns: pattern matching, input cleansing, structured data loading, persistence, batch processing, and tests.

### Original Goals vs. Actual Outcome

#### Original goals
- Validate standard California plates in the `1ABC234` format.
- Validate personalized/commercial-like plates with 2–7 alphanumeric characters.
- Reject prohibited symbols or lowercase letters.
- Return a Boolean and a specific failure message.
- Practice regex, error handling, and data cleansing.

#### Actual outcome
The final implementation goes beyond the original California-only exercise. It includes all 50 U.S. states plus UK and France examples, a JSON registry, security middleware, failure feedback, audit logging, Rich UI, CSV bulk processing, and tests.

The actual behavior differs from one early goal: lowercase input is not rejected. It is uppercased and cleaned before regex validation. This is a reasonable data-cleansing choice, but it should be documented as an intentional change from strict rejection.

The personalized/commercial California requirement is not modeled as a separate California rule category. The app uses one regex per region. That is acceptable for a simplified training project but would be insufficient for a production DMV-grade validator.

### Technical Decisions That Paid Off

#### External JSON registry
Moving patterns into `data/patterns.json` made the project extensible. Adding new regions does not require editing validation code. The `_meta` skip logic also shows maturity because it allows documentation in the JSON file without breaking the registry.

#### `re.fullmatch`
Using full-string matching prevents partial-match bugs. The regression test for `17GHT429` confirms the app does not accept extra leading characters for a California pattern.

#### Separate security validator
Keeping content filtering separate from format validation makes the system easier to test and reason about. A plate can be syntactically valid but unsafe, and the app records both `valid` and `safe` in the audit entry.

#### Leetspeak normalization
Normalizing inputs such as `B4D` before restricted-word checks is a meaningful validation upgrade. It shows awareness that user input can be adversarial or intentionally obfuscated.

#### Failure explanation helper
The failure reason logic makes regex validation more user-friendly. Instead of saying only “invalid,” it can explain length mismatches, digit/letter mismatches, exact character mismatches, and pattern-only mismatches.

#### CSV bulk validation seam
`bulk_validate_csv()` is testable because it accepts explicit input and output paths and returns both results and warnings. This makes batch validation more reliable than a purely interactive flow.

### Technical Decisions That Created Debt

#### Single-file architecture
The file is readable for a small project, but it now contains registry, security, engine, audit, UI, app orchestration, and CSV logic. That is too many responsibilities for one file if the project grows further.

#### Audit log as rewritten JSON array
Rewriting a JSON list is simple, but it is not ideal for logs. It loses older entries, has no timestamp, and can be corrupted if a write is interrupted. JSONL would be more robust.

#### Hardcoded blacklist
The blacklist and allowlist live in code. This makes it harder to update policy without editing source. A JSON or TOML content policy file would match the Provider Pattern used for plate formats.

#### Example-based failure feedback
The failure explainer depends heavily on the example string. If the example does not represent the regex accurately, feedback may be misleading. The engine does not parse regex structure; it compares against a representative example.

#### Rich dependency at import time
Because Rich is imported at the top level, core validation classes cannot be imported without Rich installed. Splitting UI code from core logic would let engine tests and library use avoid UI dependencies.

#### No formal noninteractive CLI
The app is useful interactively, but it lacks a command like:

```bash
python plate_validator.py validate CA 1ABC234 --json
```

That limits scripting, automation, and shell integration.

### What Was Harder Than Expected

#### Balancing cleansing and strictness
The early goal said to reject lowercase and prohibited symbols, but the final engine normalizes and strips non-alphanumeric characters. That design is user-friendly, but it changes semantics. Validation projects often need a clear decision: reject messy input, or normalize it and validate the cleaned value.

#### Content filtering without false positives
The `HELL` versus `HELLO` case shows why naive substring checks can be too aggressive. The allowlist for `BUMBLE` shows the same issue for short words. Content filtering is deceptively difficult because users can create edge cases easily.

#### Explaining regex failures
Regexes are compact, but explaining them is hard. The project uses the example as a proxy for the pattern, which is good enough for teaching but not comprehensive.

#### Batch validation edge cases
CSV files can have missing headers, differently cased headers, blank regions, unknown regions, and no data rows. The test suite shows that robust batch handling requires more than looping through rows.

### What Was Easier Than Expected

#### Adding regions through JSON
Once `PatternRegistry` existed, expanding the registry was straightforward. The validation engine did not need to change for each new state.

#### Testing core classes directly
Because the classes are importable and most methods return values, tests can bypass the Rich interactive UI and verify behavior directly.

#### Using `re.fullmatch`
Full matching simplified correctness. It avoided needing to manually anchor every validation call beyond the patterns themselves.

#### Keeping CSV output simple
A three-column result file is easy to understand and easy to test. It is limited but practical for a first batch-processing feature.

### Python-Specific Learnings

#### Regular expressions
The project practices:
- anchored patterns such as `^[0-9][A-Z]{3}[0-9]{3}$`;
- `re.fullmatch` for whole-string validation;
- `re.sub` for input cleansing;
- dynamically compiled patterns for boundary matching;
- regex-driven splitting of normalized letter runs.

#### JSON I/O
The project uses `json.load` and `json.dump` for both configuration-like pattern data and audit persistence. It also handles missing and malformed JSON defensively.

#### CSV I/O
`csv.DictReader` and `csv.DictWriter` provide a clean way to support batch processing. The project also shows how to normalize headers case-insensitively.

#### Rich terminal output
Rich panels, tables, and columns make terminal output much more readable. The trade-off is dependency coupling if UI imports live in the same module as core validation logic.

#### Testable class design
Even in a single file, separating behavior into classes made tests easier. `ValidatorEngine`, `SecurityValidator`, `PatternRegistry`, and `AuditManager` can be tested without running the interactive loop.

### Architecture Insights

#### Data-driven validation scales better than conditional logic
The registry proves that validation rules can be data rather than code. This is a major step beyond `if state == "CA"` branching.

#### Middleware is a useful mental model
Security filtering is separate from regex validation. This resembles middleware in larger systems: input passes through safety checks and domain checks before a final decision is shown.

#### Small persistence is still persistence
Even a ten-entry audit file introduces file paths, missing files, corrupted JSON, and write behavior. Persistence should be designed intentionally even when small.

#### User feedback is part of architecture
The failure explainer is not cosmetic. It changes the system from a Boolean validator into a teaching tool. For validation software, the quality of failure messages often determines usefulness.

#### Tests reveal design seams
The presence of tests for CSV warnings, registry metadata skipping, and leetspeak false positives shows that the app has separable concerns. Those tests also point toward future module boundaries.

### Testing Gaps
The existing tests cover many important paths, including:

- California valid and invalid format;
- leetspeak filtering;
- false-positive prevention for `HELLO` and `BUMBLE`;
- failure reason branches;
- full-match behavior;
- registry `_meta` skipping;
- audit log round-trip;
- CSV success and warning paths.

Remaining gaps:

- Rich UI rendering is not deeply tested.
- The interactive `run()` loop is not tested end to end.
- `process_bulk()` prompt behavior is not tested.
- Correction suggestions are not directly tested.
- More region patterns should have representative tests.
- Corrupted `patterns.json` fallback could be tested.
- Audit truncation to last ten entries could be tested.
- Security behavior for every blacklist word could be parameterized.
- Lowercase normalization should be tested and documented as expected behavior.

### Reusable Patterns Identified

#### Registry provider
A JSON-backed provider can be reused in other validation apps: postal codes, phone numbers, product codes, SKU formats, or country-specific IDs.

#### Engine plus data dictionary
A generic engine that receives pattern data is reusable anywhere validation rules differ by category.

#### Security middleware
The content filter pattern can be applied to username validators, vanity-code validators, and user-generated identifiers.

#### Failure reason helper
Example-based failure messages are useful in many beginner validation tools. They provide a middle ground between generic failure and full parser-level diagnostics.

#### Bulk CSV method returning results and warnings
Returning structured results and warnings makes batch logic easy to test and reuse outside an interactive CLI.

### If I Built This Again
I would keep the Provider Pattern but split the code into modules:

```text
plate_validator/
├── __init__.py
├── __main__.py
├── registry.py
├── engine.py
├── security.py
├── audit.py
├── bulk.py
├── ui.py
├── cli.py
└── data/patterns.json
```

I would add an argparse CLI with subcommands:

```bash
plate-validator validate CA 1ABC234
plate-validator validate CA 1ABC234 --json
plate-validator list
plate-validator history
plate-validator bulk input.csv --output results.csv
```

I would store audit logs as JSONL with timestamps:

```json
{"timestamp":"...","region":"CA","plate":"1ABC234","valid":true,"safe":true,"reason":null}
```

I would move the blacklist and allowlist into a data file, add configurable policy loading, and make the security validator optional through a CLI flag for testing.

I would also represent multiple plate categories per region:

```json
"CA": {
  "name": "California",
  "patterns": [
    {"kind": "standard", "pattern": "...", "example": "1ABC234"},
    {"kind": "personalized", "pattern": "...", "example": "MYCAR"}
  ]
}
```

That would better match the original standard-versus-personalized goal.

### Open Questions
- Should the engine reject lowercase and symbols strictly, or should it continue cleansing them?
- Should validation output include both original and cleaned plate values?
- Should audit logs include timestamps and failure reasons?
- Should CSV output include the reason for `FAIL`?
- Should security policy be externalized into JSON?
- Should each state support multiple plate classes?
- Should the README more clearly distinguish original tutorial goals from final implemented behavior?
- Should Rich UI be optional so core validation works without UI dependencies?
- Should official DMV references be explicitly out of scope in every command output?

### Constitution Checklist
- **Authenticity:** The code reflects a learner-built validation project with comments, incremental feature growth, and tests.
- **Scope discipline:** The app grew beyond California validation, but it remains a small CLI and documents that its patterns are simplified.
- **Engineering quality:** Provider Pattern, separated classes, and tests show appropriate architecture for the project size.
- **Verification:** Tests cover engine, security, registry, audit, and CSV behavior.
- **Reflection:** The main trade-offs are clear: simplified data, single-file structure, and human-first interactive design.
- **Roadmap progression:** This app demonstrates regex, data-driven validation, persistence, CSV processing, UI formatting, and regression testing.

-e
