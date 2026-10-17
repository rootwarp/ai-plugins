---
description: 'Look up Python documentation from docs.python.org or PyPI. Use when verifying Python stdlib or package APIs.'
name: python-doc
---

# /python:python-doc

Arguments: `<module> [symbol] [version]`, e.g. `asyncio`, `collections.abc Mapping`,
`typing 3.11`, `requests`.

1. **Stdlib or third-party?** A name is stdlib when its top-level part is in
   `sys.stdlib_module_names`:

   ```bash
   python3 -c "import sys; print('<module>'.split('.')[0] in sys.stdlib_module_names)"
   ```

2. **Stdlib:** fetch `https://docs.python.org/{version}/library/{module}.html` with WebFetch.
   `version` defaults to `3` (latest); when the question concerns this project's code, use its
   minor version from `.python-version`. Submodules use the lowercase dotted name
   (`collections.abc.html`, `xml.etree.elementtree.html`).

3. **Third-party:** the import name can differ from the distribution name (`yaml` is `PyYAML`,
   `PIL` is `pillow`, `sklearn` is `scikit-learn`); in a uv project, find the distribution and
   its locked version in `pyproject.toml` or `uv.lock`. Fetch
   `https://pypi.org/pypi/{package}/json` (or `/pypi/{package}/{version}/json`) and follow
   `info.project_urls`: `Documentation`, else `Homepage` or `Source`. Don't guess a Read the
   Docs URL, since many packages host docs elsewhere. With no docs URL listed, fall back to
   `https://pypi.org/project/{package}/`.

4. **Answer** with what was asked: for a symbol, its signature, behavior, and an example; for a
   module, an overview of its key classes and functions.

Not found: check the spelling, whether it is a submodule (`collections.abc`), and whether the
import name differs from the distribution name; then search PyPI and suggest close matches.
