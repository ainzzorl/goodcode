---
title:  "did_you_mean - Correcting Typos in Ruby [Ruby]"
layout: default
last_modified_date: 2021-08-26T16:52:00+0300
nav_order: 16

status: PUBLISHED
language: Ruby
short-title: Correcting typos in Ruby
project:
    name: did_you_mean
    key: did_you_mean
    home-page: https://github.com/ruby/did_you_mean
tags: ['spell-checking']
---

{% include article-meta.html article=page %}

## Context

did_you_mean is *the gem that has been saving people from typos since 2014*. When a Ruby program fails because a name of a class, method or something else is mistyped, did_you_mean augments the error message with suggested corrections.

Example from its README:

```ruby
full_name = "Yuki Nishijima"
full_name.starts_with?("Y")
# => NoMethodError: undefined method `starts_with?' for "Yuki Nishijima":String
#    Did you mean?  start_with?
```

Ruby 2.3 and later [ship with this gem](https://github.com/ruby/ruby/tree/master/lib/did_you_mean).

## Problem

When a Ruby program fails with a `NameError`, how do you suggest corrections?

## Overview

did_you_mean adds corrections [by overriding](https://github.com/ruby/did_you_mean/blob/master/lib/did_you_mean/core_ext/name_error.rb#L14-L30) `NameError`'s `to_s` method.

Corrections are generated by spell-checking against a dictionary populated with all symbols that could be used in that place.

The spell-checker uses two similarity metrics: [Jaro–Winkler](https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance) and [Levenshtein](https://en.wikipedia.org/wiki/Levenshtein_distance).

## Implementation details

[Extending](https://github.com/ruby/did_you_mean/blob/9c4ccac47de4722ed22d1a33791e641bc4ba9664/lib/did_you_mean/core_ext/name_error.rb) `NameError#to_s`. Suggestions are generated by spell-checkers and then formatted and appended to the original message.

```ruby
    def to_s
      msg = super.dup
      suggestion = DidYouMean.formatter.message_for(corrections)

      msg << suggestion if !msg.include?(suggestion)
      msg
    rescue
      super
    end

    def corrections
      @corrections ||= spell_checker.corrections
    end

    def spell_checker
      SPELL_CHECKERS[self.class.to_s].new(self)
    end
```

