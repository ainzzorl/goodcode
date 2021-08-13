---
title:  "Buck - Artifact Cache Decorators [Java]"
layout: default
last_modified_date: 2021-08-13T12:18:00+0300
nav_order: 15

status: PUBLISHED
language: Java
short-title: Artifact Cache Decorators
project:
    name: Buck
    key: buck
    home-page: https://github.com/facebook/buck
tags: ['decorator']
---

{% include article-meta.html article=page %}

## Context

[Buck](https://buck.build/) is a multi-language build system developed and used by Facebook.

Buck avoids rebuilding the same module twice by caching build artifacts and metadata. It employs various caching strategies, such as caching on the local disk, in SQLite database or in a shared cache over HTTP.

Caches obey the [`ArtifactCache`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/ArtifactCache.java#) interface, which defines methods such as `fetchAsync` or `store`.

## Problem

Various embellishments, such as retries or even logging, need to be added to some, but not all, cache client instances.

## Overview

The solution uses the classical [Decorator](https://en.wikipedia.org/wiki/Decorator_pattern) design pattern.

> *The decorator pattern is a design pattern that allows behavior to be added to an individual object, dynamically, without affecting the behavior of other objects from the same class.* - [Wikipedia](https://en.wikipedia.org/wiki/Decorator_pattern)

Buck implements 3 decorators for `ArtifactCache`:
1. [`LoggingArtifactCacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/LoggingArtifactCacheDecorator.java) - logs caching events to an event bus;
1. [`RetryingCacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/RetryingCacheDecorator.java) - retries failed requests;
1. [`TwoLevelArtifactCacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/TwoLevelArtifactCacheDecorator.java) - adds a two-level caching scheme, the details of which are not in the scope of this article.

The decorators implement the same `ArtifactCache` interface as the actual caches. Their constructors accept `ArtifactCache delegate` and some decorator-specific parameters. Most of the work is delegated to the `delegate`, while delegators provide additional functionality. Users of the `ArtifactCache` interface don't distinguish between "actual" caches and decorators.

The decorators also implement the [`CacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/CacheDecorator.java) interface with the only method `ArtifactCache getDelegate()`. As far as we can tell, this method is only used in testing.

Concrete instances of `ArtifactCache` with appropriate decorators are instanciated by [`ArtifactCaches`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/ArtifactCaches.java) factory class.

## Implementation details

Let's look at [`LoggingArtifactCacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/LoggingArtifactCacheDecorator.java). All it does is call `eventBus.post()` before and after fetching or storing artifacts in the underlying cache.

```java
/**
 * Decorator for wrapping a {@link ArtifactCache} to log a {@link ArtifactCacheEvent} for the start
 * and finish of each event. The underlying cache must only provide synchronous operations.
 */
public class LoggingArtifactCacheDecorator implements ArtifactCache, CacheDecorator {
  private final BuckEventBus eventBus;
  private final ArtifactCache delegate;
  private final ArtifactCacheEventFactory eventFactory;

  public LoggingArtifactCacheDecorator(
      BuckEventBus eventBus, ArtifactCache delegate, ArtifactCacheEventFactory eventFactory) {
    this.eventBus = eventBus;
    this.delegate = delegate;
    this.eventFactory = eventFactory;
  }

  @Override
  public ListenableFuture<CacheResult> fetchAsync(
      @Nullable BuildTarget target, RuleKey ruleKey, LazyPath output) {
    ArtifactCacheEvent.Started started =
        eventFactory.newFetchStartedEvent(ImmutableSet.of(ruleKey));
    eventBus.post(started);
    CacheResult fetchResult = Futures.getUnchecked(delegate.fetchAsync(target, ruleKey, output));
    eventBus.post(eventFactory.newFetchFinishedEvent(started, fetchResult));
    return Futures.immediateFuture(fetchResult);
  }

  @Override
  public void skipPendingAndFutureAsyncFetches() {
    delegate.skipPendingAndFutureAsyncFetches();
  }

  @Override
  public ListenableFuture<Unit> store(ArtifactInfo info, BorrowablePath output) {
    ArtifactCacheEvent.Started started =
        eventFactory.newStoreStartedEvent(info.getRuleKeys(), info.getMetadata());
    eventBus.post(started);
    ListenableFuture<Unit> storeFuture = delegate.store(info, output);
    eventBus.post(eventFactory.newStoreFinishedEvent(started));
    return storeFuture;
  }

  @Override
  public ListenableFuture<ImmutableMap<RuleKey, CacheResult>> multiContainsAsync(
      ImmutableSet<RuleKey> ruleKeys) {
    ArtifactCacheEvent.Started started = eventFactory.newContainsStartedEvent(ruleKeys);
    eventBus.post(started);

    return Futures.transform(
        delegate.multiContainsAsync(ruleKeys),
        results -> {
          eventBus.post(eventFactory.newContainsFinishedEvent(started, results));
          return results;
        },
        MoreExecutors.directExecutor());
  }

  @Override
  public ListenableFuture<CacheDeleteResult> deleteAsync(List<RuleKey> ruleKeys) {
    return delegate.deleteAsync(ruleKeys);
  }

  @Override
  public CacheReadMode getCacheReadMode() {
    return delegate.getCacheReadMode();
  }

  @Override
  public void close() {
    delegate.close();
  }

  @Override
  public ArtifactCache getDelegate() {
    return delegate;
  }
}
```

Similarly, [`RetryingCacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/RetryingCacheDecorator.java) passes everything down to the `delegate`, retrying failed fetches:

```java
public class RetryingCacheDecorator implements ArtifactCache, CacheDecorator {

  private static final Logger LOG = Logger.get(RetryingCacheDecorator.class);

  private final ArtifactCache delegate;
  private final int maxFetchRetries;
  private final BuckEventBus buckEventBus;
  private final ArtifactCacheMode cacheMode;

  public RetryingCacheDecorator(
      ArtifactCacheMode cacheMode,
      ArtifactCache delegate,
      int maxFetchRetries,
      BuckEventBus buckEventBus) {
    Preconditions.checkArgument(maxFetchRetries > 0);

    this.cacheMode = cacheMode;
    this.delegate = delegate;
    this.maxFetchRetries = maxFetchRetries;
    this.buckEventBus = buckEventBus;
  }

  @Override
  public ListenableFuture<CacheResult> fetchAsync(
      @Nullable BuildTarget target, RuleKey ruleKey, LazyPath output) {
    List<String> allCacheErrors = new ArrayList<>();
    ListenableFuture<CacheResult> resultFuture = delegate.fetchAsync(target, ruleKey, output);
    for (int retryCount = 1; retryCount < maxFetchRetries; retryCount++) {
      int retryCountForLambda = retryCount;
      resultFuture =
          Futures.transformAsync(
              resultFuture,
              result -> {
                if (result.getType() != CacheResultType.ERROR) {
                  return Futures.immediateFuture(result);
                }
                result.cacheError().ifPresent(allCacheErrors::add);
                LOG.info(
                    "Failed to fetch %s after %d/%d attempts, exception: %s",
                    ruleKey, retryCountForLambda + 1, maxFetchRetries, result.cacheError());
                return delegate.fetchAsync(target, ruleKey, output);
              });
    }
    return Futures.transform(
        resultFuture,
        result -> {
          if (result.getType() != CacheResultType.ERROR) {
            return result;
          }
          String msg = String.join("\n", allCacheErrors);
          if (!msg.contains(NoHealthyServersException.class.getName())) {
            buckEventBus.post(
                ConsoleEvent.warning(
                    "Failed to fetch %s over %s after %d attempts.",
                    ruleKey, cacheMode.name(), maxFetchRetries));
          }
          return result.withCacheError(Optional.of(msg));
        },
        MoreExecutors.directExecutor());
  }

  // ...
  // Just delegates
}
```

[`TwoLevelArtifactCacheDecorator`](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/TwoLevelArtifactCacheDecorator.java) is a lot more involved, but its details are not in the scope of this article.

## Testing

`RetryingCacheDecorator` and `LoggingArtifactCacheDecorator` aren't tested directly. `TwoLevelArtifactCacheDecorator` has [its own test](https://github.com/facebook/buck/blob/0acbcbbed7274ba654b64d97414b28183649e51a/src/com/facebook/buck/artifact_cache/TwoLevelArtifactCacheDecorator.java).

## Observations

* If most decorator methods simply delegate to the underlying cache without doing anything else, perhaps there could be a `BaseCacheDecorator` that would simply delegate all calls, and concrete decorators could inherit from `BaseCacheDecorator` and only override those methods where they need to do something.

## References

* [GitHub Repo](https://github.com/facebook/buck)
* [Buck Website](https://buck.build/)
* [Buck on Wikipedia](https://en.wikipedia.org/wiki/Buck_(software))
* [Decorator Pattern](https://en.wikipedia.org/wiki/Decorator_pattern)
* [Refactoring to Patterns: Move Embellishment to Decorator](https://www.informit.com/articles/article.aspx?p=1398607&seqNum=3)

## Copyright notice

Stockfish is licensed under the [Apache License 2.0](https://github.com/facebook/buck/blob/master/LICENSE).

Copyright (c) Facebook, Inc. and its affiliates.