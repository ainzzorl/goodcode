---
title:  "Error Prone - Testing Bug Checkers [Java]"
layout: default
last_modified_date: 2021-09-01T17:57:00+0300
nav_order: 2

status: PUBLISHED
language: Java
short-title: Testing Bug Checkers
project:
  name: Error Prone
  key: error-prone
  home-page: https://github.com/google/error-prone
tags: [dsl, test-helper, builder]
---

{% include article-meta.html article=page %}

## Context

[Error Prone](https://errorprone.info) is a static analysis tool for Java that catches common programming mistakes at compile-time.

Error Prone is comprised of hundreds of different checkers searching for different types of defects. Checkers inherit the same base class [`BugChecker`](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/check_api/src/main/java/com/google/errorprone/bugpatterns/BugChecker.java#). In the nutshell, the interface for a checker is *unit of code (class, method, etc.) in - findings out*.

## Problem

There must be an easy and uniform way to test checkers. Tests must be easy to read and easy to write. They must be agnostic to how source code is represented and how these representations (abstract syntax trees, etc.) are built, how the checker interacts with the rest of the system, etc.

## Overview

Error Prone implements a helper class, [`CompilationTestHelper`](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/test_helpers/src/main/java/com/google/errorprone/CompilationTestHelper.java), to simplify writing bug checkers.

It is initialized with the checker under test and accepts source code to run the checker on. That code can either be inlined in the test or read from another file.

The test's author annotates the input code with comments marking where the checker must fire and what it must output.

`CompilationTestHelper` then runs the checker on the provided code and compares its output with the expectation extracted from the marker comments.

[Example usage (checker input is inlined)](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/core/src/test/java/com/google/errorprone/bugpatterns/HashCodeToStringTest.java#L29-L62):

```java
public class HashCodeToStringTest {

  private final CompilationTestHelper compilationHelper =
      CompilationTestHelper.newInstance(HashCodeToString.class, getClass());

  @Test
  public void testPositiveCase() {
    compilationHelper
        .addSourceLines(
            "HashCodeOnly.java",
            "public class HashCodeOnly {",
            "  // BUG: Diagnostic contains: HashCodeToString",
            "  public int hashCode() {",
            "    return 0;",
            "  }",
            "}")
        .doTest();
  }

  @Test
  public void negative_bothHashCodeAndToString() {
    compilationHelper
        .addSourceLines(
            "HashCodeAndToString.java",
            "public class HashCodeAndToString {",
            "  public int hashCode() {",
            "    return 0;",
            "  }",
            "  public String toString() {",
            "    return \"42\";",
            "  }",
            "}")
        .doTest();
  }
```

[Example usage (checker input is read from another file)](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/core/src/test/java/com/google/errorprone/bugpatterns/ComparableTypeTest.java#L25-L38):

```java
public class ComparableTypeTest {
  private final CompilationTestHelper compilationHelper =
      CompilationTestHelper.newInstance(ComparableType.class, getClass());

  @Test
  public void testPositiveCase() {
    compilationHelper.addSourceFile("ComparableTypePositiveCases.java").doTest();
  }

  @Test
  public void testNegativeCase() {
    compilationHelper.addSourceFile("ComparableTypeNegativeCases.java").doTest();
  }
}
```

Unlike typical unit tests that interact with the public interface of the code under test, these Error Prone bug checkers' tests interact with the code under test in a very indirect way. They follow they [black-box approach](https://en.wikipedia.org/wiki/Black-box_testing) and make no assumptions about the checker's implementation.

## Implementation details

`CompilationTestHelper` [uses](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/test_helpers/src/main/java/com/google/errorprone/CompilationTestHelper.java#L168-L198) the [Builder pattern](https://refactoring.guru/design-patterns/builder/java/example) (not the same Builder as [described in the GoF book!](https://en.wikipedia.org/wiki/Builder_pattern)) to add source code and various configurations:

```java
  /**
   * Adds a source file to the test compilation, from the string content of the file.
   *
   * <p>The diagnostics expected from compiling the file are inferred from the file contents. For
   * each line of the test file that contains the bug marker pattern "// BUG: Diagnostic contains:
   * foo", we expect to see a diagnostic on that line containing "foo". For each line of the test
   * file that does <i>not</i> contain the bug marker pattern, we expect no diagnostic to be
   * generated. You can also use "// BUG: Diagnostic matches: X" in tandem with {@code
   * expectErrorMessage("X", "foo")} to allow you to programmatically construct the error message.
   *
   * @param path a path for the source file
   * @param lines the content of the source file
   */
  public CompilationTestHelper addSourceLines(String path, String... lines) {
    this.sources.add(forSourceLines(path, lines));
    return this;
  }

  /**
   * Adds a source file to the test compilation, from an existing resource file.
   *
   * <p>See {@link #addSourceLines} for how expected diagnostics should be specified.
   *
   * @param path the path to the source file
   */
  public CompilationTestHelper addSourceFile(String path) {
    this.sources.add(forResource(clazz, path));
    return this;
  }
```

The main [`doTest` method](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/test_helpers/src/main/java/com/google/errorprone/CompilationTestHelper.java#L293-L346) that compiles the supplied code and compares the output to the expectations:
```java
  /** Performs a compilation and checks that the diagnostics and result match the expectations. */
  public void doTest() {
    checkState(!sources.isEmpty(), "No source files to compile");
    checkState(!run, "doTest should only be called once");
    this.run = true;
    Result result = compile();
    for (Diagnostic<? extends JavaFileObject> diagnostic : diagnosticHelper.getDiagnostics()) {
      if (diagnostic.getCode().contains("error.prone.crash")) {
        fail(diagnostic.getMessage(Locale.ENGLISH));
      }
    }
    if (expectNoDiagnostics) {
      List<Diagnostic<? extends JavaFileObject>> diagnostics = diagnosticHelper.getDiagnostics();
      assertWithMessage(
              String.format(
                  "Expected no diagnostics produced, but found %d: %s",
                  diagnostics.size(), diagnostics))
          .that(diagnostics.size())
          .isEqualTo(0);
      assertWithMessage(
              String.format(
                  "Expected compilation result to be "
                      + expectedResult.orElse(Result.OK)
                      + ", but was %s. No diagnostics were emitted."
                      + " OutputStream from Compiler follows.\n\n%s",
                  result,
                  outputStream))
          .that(result)
          .isEqualTo(expectedResult.orElse(Result.OK));
    } else {
      for (JavaFileObject source : sources) {
        try {
          diagnosticHelper.assertHasDiagnosticOnAllMatchingLines(
              source, lookForCheckNameInDiagnostic);
        } catch (IOException e) {
          throw new UncheckedIOException(e);
        }
      }
      assertWithMessage("Unused error keys: " + diagnosticHelper.getUnusedLookupKeys())
          .that(diagnosticHelper.getUnusedLookupKeys().isEmpty())
          .isTrue();
    }

    expectedResult.ifPresent(
        expected ->
            assertWithMessage(
                    String.format(
                        "Expected compilation result %s, but was %s\n%s\n%s",
                        expected,
                        result,
                        Joiner.on('\n').join(diagnosticHelper.getDiagnostics()),
                        outputStream))
                .that(result)
                .isEqualTo(expected));
  }
```

[Extracting markers](https://github.com/google/error-prone/blob/c601758e81723a8efc4671726b8363be7a306dce/test_helpers/src/main/java/com/google/errorprone/DiagnosticTestHelper.java#L275-L353) from code and comparing them to the actual results:

```java
  /**
   * Asserts that the diagnostics contain a diagnostic on each line of the source file that matches
   * our bug marker pattern. Parses the bug marker pattern for the specific string to look for in
   * the diagnostic.
   *
   * @param source File in which to find matching lines
   */
  public void assertHasDiagnosticOnAllMatchingLines(
      JavaFileObject source, LookForCheckNameInDiagnostic lookForCheckNameInDiagnostic)
      throws IOException {
    final List<Diagnostic<? extends JavaFileObject>> diagnostics = getDiagnostics();
    final LineNumberReader reader =
        new LineNumberReader(CharSource.wrap(source.getCharContent(false)).openStream());
    do {
      String line = reader.readLine();
      if (line == null) {
        break;
      }

      List<Predicate<? super String>> predicates = null;
      if (line.contains(BUG_MARKER_COMMENT_INLINE)) {
        // Diagnostic must contain all patterns from the bug marker comment.
        List<String> patterns = extractPatterns(line, reader, BUG_MARKER_COMMENT_INLINE);
        predicates = new ArrayList<>(patterns.size());
        for (String pattern : patterns) {
          predicates.add(new SimpleStringContains(pattern));
        }
      } else if (line.contains(BUG_MARKER_COMMENT_LOOKUP)) {
        int markerLineNumber = reader.getLineNumber();
        List<String> lookupKeys = extractPatterns(line, reader, BUG_MARKER_COMMENT_LOOKUP);
        predicates = new ArrayList<>(lookupKeys.size());
        for (String lookupKey : lookupKeys) {
          assertWithMessage(
                  "No expected error message with key [%s] as expected from line [%s] "
                      + "with diagnostic [%s]",
                  lookupKey, markerLineNumber, line.trim())
              .that(expectedErrorMsgs.containsKey(lookupKey))
              .isTrue();
          predicates.add(expectedErrorMsgs.get(lookupKey));
          usedLookupKeys.add(lookupKey);
        }
      }

      if (predicates != null) {
        int lineNumber = reader.getLineNumber();
        for (Predicate<? super String> predicate : predicates) {
          Matcher<? super Iterable<Diagnostic<? extends JavaFileObject>>> patternMatcher =
              hasItem(diagnosticOnLine(source.toUri(), lineNumber, predicate));
          assertWithMessage(
                  "Did not see an error on line %s matching %s. %s",
                  lineNumber, predicate, allErrors(diagnostics))
              .that(patternMatcher.matches(diagnostics))
              .isTrue();
        }

        if (checkName != null && lookForCheckNameInDiagnostic == LookForCheckNameInDiagnostic.YES) {
          // Diagnostic must contain check name.
          Matcher<? super Iterable<Diagnostic<? extends JavaFileObject>>> checkNameMatcher =
              hasItem(
                  diagnosticOnLine(
                      source.toUri(), lineNumber, new SimpleStringContains("[" + checkName + "]")));
          assertWithMessage(
                  "Did not see an error on line %s containing [%s]. %s",
                  lineNumber, checkName, allErrors(diagnostics))
              .that(checkNameMatcher.matches(diagnostics))
              .isTrue();
        }

      } else {
        int lineNumber = reader.getLineNumber();
        Matcher<? super Iterable<Diagnostic<? extends JavaFileObject>>> matcher =
            hasItem(diagnosticOnLine(source.toUri(), lineNumber));
        if (matcher.matches(diagnostics)) {
          fail("Saw unexpected error on line " + lineNumber + ". " + allErrors(diagnostics));
        }
      }
    } while (true);
    reader.close();
  }
```

## Related

Many related products - linters, bug checkers, etc. - implement very similar patterns. For example, see SonarQube's [CheckVerifier](https://github.com/SonarSource/sonar-java/blob/434f170b9667df33eb7355c0e7e62147c48a7da8/java-checks-testkit/src/main/java/org/sonar/java/checks/verifier/CheckVerifier.java#) or Rubocop's [ExpectOffence](https://github.com/rubocop/rubocop/blob/dc858b7ba893ffeae5edfe7b8012d8f13afd6903/lib/rubocop/rspec/expect_offense.rb).

Some other Java code analysis tools are built on top of Error Prone, making use of its extensible design. For example, Uber's [NullAway](https://github.com/uber/NullAway). NullAway also uses Error Prone's test helper, e.g. [here](https://github.com/uber/NullAway/blob/067c31dca42a7d2302c3058cb7a626bc02574c39/nullaway/src/test/java/com/uber/nullaway/NullAwayTest.java#L294-L331).

## References

* [GitHub Repo](https://github.com/google/error-prone)
* [Error Prone](https://errorprone.info)
* [Writing a check](https://github.com/google/error-prone/wiki/Writing-a-check)

## Copyright notice

Error Prone is licensed under the [Apache License 2.0](https://github.com/google/error-prone/blob/master/COPYING).
