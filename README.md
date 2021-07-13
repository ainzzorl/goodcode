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

### Planned

| Title     | Project | Language(s)
| ----------- | ----------- | -- |
| [Chaos Monkey - Database access layer](https://github.com/ainzzorl/goodcode/issues/4)      | [Chaos Monkey](https://github.com/Netflix/chaosmonkey) | Go
| [AWS CDK - Diff computation](https://github.com/ainzzorl/goodcode/issues/5)      | [AWS CDK](https://github.com/aws/aws-cdk) | TypeScript
| [Stockfish - Chess board representation](https://github.com/ainzzorl/goodcode/issues/6)      | [Stockfish](https://github.com/official-stockfish/Stockfish) | C++
| [AWS CLI - Shorthand parser](https://github.com/ainzzorl/goodcode/issues/7)      | [AWS CLI](https://github.com/aws/aws-cli) | Python


Also see [all proposed examples](https://github.com/ainzzorl/goodcode/issues?q=is%3Aissue+is%3Aopen+label%3A%22new+example%22).

# Contributing

If you know a code example the satisfies the criteria above, suggest adding it to the collection by [creating an issue](https://github.com/ainzzorl/goodcode/issues/new?assignees=&labels=new+example&template=new-example-proposal.md&title=%5BNEW+EXAMPLE%5D+PROJECT+-+TITLE).
