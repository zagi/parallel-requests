# parallel-requests

A Claude Code skill that detects sequential independent HTTP/API calls and refactors them to parallel execution patterns.

## Structure

- `skill/SKILL.md` — Core skill definition with detection rules, patterns, and anti-patterns
- `skill/README.md` — Public documentation with installation instructions
- `skill/references/patterns.md` — Copy-pasteable code patterns for 9 languages
- `skill/LICENSE` — MIT license

## Primary Languages

JS/TS (`Promise.all`, `Promise.allSettled`, `p-limit`) and Python (`asyncio.gather`, `TaskGroup`, `Semaphore`).

Quick reference table covers Go, Rust, C#, Java, PHP, Ruby, Shell.

## Testing

1. `ln -s $(pwd)/skill ~/.agents/skills/parallel-requests`
2. Open a new Claude Code session
3. Paste sequential fetch code — Claude should suggest `Promise.all`
4. Paste code where request B depends on A — Claude should warn and keep sequential
5. Paste 50 sequential requests — Claude should suggest concurrency-limited pattern
