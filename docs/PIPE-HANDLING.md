# Pipe Handling

When a bash command contains a pipe (`|`), the plugin must determine which parts of
the pipeline to prefix with `snip run --`.

## Pipeline Segment Resolution

Every segment separated by `|` is independently evaluated via `snip check`. Only
segments whose first command has a snip filter are wrapped.

### Behavior

| Command | Result | Explanation |
|---------|--------|-------------|
| `git log \| head` | `snip run -- git log \| head` | `git` has a filter, `head` does not |
| `cat file.json \| jq '.a \| .b'` | `cat file.json \| snip run -- jq '.a \| .b'` | `jq` has a filter, `cat` does not; pipe in quotes is preserved |
| `cd /tmp && git log \| head` | `cd /tmp && snip run -- git log \| head` | compound `&&` within first pipe segment |
| `cmd1 \|\| cmd2 \| cmd3 && cmd4` | `cmd1 \|\| cmd2 \| cmd3 && cmd4` | `\|\|` treated as compound, not pipe; nothing wrapped (no filters) |

## Quote Awareness

The plugin tracks single quotes (`'`) and double quotes (`"`) while scanning for
pipe characters. A `|` inside quotes is **never** treated as a pipe separator.
This is essential for commands like `jq` that use `|` inside expression strings:

```
cat file.json | jq '.content | fromjson'   # | inside '' is NOT a pipe
cat file.json | jq ".content | fromjson"   # | inside "" is NOT a pipe
```

## Compound Operators Within Segments

Each pipe segment is further split on compound operators (`&&`, `||`, `;`, `&`)
before being checked. This ensures that `cd /tmp && git log` only wraps `git log`,
not `cd /tmp`.

Shell redirections like `2>&1` or `1>&2` are not mistaken for the `&` compound
operator.

## Implementation

The pipe splitting logic lives in `createToolExecuteBefore()` in `src/index.ts`.
It replaces the original `findFirstPipe()` helper which only checked the first
pipe segment.
