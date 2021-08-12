---
title:  "Protocol Buffers - Stream Tokenizer [C++]"
layout: default
last_modified_date: 2021-08-12T16:43:00+0300
nav_order: 14

status: PUBLISHED
language: C++
short-title: Stream Tokenizer
project:
    name: Protocol Buffers
    key: protobuf
    home-page: https://github.com/protocolbuffers/protobuf/
tags: ['tokenizer', 'parsing']
---

{% include article-meta.html article=page %}

## Context

[Protocol buffers](https://developers.google.com/protocol-buffers/) (Protobuf) are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages.

## Problem

Parsing serialized data requires [tokenizing](https://en.wikipedia.org/wiki/Lexical_analysis) (lexing) data streams.

## Overview

Protobuf implements a custom tokenizer. Why it needs to exist and why it's implemented this way is explained in much detail in [this comment](https://github.com/protocolbuffers/protobuf/blob/8a3c4948a49d3b38effea499fd9dee66f28cb0c4/src/google/protobuf/io/tokenizer.cc#L35-L89):

```c++
// Here we have a hand-written lexer.  At first you might ask yourself,
// "Hand-written text processing?  Is Kenton crazy?!"  Well, first of all,
// yes I am crazy, but that's beside the point.  There are actually reasons
// why I ended up writing this this way.
//
// The traditional approach to lexing is to use lex to generate a lexer for
// you.  Unfortunately, lex's output is ridiculously ugly and difficult to
// integrate cleanly with C++ code, especially abstract code or code meant
// as a library.  Better parser-generators exist but would add dependencies
// which most users won't already have, which we'd like to avoid.  (GNU flex
// has a C++ output option, but it's still ridiculously ugly, non-abstract,
// and not library-friendly.)
//
// The next approach that any good software engineer should look at is to
// use regular expressions.  And, indeed, I did.  I have code which
// implements this same class using regular expressions.  It's about 200
// lines shorter.  However:
// - Rather than error messages telling you "This string has an invalid
//   escape sequence at line 5, column 45", you get error messages like
//   "Parse error on line 5".  Giving more precise errors requires adding
//   a lot of code that ends up basically as complex as the hand-coded
//   version anyway.
// - The regular expression to match a string literal looks like this:
//     kString  = new RE("(\"([^\"\\\\]|"              // non-escaped
//                       "\\\\[abfnrtv?\"'\\\\0-7]|"   // normal escape
//                       "\\\\x[0-9a-fA-F])*\"|"       // hex escape
//                       "\'([^\'\\\\]|"        // Also support single-quotes.
//                       "\\\\[abfnrtv?\"'\\\\0-7]|"
//                       "\\\\x[0-9a-fA-F])*\')");
//   Verifying the correctness of this line noise is actually harder than
//   verifying the correctness of ConsumeString(), defined below.  I'm not
//   even confident that the above is correct, after staring at it for some
//   time.
// - PCRE is fast, but there's still more overhead involved than the code
//   below.
// - Sadly, regular expressions are not part of the C standard library, so
//   using them would require depending on some other library.  For the
//   open source release, this could be really annoying.  Nobody likes
//   downloading one piece of software just to find that they need to
//   download something else to make it work, and in all likelihood
//   people downloading Protocol Buffers will already be doing so just
//   to make something else work.  We could include a copy of PCRE with
//   our code, but that obligates us to keep it up-to-date and just seems
//   like a big waste just to save 200 lines of code.
//
// On a similar but unrelated note, I'm even scared to use ctype.h.
// Apparently functions like isalpha() are locale-dependent.  So, if we used
// that, then if this code is being called from some program that doesn't
// have its locale set to "C", it would behave strangely.  We can't just set
// the locale to "C" ourselves since we might break the calling program that
// way, particularly if it is multi-threaded.  WTF?  Someone please let me
// (Kenton) know if I'm missing something here...
//
// I'd love to hear about other alternatives, though, as this code isn't
// exactly pretty.
```

## Implementation details

Let's look at the method [`Next`](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/io/tokenizer.cc#L565-L659), which reads the next token. It looks at the beginning of the token and identifies its type (string, number, identifier, etc.) and then, knowing the type, delegates reading the rest of the token to specialized methods.

```c++
bool Tokenizer::Next() {
  previous_ = current_;

  while (!read_error_) {
    ConsumeZeroOrMore<Whitespace>();

    switch (TryConsumeCommentStart()) {
      case LINE_COMMENT:
        ConsumeLineComment(NULL);
        continue;
      case BLOCK_COMMENT:
        ConsumeBlockComment(NULL);
        continue;
      case SLASH_NOT_COMMENT:
        return true;
      case NO_COMMENT:
        break;
    }

    // Check for EOF before continuing.
    if (read_error_) break;

    if (LookingAt<Unprintable>() || current_char_ == '\0') {
      AddError("Invalid control characters encountered in text.");
      NextChar();
      // Skip more unprintable characters, too.  But, remember that '\0' is
      // also what current_char_ is set to after EOF / read error.  We have
      // to be careful not to go into an infinite loop of trying to consume
      // it, so make sure to check read_error_ explicitly before consuming
      // '\0'.
      while (TryConsumeOne<Unprintable>() ||
             (!read_error_ && TryConsume('\0'))) {
        // Ignore.
      }

    } else {
      // Reading some sort of token.
      StartToken();

      if (TryConsumeOne<Letter>()) {
        ConsumeZeroOrMore<Alphanumeric>();
        current_.type = TYPE_IDENTIFIER;
      } else if (TryConsume('0')) {
        current_.type = ConsumeNumber(true, false);
      } else if (TryConsume('.')) {
        // This could be the beginning of a floating-point number, or it could
        // just be a '.' symbol.

        if (TryConsumeOne<Digit>()) {
          // It's a floating-point number.
          if (previous_.type == TYPE_IDENTIFIER &&
              current_.line == previous_.line &&
              current_.column == previous_.end_column) {
            // We don't accept syntax like "blah.123".
            error_collector_->AddError(
                line_, column_ - 2,
                "Need space between identifier and decimal point.");
          }
          current_.type = ConsumeNumber(false, true);
        } else {
          current_.type = TYPE_SYMBOL;
        }
      } else if (TryConsumeOne<Digit>()) {
        current_.type = ConsumeNumber(false, false);
      } else if (TryConsume('\"')) {
        ConsumeString('\"');
        current_.type = TYPE_STRING;
      } else if (TryConsume('\'')) {
        ConsumeString('\'');
        current_.type = TYPE_STRING;
      } else {
        // Check if the high order bit is set.
        if (current_char_ & 0x80) {
          error_collector_->AddError(
              line_, column_,
              StringPrintf("Interpreting non ascii codepoint %d.",
                           static_cast<unsigned char>(current_char_)));
        }
        NextChar();
        current_.type = TYPE_SYMBOL;
      }

      EndToken();
      return true;
    }
  }

  // EOF
  current_.type = TYPE_END;
  current_.text.clear();
  current_.line = line_;
  current_.column = column_;
  current_.end_column = column_;
  return false;
}
```

Let's now look at one of the specialized methods, [`ConsumeNumber`](https://github.com/protocolbuffers/protobuf/blob/8a3c4948a49d3b38effea499fd9dee66f28cb0c4/src/google/protobuf/io/tokenizer.cc#L427-L480). It's more difficult than just reading a sequence of digits because the number can also be negative, float, hex or written in the exponential notation. The code is written very clearly and makes it all simple.

```c++
Tokenizer::TokenType Tokenizer::ConsumeNumber(bool started_with_zero,
                                              bool started_with_dot) {
  bool is_float = false;

  if (started_with_zero && (TryConsume('x') || TryConsume('X'))) {
    // A hex number (started with "0x").
    ConsumeOneOrMore<HexDigit>("\"0x\" must be followed by hex digits.");

  } else if (started_with_zero && LookingAt<Digit>()) {
    // An octal number (had a leading zero).
    ConsumeZeroOrMore<OctalDigit>();
    if (LookingAt<Digit>()) {
      AddError("Numbers starting with leading zero must be in octal.");
      ConsumeZeroOrMore<Digit>();
    }

  } else {
    // A decimal number.
    if (started_with_dot) {
      is_float = true;
      ConsumeZeroOrMore<Digit>();
    } else {
      ConsumeZeroOrMore<Digit>();

      if (TryConsume('.')) {
        is_float = true;
        ConsumeZeroOrMore<Digit>();
      }
    }

    if (TryConsume('e') || TryConsume('E')) {
      is_float = true;
      TryConsume('-') || TryConsume('+');
      ConsumeOneOrMore<Digit>("\"e\" must be followed by exponent.");
    }

    if (allow_f_after_float_ && (TryConsume('f') || TryConsume('F'))) {
      is_float = true;
    }
  }

  if (LookingAt<Letter>() && require_space_after_number_) {
    AddError("Need space between number and identifier.");
  } else if (current_char_ == '.') {
    if (is_float) {
      AddError(
          "Already saw decimal point or exponent; can't have another one.");
    } else {
      AddError("Hex and octal numbers must be integers.");
    }
  }

  return is_float ? TYPE_FLOAT : TYPE_INTEGER;
}
```

## Testing

There's a very comprehensive [test suite](https://github.com/protocolbuffers/protobuf/blob/8a3c4948a49d3b38effea499fd9dee66f28cb0c4/src/google/protobuf/io/tokenizer_unittest.cc).

Let's look at [one of the tests](https://github.com/protocolbuffers/protobuf/blob/master/src/google/protobuf/io/tokenizer_unittest.cc#L290-L318) for tokenizing floats:

```c++
TEST_1D(TokenizerTest, FloatSuffix, kBlockSizes) {
  // Test the "allow_f_after_float" option.

  // Set up the tokenizer.
  const char* text = "1f 2.5f 6e3f 7F";
  TestInputStream input(text, strlen(text), kBlockSizes_case);
  TestErrorCollector error_collector;
  Tokenizer tokenizer(&input, &error_collector);
  tokenizer.set_allow_f_after_float(true);

  // Advance through tokens and check that they are parsed as expected.
  ASSERT_TRUE(tokenizer.Next());
  EXPECT_EQ(tokenizer.current().text, "1f");
  EXPECT_EQ(tokenizer.current().type, Tokenizer::TYPE_FLOAT);
  ASSERT_TRUE(tokenizer.Next());
  EXPECT_EQ(tokenizer.current().text, "2.5f");
  EXPECT_EQ(tokenizer.current().type, Tokenizer::TYPE_FLOAT);
  ASSERT_TRUE(tokenizer.Next());
  EXPECT_EQ(tokenizer.current().text, "6e3f");
  EXPECT_EQ(tokenizer.current().type, Tokenizer::TYPE_FLOAT);
  ASSERT_TRUE(tokenizer.Next());
  EXPECT_EQ(tokenizer.current().text, "7F");
  EXPECT_EQ(tokenizer.current().type, Tokenizer::TYPE_FLOAT);

  // There should be no more input.
  EXPECT_FALSE(tokenizer.Next());
  // There should be no errors.
  EXPECT_TRUE(error_collector.text_.empty());
}
```

## Related

* [Tokenizer](https://github.com/amzn/ion-java/blob/0eb62ba92633f03e9f3175bb77f1b3214fcc1cdd/src/com/amazon/ion/impl/IonReaderTextRawTokensX.java) in [Ion](https://en.wikipedia.org/wiki/Ion_(serialization_format)), a data serialization format developed by Amazon.

## References

* [GitHub Repo](https://github.com/protocolbuffers/protobuf)
* [Protocol Buffers on Wikipedia](https://en.wikipedia.org/wiki/Protocol_Buffers)
* [Protocol Buffers on developer.google.com](https://developers.google.com/protocol-buffers/)

## Copyright notice

Protocol Buffers is licensed under a [custom license](https://github.com/protocolbuffers/protobuf/blob/master/LICENSE).

Copyright 2008 Google Inc.
