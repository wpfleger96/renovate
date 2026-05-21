As a tool for updating dependencies, Renovate needs to interact with many different [Datasources] to determine what update(s) are available for a given repository, requiring a large amount of outbound HTTP traffic.

To avoid unnecessary stress on upstream services, [like Maven Central](https://www.sonatype.com/blog/maven-central-and-the-tragedy-of-the-commons), as well as making Renovate runs more efficient, Renovate works to heavily cache external HTTP requests where possible.

As per [RFC7232](https://www.rfc-editor.org/info/rfc7232/), Renovate supports `Cache-Control` and `ETag` HTTP **??**

<!-- TODO: https://renovatebot.slack.com/archives/C0B0JN55L6S/p1779359737078739 -->

## Factors affecting caching

There are a number of operational factors that may affect whether Renovate caches data.

Firstly, depending on how Renovate is run, it may be possible for better caching to apply.

For instance, if Renovate is run with a single repository at a time:

```sh
# newlines for readability purposes only
env RENOVATE_TOKEN=...
  renovate --platform github
  renovatebot/renovate

# then run another repo
env RENOVATE_TOKEN=...
  renovate --platform github
  containerbase/base
```

In this case, the in-memory cache Renovate holds will be lost each time the Renovate process exits.

However, if you run multiple repositories in a single Renovate process:

```sh
# newlines for readability purposes only
env RENOVATE_TOKEN=...
  renovate --platform github
  renovatebot/renovate containerbase/base
```

In this case, the in-memory cache will be shared between all repositories being processed.

In both cases, we are executing these processes on the same host (whether it's a VM, container or your personal laptop) and so the on-disk caches (if configured) will be shared between Renovate runs.

Secondly, Renovate **??**. See [] and [] below for more details.

<details>

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

</details>

## Cache types

Renovate operates **??** types of cache:

### In-memory cache

#### What is in it?

#### Where is it stored?

In-memory.

As soon as the Renovate process exits, all data is lost.

#### Which options configure it?

It is not configurable.

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

By default, there is no Repository Cache as [`repositoryCache=disabled`](./self-hosted-configuration.md#repositorycache).

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

The Package Cache includes metadata about package releases, their changelogs, and HTTP responses from [Datasources].
It is intended to be used for **??**, and the **??**.

#### What is in it?

The Package Cache contains:

- **??**
  - The full list of namespaces that are included in the [`cacheTtlOverride`](./self-hosted-configuration.md#cachettloverride) docs
- GitHub GraphQL data for GitHub releases/tags
  - If the repo is public, any tags/releases will be stored in the cache
  - If the repo is private, any tags/releases will be cached in-memory in the Renovate process (and subsequent Renovate runs will need to re-fetch the data)

#### Where is it stored?

```markdown
Backends and their on-disk formats:

Redis (recommended for shared/distributed use):

- Keys follow the pattern {prefix}{namespace}-{key} (prefix set by redisPrefix)
- Each key's value is JSON: { "value": "<base64-compressed payload>", "expiry": "<ISO timestamp>" }
- Redis native TTL (EX) is also set for automatic cleanup by Redis itself
- Supports standard (redis://) and cluster (redis+cluster:// or rediss+cluster://) modes
- From your live Redis: 895 total keys, 601 seconds TTL remaining on a sample key

Example key from your Redis: datasource-npm:cache-provider-https://registry.npmjs.org/nock
Structure: { "value": "<24KB of base64-compressed data>", "expiry": "2026-05-28T10:44:44.095+01:00" }

File (default, uses cacache library):

- Stored at {cacheDir}/renovate/renovate-cache-v1
- Content format matches Redis: JSON with { "value": "<base64-compressed>", "expiry": "<ISO>" }
- An in-memory LRU map (up to 100k entries, ~5MB) tracks expiry times to speed up cleanup
- At shutdown, scans for and deletes expired entries

SQLite (experimental, via RENOVATE_X_SQLITE_PACKAGE_CACHE=true):

- Stored at {cacheDir}/renovate/renovate-cache-sqlite/db.sqlite
- Schema: package_cache(namespace TEXT, key TEXT, expiry INTEGER, data BLOB, PRIMARY KEY(namespace, key))
- data is Brotli-compressed JSON (quality=3, text mode)
- Uses WAL journal mode for concurrency
- Lock timeout configurable via RENOVATE_X_SQLITE_BUSY_TIMEOUT (default 5000ms)
- At shutdown, DELETE FROM package_cache WHERE expiry <= unixepoch() cleans expired rows
```

#### Which options configure it?

### ...

## Recommended performance improvements

**??**

For instance, users

For topology:

```mermaid

```

## FAQs

### What's the difference between the "soft" and "hard" cache?

**??**

### **??** HTTP cache

````markdown
There's no automatic/global selection — the provider is passed **explicitly per-request** via a `cacheProvider` option in the HTTP options. Each datasource or platform chooses its own provider at the call site.

Here's how the different providers are used in practice:

**`PackageHttpCacheProvider` → cross-run, package cache backing**

Constructed with a specific namespace and TTL options. Used by datasources that make external registry calls:

```ts
// npm/get.ts
const cacheProvider = new PackageHttpCacheProvider({
  namespace: 'datasource-npm:cache-provider',
  ...
});
const options: HttpOptions = { cacheProvider };
```
````

```ts
// maven/util.ts — two separate instances for different namespaces
const cacheProvider = new PackageHttpCacheProvider({ namespace: 'datasource-maven:cache-provider', ... });
const pomCacheProvider = new PackageHttpCacheProvider({ namespace: 'datasource-maven:pom-cache-provider', ... });
```

Each call site instantiates its own `PackageHttpCacheProvider` bound to its namespace, so TTL overrides via `cacheTtlOverride` are namespace-specific.

**`repoCacheProvider` → per-run, repository cache backing**

A singleton exported from the module, used by platform code (GitHub, GitLab, etc.) for API calls that are scoped to the current repo:

```ts
// platform/github/index.ts
{
  cacheProvider: repoCacheProvider;
} // used on many GitHub API calls
```

**`memCacheProvider` → session-only, in-memory**

Another singleton, used where responses shouldn't be persisted at all — Docker registry calls and some GitHub platform calls:

```ts
// docker/common.ts
cacheProvider: memCacheProvider;

// platform/github/index.ts — mixed usage, some calls use mem, others repo
{
  cacheProvider: memCacheProvider;
} // e.g. some PR-related calls
```

**`aggressiveRepoCacheProvider`** is a variant of `repoCacheProvider` with `aggressive: true`, which adds a "synced" flag — once a URL has been fetched and written to the repo cache in the current run, subsequent calls to that URL skip the server entirely and serve directly from cache (via `bypassServer()`). The regular `repoCacheProvider` only does conditional requests (ETag/304), not full bypasses.

So in summary: **the choice of provider is hardcoded by each datasource/platform**, not configured by the user. The user can influence behaviour indirectly via `cacheTtlOverride` (affects `PackageHttpCacheProvider` TTLs) and `cachePrivatePackages` (affects whether `PackageHttpCacheProvider` caches responses without a `public` Cache-Control header).

````

### Is the data encrypted in the cache(s)?

No. Data is currently **??** and then Base64-encoded.

It is recommended to **??**.

### What happens to HTTP calls that require authentication?

### What happens to private packages being retrieved?

Private package **??**

It's [`cachePrivatePackages`](./self-hosted-configuration.md#cacheprivatepackages)


### Does Renovate store a copy of the repo?

- persistRepoData

but no


---

## Other **??** relevant

- [`cacheTtlOverride`](./self-hosted-configuration.md#cachettloverride)
- [`cacheHardTtlMinutes`](./self-hosted-configuration.md#cachehardttlminutes)
- `prCacheSyncMaxPages`
- persistRepoData


---

```markdown
Here's a summary of the cache-related questions found in the database, grouped by theme:

---

## Repo Cache questions

**S3 / storage backends**
- [#42977](https://github.com/renovatebot/renovate/discussions/42977) — S3 cache doesn't work with Cloudflare R2 ("Region is missing" error with AWS SDK v3)
- [#33612](https://github.com/renovatebot/renovate/discussions/33612) — "Clarify what kinds of cache Renovate uses and what's recommended for on-prem CI" — explicitly asks for docs distinguishing repo cache (S3) vs package cache (Redis/SQLite) vs filesystem caches

**Permissions / file access**
- [#32411](https://github.com/renovatebot/renovate/discussions/32411) — EACCES writing repo cache JSON file on GitLab CI (resolved)
- [#32317](https://github.com/renovatebot/renovate/discussions/32317) — EACCES on `__renovate-private-cache` after v38→v39 upgrade on GitLab CI (open)

**Resilience**
- [#37515](https://github.com/renovatebot/renovate/discussions/37515) — Crash from corrupt repo cache (`rawItems is not iterable`, with log line `RepoCacheBase.load() - expecting data of type 'string' received 'object'`)
- [#41752](https://github.com/renovatebot/renovate/discussions/41752) — Renovate crashes (FATAL) when Redis is unreachable, instead of treating it as a cache miss

---

## Package Cache questions

**TTL / invalidation**
- [#42535](https://github.com/renovatebot/renovate/discussions/42535) — "How can I invalidate the config cache?" — user confused about HTTP cache TTL when debugging, doesn't know how to force a refresh
- [#41320](https://github.com/renovatebot/renovate/discussions/41320) — `cacheTtlOverride` not respected for `datasource-maven:cache-provider` because the cache is initialised before config is loaded
- [#36290](https://github.com/renovatebot/renovate/discussions/36290) — Docs say docker tags TTL default is 60 min, but code shows 30 min — requests a doc fix for `cacheTtlOverride`

**Correctness / poisoning**
- [#42792](https://github.com/renovatebot/renovate/discussions/42792) — npm datasource caches private packages from registries that omit `Cache-Control` (e.g. GitHub Packages), causing stale-version issues
- [#40718](https://github.com/renovatebot/renovate/discussions/40718) — Cache poisoning: a `null` digest result caused by bad `hostRules` gets cached and then served to unrelated repos sharing Redis

**General "what caches does Renovate have?"**
- [#33612](https://github.com/renovatebot/renovate/discussions/33612) *(also listed above)* — asks what's in `$RENOVATE_CACHE_DIR/others/npm`, why it grows to 20GB+, and whether persisting it in CI is still worthwhile when S3+Redis are configured

---

## Key themes worth covering in docs

1. **What are the two distinct caches** — repo cache (per-repo state: extracted deps, branch metadata) vs package cache (datasource HTTP responses) and their storage options
2. **How to configure S3 for repo cache** including non-AWS S3-compatible endpoints and their quirks (region config)
3. **How to configure Redis/SQLite for package cache** and what happens if the backend is unreachable
4. **TTL defaults and how to override them** (`cacheTtlOverride`, `cacheHardTtlMinutes`) — the defaults are underdocumented and apparently sometimes wrong in the docs
5. **Filesystem cache (`$RENOVATE_CACHE_DIR`)** — what lives there, whether it's safe to clear/not persist in CI alongside Redis+S3, and how it relates to the npm tool cache
6. **Cache invalidation** — there's no manual invalidation button; the only options are TTL expiry, clearing the storage, or working around with `dryRun`
````
