---
title:  "Visual Studio Code - Skip Lists [TypeScript]"
layout: default
last_modified_date: 2021-09-15T09:26:00+0300
nav_order: 19

status: PUBLISHED
language: TypeScript
short-title: Skip Lists
project:
    name: Visual Studio Code
    key: vscode
    home-page: https://github.com/microsoft/vscode
tags: ['algorithm', 'data-structure']
---

{% include article-meta.html article=page %}

## Context

[Visual Studio Code](https://code.visualstudio.com/) is an extremely popular [IDE](https://en.wikipedia.org/wiki/Integrated_development_environment) made by Microsoft.

Resources manipulated by Visual Studio Code are identified by their URIs. But different URIs [can point](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/workbench/services/uriIdentity/common/uriIdentity.ts) to the same resource, e.g. `file:///c:/foo/bar.txt` and `file:///c:/FOO/BAR.txt`, because Windows paths are not case-sensitive.

Visual Studio Code has a [*URI Identity Service*](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/workbench/services/uriIdentity/common/uriIdentity.ts) to return a canonical [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier) for a given resource. The identity service is stateful. If it has seen a URI equal to the input before (it has a way to compare URIs), it returns that other URI. If it hasn't - the input URI becomes the canonical URI.

## Problem

What data structure should be used to store URIs? It should support fast insertions and fast lookups given a function comparing two URIs.

Note: it appears that URIs cannot be hashed, but it is not clear to us why. If they could be hashed, the best data structure would probably be a hash map.

## Overview

URIs are put in a [skip list](https://en.wikipedia.org/wiki/Skip_list). A skip list is a probabilistic data structure that allows `O(log ⁡n)` search complexity as well as `O(log n)` insertion complexity within an ordered sequence of `n` elements.

> *Skip lists are a simple data structure that can be used in place of balanced trees for most applications. Skip lists algorithms are very easy to implement, extend and modify. Skip lists are about as fast as highly optimized balanced tree algorithms and are substantially faster than casually implemented balanced tree algorithms.* - [William Pugh](https://www.epaperpress.com/sortsearch/download/skiplist.pdf)

> *... it can get the best features of a sorted array (for searching) while maintaining a linked list-like structure that allows insertion, which is not possible in an array. Fast search is made possible by maintaining a linked hierarchy of subsequences, with each successive subsequence skipping over fewer elements than the previous one (see the picture below on the right). Searching starts in the sparsest subsequence until two consecutive elements have been found, one smaller and one larger than or equal to the element searched for. Via the linked hierarchy, these two elements link to elements of the next sparsest subsequence, where searching is continued until finally we are searching in the full sequence. The elements that are skipped over may be chosen probabilistically or deterministically, with the former being more common.* - [Wikipedia](https://en.wikipedia.org/wiki/Skip_list)

## Implementation details

The [implementation](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/base/common/skipList.ts#L20) follows the algorithm described in the [paper](https://www.epaperpress.com/sortsearch/download/skiplist.pdf) by William Pugh.

A [node](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/base/common/skipList.ts#L20) has a key, a value, a level and a list of forward links to other nodes:

```typescript
class Node<K, V> {
	readonly forward: Node<K, V>[];
	constructor(readonly level: number, readonly key: K, public value: V) {
		this.forward = [];
	}
}
```

Skip list itself:

```typescript
export class SkipList<K, V> implements Map<K, V> {

	readonly [Symbol.toStringTag] = 'SkipList';

	private _maxLevel: number;
	private _level: number = 0;
	private _header: Node<K, V>;
	private _size: number = 0;
```

`has`, `get`, `set` and `delete` delegate to private methods. It's not entirely clear why these private methods are static: perhaps it was done for performance.

```typescript
has(key: K): boolean {
    return Boolean(SkipList._search(this, key, this.comparator));
}

get(key: K): V | undefined {
    return SkipList._search(this, key, this.comparator)?.value;
}

set(key: K, value: V): this {
    if (SkipList._insert(this, key, value, this.comparator)) {
        this._size += 1;
    }
    return this;
}

delete(key: K): boolean {
    const didDelete = SkipList._delete(this, key, this.comparator);
    if (didDelete) {
        this._size -= 1;
    }
    return didDelete;
}
```

[Search](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/base/common/skipList.ts#L123-L135). It starts with the head node and the last level. It follows forward links until it encounters an element greater than the common, then it goes to the next level.

```typescript
private static _search<K, V>(list: SkipList<K, V>, searchKey: K, comparator: Comparator<K>) {
    let x = list._header;
    for (let i = list._level - 1; i >= 0; i--) {
        while (x.forward[i] && comparator(x.forward[i].key, searchKey) < 0) {
            x = x.forward[i];
        }
    }
    x = x.forward[0];
    if (x && comparator(x.key, searchKey) === 0) {
        return x;
    }
    return undefined;
}
```

[Insert and delete](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/base/common/skipList.ts#L137-L201). *[Insertions and deletions](https://en.wikipedia.org/wiki/Skip_list) are implemented much like the corresponding linked-list operations, except that "tall" elements must be inserted into or deleted from more than one linked list.*

```typescript
private static _insert<K, V>(list: SkipList<K, V>, searchKey: K, value: V, comparator: Comparator<K>) {
    let update: Node<K, V>[] = [];
    let x = list._header;
    for (let i = list._level - 1; i >= 0; i--) {
        while (x.forward[i] && comparator(x.forward[i].key, searchKey) < 0) {
            x = x.forward[i];
        }
        update[i] = x;
    }
    x = x.forward[0];
    if (x && comparator(x.key, searchKey) === 0) {
        // update
        x.value = value;
        return false;
    } else {
        // insert
        let lvl = SkipList._randomLevel(list);
        if (lvl > list._level) {
            for (let i = list._level; i < lvl; i++) {
                update[i] = list._header;
            }
            list._level = lvl;
        }
        x = new Node<K, V>(lvl, searchKey, value);
        for (let i = 0; i < lvl; i++) {
            x.forward[i] = update[i].forward[i];
            update[i].forward[i] = x;
        }
        return true;
    }
}

private static _randomLevel(list: SkipList<any, any>, p: number = 0.5): number {
    let lvl = 1;
    while (Math.random() < p && lvl < list._maxLevel) {
        lvl += 1;
    }
    return lvl;
}

private static _delete<K, V>(list: SkipList<K, V>, searchKey: K, comparator: Comparator<K>) {
    let update: Node<K, V>[] = [];
    let x = list._header;
    for (let i = list._level - 1; i >= 0; i--) {
        while (x.forward[i] && comparator(x.forward[i].key, searchKey) < 0) {
            x = x.forward[i];
        }
        update[i] = x;
    }
    x = x.forward[0];
    if (!x || comparator(x.key, searchKey) !== 0) {
        // not found
        return false;
    }
    for (let i = 0; i < list._level; i++) {
        if (update[i].forward[i] !== x) {
            break;
        }
        update[i].forward[i] = x.forward[i];
    }
    while (list._level > 0 && list._header.forward[list._level - 1] === NIL) {
        list._level -= 1;
    }
    return true;
}
```

## Testing

[Testing](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/base/test/common/skipList.test.ts#L48-L74) insertions, deletions and searches:

```typescript
test('set/get/delete', function () {
    let list = new SkipList<number, number>((a, b) => a - b);

    assert.strictEqual(list.get(3), undefined);
    list.set(3, 1);
    assert.strictEqual(list.get(3), 1);
    assertValues(list, [1]);

    list.set(3, 3);
    assertValues(list, [3]);

    list.set(1, 1);
    list.set(4, 4);
    assert.strictEqual(list.get(3), 3);
    assert.strictEqual(list.get(1), 1);
    assert.strictEqual(list.get(4), 4);
    assertValues(list, [1, 3, 4]);

    assert.strictEqual(list.delete(17), false);

    assert.strictEqual(list.delete(1), true);
    assert.strictEqual(list.get(1), undefined);
    assert.strictEqual(list.get(3), 3);
    assert.strictEqual(list.get(4), 4);

    assertValues(list, [3, 4]);
});
```

Functional tests are accompanied by a [performance test](https://github.com/microsoft/vscode/blob/4eae83a5232ac06b29dca142f6503d695f16da13/src/vs/base/test/common/skipList.test.ts#L144-L216).

## References

* [GitHub repo](https://github.com/microsoft/vscode)
* [Skip List on Wikipedia](https://en.wikipedia.org/wiki/Skip_list)
* [Skip Lists: A Probabilistic Alternative to Balanced Trees](https://www.epaperpress.com/sortsearch/download/skiplist.pdf)
* [Skip Lists lecture](https://www.youtube.com/watch?v=kBwUoWpeH_Q) (MIT 6.046J / 18.410J Introduction to Algorithms (SMA 5503), Fall 2005)
* [URI comparisons and resources.ts](https://github.com/microsoft/vscode/issues/93368)

## Copyright notice

Visual Studio Code is licensed under the [MIT License](https://github.com/microsoft/vscode/blob/main/LICENSE.txt).

Copyright (c) Microsoft Corporation. All rights reserved.