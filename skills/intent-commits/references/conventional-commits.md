# Conventional Commits — message rules

All commit messages produced by this skill MUST follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/).

## Format

```
<type>(<scope>)<!>: <description>

[optional body]

[optional footer(s)]
```

- **type** — one of: `feat`, `fix`, `refactor`, `perf`, `docs`, `test`, `build`, `ci`, `chore`, `style`, `revert`.
- **scope** — optional, lowercase, kebab-case, derived from the touched module/path (e.g. `api`, `auth`, `users-page`).
- **`!`** — append `!` before the colon for a breaking change, AND include a `BREAKING CHANGE: <reason>` footer.
- **description** — imperative mood, lowercase, no trailing period, ≤ 72 chars.
- **body** — wrapped at 72 chars, explains _why_.
- **footers** — `Refs: #123`, `Co-authored-by: …`, `BREAKING CHANGE: …`.

## Two variants per intent

For every intent the agent MUST prepare both:

| Variant | Contents                                                        | Used for                                                                                                              |
| ------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `short` | Subject line only (`<type>(<scope>): <description>`).           | Trivial, self-evident commits.                                                                                        |
| `full`  | Subject + blank line + body (the _why_) + any required footers. | Anything non-obvious; mandatory when `!` breaking, when there is a `Refs:`, or when the body adds meaningful context. |

If `full` is mandatory (breaking change, ref, or non-trivial diff), the `short` variant is still computed and offered, but flagged: `short (not recommended)`.

## Committing with the chosen variant

```sh
# short
git commit -m "<subject>"

# full (one -m per paragraph preserves blank-line separators)
git commit -m "<subject>" -m "<body>" -m "<footer>"
```
