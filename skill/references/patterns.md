# Parallel Request Patterns

Copy-pasteable patterns for parallelizing HTTP/API requests.

---

## JavaScript / TypeScript

### Basic Promise.all

```ts
const [users, posts, comments] = await Promise.all([
  fetch('/api/users').then(r => r.json()),
  fetch('/api/posts').then(r => r.json()),
  fetch('/api/comments').then(r => r.json()),
])
```

### Promise.allSettled with result mapping

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

### Concurrency-limited with p-limit

```ts
import pLimit from 'p-limit'

const limit = pLimit(5)
const results = await Promise.all(
  urls.map(url => limit(() => fetch(url).then(r => r.json())))
)
```

### Chunked processing for large arrays

```ts
function chunk<T>(arr: T[], size: number): T[][] {
  return Array.from({ length: Math.ceil(arr.length / size) }, (_, i) =>
    arr.slice(i * size, i * size + size)
  )
}

const results: any[] = []
for (const batch of chunk(urls, 5)) {
  const batchResults = await Promise.all(
    batch.map(url => fetch(url).then(r => r.json()))
  )
  results.push(...batchResults)
}
```

---

## Python

### asyncio.gather (basic)

```python
import asyncio
import httpx

async def fetch_all():
    async with httpx.AsyncClient() as client:
        users, posts, comments = await asyncio.gather(
            client.get('/api/users'),
            client.get('/api/posts'),
            client.get('/api/comments'),
        )
    return users.json(), posts.json(), comments.json()
```

### TaskGroup (Python 3.11+)

```python
import asyncio
import httpx

async def fetch_all():
    async with httpx.AsyncClient() as client:
        async with asyncio.TaskGroup() as tg:
            users_task = tg.create_task(client.get('/api/users'))
            posts_task = tg.create_task(client.get('/api/posts'))
            comments_task = tg.create_task(client.get('/api/comments'))
    return users_task.result(), posts_task.result(), comments_task.result()
```

### Semaphore-bounded concurrency

```python
import asyncio
import httpx

async def fetch_all(urls: list[str], max_concurrent: int = 5):
    sem = asyncio.Semaphore(max_concurrent)

    async def limited_get(client, url):
        async with sem:
            return await client.get(url)

    async with httpx.AsyncClient() as client:
        return await asyncio.gather(
            *(limited_get(client, url) for url in urls)
        )
```

### aiohttp session pattern

```python
import asyncio
import aiohttp

async def fetch_all(urls: list[str]):
    async with aiohttp.ClientSession() as session:
        tasks = [session.get(url) for url in urls]
        responses = await asyncio.gather(*tasks)
        return [await r.json() for r in responses]
```

### Sync fallback with ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor
import requests

def fetch_all(urls: list[str], max_workers: int = 5):
    with ThreadPoolExecutor(max_workers=max_workers) as pool:
        responses = list(pool.map(requests.get, urls))
    return [r.json() for r in responses]
```

---

## Go

```go
import (
    "context"
    "io"
    "net/http"
    "golang.org/x/sync/errgroup"
)

func fetchAll(ctx context.Context) ([][]byte, error) {
    urls := []string{"/api/users", "/api/posts", "/api/comments"}
    results := make([][]byte, len(urls))

    g, ctx := errgroup.WithContext(ctx)
    for i, url := range urls {
        i, url := i, url // pin loop vars for Go <1.22
        g.Go(func() error {
            resp, err := http.Get(url)
            if err != nil {
                return err
            }
            defer resp.Body.Close()
            results[i], err = io.ReadAll(resp.Body)
            return err
        })
    }
    if err := g.Wait(); err != nil {
        return nil, err
    }
    return results, nil
}
```

## Rust

```rust
use reqwest::Client;
use tokio;

async fn fetch_all(client: &Client) -> Result<(String, String, String), reqwest::Error> {
    let (users, posts, comments) = tokio::join!(
        client.get("/api/users").send(),
        client.get("/api/posts").send(),
        client.get("/api/comments").send(),
    );
    Ok((
        users?.text().await?,
        posts?.text().await?,
        comments?.text().await?,
    ))
}
```

## C#

```csharp
using System.Net.Http;
using System.Threading.Tasks;

async Task<(string users, string posts, string comments)> FetchAllAsync(HttpClient client)
{
    var usersTask = client.GetStringAsync("/api/users");
    var postsTask = client.GetStringAsync("/api/posts");
    var commentsTask = client.GetStringAsync("/api/comments");

    await Task.WhenAll(usersTask, postsTask, commentsTask);

    return (usersTask.Result, postsTask.Result, commentsTask.Result);
}
```

## Java

```java
import java.net.http.*;
import java.util.concurrent.CompletableFuture;

CompletableFuture<Void> fetchAll(HttpClient client) {
    var users = client.sendAsync(
        HttpRequest.newBuilder().uri(URI.create("/api/users")).build(),
        HttpResponse.BodyHandlers.ofString());
    var posts = client.sendAsync(
        HttpRequest.newBuilder().uri(URI.create("/api/posts")).build(),
        HttpResponse.BodyHandlers.ofString());
    var comments = client.sendAsync(
        HttpRequest.newBuilder().uri(URI.create("/api/comments")).build(),
        HttpResponse.BodyHandlers.ofString());

    return CompletableFuture.allOf(users, posts, comments);
}
```

## PHP (Guzzle)

```php
use GuzzleHttp\Client;
use GuzzleHttp\Promise\Utils;

$client = new Client();
$promises = [
    'users'    => $client->getAsync('/api/users'),
    'posts'    => $client->getAsync('/api/posts'),
    'comments' => $client->getAsync('/api/comments'),
];
$results = Utils::all($promises)->wait();
```

## Ruby (Async gem)

```ruby
require 'async'
require 'async/http/internet'

Async do
  internet = Async::HTTP::Internet.new

  users_task    = Async { internet.get("https://api.example.com/users") }
  posts_task    = Async { internet.get("https://api.example.com/posts") }
  comments_task = Async { internet.get("https://api.example.com/comments") }

  users    = users_task.wait
  posts    = posts_task.wait
  comments = comments_task.wait
end
```

## Shell

```bash
# Background processes + wait
curl -s /api/users > /tmp/users.json &
curl -s /api/posts > /tmp/posts.json &
curl -s /api/comments > /tmp/comments.json &
wait

# xargs for parallel processing of a URL list
cat urls.txt | xargs -P 5 -I {} curl -s {} -o /tmp/{}.json
```
