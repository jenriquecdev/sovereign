# Contributing

Thank you for taking the time to contribute. Please read the guidelines below before opening issues or submitting pull requests.

---

## Reporting Issues

When filing a bug report, include all of the following so the issue can be reproduced and triaged efficiently:

- **OS and browser version** — e.g. "Ubuntu 22.04, Firefox 126" or "macOS 14.4, Chrome 125"
- **Steps to reproduce** — a numbered list of the exact actions that trigger the bug
- **Observed behaviour** — what actually happened
- **Expected behaviour** — what should have happened instead

Issues missing any of these fields may be closed without investigation.

---

## Branching Strategy

All development work happens on short-lived topic branches. The naming conventions are:

| Branch type | Pattern | Example |
|-------------|---------|---------|
| New feature | `feature/<short-description>` | `feature/mediaSession-controls` |
| Bug fix | `fix/<short-description>` | `fix/range-header-parsing` |

Rules:

- All branches **must** be created from `main`.
- Direct commits to `main` are **not permitted**. Every change reaches `main` through a pull request.

---

## Pull Requests

Before opening a PR:

1. Make sure your branch is up to date with `main`.
2. Ensure all CI checks pass locally (`pytest`, linting).

Each PR must:

- **Reference the related issue number** in the PR description (e.g. `Closes #42`).
- **Include a description of the change** — what was changed, why, and any relevant context.
- **Pass all CI checks** before it will be considered for merging.

PRs that do not meet these requirements will be sent back for revision.

---

## Code Style: Python

All Python code in this repository MUST comply with [PEP 8](https://peps.python.org/pep-0008/), enforced via [`flake8`](https://flake8.pycqa.org/) or [`ruff`](https://docs.astral.sh/ruff/). The maximum line length is **88 characters**, which is compatible with `black` formatting.

Configuration example for `ruff` (add to `pyproject.toml`):

```toml
[tool.ruff]
line-length = 88
```

Configuration example for `flake8` (add to `.flake8` or `setup.cfg`):

```ini
[flake8]
max-line-length = 88
```

**Type hints are required for all function signatures** in the backend. Every function must declare the types of its parameters and its return value:

```python
def extract_track_metadata(file_path: str) -> dict:
    ...

def derive_stream_id(relative_path: str) -> str:
    ...
```

PRs that introduce untyped function signatures or exceed the 88-character line limit will not be merged until these issues are resolved.

---

## Code Style: JavaScript

All JavaScript code MUST be written in plain **Vanilla JS** with no external framework dependencies (no React, Vue, Angular, or equivalent). The only permitted variable declaration keywords are `const` and `let` — `var` is not allowed.

**Every function must include a JSDoc comment block** describing its parameters and return value:

```javascript
/**
 * Sets the MediaSession metadata for the currently playing track.
 *
 * @param {string} title - The track title.
 * @param {string} artist - The artist name.
 * @param {string} album - The album name.
 * @param {string} artworkUrl - URL to the album artwork image.
 * @returns {void}
 */
const setSessionMetadata = (title, artist, album, artworkUrl) => {
    navigator.mediaSession.metadata = new MediaMetadata({
        title,
        artist,
        album,
        artwork: [{ src: artworkUrl }],
    });
};
```

PRs containing `var` declarations, framework imports, or functions without JSDoc blocks will be sent back for revision.

---

## Testing

All new Python functionality MUST be accompanied by unit tests using [`pytest`](https://docs.pytest.org/). Tests must be placed in a `tests/` directory that mirrors the structure of the source tree:

```
backend/
    metadata.py
    streaming.py
tests/
    test_metadata.py
    test_streaming.py
```

To run the full test suite locally:

```shell
pytest tests/
```

PRs that introduce new Python functions or modules without corresponding tests in `tests/` will not be merged until test coverage is added.
