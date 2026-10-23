---
description: 'Look up Python documentation on docs.python.org or PyPI. Use when you verify Python stdlib or package APIs.'
name: python-doc
---

# /python:python-doc

Arguments: `<module> [symbol] [version]`. Examples are `asyncio`, `collections.abc Mapping`,
`typing 3.11`, and `requests`.

1. **Stdlib or third-party:** A name is stdlib if its top-level part is in
   `sys.stdlib_module_names`:

   ```bash
   python3 -c "import sys; print('<module>'.split('.')[0] in sys.stdlib_module_names)"
   ```

2. **Stdlib:** Use WebFetch to get `https://docs.python.org/{version}/library/{module}.html`.
   The default `version` is `3` (latest). If the question is about the code of this project,
   use the minor version from `.python-version`. Submodules use the lowercase dotted name
   (`collections.abc.html`, `xml.etree.elementtree.html`).

3. **Third-party:** The import name can be different from the distribution name (`yaml` is
   `PyYAML`, `PIL` is `pillow`, `sklearn` is `scikit-learn`). In a uv project, find the
   distribution and its locked version in `pyproject.toml` or `uv.lock`.
   - Get `https://pypi.org/pypi/{package}/json` (or `/pypi/{package}/{version}/json`).
   - Follow `info.project_urls`: use `Documentation`. If it is not there, use `Homepage` or
     `Source`.
   - Do not guess a Read the Docs URL, because many packages keep their documentation on other
     sites.
   - If no documentation URL is in the list, use `https://pypi.org/project/{package}/`.

4. **Answer** the question. For a symbol, give its signature, its behavior, and an example. For
   a module, give an overview of its key classes and functions.

If you cannot find the documentation, do these checks:

- Check the spelling.
- Check if the name is a submodule (`collections.abc`).
- Check if the import name is different from the distribution name.

Then search PyPI and recommend names that are almost the same.
