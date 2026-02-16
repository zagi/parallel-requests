# parallel-requests

Detect sequential independent HTTP/API calls and refactor them to parallel execution.

## The Problem

```
Sequential:
  GET /users   ████████░░░░░░░░░░░░░░░░  1s
               GET /posts   ████████░░░░░░░░  1s
                             GET /comments   ████████  1s
  Total: ─────────────────────────────────────────── 3s

Parallel:
  GET /users      ████████░░░░░░░░░░░░░░░░  1s
  GET /posts      ████████░░░░░░░░░░░░░░░░  1s
  GET /comments   ████████░░░░░░░░░░░░░░░░  1s
  Total: ─────────────────────────────────── 1s
```

## What This Skill Does

When Claude encounters multiple sequential HTTP/API requests that don't depend on each other's responses, it suggests refactoring them to run in parallel — turning `sum(latencies)` into `max(latencies)`. The skill covers dependency detection (so it won't break code where request B needs response A), error handling strategies, and concurrency control for large batches.

## Primary Languages

**JavaScript/TypeScript** — `Promise.all`, `Promise.allSettled`, `p-limit`

**Python** — `asyncio.gather`, `asyncio.TaskGroup` (3.11+), `asyncio.Semaphore`

Full patterns with BEFORE/AFTER examples in `SKILL.md`.

## Other Languages

| Language | Pattern | Import |
|---|---|---|
| Go | goroutines + `errgroup.Group` | `golang.org/x/sync/errgroup` |
| Rust | `tokio::join!()` / `futures::join_all()` | `tokio`, `futures` |
| C# | `Task.WhenAll(...)` | `System.Threading.Tasks` |
| Java | `CompletableFuture.allOf(...)` | `java.util.concurrent` |
| PHP | `Utils::all()` (Guzzle Promises) | `guzzlehttp/promises` |
| Ruby | `Async { ... }` | `async` gem |
| Shell | `cmd1 & cmd2 & wait` / `xargs -P N` | built-in |

Copy-pasteable code blocks for each language in `references/patterns.md`.

## Installation

### Using npx skills

```bash
npx skills install zagi/parallel-requests
```

### Manual

```bash
git clone git@github.com:zagi/parallel-requests.git
ln -s $(pwd)/skill ~/.agents/skills/parallel-requests
```

## Example

**Before:**
```ts
const users = await fetch('/api/users').then(r => r.json())
const posts = await fetch('/api/posts').then(r => r.json())
const comments = await fetch('/api/comments').then(r => r.json())
```

**After:**
```ts
const [users, posts, comments] = await Promise.all([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json()),
  fetch('/api/comments').then(r => r.json()),
])
```

## Limitations

- Cannot detect all dependency chains — complex data flows across multiple functions may not be caught
- Rate limits and API throttling are the caller's responsibility
- Does not automatically add concurrency limits — suggests them for large batches
- Cannot determine if the target server supports concurrent connections

## License

MIT
