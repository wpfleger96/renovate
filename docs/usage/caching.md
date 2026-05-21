As a tool for updating dependencies, Renovate needs to interact with many different [Datasources](./modules/datasource/index.md) to determine what update(s) are available for a given repository, requiring a large amount of outbound HTTP traffic.

To avoid unnecessary stress on upstream services, [like Maven Central](https://www.sonatype.com/blog/maven-central-and-the-tragedy-of-the-commons), as well as making Renovate runs more efficient, Renovate works to heavily cache external HTTP requests where possible.

When Renovate encounters `Cache-Control` headers, it will abide by them, as well as perform conditional HTTP requests when `ETag` HTTP headers are received.

## Factors affecting caching

Renovate will conditionally cache data based on a few factors.

Firstly, depending on how you run Renovate, it may be possible to improve caching.

For instance, if Renovate runs against a single repository at a time:

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

In this case, the In-Memory Cache Renovate holds will be lost each time the Renovate process exits.

However, if you run multiple repositories in a single Renovate process:

```sh
# newlines for readability purposes only
env RENOVATE_TOKEN=...
  renovate --platform github
  renovatebot/renovate containerbase/base
```

In this case, the In-Memory Cache will be shared between all repositories being processed.

In both cases, the Renovate runs execute on the same host (whether it's a VM, container or your personal laptop) and so the on-disk caches (if configured) will be shared between Renovate runs.

Secondly, Renovate will conditionally cache based on whether it detects it is interacting with a private repository and/or a private package. See [What happens to HTTP calls that require authentication?](#what-happens-to-http-calls-that-require-authentication) and [What happens to private packages being retrieved?](#what-happens-to-private-packages-being-retrieved) below for more details.

Finally, if you're running Renovate across many hosts (for instance across a Kubernetes cluster or on your automated build platform like GitLab CI), [we recommend](#recommended-performance-improvements) using a persistent Package Cache, and ideally a persistent Repository Cache, too.

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

Renovate uses 3 types of cache:

### In-Memory Cache

The In-Memory Cache includes any short-lived data which is worth caching within a given Renovate run (for a single repo or against multiple), but is not worth persisting for more long-term access.

#### What is in it?

Renovate stores **??**, for instance when retrieving HTTP-/npm-based config presets, getting the public key for a Hex registry or for listing status checks on a GitHub branch.

#### Where is it stored?

In-memory.

As soon as the Renovate process exits, all data is lost.

#### Which options configure it?

It is not configurable.

### Repository Cache

The Repository Cache includes metadata about repositories to reduce the work that Renovate will need to perform on future runs.

The Repository Cache is primarily aimed at reducing the work that Renovate needs to be perform each time it executes against a repository, and limiting the API requests it needs to send to the configured Platform.

<!-- prettier-ignore -->
!!! note
    This only contains **metadata** about the repository, not the repository itself.
    <br>
    This includes similar data to what Renovate logs at `DEBUG` log level.

#### What is in it?

This cache includes (among other information):

- `configFileName`: this repository's filename i.e. `renovate.json5`
- `branches`: information about the branches Renovate is currently managing, and:
  - whether they're associated with a PR
  - what update(s) are in the given branch
  - whether the branch is conflicted/behind the base branch or if it's been modified by someone other than Renovate
- `onboardingBranchCache`, `reconfigureBranchCache`: cache for the state of the onboarding/reconfigure branches
- `platform`: specific information for the given Platform, such as a cache of all PRs
- `httpCache`/`httpCacheHead`: cached repository-specific HTTP responses
- `semanticCommits`: the current calculation for the repo's [`semanticCommits`](./configuration-options.md#semanticcommits)
- `prComments`: any PR comments that Renovate has made on PRs

This generally allows Renovate to not need to perform potentially expensive work (like extracting all package files in a repository) if the repository has not changed.

#### Where is it stored?

By default, there is no Repository Cache as [`repositoryCache=disabled`](./self-hosted-configuration.md#repositorycache) is the default.

If enabled, this cache data is stored by default in the local filesystem, under the [`cacheDir`](./self-hosted-configuration.md#cachedir) location.

It can be configured to be stored in an S3-compatible location using i.e. [`repositoryCacheType=s3://my-bucket/some-path/repo-cache`](./self-hosted-configuration.md#repositorycachetype).

#### Which options configure it?

- [`repositoryCache`](./self-hosted-configuration.md#repositorycachetype): whether to enable it
- [`repositoryCacheType`](./self-hosted-configuration.md#repositorycachetype): where the Repository Cache should be stored
- [`repositoryCacheForceLocal`](./self-hosted-configuration.md#repositorycachetype): whether to also persist it to the local filesystem if using `repositoryCacheType=s3://...`

### Package Cache

The Package Cache includes metadata about package releases, their changelogs, and HTTP responses from [Datasources](./modules/datasource/index.md).

The Package Cache is primarily aimed at improving quality-of-life for upstream providers, such as package registries.
This is the most important lever that a self-hosted administrator has to **??**.

<!-- prettier-ignore -->
!!! tip
    Tuning this **??** is a very **??**, and that helps keep the ecosystem **??**.

#### What is in it?

The Package Cache contains:

<!-- markdownlint-disable MD007 -->
<!-- prettier-ignore -->
- **??**
    - The full list of namespaces that are included in the [`cacheTtlOverride`](./self-hosted-configuration.md#cachettloverride) docs
- GitHub GraphQL data for GitHub releases/tags
    - If the repo is public, any tags/releases will be stored in the cache
    - If the repo is private, any tags/releases will be cached in-memory in the Renovate process (and subsequent Renovate runs will need to re-fetch the data)

#### Where is it stored?

By default, the Package Cache is stored in the local filesystem, under the [`cacheDir`](./self-hosted-configuration.md#cachedir) location.

When using the local filesystem, the [cacache](https://www.npmjs.com/package/cacache) library is used.

When the [`redisUrl`](./self-hosted-configuration.md#redisurl) self-hosted configuration option is set, the Package Cache will be stored in Redis.

<!-- prettier-ignore -->
!!! warning
    Experimental features might be changed or even removed at any time.

Renovate has experimental support for using SQLite as the Package Cache backend, which can be configured using [`RENOVATE_X_SQLITE_PACKAGE_CACHE`](./self-hosted-experimental.md#renovate_x_sqlite_package_cache).

#### Which options configure it?

- [`redisUrl`](./self-hosted-configuration.md#redisurl)
- [`redisPrefix`](./self-hosted-configuration.md#redisprefix)
- [`presetCachePersistence`](./self-hosted-configuration.md#presetcachepersistence)
- [`cacheTtlOverride`](./self-hosted-configuration.md#cachettloverride)
- [`cacheHardTtlMinutes`](./self-hosted-configuration.md#cachehardttlminutes)
- [`prCacheSyncMaxPages`](./self-hosted-configuration.md#prcachesyncmaxpages)
- [`RENOVATE_X_SQLITE_PACKAGE_CACHE`](./self-hosted-experimental.md#renovate_x_sqlite_package_cache).
- [`RENOVATE_X_SQLITE_BUSY_TIMEOUT`](./self-hosted-experimental.md#renovate_x_sqlite_busy_timeout)

## Other related configuration options

- [`persistRepoData`](./self-hosted-configuration.md#persistrepodata)

## Recommended performance improvements

**??**

For instance, users

For topology:

```mermaid

```

## FAQs

### What's the difference between the "soft" and "hard" cache?

**??**

### What's happens to package manager caches?

In the case that **??**, for instance, `go mod tidy`, then **??**.

Where possible, Renovate will centralise these cache locations under [`cacheDir`](./self-hosted-configuration.md#cachedir), i.e. in `$cacheDir/others`.

This directry can **??** over time https://github.com/renovatebot/renovate/discussions/33612

### How do I use the **??** with S3-compatbile?

### Where are HTTP responses cached?

HTTP responses are cached between **both** the Repository Cache and the Package Cache.

Certain HTTP requests - such as the repository's PRs and the Dependency Dashboard Issue - make more sense to be tied to the Repository Cache, and others - such as HTTP calls to Datasources - make sense to be in the Package Cache.

Additionally, some HTTP requests are only stored in the In-Memory Cache, and not persisted between Renovate runs.

The cache used for HTTP responses is not user-configurable.

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

### Does Renovate cache HTTP calls that don't return a `Cache-Control` header?

**??**

No, Renovate will not.
Renovate treats the absence of a **??**

### What happens to HTTP calls that require authentication?

### What happens to private packages being retrieved?

Private package **??**

It's [`cachePrivatePackages`](./self-hosted-configuration.md#cacheprivatepackages)

### Does Renovate store a copy of the repo?

- persistRepoData

but no

### How do I invalidate the cache?

It is not currently possible for a _user_ to invalidate the cache.

A self-hosted administrator can **??**:

- Repository Cache:
  - **??** `reset`
  - Delete the file in S3
- Package Cache:
  - foo

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

- [#33612](https://github.com/renovatebot/renovate/discussions/33612) _(also listed above)_ — asks what's in `$RENOVATE_CACHE_DIR/others/npm`, why it grows to 20GB+, and whether persisting it in CI is still worthwhile when S3+Redis are configured

---

## Key themes worth covering in docs

1. **What are the two distinct caches** — repo cache (per-repo state: extracted deps, branch metadata) vs package cache (datasource HTTP responses) and their storage options
2. **How to configure S3 for repo cache** including non-AWS S3-compatible endpoints and their quirks (region config)
3. **How to configure Redis/SQLite for package cache** and what happens if the backend is unreachable
4. **TTL defaults and how to override them** (`cacheTtlOverride`, `cacheHardTtlMinutes`) — the defaults are underdocumented and apparently sometimes wrong in the docs
5. **Filesystem cache (`$RENOVATE_CACHE_DIR`)** — what lives there, whether it's safe to clear/not persist in CI alongside Redis+S3, and how it relates to the npm tool cache
6. **Cache invalidation** — there's no manual invalidation button; the only options are TTL expiry, clearing the storage, or working around with `dryRun`
```
