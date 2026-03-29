---
description: Update documentation to match code changes. Generates READMEs, API docs, architecture diagrams, and inline docstrings. Invokes the doc-updater agent.
---

# /update-docs

Invokes the **doc-updater** agent to synchronize documentation with current codebase state.

## What This Command Does

1. **API Documentation** — updates FastAPI OpenAPI annotations, Swagger examples
2. **README Generation** — regenerates project or module READMEs based on code structure
3. **Docstrings** — adds or updates standard Python docstrings (Google, Sphinx, NumPy format)
4. **Architecture Diagrams** — updates Mermaid or PlantUML diagrams describing infrastructure or DB changes
5. **Changelog** — generates changelog entries from git commits or PR descriptions

## When to Use

- After a major feature implementation to update the README
- After changing database schemas to update Data Dictionaries
- When adding new modules that need standard docstrings
- Before a release to generate a cohesive changelog

## Required Context (provide when running)

```
Target: [Full project / Specific folder / Specific file]
Format: [Markdown / Mermaid / Docstrings / OpenAPI]
Focus:  [API changes / Architecture diagram / Installation instructions]
```

## Example Usage

- `/update-docs src/api/users.py add Google style docstrings`
- `/update-docs Write a Mermaid architecture diagram for the new AWS infrastructure we just built`
- `/update-docs Update the root README.md to include instructions on how to use the new Docker setup`
