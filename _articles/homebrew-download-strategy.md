---
title:  "Downloading software from a variety of sources in Homebrew"
---

# Downloading software from a variety of sources in Homebrew

* **Status**: DRAFT
* **Project name**: Homebrew
* **Example name**: Downloading software from a variety of sources in Homebrew
* **Project home page**: https://github.com/Homebrew/brew
* **Programming language(s)**: Ruby
* **Frameworks, libraries used:** Sorbet, rspec
* **Tags:** strategy, template-method

## Context

Homebrew is "The Missing Package Manager for macOS (or Linux)". It offers command line interface with commands like `brew install <example>`.

Available packages are described with Ruby **fomulae** that include download URLs and source code repositories, e.g.

```ruby
class Python3 < Formula
  homepage "https://www.python.org/"
  url "https://www.python.org/ftp/python/3.4.3/Python-3.4.3.tar.xz"
  sha256 "b5b3963533768d5fc325a4d7a6bd6f666726002d696f1d399ec06b043ea996b8"
  head "https://hg.python.org/cpython", :using => :hg
```

Formulae can also include URLs of resources the package depends on.

## Problem

Homebrew needs to download and unpack the resource from the specified location, but how to do it depends on the exact URL. For instance, downloading a tarball is not the same as downloading repository content from Github. Some resources need to be unpacked, some don't.

Homebrew is expected to figure out how to download the resource automatically, formulae authors should be allowed to explicitly specify the right way to download and unpack the resource.

## Overview

