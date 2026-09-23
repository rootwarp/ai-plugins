---
description: 'Look up Python documentation from docs.python.org or PyPI. Use when verifying Python stdlib or package APIs.'
name: python-doc
---

# /python:python-doc

Look up Python documentation from docs.python.org or PyPI.

## Arguments

- `module`: The Python module or package to look up (e.g., `asyncio`, `collections`, `requests`)
- `version` (optional): Python version (e.g., `3.11`). Defaults to `3`.
- `symbol` (optional): Specific function, class, or method to focus on

## Instructions

When the user wants to verify or look up Python documentation:

### Stdlib or Third-Party?

Decide before fetching. A name is stdlib when it is in `sys.stdlib_module_names`:

```bash
python3 -c "import sys; print('<module>'.split('.')[0] in sys.stdlib_module_names)"
```

For third-party code, the import name can differ from the PyPI distribution name (`yaml` is
`PyYAML`, `PIL` is `pillow`, `sklearn` is `scikit-learn`). In a uv project, find the distribution
in `pyproject.toml` or `uv.lock`; otherwise search PyPI for the import name.

### Standard Library

1. **Construct the URL** based on the module:
   - Base URL: `https://docs.python.org/3/library/`
   - With version: `https://docs.python.org/{version}/library/`
   - Example: `https://docs.python.org/3/library/asyncio.html`
   - Submodules use the dotted name in lowercase: `collections.abc.html`,
     `xml.etree.elementtree.html`

2. **Fetch the documentation** using WebFetch with a prompt tailored to the user's needs.

### Third-Party Packages (PyPI)

1. **Find the docs URL** from the PyPI JSON API: `https://pypi.org/pypi/{package}/json`. Use
   `info.project_urls` (the `Documentation` key, or `Homepage`/`Source`) and `info.version`.
   Do not guess a Read the Docs URL; many packages host docs elsewhere.

2. **Fetch the documentation** from that URL using WebFetch. Fall back to the project page
   `https://pypi.org/project/{package}/` when no docs URL is listed.

## URL Structure Reference

| Type | URL Format | Example |
|------|------------|---------|
| Stdlib | `https://docs.python.org/3/library/{module}.html` | `https://docs.python.org/3/library/asyncio.html` |
| Stdlib with version | `https://docs.python.org/{version}/library/{module}.html` | `https://docs.python.org/3.11/library/typing.html` |
| PyPI metadata | `https://pypi.org/pypi/{package}/json` | `https://pypi.org/pypi/requests/json` |
| PyPI page | `https://pypi.org/project/{package}/` | `https://pypi.org/project/requests/` |

## Example Usage

### Look up asyncio module
```
/python:python-doc asyncio
```
Fetches: `https://docs.python.org/3/library/asyncio.html`

### Look up typing with specific version
```
/python:python-doc typing 3.11
```
Fetches: `https://docs.python.org/3.11/library/typing.html`

### Look up third-party package
```
/python:python-doc requests
```
Fetches: `https://pypi.org/pypi/requests/json`, then the `Documentation` URL it lists

## Execution

When invoked, use WebFetch to retrieve the documentation:

```
WebFetch url="https://docs.python.org/3/library/{module}.html" prompt="Extract the documentation for this Python module. Include: 1) Module overview 2) Key classes and functions 3) Common usage patterns 4) Code examples. If a specific symbol was requested, focus on that item's documentation."
```

For third-party packages, first the metadata, then the listed docs URL:

```
WebFetch url="https://pypi.org/pypi/{package}/json" prompt="Return info.version, info.summary, and every entry of info.project_urls."
WebFetch url="{documentation_url}" prompt="Extract the documentation for this package. Include: 1) Overview 2) Key classes and functions 3) Usage examples. If a specific symbol was requested, focus on it."
```

## Error Handling

If the module is not found:
1. Verify the module name is correct
2. Check if it might be a submodule (e.g., `collections.abc`)
3. For third-party packages, check whether the import name differs from the distribution name,
   then search PyPI
4. Suggest related modules if exact match isn't found