[Actual spell-checking](https://github.com/ruby/did_you_mean/blob/9c4ccac47de4722ed22d1a33791e641bc4ba9664/lib/did_you_mean/spell_checker.rb) code is quite short, but there's a lot going on there.

First, it selects dictionary words within the threshold Jaro-Winkler distance from the input. The thresholds look [magic](https://en.wikipedia.org/wiki/Magic_number_(programming)), but they must be carefully tuned. Then it filters out words with high Levenshtein distance from the input. If the result is not empty, return it; otherwise, return the word with the lowest Jaro-Winkler distance from the input that has lower Levenshtein distance than the shortest of the two words.

Levenshtein and Jaro-Winkler are two different metrics measuring the edit distance between two strings. The higher the value, the more similar the strings are. We will not explain the details of these metrics - there are many resources dedicated to that, see the *References* section - but one thing worth mentioning is that Jaro-Winkler favors strings that share prefixes. This is expected to make it good for finding mistypes - the assumption is that if you mistype a word, the first character is very likely to be correct.

```ruby
module DidYouMean
  class SpellChecker
    def initialize(dictionary:)
      @dictionary = dictionary
    end

    def correct(input)
      input     = normalize(input)
      threshold = input.length > 3 ? 0.834 : 0.77

      words = @dictionary.select { |word| JaroWinkler.distance(normalize(word), input) >= threshold }
      words.reject! { |word| input == word.to_s }
      words.sort_by! { |word| JaroWinkler.distance(word.to_s, input) }
      words.reverse!

      # Correct mistypes
      threshold   = (input.length * 0.25).ceil
      corrections = words.select { |c| Levenshtein.distance(normalize(c), input) <= threshold }

      # Correct misspells
      if corrections.empty?
        corrections = words.select do |word|
          word   = normalize(word)
          length = input.length < word.length ? input.length : word.length

          Levenshtein.distance(word, input) < length
        end.first(1)
      end

      corrections
    end

    private

    def normalize(str_or_symbol) #:nodoc:
      str = str_or_symbol.to_s.downcase
      str.tr!("@", "")
      str
    end
  end
end
```

The logic populating the dictionary depends on what's mistyped: the name of a class, a method, a variable, etc. For instance, let's look at populating the dictionary with all possible method names. [The code](https://github.com/ruby/did_you_mean/blob/9c4ccac47de4722ed22d1a33791e641bc4ba9664/lib/did_you_mean/spell_checkers/method_name_checker.rb) is quite self-explanatory:

```ruby
module DidYouMean
  class MethodNameChecker
    attr_reader :method_name, :receiver

    # ...

    def initialize(exception)
      @method_name  = exception.name
      @receiver     = exception.receiver
      @private_call = exception.respond_to?(:private_call?) ? exception.private_call? : false
    end

    def corrections
      @corrections ||= begin
                         dictionary = method_names
                         dictionary = RB_RESERVED_WORDS + dictionary if @private_call

                         SpellChecker.new(dictionary: dictionary).correct(method_name) - names_to_exclude
                       end
    end

    def method_names
      if Object === receiver
        method_names = receiver.methods + receiver.singleton_methods
        method_names += receiver.private_methods if @private_call
        method_names.uniq!
        method_names
      else
        []
      end
    end

    def names_to_exclude
      Object === receiver ? NAMES_TO_EXCLUDE[receiver.class] : []
    end
  end
end
```

## Testing

[Testing spell-checking](https://github.com/ruby/did_you_mean/blob/master/test/test_spell_checker.rb). E.g. testing correcting mistypes:

```ruby
  def test_spell_checker_corrects_mistypes
    assert_spell 'foo',   input: 'doo',   dictionary: ['foo', 'fork']
    assert_spell 'email', input: 'meail', dictionary: ['email', 'fail', 'eval']
    assert_spell 'fail',  input: 'fial',  dictionary: ['email', 'fail', 'eval']
    assert_spell 'fail',  input: 'afil',  dictionary: ['email', 'fail', 'eval']
    assert_spell 'eval',  input: 'eavl',  dictionary: ['email', 'fail', 'eval']
    assert_spell 'eval',  input: 'veal',  dictionary: ['email', 'fail', 'eval']
    assert_spell 'sub!',  input: 'suv!',  dictionary: ['sub', 'gsub', 'sub!']
    assert_spell 'sub',   input: 'suv',   dictionary: ['sub', 'gsub', 'sub!']

    assert_spell %w(gsub! gsub),     input: 'gsuv!', dictionary: %w(sub gsub gsub!)
    assert_spell %w(sub! sub gsub!), input: 'ssub!', dictionary: %w(sub sub! gsub gsub!)

    group_methods = %w(groups group_url groups_url group_path)
    assert_spell 'groups', input: 'group',  dictionary: group_methods

    group_classes = %w(
      GroupMembership
      GroupMembershipPolicy
      GroupMembershipDecorator
      GroupMembershipSerializer
      GroupHelper
      Group
      GroupMailer
      NullGroupMembership
    )

    assert_spell 'GroupMembership',          dictionary: group_classes, input: 'GroupMemberhip'
    assert_spell 'GroupMembershipDecorator', dictionary: group_classes, input: 'GroupMemberhipDecorator'

    names = %w(first_name_change first_name_changed? first_name_will_change!)
    assert_spell names, input: 'first_name_change!', dictionary: names

    assert_empty DidYouMean::SpellChecker.new(dictionary: ['proc']).correct('product_path')
    assert_empty DidYouMean::SpellChecker.new(dictionary: ['fork']).correct('fooo')
  end
```

[Some tests](https://github.com/ruby/did_you_mean/blob/master/test/spell_checking/test_class_name_check.rb#L30-L46) confirming that it actually throws the right error:

```ruby
class ClassNameCheckTest < Test::Unit::TestCase
  include DidYouMean::TestHelper

  def test_corrections
    error = assert_raise(NameError) { ::Bo0k }
    assert_correction "Book", error.corrections
  end

  def test_corrections_include_case_specific_class_name
    error = assert_raise(NameError) { ::Acronym }
    assert_correction "ACRONYM", error.corrections
  end

  def test_corrections_include_top_level_class_name
    error = assert_raise(NameError) { Project.bo0k }
    assert_correction "Book", error.corrections
  end
```

## References

* [GitHub repo](https://github.com/ruby/did_you_mean/)
* [RedDotRuby 2015 - 'Did you mean?' experience in Ruby and beyond by Yuki Nishijima](https://www.youtube.com/watch?v=sca1C0Qk6ZE)
* [Levenshtein Distance](https://en.wikipedia.org/wiki/Levenshtein_distance)
* [Jaro–Winkler Distance](https://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance)
* [Deep dive into Did You Mean](https://shime.sh/deep-dive-into-did-you-mean)
* [Difference between Jaro-Winkler and Levenshtein distance?](https://stackoverflow.com/questions/25540581/difference-between-jaro-winkler-and-levenshtein-distance)
* [Jaro winkler vs Levenshtein Distance](https://srinivas-kulkarni.medium.com/jaro-winkler-vs-levenshtein-distance-2eab21832fd6)
* [https://news.ycombinator.com/item?id=8496581](https://news.ycombinator.com/item?id=8496581)
  * The author of the gem wrote a blog post about it, but it is not longer available. However, its discussion is.

## Copyright notice

did_you_mean is licensed under the [MIT License](https://github.com/ruby/did_you_mean/blob/master/LICENSE.txt).

Copyright (c) 2014-16 Yuki Nishijima.