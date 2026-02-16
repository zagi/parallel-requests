---
name: parallel-requests
description: Detect sequential independent HTTP/API calls and refactor to parallel execution. Covers Promise.all (JS/TS), asyncio.gather (Python), and 7 more languages.
license: MIT
metadata:
  author: Michal Zagalski
  version: "2026.2.16"
  trigger: When code makes multiple HTTP/API calls that don't depend on each other's responses
---

# Parallel Requests Skill

Detect sequential independent HTTP/API calls and refactor them to parallel execution patterns.

## When to Use

- Code has 2+ sequential `await fetch()`, `requests.get()`, `axios.get()`, `httpx.get()`, or similar
- Data aggregation from multiple independent APIs/endpoints
- Pages loading data from multiple microservices
- Batch operations that don't depend on each other

## Detection Rules

Before refactoring, verify requests are truly independent:

| Signal | Independent (parallelize) | Dependent (keep sequential) |
|---|---|---|
| URL construction | Static or uses only local vars | Uses response from previous call |
| Request body | No references to prior responses | Contains data from prior response |
| Headers | Static auth token | Token from prior auth call |
| Control flow | No conditional on prior result | `if (responseA.ok)` before B |

## JS/TS Patterns

### BEFORE (sequential — 3x latency)

```ts
const users = await fetch('/api/users').then(r => r.json())
const posts = await fetch('/api/posts').then(r => r.json())
const comments = await fetch('/api/comments').then(r => r.json())
```

### AFTER (parallel — 1x latency)

```ts
const [users, posts, comments] = await Promise.all([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json()),
  fetch('/api/comments').then(r => r.json()),
])
```

### AFTER (with partial failure tolerance)

```ts
const results = await Promise.allSettled([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json()),
  fetch('/api/comments').then(r => r.json()),
])
const [users, posts, comments] = results.map(r =>
  r.status === 'fulfilled' ? r.value : null
)
```

## Python Patterns

### BEFORE (synchronous — 3x latency)

```python
users = httpx.get('/api/users').json()
posts = httpx.get('/api/posts').json()
comments = httpx.get('/api/comments').json()
```

### AFTER (asyncio.gather)

```python
async with httpx.AsyncClient() as client:
    users, posts, comments = await asyncio.gather(
        client.get('/api/users'),
        client.get('/api/posts'),
        client.get('/api/comments'),
    )
```

### AFTER (Python 3.11+ TaskGroup)

```python
async with asyncio.TaskGroup() as tg:
    users_task = tg.create_task(client.get('/api/users'))
    posts_task = tg.create_task(client.get('/api/posts'))
    comments_task = tg.create_task(client.get('/api/comments'))
users, posts, comments = users_task.result(), posts_task.result(), comments_task.result()
```

## Other Languages — Quick Reference

| Language | Pattern | Import |
|---|---|---|
| Go | goroutines + `errgroup.Group` | `golang.org/x/sync/errgroup` |
| Rust | `tokio::join!()` / `futures::join_all()` | `tokio`, `futures` |
| C# | `Task.WhenAll(...)` | `System.Threading.Tasks` |
| Java | `CompletableFuture.allOf(...)` | `java.util.concurrent` |
| PHP | `Utils::all()` (Guzzle Promises) | `guzzlehttp/promises` |
| Ruby | `Async { ... }` | `async` gem |
| Shell | `cmd1 & cmd2 & wait` / `xargs -P N` | built-in |

See `skill/patterns.md` for copy-pasteable code blocks in each language.

## Dependency Detection & Restructuring

When requests have dependencies, don't blindly parallelize — restructure:

1. **Response of A used in B's URL/body** — Keep sequential, but suggest: "Can these be a single batch API call?"
2. **Auth token from A needed for B, C, D** — Get token first, then parallelize B+C+D:
   ```ts
   const token = await getAuthToken()
   const [users, posts, comments] = await Promise.all([
     fetch('/api/users', { headers: { Authorization: token } }).then(r => r.json()),
     fetch('/api/posts', { headers: { Authorization: token } }).then(r => r.json()),
     fetch('/api/comments', { headers: { Authorization: token } }).then(r => r.json()),
   ])
   ```
3. **Pagination** — Sequential pages, but parallelize processing of each page's data
4. **Always ask:** "Is there a batch/bulk endpoint? A GraphQL query that combines these?"

## Error Handling

| Strategy | JS/TS | Python | When |
|---|---|---|---|
| Fail-fast | `Promise.all` | `asyncio.gather()` | All results required |
| Settle all | `Promise.allSettled` | `gather(return_exceptions=True)` | Partial results OK |
| Per-request retry | wrap each promise | wrap each coroutine | Flaky endpoints |

### JS/TS — per-request retry example

```ts
const withRetry = (fn, retries = 3) =>
  fn().catch(err => retries > 0 ? withRetry(fn, retries - 1) : Promise.reject(err))

const [users, posts] = await Promise.all([
  withRetry(() => fetch('/api/users').then(r => r.json())),
  withRetry(() => fetch('/api/posts').then(r => r.json())),
])
```

### Python — per-request retry example

```python
async def with_retry(coro_fn, retries=3):
    for attempt in range(retries):
        try:
            return await coro_fn()
        except Exception:
            if attempt == retries - 1:
                raise

users, posts = await asyncio.gather(
    with_retry(lambda: client.get('/api/users')),
    with_retry(lambda: client.get('/api/posts')),
)
```

## Concurrency Control

For 10+ parallel requests, limit concurrency to avoid overwhelming the server:

| Language | Pattern |
|---|---|
| JS/TS | `p-limit(5)` or chunk array + sequential `Promise.all` per chunk |
| Python | `asyncio.Semaphore(5)` wrapping each coroutine |

### JS/TS — p-limit

```ts
import pLimit from 'p-limit'

const limit = pLimit(5)
const results = await Promise.all(
  urls.map(url => limit(() => fetch(url).then(r => r.json())))
)
```

### JS/TS — chunked processing

```ts
function chunk<T>(arr: T[], size: number): T[][] {
  return Array.from({ length: Math.ceil(arr.length / size) }, (_, i) =>
    arr.slice(i * size, i * size + size)
  )
}

const results = []
for (const batch of chunk(urls, 5)) {
  const batchResults = await Promise.all(
    batch.map(url => fetch(url).then(r => r.json()))
  )
  results.push(...batchResults)
}
```

### Python — Semaphore

```python
sem = asyncio.Semaphore(5)

async def limited_get(client, url):
    async with sem:
        return await client.get(url)

results = await asyncio.gather(
    *(limited_get(client, url) for url in urls)
)
```

## Anti-patterns

- **Don't parallelize ordered side effects** — `POST /create` then `PUT /update` must stay sequential
- **Don't `Promise.all` unbounded arrays without concurrency limit** — you'll DDoS yourself
- **Don't ignore errors** — use `allSettled` or try/catch per request
- **Don't parallelize inside loops where iteration N depends on N-1** — e.g., cursor-based pagination
- **Don't parallelize requests that share mutable state** — race conditions are worse than slow code
