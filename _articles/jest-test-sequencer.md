---
title:  "Jest - Test Sequencer [TypeScript]"
layout: default
last_modified_date: 2021-08-09T20:18:00+0300
nav_order: 12

status: PUBLISHED
language: TypeScript
short-title: Test Sequencer
project:
  name: Jest
  key: jest
  home-page: https://github.com/facebook/jest
tags: ['test-framework']
---

{% include article-meta.html article=page %}

## Context

*Jest is a JavaScript testing framework designed to ensure correctness of any JavaScript codebase. It allows you to write tests with an approachable, familiar and feature-rich API that gives you results quickly.* - [The official website](https://jestjs.io/).

Jest runs tests in parallel. The number of workers defaults to the number of the cores available on your machine minus one for the main thread - see [`--maxWorkers`](https://jestjs.io/docs/cli#--maxworkersnumstring).

## Problem

Jest needs to decide which tests should run first. Sorting tests [is very important](https://github.com/facebook/jest/blob/master/packages/jest-test-sequencer/src/index.ts#L69-L70) because it has a great impact on the user-perceived responsiveness and speed of the test run.

## Overview

Tests are sorted based on:
1. Has it failed during the last run?
  * Since it's important to provide the most expected feedback as quickly as possible.
1. How long it took to run?
  * Because running long tests first is an effort to minimize worker idle time at the end of a long test run.

And if that information is not available, they are sorted based on file size since big test files usually take longer to complete.

## Implementation details

The [sorting method](https://github.com/facebook/jest/blob/master/packages/jest-test-sequencer/src/index.ts#L86-L113) implementing the logic described above. The code is straightforward enough to not require further explanation.

Note that newly added tests will run after failed tests but before the rest of other tests.

```typescript
sort(tests: Array<Test>): Array<Test> {
  const stats: {[path: string]: number} = {};
  const fileSize = ({path, context: {hasteFS}}: Test) =>
    stats[path] || (stats[path] = hasteFS.getSize(path) || 0);
  const hasFailed = (cache: Cache, test: Test) =>
    cache[test.path] && cache[test.path][0] === FAIL;
  const time = (cache: Cache, test: Test) =>
    cache[test.path] && cache[test.path][1];

  tests.forEach(test => (test.duration = time(this._getCache(test), test)));
  return tests.sort((testA, testB) => {
    const cacheA = this._getCache(testA);
    const cacheB = this._getCache(testB);
    const failedA = hasFailed(cacheA, testA);
    const failedB = hasFailed(cacheB, testB);
    const hasTimeA = testA.duration != null;
    if (failedA !== failedB) {
      return failedA ? -1 : 1;
    } else if (hasTimeA != (testB.duration != null)) {
      // If only one of two tests has timing information, run it last
      return hasTimeA ? 1 : -1;
    } else if (testA.duration != null && testB.duration != null) {
      return testA.duration < testB.duration ? 1 : -1;
    } else {
      return fileSize(testA) < fileSize(testB) ? 1 : -1;
    }
  });
}
```

It's [invoked](https://github.com/facebook/jest/blob/5f4dd187d89070d07617444186684c20d9213031/packages/jest-core/src/runJest.ts#L187) from [runTest.ts](https://github.com/facebook/jest/blob/5f4dd187d89070d07617444186684c20d9213031/packages/jest-core/src/runJest.ts).

[Reading](https://github.com/facebook/jest/blob/fdc74af37235354e077edeeee8aa2d1a4a863032/packages/jest-test-sequencer/src/index.ts#L45-L66) and [writing](https://github.com/facebook/jest/blob/fdc74af37235354e077edeeee8aa2d1a4a863032/packages/jest-test-sequencer/src/index.ts#L123-L140) the last run results. Note that different tests can run in different contexts.

```typescript
_getCache(test: Test): Cache {
  const {context} = test;
  if (!this._cache.has(context) && context.config.cache) {
    const cachePath = this._getCachePath(context);
    if (fs.existsSync(cachePath)) {
      try {
        this._cache.set(
          context,
          JSON.parse(fs.readFileSync(cachePath, 'utf8')),
        );
      } catch {}
    }
  }

  let cache = this._cache.get(context);
  if (!cache) {
    cache = {};
    this._cache.set(context, cache);
  }

  return cache;
}

// ...

cacheResults(tests: Array<Test>, results: AggregatedResult): void {
  const map = Object.create(null);
  tests.forEach(test => (map[test.path] = test));
  results.testResults.forEach(testResult => {
    if (testResult && map[testResult.testFilePath] && !testResult.skipped) {
      const cache = this._getCache(map[testResult.testFilePath]);
      const perf = testResult.perfStats;
      cache[testResult.testFilePath] = [
        testResult.numFailingTests ? FAIL : SUCCESS,
        perf.runtime || 0,
      ];
    }
  });

  this._cache.forEach((cache, context) =>
    fs.writeFileSync(this._getCachePath(context), JSON.stringify(cache)),
  );
}
```

There's also a [method](https://github.com/facebook/jest/blob/master/packages/jest-test-sequencer/src/index.ts#L115-L121), supporting the `--onlyFailures` option, to get tests that failed during the last run. It must have been placed in `TestSequencer` because it has access to the data about the last run.

## Testing

The ordering logic is covered by unit tests. E.g. testing that it "sorts based on failures, timing information and file size":

```typescript
test('sorts based on failures, timing information and file size', () => {
  fs.readFileSync.mockImplementationOnce(() =>
    JSON.stringify({
      '/test-a.js': [SUCCESS, 5],
      '/test-ab.js': [FAIL, 1],
      '/test-c.js': [FAIL],
      '/test-d.js': [SUCCESS, 2],
      '/test-efg.js': [FAIL],
    }),
  );
  expect(
    sequencer.sort(
      toTests([
        '/test-a.js',
        '/test-ab.js',
        '/test-c.js',
        '/test-d.js',
        '/test-efg.js',
      ]),
    ),
  ).toEqual([
    {context, duration: undefined, path: '/test-efg.js'},
    {context, duration: undefined, path: '/test-c.js'},
    {context, duration: 1, path: '/test-ab.js'},
    {context, duration: 5, path: '/test-a.js'},
    {context, duration: 2, path: '/test-d.js'},
  ]);
});
```

## Related

* Ordering tests [in jUnit4](https://github.com/junit-team/junit4/blob/9ad61c6bf757be8d8968fd5977ab3ae15b0c5aba/src/main/java/org/junit/runner/manipulation/Sorter.java). As far as we can see, it doesn't do similar tricks.
* Ordering tests [in RSpec](https://github.com/rspec/rspec-core/blob/dc898adc3f98d841a43e22cdf62ae2250266c7b6/lib/rspec/core/ordering.rb). It supports *random* and *recently modified* modes.

## References

* [GitHub Repo](https://github.com/facebook/jest)
* [Official Website](https://jestjs.io/)

## Copyright notice

Jest is licensed under the [MIT License](https://github.com/facebook/jest/blob/master/LICENSE).

Copyright (c) Facebook, Inc. and its affiliates. All Rights Reserved.
