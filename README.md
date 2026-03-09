# blackroad-xml-parser

[![PyPI version](https://img.shields.io/pypi/v/blackroad-xml-parser.svg)](https://pypi.org/project/blackroad-xml-parser/)
[![Python](https://img.shields.io/pypi/pyversions/blackroad-xml-parser.svg)](https://pypi.org/project/blackroad-xml-parser/)
[![License](https://img.shields.io/badge/license-Proprietary-red.svg)](LICENSE)
[![Tests](https://github.com/BlackRoad-OS/blackroad-xml-parser/actions/workflows/ci.yml/badge.svg)](https://github.com/BlackRoad-OS/blackroad-xml-parser/actions)
[![Coverage](https://img.shields.io/badge/coverage-100%25-brightgreen.svg)](#testing)

**BlackRoad XML Parser** is a production-ready Python library and CLI for parsing, validating, querying, transforming, and diffing XML, RSS 2.0, and Atom feed documents — backed by SQLite persistence.

---

## Table of Contents

1. [Overview](#overview)
2. [Features](#features)
3. [Installation](#installation)
4. [Quick Start](#quick-start)
5. [CLI Reference](#cli-reference)
   - [parse](#parse)
   - [xpath](#xpath)
   - [transform](#transform)
   - [validate](#validate)
   - [diff](#diff)
   - [list](#list)
6. [API Reference](#api-reference)
   - [XMLDB](#xmldb)
   - [XMLParser](#xmlparser)
     - [parse](#parse-1)
     - [xpath_query](#xpath_query)
     - [add_schema_rule](#add_schema_rule)
     - [validate_schema](#validate_schema)
     - [transform_to_json](#transform_to_json)
     - [diff_documents](#diff_documents)
     - [list_documents](#list_documents)
7. [Data Models](#data-models)
   - [XMLDocument](#xmldocument)
   - [SchemaRule](#schemarule)
   - [XPathResult](#xpathresult)
   - [TransformResult](#transformresult)
   - [DiffResult](#diffresult)
8. [Database Schema](#database-schema)
9. [Configuration](#configuration)
10. [Testing](#testing)
11. [Contributing](#contributing)
12. [License](#license)

---

## Overview

BlackRoad XML Parser provides a single-file Python module (`src/xml_parser.py`) with zero external runtime dependencies. It combines a rich programmatic API with a colourised CLI, and persists all parsed documents, schema rules, XPath query history, and transform results to a local SQLite database so you can query and audit your data at any time.

---

## Features

| Category | Details |
|---|---|
| 📄 **Multi-format parsing** | XML, RSS 2.0, Atom feeds — auto-detected at parse time |
| 🔍 **XPath queries** | ElementTree `findall`-based queries with full result history stored in SQLite |
| ✅ **Schema validation** | Four rule types: `required_element`, `attribute_required`, `text_pattern`, `max_occurrences` |
| 🔄 **JSON transformation** | Structured tree, flattened row, and normalised RSS/Atom feed output |
| ↔️ **Document diffing** | Tree-level diff reporting added, removed, and modified elements |
| 🔖 **Namespace extraction** | Automatic `xmlns` prefix-to-URI mapping |
| 💾 **SQLite persistence** | Four-table schema: `documents`, `schema_rules`, `xpath_history`, `transforms` |
| 🎨 **ANSI CLI** | Six subcommands with full colour output |

---

## Installation

**Requirements:** Python 3.8+

### From PyPI

```bash
pip install blackroad-xml-parser
```

### From source

```bash
git clone https://github.com/BlackRoad-OS/blackroad-xml-parser.git
cd blackroad-xml-parser
pip install .
```

### Development / testing dependencies

```bash
pip install pytest pytest-cov
```

---

## Quick Start

```python
from xml_parser import XMLDB, XMLParser

db = XMLDB()                          # creates ~/.blackroad/xml_parser.db
parser = XMLParser(db)

# Parse an XML document
doc = parser.parse('<catalog><book id="1"><title>Pro XML</title></book></catalog>',
                   name="catalog")
print(doc.doc_type, doc.element_count)   # xml  3

# XPath query
result = parser.xpath_query(doc.id, ".//book")
print(result.match_count)               # 1

# Transform to JSON
transform = parser.transform_to_json(doc.id)
print(transform.content["tag"])         # catalog

# Schema validation
parser.add_schema_rule("has-book", "required_element", ".//book", "exists")
report = parser.validate_schema(doc.id)
print(report["valid"])                  # True
```

---

## CLI Reference

All subcommands are available via:

```bash
python src/xml_parser.py <subcommand> [options]
```

### `parse`

Parse an XML, RSS, or Atom document and store it in the local database.

```bash
python src/xml_parser.py parse '<root><item/></root>' --name my-doc
python src/xml_parser.py parse feed.rss --file --name my-feed
```

| Argument | Description |
|---|---|
| `source` | XML string or file path |
| `--name` | Human-readable document name (optional) |
| `--file` | Treat `source` as a file path |

---

### `xpath`

Execute an XPath-like query against a previously parsed document.

```bash
python src/xml_parser.py xpath my-doc ".//item"
python src/xml_parser.py xpath my-doc ".//title" --limit 5
```

| Argument | Description |
|---|---|
| `document` | Document name (as stored) |
| `query` | XPath expression (ElementTree `findall` syntax) |
| `--limit` | Maximum number of results to display (default: `20`) |

---

### `transform`

Transform a stored document to JSON and optionally write it to a file.

```bash
python src/xml_parser.py transform my-doc
python src/xml_parser.py transform my-doc --output out.json
python src/xml_parser.py transform my-doc --flatten
python src/xml_parser.py transform my-doc --no-attrs
```

| Argument | Description |
|---|---|
| `document` | Document name |
| `--output`, `-o` | Write JSON to this file path |
| `--flatten` | Emit a flat list of `{tag, text, attribs}` rows instead of a nested tree |
| `--no-attrs` | Exclude element attributes from the output |

---

### `validate`

Validate a stored document against all (or named) schema rules.

```bash
python src/xml_parser.py validate my-doc
python src/xml_parser.py validate my-doc --schema catalog-rules
```

| Argument | Description |
|---|---|
| `document` | Document name |
| `--schema` | Filter to rules matching this name prefix (optional) |

Exit code `1` is returned when validation fails.

---

### `diff`

Compare two stored documents and report structural differences.

```bash
python src/xml_parser.py diff catalog catalog-v2
python src/xml_parser.py diff catalog catalog-v2 --verbose
```

| Argument | Description |
|---|---|
| `doc_a` | First document name |
| `doc_b` | Second document name |
| `--verbose`, `-v` | Print individual added / removed / modified paths |

---

### `list`

List all documents stored in the local database.

```bash
python src/xml_parser.py list
```

---

## API Reference

### `XMLDB`

```python
XMLDB(db_path: str = DB_PATH)
```

Initialises the SQLite database at `db_path`, creating it and all required tables if they do not already exist.

| Parameter | Default | Description |
|---|---|---|
| `db_path` | `~/.blackroad/xml_parser.db` | Path to the SQLite database file. Override with the `XML_DB` environment variable. |

---

### `XMLParser`

```python
XMLParser(db: XMLDB)
```

Main engine class. All methods persist results to the database passed at construction.

---

#### `parse()`

```python
parse(source: str, name: str = None, from_file: bool = False) -> XMLDocument
```

Parse an XML, RSS 2.0, or Atom document from a raw string or file path. Stores the document in `documents` table.

| Parameter | Description |
|---|---|
| `source` | Raw XML string, or a file path when `from_file=True` |
| `name` | Human-readable label stored in the database |
| `from_file` | When `True`, reads `source` as a file path |

Raises `ValueError` on malformed XML. Raises `FileNotFoundError` when `from_file=True` and the path does not exist.

---

#### `xpath_query()`

```python
xpath_query(doc_id: str, query: str) -> XPathResult
```

Execute an XPath expression against a stored document. Results are stored in `xpath_history`.

| Parameter | Description |
|---|---|
| `doc_id` | UUID of the target document |
| `query` | XPath expression (ElementTree `findall` syntax) |

Raises `ValueError` when the document is not found or the query syntax is invalid.

---

#### `add_schema_rule()`

```python
add_schema_rule(
    name: str,
    rule_type: str,
    path: str,
    constraint: str,
    required: bool = True,
    description: str = ""
) -> SchemaRule
```

Add a validation rule to the `schema_rules` table.

| `rule_type` | `path` | `constraint` | Description |
|---|---|---|---|
| `required_element` | XPath to element | Any string | Fail if element is absent |
| `attribute_required` | `elem/path@attr` | Any string | Fail if attribute is missing |
| `text_pattern` | XPath to element | Regex pattern | Fail if text does not match |
| `max_occurrences` | XPath to element | Integer string | Fail if element count exceeds limit |

---

#### `validate_schema()`

```python
validate_schema(doc_id: str, schema_name: str = None) -> Dict[str, Any]
```

Validate a stored document against schema rules. Filters to `schema_name` when provided.

Returns:

```json
{
  "valid": true,
  "violations": [],
  "warnings": [],
  "rules_checked": 3,
  "rules_passed": 3,
  "document": "catalog"
}
```

---

#### `transform_to_json()`

```python
transform_to_json(
    doc_id: str,
    flatten: bool = False,
    include_attrs: bool = True
) -> TransformResult
```

Transform a stored document to a JSON-serialisable Python object. Result is stored in `transforms` table.

| Mode | Output |
|---|---|
| Default (XML) | Nested `{tag, text, attribs, children}` dict |
| `flatten=True` (XML) | Flat list of `{tag, text, attribs}` dicts |
| RSS document | `{type, title, link, description, items, item_count}` |
| Atom document | `{type, title, entries, entry_count}` |

---

#### `diff_documents()`

```python
diff_documents(doc_a_id: str, doc_b_id: str) -> DiffResult
```

Compare two stored documents at the element-tree level.

Returns a `DiffResult` with:
- `added` — elements present in B but not A
- `removed` — elements present in A but not B
- `modified` — elements present in both but with different text or attributes
- `unchanged_count` — number of identical elements

---

#### `list_documents()`

```python
list_documents() -> List[Dict]
```

Return all documents stored in the database, ordered by `created_at DESC`.

---

## Data Models

All models are Python `dataclass` instances.

### `XMLDocument`

| Field | Type | Description |
|---|---|---|
| `id` | `str` | UUID v4 |
| `name` | `str` | Human-readable label |
| `source` | `str` | Original source path or string (first 200 chars) |
| `doc_type` | `str` | One of `xml`, `rss`, `atom`, `html` |
| `content_hash` | `str` | SHA-256 hex digest (first 16 chars) |
| `element_count` | `int` | Total number of elements in the document |
| `depth` | `int` | Maximum nesting depth |
| `namespaces` | `Dict[str, str]` | `xmlns` prefix → URI mapping |
| `created_at` | `str` | ISO 8601 UTC timestamp |
| `metadata` | `Dict[str, Any]` | Arbitrary metadata |

---

### `SchemaRule`

| Field | Type | Description |
|---|---|---|
| `id` | `str` | UUID v4 |
| `name` | `str` | Rule label |
| `rule_type` | `str` | `required_element` \| `attribute_required` \| `text_pattern` \| `max_occurrences` |
| `path` | `str` | XPath expression |
| `constraint` | `str` | Rule-type-specific constraint value |
| `required` | `bool` | `True` → violation; `False` → warning |
| `description` | `str` | Human-readable description |
| `created_at` | `str` | ISO 8601 UTC timestamp |

---

### `XPathResult`

| Field | Type | Description |
|---|---|---|
| `query` | `str` | The executed XPath expression |
| `document_id` | `str` | UUID of the queried document |
| `matches` | `List[Dict]` | List of `{tag, text, attribs, child_count}` dicts |
| `match_count` | `int` | Total number of matches |
| `executed_at` | `str` | ISO 8601 UTC timestamp |

---

### `TransformResult`

| Field | Type | Description |
|---|---|---|
| `document_id` | `str` | UUID of the source document |
| `format_out` | `str` | Output format (currently `"json"`) |
| `content` | `Any` | Transformed data structure |
| `rules_applied` | `int` | Number of transformation rules applied |
| `created_at` | `str` | ISO 8601 UTC timestamp |

---

### `DiffResult`

| Field | Type | Description |
|---|---|---|
| `doc_a_id` | `str` | UUID of the base document |
| `doc_b_id` | `str` | UUID of the comparison document |
| `added` | `List[Dict]` | Elements present in B only |
| `removed` | `List[Dict]` | Elements present in A only |
| `modified` | `List[Dict]` | Elements changed between A and B |
| `unchanged_count` | `int` | Number of identical elements |
| `total_changes` | `int` | Property: `len(added) + len(removed) + len(modified)` |
| `created_at` | `str` | ISO 8601 UTC timestamp |

---

## Database Schema

The SQLite database contains four tables and two indexes.

```sql
-- Parsed documents
CREATE TABLE documents (
    id           TEXT PRIMARY KEY,
    name         TEXT NOT NULL,
    source       TEXT NOT NULL,
    doc_type     TEXT DEFAULT 'xml',
    content_hash TEXT NOT NULL,
    content      TEXT NOT NULL,
    element_count INTEGER DEFAULT 0,
    depth        INTEGER DEFAULT 0,
    namespaces   TEXT DEFAULT '{}',
    metadata     TEXT DEFAULT '{}',
    created_at   TEXT NOT NULL
);

-- Reusable validation rules
CREATE TABLE schema_rules (
    id             TEXT PRIMARY KEY,
    name           TEXT NOT NULL,
    rule_type      TEXT NOT NULL,
    path           TEXT NOT NULL,
    constraint_val TEXT NOT NULL,
    required       INTEGER DEFAULT 1,
    description    TEXT DEFAULT '',
    created_at     TEXT NOT NULL
);

-- XPath query audit trail
CREATE TABLE xpath_history (
    id          TEXT PRIMARY KEY,
    document_id TEXT NOT NULL,
    query       TEXT NOT NULL,
    match_count INTEGER DEFAULT 0,
    result_json TEXT DEFAULT '[]',
    executed_at TEXT NOT NULL
);

-- JSON transform results
CREATE TABLE transforms (
    id          TEXT PRIMARY KEY,
    document_id TEXT NOT NULL,
    format_out  TEXT NOT NULL,
    content     TEXT NOT NULL,
    rules_applied INTEGER DEFAULT 0,
    created_at  TEXT NOT NULL
);

CREATE INDEX idx_docs_name  ON documents(name, created_at DESC);
CREATE INDEX idx_xpath_doc  ON xpath_history(document_id, executed_at DESC);
```

---

## Configuration

| Environment Variable | Default | Description |
|---|---|---|
| `XML_DB` | `~/.blackroad/xml_parser.db` | Override the SQLite database file path |

Example:

```bash
XML_DB=/var/data/my_parser.db python src/xml_parser.py list
```

---

## Testing

The test suite uses **pytest** with coverage reporting.

```bash
# Install test dependencies
pip install pytest pytest-cov

# Run all tests with coverage
pytest tests/ -v --cov=src --cov-report=term-missing
```

Test file: `tests/test_xml_parser.py`

| Test | Description |
|---|---|
| `test_parse_xml` | Parse a generic XML catalog document |
| `test_parse_rss` | Parse an RSS 2.0 feed and verify type detection |
| `test_parse_invalid_xml` | Verify `ValueError` on malformed XML |
| `test_xpath_query` | XPath query returning multiple elements |
| `test_xpath_with_filter` | XPath filter returning specific child elements |
| `test_transform_to_json_xml` | Nested dict transform for a generic XML document |
| `test_transform_rss` | Normalised RSS feed transform |
| `test_schema_validation` | Validation with required and optional rules |
| `test_diff_identical_documents` | Diff of two identical documents returns zero changes |
| `test_diff_different_documents` | Diff of modified document detects changes |
| `test_flatten_transform` | Flat-row transform produces list of dicts |
| `test_list_documents` | List endpoint returns all stored documents |

---

## Contributing

1. Fork the repository and create a feature branch.
2. Make your changes with tests covering all new code paths.
3. Ensure the full test suite passes: `pytest tests/ -v --cov=src`.
4. Open a pull request against `main` with a clear description of the change.

Please follow the existing code style (PEP 8, type annotations, docstrings) and keep the zero-external-runtime-dependency constraint.

---

## License

Proprietary — © BlackRoad OS, Inc. All rights reserved.

See [LICENSE](LICENSE) for full terms.
