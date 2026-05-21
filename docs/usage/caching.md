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
