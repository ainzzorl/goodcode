---
title:  "Jest - Displaying Diffs [TypeScript]"
layout: default
last_modified_date: 2021-08-31T21:14:00+0300
nav_order: 17

status: PUBLISHED
language: TypeScript
short-title: Displaying Diffs
project:
    name: Jest
    key: jest
    home-page: https://github.com/facebook/jest
tags: ['diff', 'test-framework']
---

{% include article-meta.html article=page %}

## Context

*Jest is a JavaScript testing framework designed to ensure correctness of any JavaScript codebase. It allows you to write tests with an approachable, familiar and feature-rich API that gives you results quickly.* - [The official website](https://jestjs.io/).

## Problem

When a test assertion fails, a test framework must display the diff between the actual and the expected values clearly, so users can review the error and act accordingly. This is particularly important for long multi-line strings, arrays or objects with many, possibly nested, fields. The user must clearly see what was added compared to the expectation, what was removed and what remained the same.

Example from [README.md](https://github.com/facebook/jest/blob/master/packages/jest-diff/README.md):

```js
const a = 'common\nchanged from';
const b = 'common\nchanged to';

const difference = diffStringsUnified(a, b);
```

Output:
```diff
- Expected
+ Received

  common
- changed from
+ changed to
```

## Overview

Jest provides the [`jest-diff`](https://github.com/facebook/jest/tree/master/packages/jest-diff) package with the following API (the description is copied from [README](https://github.com/facebook/jest/blob/master/packages/jest-diff/README.md#jest-diff)):

>The `diff` named export serializes JavaScript **values**, compares them line-by-line, and returns a string which includes comparison lines.
>
> Two named exports compare **strings** character-by-character:
>
>- `diffStringsUnified` returns a string.
>- `diffStringsRaw` returns an array of `Diff` objects.
>
>Three named exports compare **arrays of strings** line-by-line:
>
>- `diffLinesUnified` and `diffLinesUnified2` return a string.
>- `diffLinesRaw` returns an array of `Diff` objects.

The implementation is based on the [`diff-sequences`](https://github.com/facebook/jest/tree/master/packages/diff-sequences) package. `diff-sequence` compares items in two sequences to find a longest common subsequence. The items not in common are the items to delete or insert in a shortest edit script. It implements a variation of the [Myers difference algorithm](http://btn1x4.inf.uni-bayreuth.de/publications/dotor_buchmann/SCM/ChefRepo/DiffUndMerge/DAlgorithmVariations.pdf).

## Implementation details

Let's look at the path comparing two JavaScript objects.

The [entry point](https://github.com/facebook/jest/blob/98f10e698ae986c19fef2d8117be2341bcfb8f7f/packages/jest-diff/src/index.ts#L60-L111) function checks edge cases (mismatching types, asymmetric matchers) and delegates the work to specialized methods:

```typescript
// Generate a string that will highlight the difference between two values
// with green and red. (similar to how github does code diffing)
// eslint-disable-next-line @typescript-eslint/explicit-module-boundary-types
export function diff(a: any, b: any, options?: DiffOptions): string | null {
  if (Object.is(a, b)) {
    return getCommonMessage(NO_DIFF_MESSAGE, options);
  }

  const aType = getType(a);
  let expectedType = aType;
  let omitDifference = false;
  if (aType === 'object' && typeof a.asymmetricMatch === 'function') {
    if (a.$$typeof !== Symbol.for('jest.asymmetricMatcher')) {
      // Do not know expected type of user-defined asymmetric matcher.
      return null;
    }
    if (typeof a.getExpectedType !== 'function') {
      // For example, expect.anything() matches either null or undefined
      return null;
    }
    expectedType = a.getExpectedType();
    // Primitive types boolean and number omit difference below.
    // For example, omit difference for expect.stringMatching(regexp)
    omitDifference = expectedType === 'string';
  }

  if (expectedType !== getType(b)) {
    return (
      '  Comparing two different types of values.' +
      ` Expected ${chalk.green(expectedType)} but ` +
      `received ${chalk.red(getType(b))}.`
    );
  }

  if (omitDifference) {
    return null;
  }

  switch (aType) {
    case 'string':
      return diffLinesUnified(a.split('\n'), b.split('\n'), options);
    case 'boolean':
    case 'number':
      return comparePrimitive(a, b, options);
    case 'map':
      return compareObjects(sortMap(a), sortMap(b), options);
    case 'set':
      return compareObjects(sortSet(a), sortSet(b), options);
    default:
      return compareObjects(a, b, options);
  }
}
```

The [comparison method](https://github.com/facebook/jest/blob/98f10e698ae986c19fef2d8117be2341bcfb8f7f/packages/jest-diff/src/index.ts#L133-L192) for objects is below. It first serializes the objects to jsons and compares them as arrays of strings. If it doesn't produce a result, it tries to do the same with an alternative serializer.

If lines in the arrays were compared as-is, the diff would look unnecessarily big because of indentation. E.g. if `expected = complexObject` and `actual = { "foo": complexObject }`, the changed indentation would make them appear as if they had nothing in common. The trick is to compare the lines ignoring indentation, but then display them with true indentation - note the `compare` and `display` variables.

The format options require some explanations:
* `FORMAT_OPTIONS` - default formatter, uses `.toJSON()` if available.
* `FORMAT_OPTIONS_0` - same as `FORMAT_OPTIONS`, but without indentation.
* `FALLBACK_FORMAT_OPTIONS` - alternative formatter not using `.toJSON()`; max depth is 10.
* `FALLBACK_FORMAT_OPTIONS_0` - same as `FALLBACK_FORMAT_OPTIONS_0`, but without indentation.

```typescript
function compareObjects(
  a: Record<string, any>,
  b: Record<string, any>,
  options?: DiffOptions,
) {
  let difference;
  let hasThrown = false;
  const noDiffMessage = getCommonMessage(NO_DIFF_MESSAGE, options);

  try {
    const aCompare = prettyFormat(a, FORMAT_OPTIONS_0);
    const bCompare = prettyFormat(b, FORMAT_OPTIONS_0);

    if (aCompare === bCompare) {
      difference = noDiffMessage;
    } else {
      const aDisplay = prettyFormat(a, FORMAT_OPTIONS);
      const bDisplay = prettyFormat(b, FORMAT_OPTIONS);

      difference = diffLinesUnified2(
        aDisplay.split('\n'),
        bDisplay.split('\n'),
        aCompare.split('\n'),
        bCompare.split('\n'),
        options,
      );
    }
  } catch {
    hasThrown = true;
  }

  // If the comparison yields no results, compare again but this time
  // without calling `toJSON`. It's also possible that toJSON might throw.
  if (difference === undefined || difference === noDiffMessage) {
    const aCompare = prettyFormat(a, FALLBACK_FORMAT_OPTIONS_0);
    const bCompare = prettyFormat(b, FALLBACK_FORMAT_OPTIONS_0);

    if (aCompare === bCompare) {
      difference = noDiffMessage;
    } else {
      const aDisplay = prettyFormat(a, FALLBACK_FORMAT_OPTIONS);
      const bDisplay = prettyFormat(b, FALLBACK_FORMAT_OPTIONS);

      difference = diffLinesUnified2(
        aDisplay.split('\n'),
        bDisplay.split('\n'),
        aCompare.split('\n'),
        bCompare.split('\n'),
        options,
      );
    }

    if (difference !== noDiffMessage && !hasThrown) {
      difference =
        getCommonMessage(SIMILAR_MESSAGE, options) + '\n\n' + difference;
    }
  }

  return difference;
}
```

We will stop here, but there's a lot more to learn from [this package](https://github.com/facebook/jest/tree/98f10e698ae986c19fef2d8117be2341bcfb8f7f/packages/jest-diff) if you decide to dive deeper. For instance, [here](https://github.com/facebook/jest/blob/98f10e698ae986c19fef2d8117be2341bcfb8f7f/packages/diff-sequences/src/index.ts) are the "guts" of the sequence difference algorithm.

## Testing

The [test coverage](https://github.com/facebook/jest/blob/98f10e698ae986c19fef2d8117be2341bcfb8f7f/packages/jest-diff/src/__tests__) for `jest-diff` is very extensive. For example, here's [one of the tests](https://github.com/facebook/jest/blob/98f10e698ae986c19fef2d8117be2341bcfb8f7f/packages/jest-diff/src/__tests__/diff.test.ts#L192-L212) for comparing two objects:

```typescript
describe('objects', () => {
  const a = {a: {b: {c: 5}}};
  const b = {a: {b: {c: 6}}};
  const expected = [
    '  Object {',
    '    "a": Object {',
    '      "b": Object {',
    '-       "c": 5,',
    '+       "c": 6,',
    '      },',
    '    },',
    '  }',
  ].join('\n');

  test('(unexpanded)', () => {
    expect(diff(a, b, unexpandedBe)).toBe(expected);
  });
  test('(expanded)', () => {
    expect(diff(a, b, expandedBe)).toBe(expected);
  });
});
```

## Related

* [difflib](https://docs.python.org/3/library/difflib.html) provides similar functionality in Python.

## References

* [GitHub Repo](https://github.com/facebook/jest)
* [Official Website](https://jestjs.io/)
* [Myers difference algorithm](http://btn1x4.inf.uni-bayreuth.de/publications/dotor_buchmann/SCM/ChefRepo/DiffUndMerge/DAlgorithmVariations.pdf)

## Copyright notice

Jest is licensed under the [MIT License](https://github.com/facebook/jest/blob/master/LICENSE).

Copyright (c) Facebook, Inc. and its affiliates. All Rights Reserved.