Homebrew employs the [***Strategy***](https://en.wikipedia.org/wiki/Strategy_pattern) pattern to describe various ways to download a resource. Various strategies, such as `GitHubGitDownloadStrategy`, `GitDownloadStrategy`, `SubversionDownloadStrategy`, `CurlDownloadStrategy` or `LocalBottleDownloadStrategy` implement `fetch(timeout)` and some other methods. Since it's written in Ruby, the interface is defined implicitly.

Strategies inherit `AbstractDownloadStrategy` that provides common functionality, e.g. unpacking the download, caching, turning logging on and off.

The exact strategy is picked based on the URL, with a way to choose a strategy explicitly.

Similar approach is applied to unpacking resourced with *UnpackStrategy*. The details are out of the scope of this article.

There are numerous intricacies around the usage of cookies, user agents, HTTP redirects, git submodules and so on.

[Choosing the download strategy](https://github.com/Homebrew/brew/blob/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/download_strategy.rb#L1306-L1372):

```ruby
# Helper class for detecting a download strategy from a URL.
#
# @api private
class DownloadStrategyDetector
  def self.detect(url, using = nil)
    if using.nil?
      detect_from_url(url)
    elsif using.is_a?(Class) && using < AbstractDownloadStrategy
      using
    elsif using.is_a?(Symbol)
      detect_from_symbol(using)
    else
      raise TypeError,
            "Unknown download strategy specification #{using.inspect}"
    end
  end

  def self.detect_from_url(url)
    case url
    when GitHubPackages::URL_REGEX
      CurlGitHubPackagesDownloadStrategy
    when %r{^https?://github\.com/[^/]+/[^/]+\.git$}
      GitHubGitDownloadStrategy
    when %r{^https?://.+\.git$},
         %r{^git://},
         %r{^https?://git\.sr\.ht/[^/]+/[^/]+$}
      GitDownloadStrategy
    when %r{^https?://www\.apache\.org/dyn/closer\.cgi},
         %r{^https?://www\.apache\.org/dyn/closer\.lua}
      CurlApacheMirrorDownloadStrategy
    when %r{^https?://(.+?\.)?googlecode\.com/svn},
         %r{^https?://svn\.},
         %r{^svn://},
         %r{^svn\+http://},
         %r{^http://svn\.apache\.org/repos/},
         %r{^https?://(.+?\.)?sourceforge\.net/svnroot/}
      SubversionDownloadStrategy
    when %r{^cvs://}
      CVSDownloadStrategy
    when %r{^hg://},
         %r{^https?://(.+?\.)?googlecode\.com/hg},
         %r{^https?://(.+?\.)?sourceforge\.net/hgweb/}
      MercurialDownloadStrategy
    when %r{^bzr://}
      BazaarDownloadStrategy
    when %r{^fossil://}
      FossilDownloadStrategy
    else
      CurlDownloadStrategy
    end
  end

  def self.detect_from_symbol(symbol)
    case symbol
    when :hg                     then MercurialDownloadStrategy
    when :nounzip                then NoUnzipCurlDownloadStrategy
    when :git                    then GitDownloadStrategy
    when :bzr                    then BazaarDownloadStrategy
    when :svn                    then SubversionDownloadStrategy
    when :curl                   then CurlDownloadStrategy
    when :cvs                    then CVSDownloadStrategy
    when :post                   then CurlPostDownloadStrategy
    when :fossil                 then FossilDownloadStrategy
    else
      raise TypeError, "Unknown download strategy #{symbol} was requested."
    end
  end
end
```

[AbstractStrategy](https://github.com/Homebrew/brew/blob/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/download_strategy.rb#L24-L267) (base class for all strategies):

```ruby
# @abstract Abstract superclass for all download strategies.
#
# @api private
class AbstractDownloadStrategy

  # ...

  # The download URL.
  #
  # @api public
  sig { returns(String) }
  attr_reader :url

  def initialize(url, name, version, **meta)
    @url = url
    @name = name
    @version = version
    @cache = meta.fetch(:cache, HOMEBREW_CACHE)
    @meta = meta
    @quiet = false
    extend Pourable if meta[:bottle]
  end

  # Download and cache the resource at {#cached_location}.
  #
  # @api public
  def fetch(timeout: nil); end

  # Unpack {#cached_location} into the current working directory.
  #
  # Additionally, if a block is given, the working directory was previously empty
  # and a single directory is extracted from the archive, the block will be called
  # with the working directory changed to that directory. Otherwise this method
  # will return, or the block will be called, without changing the current working
  # directory.
  #
  # @api public
  def stage(&block)
    UnpackStrategy.detect(cached_location,
                          prioritise_extension: true,
                          ref_type: @ref_type, ref: @ref)
                  .extract_nestedly(basename:             basename,
                                    prioritise_extension: true,
                                    verbose:              verbose? && !quiet?)
    chdir(&block) if block
  end

   # ...
```

[CurlDownloadStrategy](https://github.com/Homebrew/brew/blob/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/download_strategy.rb#L365) ultimilaty calls `curl`.

Strategies for various version control systems (Git, Mercurial, Subversion, etc.) inherit from `VCSDownloadStrategy`. It follows the [***Template Method***](https://en.wikipedia.org/wiki/Template_method_pattern) pattern: `fetch` is implemented in `VCSDownloadStrategy`, but it relies on child strategies to implement methods such as `clone_repo`.

```ruby
# @abstract Abstract superclass for all download strategies downloading from a version control system.
#
# @api private
class VCSDownloadStrategy < AbstractDownloadStrategy
  REF_TYPES = [:tag, :branch, :revisions, :revision].freeze

  def initialize(url, name, version, **meta)
    super
    @ref_type, @ref = extract_ref(meta)
    @revision = meta[:revision]
    @cached_location = @cache/"#{name}--#{cache_tag}"
  end

  # Download and cache the repository at {#cached_location}.
  #
  # @api public
  def fetch(timeout: nil)
    end_time = Time.now + timeout if timeout

    ohai "Cloning #{url}"

    if cached_location.exist? && repo_valid?
      puts "Updating #{cached_location}"
      update(timeout: timeout)
    elsif cached_location.exist?
      puts "Removing invalid repository from cache"
      clear_cache
      clone_repo(timeout: end_time)
    else
      clone_repo(timeout: end_time)
    end

    version.update_commit(last_commit) if head?

    return if @ref_type != :tag || @revision.blank? || current_revision.blank? || current_revision == @revision

    raise <<~EOS
      #{@ref} tag should be #{@revision}
      but is actually #{current_revision}
    EOS
  end

  def fetch_last_commit
    fetch
    last_commit
  end

  def commit_outdated?(commit)
    @last_commit ||= fetch_last_commit
    commit != @last_commit
  end

  def head?
    version.respond_to?(:head?) && version.head?
  end

  # @!attribute [r] last_commit
  # Return last commit's unique identifier for the repository.
  # Return most recent modified timestamp unless overridden.
  #
  # @api public
  sig { returns(String) }
  def last_commit
    source_modified_time.to_i.to_s
  end

  private

  def cache_tag
    raise NotImplementedError
  end

  def repo_valid?
    raise NotImplementedError
  end

  sig { params(timeout: T.nilable(Time)).void }
  def clone_repo(timeout: nil); end

  sig { params(timeout: T.nilable(Time)).void }
  def update(timeout: nil); end

  def current_revision; end

  def extract_ref(specs)
    key = REF_TYPES.find { |type| specs.key?(type) }
    [key, specs[key]]
  end
end
```

[GitDownloadStrategy](https://github.com/Homebrew/brew/blob/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/download_strategy.rb#L779-L981)

`clone_repo` is called by `GitDownloadStrategy#fetch`.

```ruby
# Strategy for downloading a Git repository.
#
# @api public
class GitDownloadStrategy < VCSDownloadStrategy
  def initialize(url, name, version, **meta)
    super
    @ref_type ||= :branch
    @ref ||= "master"
  end
  
  # ...

  sig { params(timeout: T.nilable(Time)).void }
  def clone_repo(timeout: nil)
    command! "git", args: clone_args, timeout: timeout&.remaining

    command! "git",
             args:    ["config", "homebrew.cacheversion", cache_version],
             chdir:   cached_location,
             timeout: timeout&.remaining
    checkout(timeout: timeout)
    update_submodules(timeout: timeout) if submodules?
  end
end
```

[Code to resolve a URL (follow redirects), get file name, modification time and size](https://github.com/Homebrew/brew/blob/master/Library/Homebrew/download_strategy.rb#L447-L512):
```ruby
  def resolve_url_basename_time_file_size(url, timeout: nil)
    @resolved_info_cache ||= {}
    return @resolved_info_cache[url] if @resolved_info_cache.include?(url)

    if (domain = Homebrew::EnvConfig.artifact_domain)
      url = url.sub(%r{^((ht|f)tps?://)?}, "#{domain.chomp("/")}/")
    end

    out, _, status= curl_output("--location", "--silent", "--head", "--request", "GET", url.to_s, timeout: timeout)

    lines = status.success? ? out.lines.map(&:chomp) : []

    locations = lines.map { |line| line[/^Location:\s*(.*)$/i, 1] }
                     .compact

    redirect_url = locations.reduce(url) do |current_url, location|
      if location.start_with?("//")
        uri = URI(current_url)
        "#{uri.scheme}:#{location}"
      elsif location.start_with?("/")
        uri = URI(current_url)
        "#{uri.scheme}://#{uri.host}#{location}"
      elsif location.start_with?("./")
        uri = URI(current_url)
        "#{uri.scheme}://#{uri.host}#{Pathname(uri.path).dirname/location}"
      else
        location
      end
    end

    content_disposition_parser = Mechanize::HTTP::ContentDispositionParser.new

    parse_content_disposition = lambda do |line|
      next unless (content_disposition = content_disposition_parser.parse(line.sub(/; *$/, ""), true))

      filename = nil

      if (filename_with_encoding = content_disposition.parameters["filename*"])
        encoding, encoded_filename = filename_with_encoding.split("''", 2)
        filename = URI.decode_www_form_component(encoded_filename).encode(encoding) if encoding && encoded_filename
      end

      # Servers may include '/' in their Content-Disposition filename header. Take only the basename of this, because:
      # - Unpacking code assumes this is a single file - not something living in a subdirectory.
      # - Directory traversal attacks are possible without limiting this to just the basename.
      File.basename(filename || content_disposition.filename)
    end

    filenames = lines.map(&parse_content_disposition).compact

    time =
      lines.map { |line| line[/^Last-Modified:\s*(.+)/i, 1] }
           .compact
           .map { |t| t.match?(/^\d+$/) ? Time.at(t.to_i) : Time.parse(t) }
           .last

    file_size =
      lines.map { |line| line[/^Content-Length:\s*(\d+)/i, 1] }
           .compact
           .map(&:to_i)
           .last

    basename = filenames.last || parse_basename(redirect_url)

    @resolved_info_cache[url] = [redirect_url, basename, time, file_size]
  end
```

## Testing

[Testing inferring strategies from URLs](https://github.com/Homebrew/brew/blob/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/test/download_strategies/detector_spec.rb#L6-L35) is rather straightforward:

```ruby
describe DownloadStrategyDetector do
  describe "::detect" do
    subject(:strategy_detector) { described_class.detect(url, strategy) }

    let(:url) { Object.new }
    let(:strategy) { nil }

    context "when given Git URL" do
      let(:url) { "git://example.com/foo.git" }

      it { is_expected.to eq(GitDownloadStrategy) }
    end

    context "when given a GitHub Git URL" do
      let(:url) { "https://github.com/homebrew/brew.git" }

      it { is_expected.to eq(GitHubGitDownloadStrategy) }
    end

    it "defaults to curl" do
      expect(strategy_detector).to eq(CurlDownloadStrategy)
    end

    it "raises an error when passed an unrecognized strategy" do
      expect {
        described_class.detect("foo", Class.new)
      }.to raise_error(TypeError)
    end
  end
end
```

[Testing CurlDownloadStrategy](https://github.com/Homebrew/brew/blob/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/test/download_strategies/curl_spec.rb) boils down to checking that `curl` is called with correct parameters.

```ruby
describe CurlDownloadStrategy do
  subject(:strategy) { described_class.new(url, name, version, **specs) }

  let(:name) { "foo" }
  let(:url) { "https://example.com/foo.tar.gz" }
  let(:version) { "1.2.3" }
  let(:specs) { { user: "download:123456" } }

  it "parses the opts and sets the corresponding args" do
    expect(strategy.send(:_curl_args)).to eq(["--user", "download:123456"])
  end

  describe "#fetch" do
    before do
      strategy.temporary_path.dirname.mkpath
      FileUtils.touch strategy.temporary_path
    end

    it "calls curl with default arguments" do
      expect(strategy).to receive(:curl).with(
        # example.com supports partial requests.
        "--continue-at", "-",
        "--location",
        "--remote-time",
        "--output", an_instance_of(Pathname),
        url,
        an_instance_of(Hash)
      )

      strategy.fetch
    end

  # ...
end
```

[Tests for the rest of DownloadStrategies](https://github.com/Homebrew/brew/tree/09f7bc27a99469cf947431df4754737dfbadb31d/Library/Homebrew/test/download_strategies)

## Observations

- We see a use of the strategy pattern to select a downloading algorithm at runtime. `DownloadStrategy` effectively hides the underlying complexity behind a simple interface.
- Using the template method pattern streamlines strategies for different version control systems.
- It's a little surpsising that all strategies are implemented in the same file. Interestingly, test files for different strategies are different.
- The name of the default Git branch is [hard-coded to "master"](https://github.com/Homebrew/brew/blob/625d9db5f413c0174267442cb175ec0b8b5b4892/Library/Homebrew/download_strategy.rb#L783). Since that, git has changed it to "main".
- *Unit* test coverage differs per strategy. [CurlDownloadStrategy spec](https://github.com/Homebrew/brew/blob/04532cb6216b69a5b067aa7a4e22cff0944b257d/Library/Homebrew/test/download_strategies/curl_spec.rb) appears rather comprehensive, specs for git-based strategies - not so much. Perhaps there are integration tests verifying the behavior.
- In [DowloadStrategyDetector spec](https://github.com/Homebrew/brew/blob/04532cb6216b69a5b067aa7a4e22cff0944b257d/Library/Homebrew/test/download_strategies/detector_spec.rb), they did not bother to test some of the seemingly trivial scenarios like explicitly choosing a strategy, or exercising all URL patterns.
- To interact with git, they actually call the git command, rather than using a library like [ruby-git](https://github.com/ruby-git/ruby-git).

## References

* [Github Repo](https://github.com/Homebrew/brew)
* [Documentation](https://docs.brew.sh)
* [Homebrew Terminology](https://docs.brew.sh/Formula-Cookbook#homebrew-terminology)
