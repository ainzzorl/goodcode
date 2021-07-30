---
title:  "Chaos Monkey - MySQL-backed Store for Schedules and Terminations [Go]"
layout: default
last_modified_date: 2021-07-30T12:25:00+0300
nav_order: 4

status: PUBLISHED
language: Go
project:
  name: Chaos Monkey
  key: chaos-monkey
  home-page: https://github.com/Netflix/chaosmonkey
tags: [data-access, sql, dao, chaos-engineering]
---

{% include article-meta.html article=page %}

## Context

Chaos Monkey randomly terminates virtual machine instances and containers that run inside of your production environment. Exposing engineers to failures more frequently incentivizes them to build resilient services. Chaos Monkey is an example of a tool that follows the [Principles of Chaos Engineering](https://principlesofchaos.org/).

## Problem

Chaos Monkey stores performed instance terminations and termination schedules in MySQL. The persistence logic should be separated from the rest of the application.

## Overview

The persistence logic resides in [`MySQL` struct](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L40-L42) and its methods. This is a variation of the [Data Access Object (DAO)](https://www.oracle.com/java/technologies/dataaccessobject.html) pattern. It provides a convenient interface to the application and hides the details of the persistence logic.

Public methods:
* `func New(host string, port int, user string, password string, dbname string) (MySQL, error)`
* `func (m MySQL) Retrieve(date time.Time) (sched *schedule.Schedule, err error)`
* `func (m MySQL) Publish(date time.Time, sched *schedule.Schedule) error`
* `func (m MySQL) PublishWithDelay(date time.Time, sched *schedule.Schedule, delay time.Duration) (err error)`
* `func (m MySQL) Check(term chaosmonkey.Termination, appCfg chaosmonkey.AppConfig, endHour int, loc *time.Location) error`
* `func (m MySQL) CheckWithDelay(term chaosmonkey.Termination, appCfg chaosmonkey.AppConfig, endHour int, loc *time.Location, delay time.Duration) error`
* `func Migrate(mysqlDb MySQL) error`

## Implementation details

[Structure definition](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L40-L42):
```go
// MySQL represents a MySQL-backed store for schedules and terminations
type MySQL struct {
	db *sql.DB
}
```

[Retrieving the schedule for a given date](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L109-L143):
```go
// Retrieve  retrieves the schedule for the given date
func (m MySQL) Retrieve(date time.Time) (sched *schedule.Schedule, err error) {
	rows, err := m.db.Query("SELECT time, app, account, region, stack, cluster FROM schedules WHERE date = DATE(?)", utcDate(date))
	if err != nil {
		return nil, errors.Wrapf(err, "failed to retrieve schedule for %s", date)
	}

	sched = schedule.New()

	defer func() {
		if cerr := rows.Close(); cerr != nil && err == nil {
			err = errors.Wrap(cerr, "rows.Close() failed")
		}
	}()

	for rows.Next() {
		var tm time.Time
		var app, account, region, stack, cluster string

		err = rows.Scan(&tm, &app, &account, &region, &stack, &cluster)
		if err != nil {
			return nil, errors.Wrap(err, "failed to scan row")
		}

		sched.Add(tm, grp.New(app, account, region, stack, cluster))
	}

	err = rows.Err()
	if err != nil {
		return nil, errors.Wrap(err, "rows.Err() errored")
	}

	return sched, nil

}
```

[Publishing a schedule](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L145-L212). Note the delay for testing race conditions.

```go
// Publish publishes the schedule for the given date
func (m MySQL) Publish(date time.Time, sched *schedule.Schedule) error {
	return m.PublishWithDelay(date, sched, 0)
}

// PublishWithDelay publishes the schedule with a delay between checking the schedule
// exists and writing it. The delay is used only for testing race conditions
func (m MySQL) PublishWithDelay(date time.Time, sched *schedule.Schedule, delay time.Duration) (err error) {
	// First, we check to see if there is a schedule present
	tx, err := m.db.Begin()
	if err != nil {
		return errors.Wrap(err, "failed to begin transaction")
	}

	// We must either commit or rollback at the end
	defer func() {
		switch err {
		case nil:
			err = tx.Commit()
		case schedstore.ErrAlreadyExists:
			// We want to return ErrAlreadyExists even if the transaction commit
			// fails
			_ = tx.Commit()
		default:
			_ = tx.Rollback()
		}
	}()

	exists, err := schedExists(tx, date)
	if err != nil {
		return err
	}

	if exists {
		return schedstore.ErrAlreadyExists
	}

	if delay > 0 {
		time.Sleep(delay)
	}
	query := "INSERT INTO schedules (date, time, app, account, region, stack, cluster) VALUES (?, ?, ?, ?, ?, ?, ?)"
	stmt, err := tx.Prepare(query)
	if err != nil {
		return errors.Wrapf(err, "failed to prepare sql statement: %s", query)
	}

	for _, entry := range sched.Entries() {
		var app, account, region, stack, cluster string
		app = entry.Group.App()
		account = entry.Group.Account()
		if val, ok := entry.Group.Region(); ok {
			region = val
		}
		if val, ok := entry.Group.Stack(); ok {
			stack = val
		}
		if val, ok := entry.Group.Cluster(); ok {
			cluster = val
		}

		_, err = stmt.Exec(utcDate(date), entry.Time.In(time.UTC), app, account, region, stack, cluster)
		if err != nil {
			return errors.Wrapf(err, "failed to execute prepared query")
		}
	}

	return nil
}
```

[Checking if a termination is permitted](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L262-L297):
```go
/ Check checks if a termination is permitted and, if so, records the
// termination time on the server
func (m MySQL) Check(term chaosmonkey.Termination, appCfg chaosmonkey.AppConfig, endHour int, loc *time.Location) error {
	return m.CheckWithDelay(term, appCfg, endHour, loc, 0)
}

// CheckWithDelay is the same as Check, but adds a delay between reading and
// writing to the database (used for testing only)
func (m MySQL) CheckWithDelay(term chaosmonkey.Termination, appCfg chaosmonkey.AppConfig, endHour int, loc *time.Location, delay time.Duration) error {
	tx, err := m.db.Begin()
	if err != nil {
		return errors.Wrap(err, "failed to begin transaction")
	}

	defer func() {
		switch err {
		case nil:
			err = tx.Commit()
		default:
			_ = tx.Rollback()
		}
	}()

	err = respectsMinTimeBetweenKills(tx, term.Time, term, appCfg, endHour, loc)
	if err != nil {
		return err
	}

	if delay > 0 {
		time.Sleep(delay)
	}

	err = recordTermination(tx, term, loc)
	return err

}
```

## Testing

[Basic happy test](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/schedstore_test.go#L35-L79):

```go
// Test we can publish and then retrieve a schedule
func TestPublishRetrieve(t *testing.T) {
	err := initDB()
	if err != nil {
		t.Fatal(err)
	}

	m, err := mysql.New("localhost", port, "root", password, "chaosmonkey")
	if err != nil {
		t.Fatal(err)
	}

	loc, err := time.LoadLocation("America/Los_Angeles")
	if err != nil {
		t.Fatal(err)
	}

	sched := schedule.New()

	t1 := time.Date(2016, time.June, 20, 11, 40, 0, 0, loc)
	sched.Add(t1, grp.New("chaosguineapig", "test", "us-east-1", "", "chaosguineapig-test"))

	date := time.Date(2016, time.June, 20, 0, 0, 0, 0, loc)

	// Code under test:
	err = m.Publish(date, sched)
	if err != nil {
		t.Fatal(err)
	}
	sched, err = m.Retrieve(date)
	if err != nil {
		t.Fatal(err)
	}

	entries := sched.Entries()
	if got, want := len(entries), 1; got != want {
		t.Fatalf("got len(entries)=%d, want %d", got, want)
	}

	entry := entries[0]

	if !t1.Equal(entry.Time) {
		t.Errorf("%s != %s", t1, entry.Time)
	}
}
```

[Testing for race conditions](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/schedstore_test.go#L185-L253). Note that Go's [built-in race detector](https://golang.org/doc/articles/race_detector) wouldn't catch race conditions like this.

```go
func TestScheduleAlreadyExistsConcurrency(t *testing.T) {
    // ...

	// Try to publish the schedule twice. At least one schedule should return an
	// error
	ch := make(chan error, 2)

	date := time.Date(2016, time.June, 20, 0, 0, 0, 0, loc)

	go func() {
		ch <- m.PublishWithDelay(date, psched1, 3*time.Second)
	}()

	go func() {
		ch <- m.PublishWithDelay(date, psched2, 0)
	}()

	// Retrieve the two error values from the two calls

	var success int
	var txDeadlock int
	for i := 0; i < 2; i++ {
		err := <-ch
		switch {
		case err == nil:
			success++
		case mysql.TxDeadlock(err):
			txDeadlock++
		default:
			t.Fatalf("Unexpected error: %+v", err)
		}
	}

	if got, want := success, 1; got != want {
		t.Errorf("got %d succeses, want: %d", got, want)
	}

	// Should cause a deadlock
	if got, want := txDeadlock, 1; got != want {
		t.Errorf("got %d txDeadlock, want: %d", got, want)
	}
}
```

## Observations

* It's common for DAOs to try to abstract away the nature of the storage as much as possible. Here, while the interface does not give away the nature of the storage, the name `MySQL` does.
* The name of the [`Check`](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L262-L266) method seems to imply that it's side-effect free, but it actually records the termination time.
* It could run a little faster if [statements were prepared](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L185-L186) during the [initialization](https://github.com/Netflix/chaosmonkey/blob/c16d769a82bb765f6544627ef6f08305791e8895/mysql/mysql.go#L85-L93). Read more about [using prepared statements in Go](https://golang.org/doc/database/prepared-statements).

## Related

DAO implementations are [numerous on GitHub](https://github.com/search?q=%22data+access+object%22).

## References

* [GitHub repo](https://github.com/Netflix/chaosmonkey)
* [Documentation](https://netflix.github.io/chaosmonkey/)
* [Chaos Engineering](https://en.wikipedia.org/wiki/Chaos_engineering)
* [Principles of Chaos Engineering](https://principlesofchaos.org/)

## Copyright notice

Bat is licensed under the [Apache License 2.0](https://github.com/Netflix/chaosmonkey/blob/master/LICENSE).

Copyright 2015 Netflix, Inc.
