# CLI Reference

## Search

```bash
mix ex_ast.search 'PATTERN' PATH [OPTIONS]
```

Search files for AST pattern matches. PATH can be a file, directory, or glob.

### Options

| Flag | Meaning |
|------|---------|
| `-e`, `--pattern PATTERN` | Add a pattern to a multi-pattern batch (repeatable). See [Multiple patterns](#multiple-patterns) |
| `--count` | Print match count only |
| `--count-by-file` | Print per-file match counts, most matches first |
| `--debug-query` | Print how the pattern parsed — signature, `broad?`, retrieval terms, and per-node structure — before searching; with `-e`, one block per pattern (see [Debugging a query](#debugging-a-query)) |
| `--limit N` | Stop after N matches |
| `--allow-broad` | Allow patterns like `_` that match everything |
| `--expand-imports` | Resolve `import Mod` to `Mod`'s real exports, scoped per module, so `map(a, b)` matches `Mod.map(_, _)`. Requires `Mod` to be loadable |
| `--format json` / `--json` | Print structured JSON output |
| `--inside PATTERN` | Only match inside ancestors matching pattern |
| `--not-inside PATTERN` | Reject matches inside ancestors matching pattern |
| `--parent PATTERN` | Direct semantic parent matches pattern |
| `--not-parent PATTERN` | Direct semantic parent does not match pattern |
| `--ancestor PATTERN` | Any semantic ancestor matches pattern |
| `--not-ancestor PATTERN` | No ancestor matches pattern |
| `--has-child PATTERN` | Has a direct child matching pattern |
| `--not-has-child PATTERN` | No direct child matches pattern |
| `--contains PATTERN` | Has a descendant matching pattern |
| `--not-contains PATTERN` | No descendant matches pattern |
| `--follows PATTERN` | Previous sibling matches pattern |
| `--not-follows PATTERN` | No previous sibling matches pattern |
| `--precedes PATTERN` | Following sibling matches pattern |
| `--not-precedes PATTERN` | No following sibling matches pattern |
| `--immediately-follows PATTERN` | Immediately previous sibling matches pattern |
| `--immediately-precedes PATTERN` | Immediately following sibling matches pattern |
| `--first` | First sibling in parent |
| `--not-first` | Not first sibling |
| `--last` | Last sibling in parent |
| `--not-last` | Not last sibling |
| `--nth N` | Nth sibling (1-based) |
| `--not-nth N` | Not nth sibling |
| `--comment TEXT` | Associated comments contain TEXT |
| `--not-comment TEXT` | No associated comments contain TEXT |
| `--comment-before TEXT` | Comment immediately before contains TEXT |
| `--comment-after TEXT` | Comment immediately after contains TEXT |
| `--comment-inside TEXT` | Comment inside range contains TEXT |
| `--comment-inline TEXT` | Inline comment on start line contains TEXT |

Comment values are substring matches. Use `/.../` for regex:

```bash
mix ex_ast.search 'def _ do ... end' --comment-inside '/TODO|FIXME/'
```

### Examples

```bash
# Find all IO.inspect calls
mix ex_ast.search 'IO.inspect(_)' lib/

# Find structs by field
mix ex_ast.search '%Step{id: "subject"}' lib/

# Only inside private functions
mix ex_ast.search --inside 'defp _ do _ end' 'Repo.get!(_, _)'

# Count matches
mix ex_ast.search --count 'dbg(_)' lib/

# Per-file match counts, most matches first
mix ex_ast.search 'IO.inspect(_)' lib/ --count-by-file

# See how a pattern parsed before running it
mix ex_ast.search 'IO.inspect(expr)' lib/ --debug-query

# Resolve a bare `import Enum` to its real exports
mix ex_ast.search 'Enum.map(_, _)' lib/ --expand-imports
```

### Debugging a query

A surprising `0 matches` usually means the pattern parsed differently than you
intended, not that the code isn't there. `--debug-query` prints the parse so you
can see what actually runs:

```console
$ mix ex_ast.search 'Enum.map(data, _)' lib/ --debug-query --count
pattern:    "Enum.map(data, _)"
parsed:     Enum.map(data, _)
signature:  {:call, :map, 2}
multi-node: false
broad?:     false
terms:      alias:Enum, atom:Enum, atom:map, call.remote:Enum.map/2

structure:
  remote call Enum.map, arity 2
    data — capture (binds one node under :data)
    _ — wildcard (matches anything, not captured)

52
```

Each header line answers a different "why zero?":

- **`parsed`** — the pattern after normalization (pipes rewritten, blocks
  unwrapped). If this doesn't look like what you meant, nothing below matters.
- **`signature`** — the `{:call, name, arity}` key used to prefilter candidate
  nodes. A wrong arity or name here (often from a missing/extra arg) rules out
  every node before matching runs.
- **`broad?`** — whether the pattern matches essentially everything (`_`, `...`,
  `[...]`). A broad pattern is *refused before it runs* unless you pass `--limit`
  or `--allow-broad`, so a broad-looking zero is really a refusal.
- **`terms`** — the high-signal terms used to retrieve candidate files from the
  index. A file whose terms don't include one of these is never opened, so a
  term that no file contains (a typo'd module, a struct that doesn't exist)
  yields zero. `(none …)` means retrieval falls back to the signature alone.
- **`structure`** — what each node binds to, distinguishing a **capture**
  (`data`) from a **wildcard** (`_`), an **ellipsis** from a fixed arg, a
  bitstring segment's type from a bound name, and a named callee from a wildcard
  one. Nesting deeper than six levels is elided with `… (deeper nodes not
  shown)`.

The classic false zero is an aliased module. `lib/ex_ast/index.ex` calls
`Terms.from_pattern(pattern)`, but ex_ast indexes calls under their resolved
module, so the short-alias pattern finds none while the fully-qualified one
finds all four — `terms` shows why, looking for `Terms.from_pattern/1` where no
file carries it:

```console
$ mix ex_ast.search 'Terms.from_pattern(_)' lib/ --debug-query --count
pattern:    "Terms.from_pattern(_)"
parsed:     Terms.from_pattern(_)
signature:  {:call, :from_pattern, 1}
multi-node: false
broad?:     false
terms:      alias:Terms, atom:Terms, atom:from_pattern, call.remote:Terms.from_pattern/1

structure:
  remote call Terms.from_pattern, arity 1
    _ — wildcard (matches anything, not captured)

0
$ mix ex_ast.search 'ExAST.Index.Terms.from_pattern(_)' lib/ --count
4
```

A bare `_` shows the refusal case — `broad?: true`, and no terms to narrow on:

```console
$ mix ex_ast.search '_' lib/ --debug-query --count
pattern:    "_"
parsed:     _
signature:  :unknown
multi-node: false
broad?:     true
terms:      (none — retrieval falls back to the signature)

structure:
  _ — wildcard (matches anything, not captured)
```

Special forms are rendered as themselves rather than as raw calls, so
`case`/`cond`/`with`/`for`, anonymous functions, `->` clauses, comprehension and
`do`/`else` blocks, operators, bitstrings, sigils, and ranges read the way you
wrote them. The `signature` line still shows the internal `{:call, ...}` key
these match under, annotated so it isn't mistaken for an ordinary function call:

```console
$ mix ex_ast.search 'case x do _ -> _ end' lib/ --debug-query --count
signature:  {:call, :case, 2}  (case is a special form, matched structurally as a call)
...
structure:
  case expression
    x — capture (binds one node under :x)
    do:
      clause (1 head arg(s)) ->
        _ — wildcard (matches anything, not captured)
        _ — wildcard (matches anything, not captured)
```

Some pattern shapes can't be matched at all — notably the map-update form
`%{map | k: v}`. `--debug-query` flags these up front and skips the search
instead of raising mid-run:

```console
$ mix ex_ast.search '%{map | key: value}' lib/ --debug-query --count
...
unsupported: this pattern's shape crashes the matcher — `mix ex_ast.search`
             will raise instead of returning matches (e.g. the map-update
             form `%{map | k: v}` is not supported for matching).

structure:
  map %{}, 1 entry(ies)
    operator |
      map — capture (binds one node under :map)
      pair:
        literal :key
        value — capture (binds one node under :value)
Skipping search — pattern is unsupported (see above).
```

`ExAST.Pattern.explain/1` returns this same text for use outside the CLI, and
`ExAST.Pattern.matchable?/1` reports whether a pattern's shape can be matched:

```elixir
IO.puts(ExAST.Pattern.explain("Enum.map(data, _)"))
ExAST.Pattern.matchable?("%{map | k: v}")  #=> false
```

## Multiple patterns

Pass a repeatable `-e` / `--pattern` flag to search several patterns in one
invocation. Each file is read and parsed once for the whole batch, avoiding BEAM
startup and per-file re-parsing per pattern — useful for analyzers that run many
checks over the same tree.

```bash
mix ex_ast.search -e 'IO.inspect(_)' -e 'dbg(_)' lib/
```

In this mode there is no positional pattern; remaining positional args are
paths. Combining a positional pattern with `-e` is an error.

### Per-pattern selector filters

Selector-scoping flags (`--inside`, `--not-inside`, `--parent`, `--contains`,
etc.) are *per-pattern*: a filter binds to the most recent preceding `-e`,
mirroring `grep -e`. Filters do not bleed across patterns.

```bash
mix ex_ast.search \
  -e 'App.Repo.get!(_, _)' --inside 'def handle_call(_, _, _) do _ end' \
  -e 'IO.inspect(_)' --not-inside 'test _ do _ end' \
  lib/ test/
```

Global flags (`--count`, `--json`, `--expand-imports`, `--limit`,
`--allow-broad`, paths) apply to the whole batch.

### Debugging a batch

`--debug-query` explains every pattern in the batch, in order, before any file is
read — the same output as [Debugging a query](#debugging-a-query), one block per
pattern:

```bash
mix ex_ast.search -e 'IO.inspect(expr)' -e 'dbg(_)' lib/ --debug-query
```

A pattern flagged `unsupported:` is dropped from the batch rather than taking the
whole run down with it: the remaining patterns still search, and the dropped one
is left out of the tally, so `N pattern(s)` counts only what ran. If every
pattern is unsupported, no search happens at all.

### Output

Each match is tagged by its pattern string:

```
[IO.inspect(_)] lib/foo.ex:12
  IO.inspect(result)

[dbg(_)] lib/bar.ex:88
  dbg(value)

2 pattern(s), 2 match(es)
```

`--count` prints a per-pattern tally plus a total; `--json` includes the
`pattern` field on each match. Duplicate `-e` patterns raise an error, and
`--count-by-file` is not supported with `-e`.

Multi-pattern search uses one shared path list for all patterns, so per-pattern
path include/exclude is not expressible in a single call — group patterns by
shared path scope into separate invocations. Per-pattern *selector filters* do
work, since they live in each pattern's selector.

## Replace

```bash
mix ex_ast.replace 'PATTERN' 'REPLACEMENT' PATH [OPTIONS]
```

Replace AST pattern matches in files. Captures from the pattern are substituted
into the replacement by name.

### Options

Same relationship filters as `search`. Additional:

| Flag | Meaning |
|------|---------|
| `--dry-run` | Preview changes without writing files |
| `--format json` / `--json` | Print structured JSON summary |
| `--format-output` | Run the Elixir formatter on modified files |

### Examples

```bash
# Remove debug calls
mix ex_ast.replace 'dbg(expr)' 'expr' lib/

# Migrate API
mix ex_ast.replace 'Repo.get!(mod, id)' 'Repo.get!(mod, id) || raise NotFoundError' lib/

# Preview without writing
mix ex_ast.replace --dry-run 'use Mix.Config' 'import Config' lib/

# Preview as JSON
mix ex_ast.replace --dry-run --format json 'dbg(expr)' 'expr' lib/

# Format changed files
mix ex_ast.replace --format-output 'dbg(expr)' 'expr' lib/

# Only outside tests
mix ex_ast.replace --not-inside 'test _ do _ end' 'IO.inspect(expr)' 'expr' lib/
```

## Diff

```bash
mix ex_ast.diff FILE1 FILE2 [OPTIONS]
```

Syntax-aware diff between two Elixir files.

### Options

| Flag | Meaning |
|------|---------|
| `--summary` | Print summary lines only |
| `--no-moves` | Disable move detection |
| `--no-color` | Disable colored output |
| `--json` / `--format json` | Print edits as JSON |

### Example output

```
lib/old.ex ↔ lib/new.ex

L2 UPDATE updated function def first/0
  - def first, do: 1
  + def first, do: 10

L5 INSERT inserted function def fourth/0
  + def fourth, do: 4

2 edit(s)
```
