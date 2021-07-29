# Learning from Open Source

If you write code, it's natural to want to get better at it. When you ask someone how to get better at coding, the second most common response (right after writing code) is to read other people's code and to learn from it.

While it sounds very reasonable, it's hard to implement it in practice. So you find the source for your favorite software on GitHub and start reading it. Firstly, for any established and mature product, the codebase is probably huge, complex, and very hard to get started with. It's unlikely that the design is documented anywhere. Even if the high-level architecture is documented, the structure of the codebase almost certainly isn't. It probably depends on some libraries and frameworks you've never heard of. Unfamiliar terms and creative codenames are all over the place. Making sense of it can be a challenge even for seasoned professionals. And where do you start exploring the code? If you open a random source file, chances are that it's mostly boilerplate code from which you won't learn much.

It doesn't mean that you should not try to understand good open source projects. You should, and the experience can be very rewarding. Yet, it's not an easy way, and the learning curve will be steep. 

[Code Catalog](https://codecatalog.org) offers a more straightforward way to learn from open-source projects. We present a curated collection of annotated code examples that we find instructive. The examples are:
* Taken from **popular, established open-source projects**.
* **Instructive**. They solve somewhat general problems, similar to what some other coders on some projects could be facing. They use some pattern you could use some day.
* Somewhat **self-contained**. They can be understood with little knowledge of the surrounding context.
* **Small**-ish. One example can be read in one sitting.
* **Non-trivial**.
* **Good code!** At least in our opinion.

Please note that the annotations are written by people who did not author the code they are describing. The article author's understanding of the code can be incomplete or even completely wrong. Your discretion is adviced.

# Articles

Read the articles on https://codecatalog.org. The sources are located under [_articles](./_articles).

Also see [proposed examples](https://github.com/ainzzorl/goodcode/issues?q=is%3Aissue+is%3Aopen+label%3A%22new+example%22).

# FAQ

### Who is this project for?

Software practicioners and intermediate+ learners of programming.

The reader is expected to be able to read non-trivial code in the language used by the example. Most examples expect little domain knowledge and provide context necessary to understand them.

### What can I learn from this?

* Style and best practices from prominent open-source projects.
* Applications of common patterns.
* Battle-tested solutions to typical problems.

### Who writes these articles?

The founder of the project, [Anton Emelyanov](https://github.com/ainzzorl). Future content is expected to be crowd-sourced. Read more about [contributing to the project](#Contributing).

We believe that the code speaks for itself, though. Our job is to find interesting examples, provide necessary context and point out certain details.

### Why did you choose *Example X* from *Project Y* and not something else? It's not nearly the most interesting thing in *Project Y*.

Because we find that *Example X* satisfies the criteria described above. But there's no reason why there can only be one example from *Project Y*. Please [suggest adding another example](https://github.com/ainzzorl/goodcode/issues/new?assignees=&labels=new+example&template=new-example-proposal.md&title=%5BNEW+EXAMPLE%5D+PROJECT+-+TITLE).

# Contributing

This project cannot succeed without volunteer effort. Your help in improving existing or adding new articles is very welcome. See [CONTRIBUTING.md](./CONTRIBUTING.md) for details.
