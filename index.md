---
layout: default
title: A collection of code examples from prominent open-source projects
---

# Learning from Open Source

[Code Catalog](https://codecatalog.org) is a collection of instructive code examples with annotations. The examples are:
* Taken from **popular, established open-source projects**.
* **Instructive**. They solve general problems, similar to what other coders could be facing in their projects. They use patterns that you could apply one day.
* Mostly **self-contained**. They can be understood with little knowledge of the surrounding context.
* **Small**-ish. One example can be read in one sitting.
* **Non-trivial**.
* **Good code!** At least in our opinion.

# Catalog

{% include article-list.html %}

# FAQ

### Why do we need this?

When you ask someone how to get better at coding, the second most common response - right after writing more code - is to read other people's code and learn from it.

While it sounds very reasonable, it's hard to implement it in practice. So you find the source for your favorite software on GitHub and start reading it. Firstly, for any established and mature product, the codebase is probably huge, complex and very hard to get started with. It's unlikely that the design is documented anywhere. Even if the high-level architecture is documented, the structure of the codebase almost certainly isn't. It probably depends on some libraries and frameworks you've never heard of. Unfamiliar terms and creative code names are all over the place. Making sense of it can be a challenge even for seasoned professionals. And how do you even start exploring the code? Open a random source file?

It doesn't mean that you should not try to understand good open source projects. You should, and the experience can be very rewarding. Yet, it's not an easy way, and the learning curve will be steep.

Code Catalog strives to make learning from other people's code easier by finding good examples, explaining, annotating and categorizing them.

If you find an example interesting, we encourage you to study the original codebase in more detail. You should have a better understanding of how it works by then.

### Who is this project for?

Software practitioners and intermediate+ learners of programming.

The reader is expected to be able to read non-trivial code in the language used by the example. Most examples expect little domain knowledge and provide context necessary to understand them.

### What can I learn from this?

* Style and best practices from prominent open-source projects.
* Applications of common patterns.
* Battle-tested solutions to typical problems.

### Why so few examples?

You'd be surprised how much time and effort it takes to find examples meeting our criteria, understand and then explain them. If it was easy, this project would add no value.

We are working on adding more. See [proposed examples](https://github.com/ainzzorl/goodcode/issues?q=is%3Aopen+is%3Aissue+label%3A%22new+example%22).

### Who writes these articles?

The founder of the project, [Anton Emelyanov](https://github.com/ainzzorl). Future content is expected to be crowd-sourced. Read more about [contributing to the project](#contributing).

We believe that the code speaks for itself, though. Our job is to find interesting examples, provide necessary context and point out certain details.

Please note that the articles are not written by the authors of the code they are describing. The article author's understanding of the code can be incomplete or even completely wrong. Your discretion is advised.

### Why did you choose *Example X* from *Project Y* and not something else? It's not nearly the most interesting thing in *Project Y*.

Because we find that *Example X* satisfies the criteria above. But there's no reason why there can only be one example from *Project Y*. Please [suggest adding another example](https://github.com/ainzzorl/goodcode/issues/new?assignees=&labels=new+example&template=new-example-proposal.md&title=%5BNEW+EXAMPLE%5D+PROJECT+-+TITLE).

# Contributing

This project cannot succeed without volunteer effort. Your help in improving existing or adding new articles is very welcome. Read the [contributing guide](https://github.com/ainzzorl/goodcode/blob/main/CONTRIBUTING.md) for details.
