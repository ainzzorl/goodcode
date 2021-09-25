---
title:  "Scylla - Leader Election with Raft [C++]"
layout: default
last_modified_date: 2021-09-25T18:00:00+0300
nav_order: 20

status: PUBLISHED
language: C++
short-title: Leader Election with Raft
project:
    name: Scylla
    key: scylla
    home-page: https://github.com/scylladb/scylla
tags: ['algorithm', 'leader-election']
---

{% include article-meta.html article=page %}

## Context

*[Scylla](https://www.scylladb.com/) is the real-time big data database that is API-compatible with Apache Cassandra and Amazon DynamoDB. Scylla embraces a shared-nothing approach that increases throughput and storage capacity to realize order-of-magnitude performance improvements and reduce hardware costs.*

Scylla implements the [Raft](https://en.wikipedia.org/wiki/Raft_(algorithm)) consensus algorithm for schema changes, topology changes and tablets.

*Raft is a consensus algorithm designed as an alternative to the [Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science)) family of algorithms. It was meant to be more understandable than Paxos by means of separation of logic, but it is also formally proven safe and offers some additional features. Raft offers a generic way to distribute a state machine across a cluster of computing systems, ensuring that each node in the cluster agrees upon the same series of state transitions. It has a number of open-source reference implementations, with full-specification implementations in Go, C++, Java, and Scala. It is named after Reliable, Replicated, Redundant, And Fault-Tolerant.* - [Wikipedia](https://en.wikipedia.org/wiki/Raft_(algorithm)).

All communications with clients in Raft go through the leader node. The leader is elected by the majority of nodes in the cluster.

## Problem

How does Scylla implement Raft leader election?

The reason we focus on leader election for the scope of this article is that it's, arguably, the simplest part of Raft and that it's mostly independent from the rest. Note that, even though Raft is considered a simple algorithm (moreover, it was specifically designed to be understandable), its simplicity is relative to other distributed consensus algorithms.

## Overview

Scylla's implementation is based on the [paper](https://raft.github.io/raft.pdf) by Diego Ongaro and John Ousterhout.

The general idea is this. Each node in a cluster can be in one of the three states: `FOLLOWER`, `LEADER` or `CANDIDATE`. Assume a node is a follower. Follower nodes receive regular updates and/or heartbeats from their leader. When a node doesn't hear from the leader for a certain interval of time, it assumes that the leader has become unavailable, transitions to `CANDIDATE` and starts an election by requesting votes from its peers. If the majority of nodes in the cluster vote for it, the node becomes the new `LEADER`. If another node wins, our node becomes a `FOLLOWER`. If votes are split, it starts another vote.

Time is divided in *terms*, arbitrary periods of time on the server for which a new leader needs to be elected. When a node starts an election, it starts a new term. Raft guarantees election safety: at most one leader can be elected in a given term.

One trick employed by Scylla and not described in the original paper is the Prevote phase where a candidate ensures it *can* become a leader before starting an actual election. This does not affect the correctness, but allows preventing interruptions when a node is disconnected from the rest of the cluster and then reconnected.

Please keep in mind that the description above is a gross oversimplification. Raft guarantees only hold if the [protocol](https://raft.github.io/raft.pdf) is followed very precisely. If you try to implement Raft on your own, you will find out that misinterpreting even seemingly irrelevant details will result in incorrect behavior. See the [References](#references) section for resources to learn more about Raft.

## Implementation details

The Raft implementation in Scylla is very clear and well commented. As long as you understand the protocol, you should be able to follow the code easily. We just list the relevant code sections.

[Transitioning](https://github.com/scylladb/scylla/blob/acf8da2bcef47168376e876e2fdc37fc26ebc82e/raft/fsm.cc#L187-L279) to `CANDIDATE` and starting an election:

```c++
void fsm::become_candidate(bool is_prevote, bool is_leadership_transfer) {
    // When starting a campain we need to reset current leader otherwise
    // disruptive server prevention will stall an election if quorum of nodes
    // start election together since each one will ignore vote requests from others

    // Note that current state should be destroyed only after the new one is
    // assigned. The exchange here guarantis that.
    std::exchange(_state, candidate(_log.get_configuration(), is_prevote));

    reset_election_timeout();

    // 3.4 Leader election
    //
    // A possible outcome is that a candidate neither wins nor
    // loses the election: if many followers become candidates at
    // the same time, votes could be split so that no candidate
    // obtains a majority. When this happens, each candidate will
    // time out and start a new election by incrementing its term
    // and initiating another round of RequestVote RPCs.
    _last_election_time = _clock.now();

    auto& votes = candidate_state().votes;

    const auto& voters = votes.voters();
    if (!voters.contains(server_address{_my_id})) {
        // We're not a voter in the current configuration (perhaps we completely left it).
        //
        // But sometimes, if we're transitioning between configurations
        // such that we were a voter in the previous configuration, we may still need
        // to become a candidate: the new configuration may be unable to proceed without us.
        //
        // For example, if Cold = {A, B}, Cnew = {B}, A is a leader, switching from Cold to Cnew,
        // and Cnew wasn't yet received by B, then B won't be able to win an election:
        // B will ask A for a vote because it is still in the joint configuration
        // and A won't grant it because B has a shorter log. A is the only node
        // that can become a leader at this point.
        //
        // However we can easily determine when we don't need to become a candidate:
        // If Cnew is already committed, that means that a quorum in Cnew had to accept
        // the Cnew entry, so there is a quorum in Cnew that can proceed on their own.
        //
        // Ref. Raft PhD 4.2.2.
        if (_log.last_conf_idx() <= _commit_idx) {
            // Cnew already committed, no need to become a candidate.
            become_follower(server_id{});
            return;
        }

        // The last configuration is not committed yet.
        // This means we must still have access to the previous configuration.
        // Become a candidate only if we were previously a voter.
        auto prev_cfg = _log.get_prev_configuration();
        assert(prev_cfg);
        if (!prev_cfg->can_vote(_my_id)) {
            // We weren't a voter before.
            become_follower(server_id{});
            return;
        }
    }

    term_t term{_current_term + 1};
    if (!is_prevote) {
        update_current_term(term);
    }
    // Replicate RequestVote
    for (const auto& server : voters) {
        if (server.id == _my_id) {
            // Vote for self.
            votes.register_vote(server.id, true);
            if (!is_prevote) {
                // Only record real votes
                _voted_for = _my_id;
            }
            // Already signaled _sm_events in update_current_term()
            continue;
        }
        logger.trace("{} [term: {}, index: {}, last log term: {}{}{}] sent vote request to {}",
            _my_id, term, _log.last_idx(), _log.last_term(), is_prevote ? ", prevote" : "",
            is_leadership_transfer ? ", force" : "", server.id);

        send_to(server.id, vote_request{term, _log.last_idx(), _log.last_term(), is_prevote, is_leadership_transfer});
    }
    if (votes.tally_votes() == vote_result::WON) {
        // A single node cluster.
        if (is_prevote) {
            logger.trace("become_candidate[{}] won prevote", _my_id);
            become_candidate(false);
        } else {
            logger.trace("become_candidate[{}] won vote", _my_id);
            become_leader();
        }
    }
}
```

[Handling](https://github.com/scylladb/scylla/blob/acf8da2bcef47168376e876e2fdc37fc26ebc82e/raft/fsm.cc#L725-L778) vote requests:

```c++
void fsm::request_vote(server_id from, vote_request&& request) {

    // We can cast a vote in any state. If the candidate's term is
    // lower than ours, we ignore the request. Otherwise we first
    // update our current term and convert to a follower.
    assert(request.is_prevote || _current_term == request.current_term);

    bool can_vote =
        // We can vote if this is a repeat of a vote we've already cast...
        _voted_for == from ||
        // ...we haven't voted and we don't think there's a leader yet in this term...
        (_voted_for == server_id{} && current_leader() == server_id{}) ||
        // ...this is prevote for a future term...
        // (we will get here if the node does not know any leader yet and already
        //  voted for some other node, but now it get even newer prevote request)
        (request.is_prevote && request.current_term > _current_term);

    // ...and we believe the candidate is up to date.
    if (can_vote && _log.is_up_to_date(request.last_log_idx, request.last_log_term)) {

        logger.trace("{} [term: {}, index: {}, last log term: {}, voted_for: {}] "
            "voted for {} [log_term: {}, log_index: {}]",
            _my_id, _current_term, _log.last_idx(), _log.last_term(), _voted_for,
            from, request.last_log_term, request.last_log_idx);
        if (!request.is_prevote) { // Only record real votes
            // If a server grants a vote, it must reset its election
            // timer. See Raft Summary.
            _last_election_time = _clock.now();
            _voted_for = from;
        }
        // The term in the original message and current local term are the
        // same in the case of regular votes, but different for pre-votes.
        //
        // When responding to {Pre,}Vote messages we include the term
        // from the message, not the local term. To see why, consider the
        // case where a single node was previously partitioned away and
        // its local term is now out of date. If we include the local term
        // (recall that for pre-votes we don't update the local term), the
        // (pre-)campaigning node on the other end will proceed to ignore
        // the message (it ignores all out of date messages).
        send_to(from, vote_reply{request.current_term, true, request.is_prevote});
    } else {
        // If a vote is not granted, this server is a potential
        // viable candidate, so it should not reset its election
        // timer, to avoid election disruption by non-viable
        // candidates.
        logger.trace("{} [term: {}, index: {}, log_term: {}, voted_for: {}] "
            "rejected vote for {} [current_term: {}, log_term: {}, log_index: {}, is_prevote: {}]",
            _my_id, _current_term, _log.last_idx(), _log.last_term(), _voted_for,
            from, request.current_term, request.last_log_term, request.last_log_idx, request.is_prevote);

        send_to(from, vote_reply{_current_term, false, request.is_prevote});
    }
}
```

On [receiving](https://github.com/scylladb/scylla/blob/acf8da2bcef47168376e876e2fdc37fc26ebc82e/raft/fsm.cc#L780-L809) a vote:

```c++
void fsm::request_vote_reply(server_id from, vote_reply&& reply) {
    assert(is_candidate());

    logger.trace("request_vote_reply[{}] received a {} vote from {}", _my_id, reply.vote_granted ? "yes" : "no", from);

    auto& state = std::get<candidate>(_state);
    // Should not register a reply to prevote as a real vote
    if (state.is_prevote != reply.is_prevote) {
        logger.trace("request_vote_reply[{}] ignoring prevote from {} as state is vote", _my_id, from);
        return;
    }
    state.votes.register_vote(from, reply.vote_granted);

    switch (state.votes.tally_votes()) {
    case vote_result::UNKNOWN:
        break;
    case vote_result::WON:
        if (state.is_prevote) {
            logger.trace("request_vote_reply[{}] won prevote", _my_id);
            become_candidate(false);
        } else {
            logger.trace("request_vote_reply[{}] won vote", _my_id);
            become_leader();
        }
        break;
    case vote_result::LOST:
        become_follower(server_id{});
        break;
    }
}
```

[Counting](https://github.com/scylladb/scylla/blob/acf8da2bcef47168376e876e2fdc37fc26ebc82e/raft/tracker.hh#L189-L197) votes:

```c++
vote_result tally_votes() const {
    auto quorum = _suffrage.size() / 2 + 1;
    if (_granted >= quorum) {
        return vote_result::WON;
    }
    assert(_responded.size() <= _suffrage.size());
    auto unknown = _suffrage.size() - _responded.size();
    return _granted + unknown >= quorum ? vote_result::UNKNOWN : vote_result::LOST;
}
```

## Testing

There's a great number of various tests for the Raft implementation, including the leader election. For instance, here's a [test](https://github.com/scylladb/scylla/blob/acf8da2bcef47168376e876e2fdc37fc26ebc82e/test/raft/fsm_test.cc#L474-L517) for leader election between four nodes:

```c++
BOOST_AUTO_TEST_CASE(test_election_four_nodes) {

    discrete_failure_detector fd;

    server_id id1 = id(), id2 = id(), id3 = id(), id4 = id();

    raft::configuration cfg({id1, id2, id3, id4});
    raft::log log{raft::snapshot{.config = cfg}};

    auto fsm = create_follower(id1, std::move(log), fd);

    // Initial state is follower
    BOOST_CHECK(fsm.is_follower());

    // Inform FSM about a new leader at a new term
    fsm.step(id4, raft::append_request{term_t{1}, index_t{1}, term_t{1}});

    (void) fsm.get_output();

    // Request a vote during the same term. Even though
    // we haven't voted, we should deny a vote because we
    // know about a leader for this term.
    fsm.step(id3, raft::vote_request{term_t{1}, index_t{1}, term_t{1}});

    auto output = fsm.get_output();
    auto reply = std::get<raft::vote_reply>(output.messages.back().second);
    BOOST_CHECK(!reply.vote_granted);

    // Run out of steam for this term. Start a new one.
    fd.mark_all_dead();
    election_timeout(fsm);
    BOOST_CHECK(fsm.is_candidate());

    output = fsm.get_output();
    BOOST_CHECK(output.term_and_vote);
    auto current_term = output.term_and_vote->first;
    // Add a favourable reply, not enough for quorum
    fsm.step(id2, raft::vote_reply{current_term, true});
    BOOST_CHECK(fsm.is_candidate());

    // Add another one, this adds up to quorum
    fsm.step(id3, raft::vote_reply{current_term, true});
    BOOST_CHECK(fsm.is_leader());
}
```

## Related

Raft is implemented in many other distributed systems, e.g. [etcd](https://github.com/etcd-io/etcd) and [RethinkDB](https://github.com/rethinkdb/rethinkdb). See a long list of other Raft implementations [here](https://raft.github.io/#implementations).

## References

* [GitHub Repo](https://github.com/scylladb/scylla)
* [Scylla](https://www.scylladb.com/)
* [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf) ("The Raft Paper")
* [Raft in Scylla](https://www.youtube.com/watch?v=n_NItGXGcCs) (video)
* [Making Scylla a Monstrous Database: Introducing Project Circe](https://www.scylladb.com/2021/01/12/making-scylla-a-monstrous-database-introducing-project-circe/)
* [Project Circe February Update](https://www.scylladb.com/2021/03/01/project-circe-february-update/). Talks about what Raft is used for in Scylla.
* [Raft](https://raft.github.io/)
* [Courses teaching Raft](https://raft.github.io/#courses)
  * We highly recommend MIT [6.824: Distributed Systems](http://nil.csail.mit.edu/6.824/2020/index.html). All materials (videos, papers, textbooks, homeworks) are available online for free. Three labs are dedicated to Raft and implementing a sharded key-value storage on top of it.

## Copyright notice

Scylla is licensed under the [GNU Affero General Public License v3.0](https://github.com/scylladb/scylla/blob/master/LICENSE.AGPL).

Copyright (C) 2020-present ScyllaDB.
