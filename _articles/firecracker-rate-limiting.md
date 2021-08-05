---
title: "Firecracker - Rate Limiting [Rust]"
layout: default
last_modified_date: 2021-08-05T13:02:00+0300
nav_order: 11

status: PUBLISHED
language: Rust
project:
  name: Firecracker
  key: firecracker
  home-page: https://github.com/firecracker-microvm/firecracker
tags: ['rate-limiting', 'token-bucket']
---

{% include article-meta.html article=page %}

## Context

Firecracker is an open source virtualization technology that is purpose-built for creating and managing secure, multi-tenant container and function-based services that provide serverless operational models. Firecracker runs workloads in lightweight virtual machines, called microVMs, which combine the security and isolation properties provided by hardware virtualization technology with the speed and flexibility of containers.

## Problem

Firecracker API allows [configuring rate limiters](https://github.com/firecracker-microvm/firecracker/blob/a367796e66eeac42d9ce1294c0fbbca6191e9cf3/src/api_server/swagger/firecracker.yaml#L962-L973) for virtual devices which can limit the bandwidth, operations per second, or both.

## Overview

The implementation is based on the [Token Bucket algorithm](https://en.wikipedia.org/wiki/Token_bucket). The gist of the algorithm is that each operation consumes from a budget which is replenished with a configured rate, but cannot grow over a configured max value often referred to as *burst*. If there's no budget left, the operation is rejected.

Firecracker's implementation deviates from the classical algorithm in a few places. It adds the initial `one_time_burst`, which doesn't replenish. It also allows consuming more tokens than the size of the bucket, but it will force the client to wait accordingly - search the code for `OverConsumption`.

There are two independent buckets: operations and bandwidth. The buckets are *passively* replenished as they're being used. A timer is enabled and used to *actively* replenish the token buckets when limiting is in effect and `consume()` operations are disabled.

There are two main structs: [`TokenBucket`](https://github.com/firecracker-microvm/firecracker/blob/2f92f4a2d189c504aa358928b3ad490aafa61cff/src/rate_limiter/src/lib.rs#L93-L113), which basically just counts tokens, and [`RateLimiter`](https://github.com/firecracker-microvm/firecracker/blob/2f92f4a2d189c504aa358928b3ad490aafa61cff/src/rate_limiter/src/lib.rs#L286-L293), which serves as the primary user interface, routes requests to the right bucket and handles events.

The implementation [leans more on efficiency at the expense of accuracy](https://github.com/firecracker-microvm/firecracker/commit/3aa643406bf6d715f17e7113f2e19f804174b386) and avoids floating point operations.

[Doc comment](https://github.com/firecracker-microvm/firecracker/blob/2f92f4a2d189c504aa358928b3ad490aafa61cff/src/rate_limiter/src/lib.rs#L5-L44):

```rust
//! # Rate Limiter
//!
//! Provides a rate limiter written in Rust useful for IO operations that need to
//! be throttled.
//!
//! ## Behavior
//!
//! The rate limiter starts off as 'unblocked' with two token buckets configured
//! with the values passed in the `RateLimiter::new()` constructor.
//! All subsequent accounting is done independently for each token bucket based
//! on the `TokenType` used. If any of the buckets runs out of budget, the limiter
//! goes in the 'blocked' state. At this point an internal timer is set up which
//! will later 'wake up' the user in order to retry sending data. The 'wake up'
//! notification will be dispatched as an event on the FD provided by the `AsRawFD`
//! trait implementation.
//!
//! The contract is that the user shall also call the `event_handler()` method on
//! receipt of such an event.
//!
//! The token buckets are replenished when a called `consume()` doesn't find enough
//! tokens in the bucket. The amount of tokens replenished is automatically calculated
//! to respect the `complete_refill_time` configuration parameter provided by the user.
//! The token buckets will never replenish above their respective `size`.
//!
//! Each token bucket can start off with a `one_time_burst` initial extra capacity
//! on top of their `size`. This initial extra credit does not replenish and
//! can be used for an initial burst of data.
//!
//! The granularity for 'wake up' events when the rate limiter is blocked is
//! currently hardcoded to `100 milliseconds`.
//!
//! ## Limitations
//!
//! This rate limiter implementation relies on the *Linux kernel's timerfd* so its
//! usage is limited to Linux systems.
//!
//! Another particularity of this implementation is that it is not self-driving.
//! It is meant to be used in an external event loop and thus implements the `AsRawFd`
//! trait and provides an *event-handler* as part of its API. This *event-handler*
//! needs to be called by the user on every event on the rate limiter's `AsRawFd` FD.
```

## Implementation details

[The implementation](https://github.com/firecracker-microvm/firecracker/blob/2f92f4a2d189c504aa358928b3ad490aafa61cff/src/rate_limiter/src/lib.rs) is documented so extensively that we see no reason to explain it any further. We will just list it as-is, with some boilerplate excluded:

```rust
/// Enum describing the outcomes of a `reduce()` call on a `TokenBucket`.
#[derive(Clone, Debug, PartialEq)]
pub enum BucketReduction {
    /// There are not enough tokens to complete the operation.
    Failure,
    /// A part of the available tokens have been consumed.
    Success,
    /// A number of tokens `inner` times larger than the bucket size have been consumed.
    OverConsumption(f64),
}

/// TokenBucket provides a lower level interface to rate limiting with a
/// configurable capacity, refill-rate and initial burst.
#[derive(Clone, Debug, PartialEq)]
pub struct TokenBucket {
    // Bucket defining traits.
    size: u64,
    // Initial burst size.
    initial_one_time_burst: u64,
    // Complete refill time in milliseconds.
    refill_time: u64,

    // Internal state descriptors.

    // Number of free initial tokens, that can be consumed at no cost.
    one_time_burst: u64,
    // Current token budget.
    budget: u64,
    // Last time this token bucket saw activity.
    last_update: Instant,

    // Fields used for pre-processing optimizations.
    processed_capacity: u64,
    processed_refill_time: u64,
}

impl TokenBucket {
    /// Creates a `TokenBucket` wrapped in an `Option`.
    ///
    /// TokenBucket created is of `size` total capacity and takes `complete_refill_time_ms`
    /// milliseconds to go from zero tokens to total capacity. The `one_time_burst` is initial
    /// extra credit on top of total capacity, that does not replenish and which can be used
    /// for an initial burst of data.
    ///
    /// If the `size` or the `complete refill time` are zero, then `None` is returned.
    pub fn new(size: u64, one_time_burst: u64, complete_refill_time_ms: u64) -> Option<Self> {
        // If either token bucket capacity or refill time is 0, disable limiting.
        if size == 0 || complete_refill_time_ms == 0 {
            return None;
        }
        // Formula for computing current refill amount:
        // refill_token_count = (delta_time * size) / (complete_refill_time_ms * 1_000_000)
        // In order to avoid overflows, simplify the fractions by computing greatest common divisor.

        let complete_refill_time_ns = complete_refill_time_ms * NANOSEC_IN_ONE_MILLISEC;
        // Get the greatest common factor between `size` and `complete_refill_time_ns`.
        let common_factor = gcd(size, complete_refill_time_ns);
        // The division will be exact since `common_factor` is a factor of `size`.
        let processed_capacity: u64 = size / common_factor;
        // The division will be exact since `common_factor` is a factor of `complete_refill_time_ns`.
        let processed_refill_time: u64 = complete_refill_time_ns / common_factor;

        Some(TokenBucket {
            size,
            one_time_burst,
            initial_one_time_burst: one_time_burst,
            refill_time: complete_refill_time_ms,
            // Start off full.
            budget: size,
            // Last updated is now.
            last_update: Instant::now(),
            processed_capacity,
            processed_refill_time,
        })
    }

    // Replenishes token bucket based on elapsed time. Should only be called internally by `Self`.
    fn auto_replenish(&mut self) {
        // Compute time passed since last refill/update.
        let time_delta = self.last_update.elapsed().as_nanos() as u64;
        self.last_update = Instant::now();

        // At each 'time_delta' nanoseconds the bucket should refill with:
        // refill_amount = (time_delta * size) / (complete_refill_time_ms * 1_000_000)
        // `processed_capacity` and `processed_refill_time` are the result of simplifying above
        // fraction formula with their greatest-common-factor.
        let tokens = (time_delta * self.processed_capacity) / self.processed_refill_time;
        self.budget = std::cmp::min(self.budget + tokens, self.size);
    }

    /// Attempts to consume `tokens` from the bucket and returns whether the action succeeded.
    pub fn reduce(&mut self, mut tokens: u64) -> BucketReduction {
        // First things first: consume the one-time-burst budget.
        if self.one_time_burst > 0 {
            // We still have burst budget for *all* tokens requests.
            if self.one_time_burst >= tokens {
                self.one_time_burst -= tokens;
                self.last_update = Instant::now();
                // No need to continue to the refill process, we still have burst budget to consume from.
                return BucketReduction::Success;
            } else {
                // We still have burst budget for *some* of the tokens requests.
                // The tokens left unfulfilled will be consumed from current `self.budget`.
                tokens -= self.one_time_burst;
                self.one_time_burst = 0;
            }
        }

        if tokens > self.budget {
            // Hit the bucket bottom, let's auto-replenish and try again.
            self.auto_replenish();

            // This operation requests a bandwidth higher than the bucket size
            if tokens > self.size {
                error!(
                    "Consumed {} tokens from bucket of size {}",
                    tokens, self.size
                );
                // Empty the bucket and report an overconsumption of
                // (remaining tokens / size) times larger than the bucket size
                tokens -= self.budget;
                self.budget = 0;
                return BucketReduction::OverConsumption(tokens as f64 / self.size as f64);
            }

            if tokens > self.budget {
                // Still not enough tokens, consume() fails, return false.
                return BucketReduction::Failure;
            }
        }

        self.budget -= tokens;
        BucketReduction::Success
    }

    /// "Manually" adds tokens to bucket.
    pub fn force_replenish(&mut self, tokens: u64) {
        // This means we are still during the burst interval.
        // Of course there is a very small chance  that the last reduce() also used up burst
        // budget which should now be replenished, but for performance and code-complexity
        // reasons we're just gonna let that slide since it's practically inconsequential.
        if self.one_time_burst > 0 {
            self.one_time_burst += tokens;
            return;
        }
        self.budget = std::cmp::min(self.budget + tokens, self.size);
    }

    // Getters
    // ...
}

/// Enum that describes the type of token used.
pub enum TokenType {
    /// Token type used for bandwidth limiting.
    Bytes,
    /// Token type used for operations/second limiting.
    Ops,
}

/// Enum that describes the type of token bucket update.
pub enum BucketUpdate {
    /// No Update - same as before.
    None,
    /// Rate Limiting is disabled on this bucket.
    Disabled,
    /// Rate Limiting enabled with updated bucket.
    Update(TokenBucket),
}

/// Rate Limiter that works on both bandwidth and ops/s limiting.
///
/// Bandwidth (bytes/s) and ops/s limiting can be used at the same time or individually.
///
/// Implementation uses a single timer through TimerFd to refresh either or
/// both token buckets.
///
/// Its internal buckets are 'passively' replenished as they're being used (as
/// part of `consume()` operations).
/// A timer is enabled and used to 'actively' replenish the token buckets when
/// limiting is in effect and `consume()` operations are disabled.
///
/// RateLimiters will generate events on the FDs provided by their `AsRawFd` trait
/// implementation. These events are meant to be consumed by the user of this struct.
/// On each such event, the user must call the `event_handler()` method.
pub struct RateLimiter {
    bandwidth: Option<TokenBucket>,
    ops: Option<TokenBucket>,

    timer_fd: TimerFd,
    // Internal flag that quickly determines timer state.
    timer_active: bool,
}

// ParticalEq, fmt::Debug
// ...

impl RateLimiter {
    /// Creates a new Rate Limiter that can limit on both bytes/s and ops/s.
    ///
    /// # Arguments
    ///
    /// * `bytes_total_capacity` - the total capacity of the `TokenType::Bytes` token bucket.
    /// * `bytes_one_time_burst` - initial extra credit on top of `bytes_total_capacity`,
    /// that does not replenish and which can be used for an initial burst of data.
    /// * `bytes_complete_refill_time_ms` - number of milliseconds for the `TokenType::Bytes`
    /// token bucket to go from zero Bytes to `bytes_total_capacity` Bytes.
    /// * `ops_total_capacity` - the total capacity of the `TokenType::Ops` token bucket.
    /// * `ops_one_time_burst` - initial extra credit on top of `ops_total_capacity`,
    /// that does not replenish and which can be used for an initial burst of data.
    /// * `ops_complete_refill_time_ms` - number of milliseconds for the `TokenType::Ops` token
    /// bucket to go from zero Ops to `ops_total_capacity` Ops.
    ///
    /// If either bytes/ops *size* or *refill_time* are **zero**, the limiter
    /// is **disabled** for that respective token type.
    ///
    /// # Errors
    ///
    /// If the timerfd creation fails, an error is returned.
    pub fn new(
        bytes_total_capacity: u64,
        bytes_one_time_burst: u64,
        bytes_complete_refill_time_ms: u64,
        ops_total_capacity: u64,
        ops_one_time_burst: u64,
        ops_complete_refill_time_ms: u64,
    ) -> io::Result<Self> {
        let bytes_token_bucket = TokenBucket::new(
            bytes_total_capacity,
            bytes_one_time_burst,
            bytes_complete_refill_time_ms,
        );

        let ops_token_bucket = TokenBucket::new(
            ops_total_capacity,
            ops_one_time_burst,
            ops_complete_refill_time_ms,
        );

        // We'll need a timer_fd, even if our current config effectively disables rate limiting,
        // because `Self::update_buckets()` might re-enable it later, and we might be
        // seccomp-blocked from creating the timer_fd at that time.
        let timer_fd = TimerFd::new_custom(ClockId::Monotonic, true, true)?;

        Ok(RateLimiter {
            bandwidth: bytes_token_bucket,
            ops: ops_token_bucket,
            timer_fd,
            timer_active: false,
        })
    }

    // Arm the timer of the rate limiter with the provided `TimerState`.
    fn activate_timer(&mut self, timer_state: TimerState) {
        // Register the timer; don't care about its previous state
        self.timer_fd.set_state(timer_state, SetTimeFlags::Default);
        self.timer_active = true;
    }

    /// Attempts to consume tokens and returns whether that is possible.
    ///
    /// If rate limiting is disabled on provided `token_type`, this function will always succeed.
    pub fn consume(&mut self, tokens: u64, token_type: TokenType) -> bool {
        // If the timer is active, we can't consume tokens from any bucket and the function fails.
        if self.timer_active {
            return false;
        }

        // Identify the required token bucket.
        let token_bucket = match token_type {
            TokenType::Bytes => self.bandwidth.as_mut(),
            TokenType::Ops => self.ops.as_mut(),
        };
        // Try to consume from the token bucket.
        if let Some(bucket) = token_bucket {
            let refill_time = bucket.refill_time_ms();
            match bucket.reduce(tokens) {
                // When we report budget is over, there will be no further calls here,
                // register a timer to replenish the bucket and resume processing;
                // make sure there is only one running timer for this limiter.
                BucketReduction::Failure => {
                    if !self.timer_active {
                        self.activate_timer(TIMER_REFILL_STATE);
                    }
                    false
                }
                // The operation succeeded and further calls can be made.
                BucketReduction::Success => true,
                // The operation succeeded as the tokens have been consumed
                // but the timer still needs to be armed.
                BucketReduction::OverConsumption(ratio) => {
                    // The operation "borrowed" a number of tokens `ratio` times
                    // greater than the size of the bucket, and since it takes
                    // `refill_time` milliseconds to fill an empty bucket, in
                    // order to enforce the bandwidth limit we need to prevent
                    // further calls to the rate limiter for
                    // `ratio * refill_time` milliseconds.
                    self.activate_timer(TimerState::Oneshot(Duration::from_millis(
                        (ratio * refill_time as f64) as u64,
                    )));
                    true
                }
            }
        } else {
            // If bucket is not present rate limiting is disabled on token type,
            // consume() will always succeed.
            true
        }
    }

    /// Adds tokens of `token_type` to their respective bucket.
    ///
    /// Can be used to *manually* add tokens to a bucket. Useful for reverting a
    /// `consume()` if needed.
    pub fn manual_replenish(&mut self, tokens: u64, token_type: TokenType) {
        // Identify the required token bucket.
        let token_bucket = match token_type {
            TokenType::Bytes => self.bandwidth.as_mut(),
            TokenType::Ops => self.ops.as_mut(),
        };
        // Add tokens to the token bucket.
        if let Some(bucket) = token_bucket {
            bucket.force_replenish(tokens);
        }
    }

    /// Returns whether this rate limiter is blocked.
    ///
    /// The limiter 'blocks' when a `consume()` operation fails because there was not enough
    /// budget for it.
    /// An event will be generated on the exported FD when the limiter 'unblocks'.
    pub fn is_blocked(&self) -> bool {
        self.timer_active
    }

    /// This function needs to be called every time there is an event on the
    /// FD provided by this object's `AsRawFd` trait implementation.
    ///
    /// # Errors
    ///
    /// If the rate limiter is disabled or is not blocked, an error is returned.
    pub fn event_handler(&mut self) -> Result<(), Error> {
        match self.timer_fd.read() {
            0 => Err(Error::SpuriousRateLimiterEvent(
                "Rate limiter event handler called without a present timer",
            )),
            _ => {
                self.timer_active = false;
                Ok(())
            }
        }
    }

    /// Updates the parameters of the token buckets associated with this RateLimiter.
    // TODO: Please note that, right now, the buckets become full after being updated.
    pub fn update_buckets(&mut self, bytes: BucketUpdate, ops: BucketUpdate) {
        match bytes {
            BucketUpdate::Disabled => self.bandwidth = None,
            BucketUpdate::Update(tb) => self.bandwidth = Some(tb),
            BucketUpdate::None => (),
        };
        match ops {
            BucketUpdate::Disabled => self.ops = None,
            BucketUpdate::Update(tb) => self.ops = Some(tb),
            BucketUpdate::None => (),
        };
    }

    /// Returns an immutable view of the inner bandwidth token bucket.
    pub fn bandwidth(&self) -> Option<&TokenBucket> {
        self.bandwidth.as_ref()
    }

    /// Returns an immutable view of the inner ops token bucket.
    pub fn ops(&self) -> Option<&TokenBucket> {
        self.ops.as_ref()
    }
}

// AsRawFd, Default
// ...
```

## Testing

The code is [thoroughly tested](https://github.com/firecracker-microvm/firecracker/blob/2f92f4a2d189c504aa358928b3ad490aafa61cff/src/rate_limiter/src/lib.rs#L556-L882).

One of the tests, for [rate-limiting bandwidth](https://github.com/firecracker-microvm/firecracker/blob/2f92f4a2d189c504aa358928b3ad490aafa61cff/src/rate_limiter/src/lib.rs#L680-L711):

```rust
#[test]
fn test_rate_limiter_bandwidth() {
    // rate limiter with limit of 1000 bytes/s
    let mut l = RateLimiter::new(1000, 0, 1000, 0, 0, 0).unwrap();

    // limiter should not be blocked
    assert!(!l.is_blocked());
    // raw FD for this disabled should be valid
    assert!(l.as_raw_fd() > 0);

    // ops/s limiter should be disabled so consume(whatever) should work
    assert!(l.consume(u64::max_value(), TokenType::Ops));

    // do full 1000 bytes
    assert!(l.consume(1000, TokenType::Bytes));
    // try and fail on another 100
    assert!(!l.consume(100, TokenType::Bytes));
    // since consume failed, limiter should be blocked now
    assert!(l.is_blocked());
    // wait half the timer period
    thread::sleep(Duration::from_millis(REFILL_TIMER_INTERVAL_MS / 2));
    // limiter should still be blocked
    assert!(l.is_blocked());
    // wait the other half of the timer period
    thread::sleep(Duration::from_millis(REFILL_TIMER_INTERVAL_MS / 2));
    // the timer_fd should have an event on it by now
    assert!(l.event_handler().is_ok());
    // limiter should now be unblocked
    assert!(!l.is_blocked());
    // try and succeed on another 100 bytes this time
    assert!(l.consume(100, TokenType::Bytes));
}
```

When testing code that depends on the clock, it's common to use a fake clock. The downside of it is that it often complicates the code under test, but on the upside it usually simplifies the test and makes it more reliable.

Here no fake clock is used. Instead, when the test needs some time to pass, it just sleeps. It's unclear if it makes the test slow and/or flaky.

## Related

* [`RateLimiter` in Apache Lucene](https://github.com/apache/lucene/blob/acf45d8a315f94c4bf685458faa1aae24c1e8599/lucene/core/src/java/org/apache/lucene/store/RateLimiter.java)

## References

* [GitHub repo](https://github.com/firecracker-microvm/firecracker)
* [Firecracker](https://firecracker-microvm.github.io/)
* [Token Bucket](https://en.wikipedia.org/wiki/Token_bucket)

## Copyright notice

Firecracker is licensed under the [Apache-2.0 License](https://github.com/firecracker-microvm/firecracker/blob/main/LICENSE).

Copyright 2018 Amazon.com, Inc. or its affiliates. All Rights Reserved.
