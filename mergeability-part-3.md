# Mergability Part 3 - The Problem with Test Merges


### The Problems with Test Merges
- Performing constant attempts to merge and then discarding objects
- Unreachable vs Reachable Objects
- Spamming requests, kick off these jobs
- Unreachable object growth for monorepos spamming checks and fast moving head
- Git nw-gc --pristine

### What To Do
- reduce API requests
- Graphql remove merge
- Fewer open PRs
- Feature requests

### Addendum
- CPRMC no more, Same problems exist

### Takeaways
- Deeper understanding of how git merges work
- Deeper understanding of concept of a test merge, and how it works in context of Github.com and GHES.
- Problems created by test merge and potential mitigations