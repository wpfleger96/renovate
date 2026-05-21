As a tool for updating dependencies, Renovate needs to interact with a number of different [Datasources] to be able to determine what update(s) are available for a given repository.

To avoid unnecessary stress on upstream services, [like Maven Central](https://www.sonatype.com/blog/maven-central-and-the-tragedy-of-the-commons), as well as making Renovate runs more efficient, Renovate works to heavily cache external HTTP requests where possible.

As per [RFC7232](https://www.rfc-editor.org/info/rfc7232/), Renovate supports `Cache-Control` and `ETag` HTTP **??**

<!-- TODO: https://renovatebot.slack.com/archives/C0B0JN55L6S/p1779359737078739 -->

In addition, there are internal caches set on **??**

In addition,

Renovate's operation model leads to a lot of external traffic being sent.

With even a "medium sized" deployment, **??**.

When operating, Renovate **??**

Where possible, Renovate works in a **??**.

However, **??**

<!-- prettier-ignore -->
!!! tip
    It is strongly recommended to set up caching if you're running **??**.
    <br>
    It is ideal if you have a proxy/etc to handle package cahhing to avoid **??**

Renovate operates **??** types of cache:

## Cache types

### Repository Cache

The Repository Cache includes metadata about repositories to reduce the work that Renovate will need to perform on future runs.

<!-- prettier-ignore -->
!!! note
    This only contains **metadata** about the repository, not the repository itself.
    <br>
    This data includes very similar data to what is seen in the debug logs.

#### What is in it?

This cache includes:

- `branches`: information about the branches Renovate is currently managing, and:
  - whether they're associated with a PR
  - what update(s) are in the given branch
  - whether the branch is conflicted/behind the base branch or if it's been modified by someone other than Renovate
  -
- `scan`: **??**
- `platform.{github,gitlab,gitea,bitbucket,...}.pullRequestsCache`
- `httpCache`/`httpCacheHead`:
- `semanticCommits`: a **??** for [`semanticCommits`](./configuration-options.md#semanticcommits) detection
- `onboardingBranchCache`, `reconfigureBranchCache`: cache for when using **??**
- `prComments`: any PR comments that Renovate has made on PRs

This generally allows Renovate to not need to perform potentially expensive work (like extracting all package files in a repository) if the repository has not changed.

#### Where is it stored?

The presence of the Repository Cache is defined by the global self-hosted configuration [`repositoryCache`](./self-hosted-configuration.md#repositorycache), which is disabled by default.

If enabled, this cache data is stored by default in the local filesystem, under the [`cacheDir`](./self-hosted-configuration.md#cachedir) location.

It can be configured to be stored in an S3-compatible location using i.e. [`repositoryCacheType=s3://my-bucket/some-path/repo-cache`](./self-hosted-configuration.md#repositorycachetype).

Configuration:
Backends:

- Local filesystem (default)
- S3-compatible interface

#### Which options configure it?

[`repositoryCache`](./self-hosted-configuration.md#repositorycachetype)
[`repositoryCacheType`](./self-hosted-configuration.md#repositorycachetype)
[`repositoryCacheForceLocal`](./self-hosted-configuration.md#repositorycachetype)

### Package Cache

### ...

## Recommended performance improvements

**??**

For instance, users

For topology:

```mermaid

```

## FAQs

## Other **??** relevant

- [`cacheTtlOverride`](./self-hosted-configuration.md#cachettloverride)
- [`cacheHardTtlMinutes`](./self-hosted-configuration.md#cachehardttlminutes)
- `prCacheSyncMaxPages`
- persistRepoData

### Is the data encrypted in the cache(s)?

No. Data is currently **??** and then Base64-encoded.

It is recommended to **??**.

### Does Renovate store a copy of the repo?

- persistRepoData

but no

---

```markdown
Renovate has a rich caching system with several distinct layers:

---

## Renovate Cache Types

### 1. Repository Cache (`lib/util/cache/repository/`)

Persists per-repository metadata across Renovate runs. Stores branch states, PR info, HTTP responses, and extraction fingerprints.

**Backends**: Local filesystem (default), S3, or disabled
**Config**: `repositoryCache` (`enabled`/`disabled`/`reset`), `repositoryCacheType` (e.g. `s3://...`)

### 2. Package Cache (`lib/util/cache/package/`)

Caches datasource/HTTP responses with configurable TTLs. This is the main cache for version lookups across 100+ package manager namespaces.

**Backends**: File (cacache), Redis, or SQLite (experimental via `RENOVATE_X_SQLITE_PACKAGE_CACHE`)
**Config**: `redisUrl`, `cacheTtlOverride`, `cacheHardTtlMinutes`, `cacheDir`

### 3. Memory Cache (`lib/util/cache/memory/`)

In-memory key-value store scoped to a single Renovate run. No persistence, used for deduplication within a session.

### 4. HTTP Cache (`lib/util/http/cache/`)

Caches HTTP responses using ETags/Last-Modified headers. Has three provider strategies:

- **Memory**: volatile, session-only
- **Repository**: persisted in the repository cache
- **Package**: persisted in the package cache with TTL

**Config**: `httpCacheTtlDays`, `cachePrivatePackages`

### 5. Git Branch Caches (within Repository Cache)

Four specialized caches stored inside the repository cache, each keyed by branch + commit SHA:

- Behind-base-branch status
- Merge conflict status
- External modification status
- Last update date

### 6. Platform PR Caches (within Repository Cache)

Per-platform PR list caches (GitLab, Gitea, Bitbucket, Bitbucket Server, Forgejo) stored under `platform.{provider}.pullRequestsCache` in the repository cache.

### 7. GitHub GraphQL Cache (`lib/util/github/graphql/cache-strategies/`)

Specialized cache for GitHub releases/tags with a 30-day TTL and reconciliation logic. Can use either memory or package cache as its backend.

---

## Summary

| Type           | Persistence  | Backends              | Purpose                      |
| -------------- | ------------ | --------------------- | ---------------------------- |
| Repository     | Per-run      | Local FS, S3          | Branch/PR metadata per repo  |
| Package        | Cross-run    | File, Redis, SQLite   | Datasource version lookups   |
| Memory         | Session only | RAM                   | Deduplication within a run   |
| HTTP           | Varies       | Memory, Repo, Package | HTTP response caching        |
| Git            | Per-run      | (in repo cache)       | Branch state tracking        |
| Platform PR    | Per-run      | (in repo cache)       | PR list caching per platform |
| GitHub GraphQL | Cross-run    | Memory/Package        | GitHub release/tag data      |

The two main ones you were thinking of are **Repository Cache** (repo metadata, branches, PRs) and **Package Cache** (datasource/version data) — they're also the ones with the most configuration surface.
```
