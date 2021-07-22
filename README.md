# Learning from Open Source

If you write code, it's natural to want to get better at it. When you ask someone how to get better at coding, the second most common response (right after writing code) is to read other people's code and to learn from it.

While it sounds very reasonable, it's hard to implement it in practice. So you find the source for your favorite software on Github and start reading it. Firstly, for any established and mature product, the codebase is probably huge, complex, and very hard to get started with. It's unlikely that the design is documented anywhere. Even if the high-level architectureis documented, the structure of the codebase almost certainly isn't. It probably depends on some libraries and frameworks you've never heard of. Unfamiliar terms and creative codenames are all over the place. Making sense of it can be a challenge even for seasoned professionals. And where do you start exploring the code? If you open a random source file, chances are that it's mostly boilerplate code from which you won't learn much.

It doesn't mean that you should not try to understand good open source projects. You should, and the experience can be very rewarding. Yet, it's not an easy way, and the learning curve will be steep. 

Here we offer a more straightforward way to learn from open-source projects. We present a curated collection of annotated code examples that we find instructive. The examples are:
* Taken from **popular, established open-source projects**.
* **Instructive**. They solve somewhat general problems, similar to what some other coders on some projects could be facing. They use some pattern you could use some day.
* Somewhat **self-contained**. They can be understood with little knowledge of the surrounding context.
* **Small**-ish. One example can be read in one sitting.
* **Non-trivial**.
* **Good code!** At least in our opinion.

Please note that the annotations are written by people who did not author the code they are describing. The article author's understanding of the code can be incomplete or even completely wrong. Your discretion is adviced.

# Articles

The collection is small but growing.

| Title     | Project | Language(s) | Status
| ----------- | ----------- | -- | -- |
| [Homebrew - Downloading software from a variety of sources](./articles/homebrew-download-strategy.md)      | [Homebrew](https://github.com/Homebrew/brew) | Ruby   | DRAFT |
| [ErrorProne - Testing bug checkers](./articles/error-prone-test-helper.md)      | [ErrorProne](https://github.com/google/error-prone/) | Java   | DRAFT |
| [Terraform - Algorithms on graphs](./articles/terraform-graph-algorithms.md)      | [Terraform](https://github.com/hashicorp/terraform) | Go | DRAFT |
| [Chaos Monkey - Database access layer](./articles/chaos-monkey-store.md)      | [Chaos Monkey](https://github.com/Netflix/chaosmonkey) | Go | DRAFT
| [AWS CDK - Diff computation](./articles/aws-cdk-template-diff.md)      | [AWS CDK](https://github.com/aws/aws-cdk) | TypeScript | DRAFT
| [Stockfish - Board representation](./articles/stockfish-board-representation.md)      | [Stockfish](https://github.com/official-stockfish/Stockfish) | C++ | DRAFT
| [AWS CLI - Shorthand parser](./articles/aws-cli-shorthand-parser.md)      | [AWS CLI](https://github.com/aws/aws-cli) | Python | DRAFT
| [Bat - Decorations](./articles/bat-decorations.md)      | [Bat](https://github.com/sharkdp/bat) | Rust | DRAFT
| [Httpie - Status reporting](./articles/httpie-status-reporting.md)      | [Httpie](https://github.com/httpie/httpie) | Python | DRAFT
| [ZooKeeper - Trie](./articles/zookeeper-trie.md)      | [ZooKeeper](https://github.com/apache/zookeeper) | Java | DRAFT

### Planned

| Title     | Project | Language(s)
| ----------- | ----------- | -- |
| [Spark - Cross Validation](https://github.com/ainzzorl/goodcode/issues/10)      | [Spark](https://github.com/apache/spark) | Scala
| [Lucene - Rate limiting](https://github.com/ainzzorl/goodcode/issues/13)      | [Lucene](https://github.com/apache/lucene) | Java
| [ZooKeeper - Snapshot comparison](https://github.com/ainzzorl/goodcode/issues/17)      | [ZooKeeper](https://github.com/apache/zookeeper) | Java
| [ZooKeeper - Client connection](https://github.com/ainzzorl/goodcode/issues/18)      | [ZooKeeper](https://github.com/apache/zookeeper) | Java
| [Ionic Framework - Gestures](https://github.com/ainzzorl/goodcode/issues/19)      | [Ionic Framework](https://github.com/ainzzorl/goodcode/issues/19) | TypeScript


Also see [all proposed examples](https://github.com/ainzzorl/goodcode/issues?q=is%3Aissue+is%3Aopen+label%3A%22new+example%22).

# FAQ

### Who is this project for?

Software practicioners and intermediate+ learners of programming.

The reader is expected to be able to read non-trivial code in the language used by the example. Most examples expect little domain knowledge and provide context necessary to understand them.

### What can I learn from this?

* Style and best practices from prominent open-source projects.
* Applications of common patterns.
* Battle-tested solutions to typical problems.

### Who writes these articles?

The founder of the project, [ainzzorl](https://github.com/ainzzorl/goodcode). Future content is expected to be crowd-sourced. Read more about [contributing to the project](#Contributing).

We believe that the code speaks for itself, though. Our job is to find interesting examples, provide necessary context and point out certain details.

# Contributing

If you know a code example that satisfies the criteria above, suggest adding it to the collection by [creating an issue](https://github.com/ainzzorl/goodcode/issues/new?assignees=&labels=new+example&template=new-example-proposal.md&title=%5BNEW+EXAMPLE%5D+PROJECT+-+TITLE).
