---
title:  "Puppet - HTTP Connection Pool [Ruby]"
layout: default
last_modified_date: 2021-08-10T12:10:00+0300
nav_order: 13

status: PUBLISHED
language: Ruby
short-title: HTTP Connection Pool
project:
    name: Puppet
    key: puppet
    home-page: https://github.com/puppetlabs/puppet
tags: ['connection-pool']
---

{% include article-meta.html article=page %}

## Context

Puppet, an automated administrative engine for your Linux, Unix, and Windows systems, performs administrative tasks (such as adding users, installing packages, and updating server configurations) based on a centralized specification.

Puppet usually [follows client-server architecture](https://en.wikipedia.org/wiki/Puppet_(software)#Architecture). The client is known as an agent and the server is known as the master. For testing and simple configuration, it can also be used as a stand-alone application run from the command line.

Puppet Server is installed on one or more servers, and Puppet Agent is installed on all the machines that the user wants to manage. Puppet Agents communicate with the server and fetch configuration instructions. The Agent then applies the configuration on the system and sends a status report to the server.

[Persistent HTTP connections](https://en.wikipedia.org/wiki/HTTP_persistent_connection) allow Puppet to establish an HTTP(S) connection once and reuse it for multiple HTTP requests. This avoids making a new TCP connection and SSL handshake for each request.

## Problem

Puppet needs to maintain a pool of persistent connections, keeping track of when idle connections expire.

## Overview

Puppet's [Puppet::HTTP::Pool](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/lib/puppet/http/pool.rb) implements the [connection pool](https://en.wikipedia.org/wiki/Pool_(computer_science)) pattern.

Connections are borrowed from the pool, yielded to the caller, and released back into the pool. If a connection is expired, it will be closed either when a connection to that site is requested, or when the pool is closed. The pool can store multiple connections to the same site, and will be reused in MRU (Most Recently Used) order.

The pool delegates connection creation to a factory class, which configures SSL settings, timeouts and retries.

## Implementation details

The key method [`with_connection`](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/lib/puppet/http/pool.rb#L19-L39) follows an idiomatic Ruby pattern by accepting a block of code and passing the connection to it.

It borrows the connection and, after it's used, releases it back or closes it.

```ruby
  def with_connection(site, verifier, &block)
    reuse = true

    http = borrow(site, verifier)
    begin
      if http.use_ssl? && http.verify_mode != OpenSSL::SSL::VERIFY_PEER
        reuse = false
      end

      yield http
    rescue => detail
      reuse = false
      raise detail
    ensure
      if reuse && http.started?
        release(site, verifier, http)
      else
        close_connection(site, http)
      end
    end
  end
```

[Borrowing](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/lib/puppet/http/pool.rb#L88-L111) a connection:

```ruby
  # Borrow and take ownership of a persistent connection. If a new
  # connection is created, it will be started prior to being returned.
  #
  # @api private
  def borrow(site, verifier)
    @pool[site] = active_entries(site)
    index = @pool[site].index do |entry|
      (verifier.nil? && entry.verifier.nil?) ||
        (!verifier.nil? && verifier.reusable?(entry.verifier))
    end
    entry = index ? @pool[site].delete_at(index) : nil
    if entry
      @pool.delete(site) if @pool[site].empty?

      Puppet.debug("Using cached connection for #{site}")
      entry.connection
    else
      http = @factory.create_connection(site)

      start(site, verifier, http)
      setsockopts(http.instance_variable_get(:@socket))
      http
    end
  end
```

[Releasing a connection](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/lib/puppet/http/pool.rb#L123-L137):

```ruby
  # Release a connection back into the pool.
  #
  # @api private
  def release(site, verifier, http)
    expiration = Time.now + @keepalive_timeout
    entry = Puppet::HTTP::PoolEntry.new(http, verifier, expiration)
    Puppet.debug("Caching connection for #{site}")

    entries = @pool[site]
    if entries
      entries.unshift(entry)
    else
      @pool[site] = [entry]
    end
  end
```

[Expirations are checked](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/lib/puppet/http/pool.rb#L139-L154) when polling active connections:

```ruby
  # Returns an Array of entries whose connections are not expired.
  #
  # @api private
  def active_entries(site)
    now = Time.now

    entries = @pool[site] || []
    entries.select do |entry|
      if entry.expired?(now)
        close_connection(site, entry.connection)
        false
      else
        true
      end
    end
  end
```

Creating connections is delegated to [Puppet::HTTP::Factory](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/lib/puppet/http/factory.rb#L11):

```ruby
  def create_connection(site)
    Puppet.debug("Creating new connection for #{site}")

    http = Puppet::HTTP::Proxy.proxy(URI(site.addr))
    http.use_ssl = site.use_ssl?
    if site.use_ssl?
      http.min_version = OpenSSL::SSL::TLS1_VERSION if http.respond_to?(:min_version)
      http.ciphers = Puppet[:ciphers]
    end
    http.read_timeout = Puppet[:http_read_timeout]
    http.open_timeout = Puppet[:http_connect_timeout]
    http.keep_alive_timeout = KEEP_ALIVE_TIMEOUT if http.respond_to?(:keep_alive_timeout=)

    # 0 means make one request and never retry
    http.max_retries = 0

    if Puppet[:sourceaddress]
      Puppet.debug("Using source IP #{Puppet[:sourceaddress]}")
      http.local_host = Puppet[:sourceaddress]
    end

    if Puppet[:http_debug]
      http.set_debug_output($stderr)
    end

    http
  end
```

## Testing

There's a comprehensive [suite](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/spec/unit/http/pool_spec.rb) of unit tests for the connection pool.

Example: [multiple connections to the same site](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/spec/unit/http/pool_spec.rb#L83-L95).

```ruby
    it 'can yield multiple connections to the same site' do
      lru_conn = create_connection(site)
      mru_conn = create_connection(site)
      pool = create_pool_with_connections(site, lru_conn, mru_conn)

      pool.with_connection(site, verifier) do |a|
        expect(a).to eq(mru_conn)

        pool.with_connection(site, verifier) do |b|
          expect(b).to eq(lru_conn)
        end
      end
    end
```

The trick to [test expirations](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/spec/unit/http/pool_spec.rb#L83-L95) without depending on the clock is to use negative timeouts:

```ruby
  def create_pool_with_expired_connections(site, *connections)
    # setting keepalive timeout to -1 ensures any newly added
    # connections have already expired
    pool = Puppet::HTTP::Pool.new(-1)
    connections.each do |conn|
      pool.release(site, verifier, conn)
    end
    pool
  end
```

## Related

* [Generic connection pool](https://github.com/mperham/connection_pool/blob/c5aef742642def23664c4d9c15d12f0786347fb8/lib/connection_pool.rb) in [connection_pool](https://github.com/mperham/connection_pool) library for Ruby.
* [Connection pooling](https://github.com/rails/rails/blob/83217025a171593547d1268651b446d3533e2019/activerecord/lib/active_record/connection_adapters/abstract/connection_pool.rb) in [Rails](https://rubyonrails.org/), a server-side web application framework.

## References

* [GitHub Repo](https://github.com/puppetlabs/puppet)
* [Puppet Website](https://puppet.com/)
* [Puppet's HTTP Client Design](https://github.com/puppetlabs/puppet/blob/8ed2564a3b0978ce0880af904df56d79637c15e8/docs/http.md)

## Copyright notice

Puppet is licensed under the [Apache-2.0 License](https://github.com/puppetlabs/puppet/blob/main/LICENSE).

Copyright (c) 2011 Puppet Inc.
