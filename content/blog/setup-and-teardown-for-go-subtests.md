+++
draft = false
title = "Setup and teardown for Go subtests"
date = '2026-10-25'
description = "PHPUnit-style setUp() and tearDown() for Go subtests with an anonymous struct and t.Cleanup()."
tags = [
"php",
"go",
"phpunit",
"testing",
]
+++

Go's `testing` package has no `setUp()` and `tearDown()` hooks, and you don't need a framework to get them back. An
anonymous struct with 2 function fields, plus `t.Cleanup()`, gives each subtest a fresh fixture and a guaranteed
cleanup. This article presents a pattern, explains why it uses `t.Cleanup()` instead of `defer`, and points out where to
pay attention.

## The equivalents of PHPUnit's Data Providers in Go: Table-Driven Tests

The [Go Wiki recommends table-driven tests](https://go.dev/wiki/TableDrivenTests). You list the test cases in a
`[]struct {...}` slice or a `map[string]struct {...}` map and run each one as a [subtest](https://go.dev/blog/subtests)
with [`t.Run()`](https://pkg.go.dev/testing#T.Run), which Go has supported since 1.7. Dave Cheney's article
["Prefer table driven tests"](https://dave.cheney.net/2019/05/07/prefer-table-driven-tests) goes into much detail about
using this nice technique.

```go
func TestReverse(t *testing.T) {
	tests := map[string]struct {
		actual   string
		expected string
	}{
		"empty string":      {actual: "", expected: ""},
		"one character":     {actual: "x", expected: "x"},
		"palindrome string": {actual: "level", expected: "level"},
		"plain word":        {actual: "abc", expected: "cba"},
	}

	for name, tc := range tests {
		t.Run(name, func(t *testing.T) {
			t.Parallel() // Marks each test case as capable of running in parallel with each other.
			if got := reverse(tc.actual); got != tc.expected {
				t.Fatalf("reverse(%q) = %q, expected %q", tc.actual, got, tc.expected)
			}
		})
	}
}
```

> [!WARNING]
> Storing the cases in a map has a side effect worth keeping.
> In Go [the iteration order over maps is not specified](https://go.dev/blog/maps#iteration-order) and is not guaranteed
> to be the same from one iteration to the next. If one case quietly depends on another running first, the test fails
> sooner or later instead of hiding the dependency for years.

If you come from PHP like me, this is what
[PHPUnit **Data
Providers**](https://docs.phpunit.de/en/13.4/writing-tests-for-phpunit.html#writing-tests-for-phpunit-data-providers)
do, and the Go version covers the same ground with less machinery. One PHPUnit feature has no counterpart, though:
[fixtures with `setUp()` and `tearDown()`](https://docs.phpunit.de/en/13.4/fixtures.html).

## Keep the setup logic out of the test body

A fixture can be a single object or a whole object graph. As it grows, the code that builds it grows too, and the
assertions drown in setup noise. PHPUnit solves this with 2 **template methods**. It calls `setUp()` before each test
method and `tearDown()` after it, whether the test passed or failed.

In practice the 2 methods are not symmetric. You need `tearDown()` only when `setUp()` touched something outside the
test object: a static property, a database row, a file, a socket, or an environment variable. Database fixtures are the
common case. You insert rows before the test and delete them after it, so the next test starts from a clean state.

Go's standard library has no hook system that runs before and after each subtest, but you can still build one in a few
lines.

## An anonymous struct with setUp & tearDown fields

The test below checks a `StockRepository` against a real database. Each subtest needs the same two products in the
`stock` table, and each must leave the table empty.

```go
func TestStockRepository_Reserve(t *testing.T) {
	db := openTestDB(t)
	repo := NewStockRepository(db)

	suite := struct {
		setUp    func(t *testing.T)
		tearDown func(t *testing.T)
	}{
		setUp: func(t *testing.T) {
			t.Helper()

			query := "INSERT INTO `stock` (`sku`, `quantity`) VALUES ('BOOK-1', 3), ('PEN-1', 0)"
			if _, err := db.Exec(query); err != nil {
				t.Fatalf("failed to seed stock: %v", err)
			}
		},
		tearDown: func(t *testing.T) {
			if _, err := db.Exec("TRUNCATE TABLE `stock`"); err != nil {
				t.Errorf("failed to truncate stock: %v", err)
			}
		},
	}

	t.Run("reserves available items", func(t *testing.T) {
		t.Cleanup(func() { suite.tearDown(t) })
		suite.setUp(t)

		if err := repo.Reserve(t.Context(), "BOOK-1", 2); err != nil {
			t.Fatalf("Reserve() error = %v", err)
		}

		if got := repo.Quantity(t.Context(), "BOOK-1"); got != 1 {
			t.Errorf("Quantity() = %d, expected 1", got)
		}
	})

	t.Run("rejects an out-of-stock item", func(t *testing.T) {
		t.Cleanup(func() { suite.tearDown(t) })
		suite.setUp(t)

		if err := repo.Reserve(t.Context(), "PEN-1", 1); !errors.Is(err, ErrOutOfStock) {
			t.Fatalf("Reserve() error = %v, expected ErrOutOfStock", err)
		}
	})
}
```

The `openTestDB()` function is an ordinary helper. It opens the connection and registers `db.Close()` on the parent
test:

```go
func openTestDB(t *testing.T) *sql.DB {
	t.Helper()

	db, err := sql.Open("mysql", os.Getenv("TEST_DATABASE_DSN"))
	if err != nil {
		t.Fatalf("failed to open test database: %v", err)
	}

	t.Cleanup(func() { db.Close() })

	return db
}
```

## How the pattern works

Both closures capture `db` from the enclosing test function. The struct exists only inside
`TestStockRepository_Reserve`, so the fixture sits next to the tests that use it and nothing leaks into the package
scope - this keeps the things nicely [encapsulated](https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)).

A package-level helper could do the same work, but it would need `db` passed in and would be visible to every other test
in the package. I prefer the struct: **the setup and the teardown read as one unit**, the way they do in a PHPUnit test
class.

Each subtest starts with the same two lines:

1. `t.Cleanup(func() { suite.tearDown(t) })` - registers the teardown.
2. `suite.setUp(t)` - seeds the table.

The order of these two lines matters. If `setUp()` inserts one row and then fails, the teardown is already registered
and still empties the table. The next subtest starts clean either way.

Both functions take the subtest's `t` as a parameter instead of capturing the parent's `t`. The reason is a rule of the
`testing` package: `t.Fatalf()` must be called from the goroutine that runs the test. A subtest runs in its own
goroutine, so a `setUp()` that calls the parent's `t.Fatalf()` breaks that rule. The subtest's `t` also makes the
failure show up under the subtest's name in the output. [`t.Helper()`](https://pkg.go.dev/testing#T.Helper) in `setUp()`
makes the failure point at the subtest's line, not at the line inside the closure.

The one difference from PHPUnit is that nothing invokes these functions for you. You have to manually write that 2 lines
for each subtest. One advantage of this is its explicitness: anyone who reads a subtest sees exactly what state it
starts from.

## Use `t.Cleanup` instead of `defer`

[`t.Cleanup()`](https://pkg.go.dev/testing#T.Cleanup) registers a function that runs when the test and all its subtests
complete. Functions registered on the same test run in reverse order of registration, like `defer`. They also run when
the test stops early with `t.Fatal()`.

In the example the subtest's teardown runs first, when the subtest ends. The parent's `db.Close()` runs last, after
every subtest has finished. The connection is still open when each teardown needs it.

A `defer` in a subtest would work here too. The difference shows up with parallel subtests. Brandur Leach explains it in
his ["Prefer t.Cleanup with parallel subtests"](https://brandur.org/fragments/go-prefer-t-cleanup-with-parallel-subtests)
article. When a subtest calls `t.Parallel()`, the test framework pauses it and lets the parent function return. A
`defer` in the parent then runs at that return, before the paused subtests have run at all. If that `defer` closes the
database, every parallel subtest gets a closed connection. `t.Cleanup()` is part of the testing framework, so the
framework waits for the subtests before it calls the cleanup.

Use `t.Cleanup()` for anything a test sets up, then you never have to think about whether the test runs in parallel.

## Combine the suite with a table

The test suite struct and a table can work together. Each case calls the same 2 lines before its own logic:

```go
// This is a slice, the order fixed.
tests := []struct {
	name     string
	sku      string
	quantity int
	expected error
}{
	{name: "reserves available items", sku: "BOOK-1", quantity: 2},
	{name: "rejects an out-of-stock item", sku: "PEN-1", quantity: 1, expected: ErrOutOfStock},
	{name: "rejects more than in stock", sku: "BOOK-1", quantity: 4, expected: ErrOutOfStock},
}

for _, tc := range tests {
	t.Run(tc.name, func(t *testing.T) {
		t.Cleanup(func() { suite.tearDown(t) })
		suite.setUp(t)

		if err := repo.Reserve(t.Context(), tc.sku, tc.quantity); !errors.Is(err, tc.expected) {
			t.Fatalf("Reserve(%q, %d) error = %v, expected %v", tc.sku, tc.quantity, err, tc.expected)
		}
	})
}
```

## Run these subtests in parallel?

All subtests in the example share one `stock` table, and each teardown truncates it. If two of them ran in parallel,
one subtest's `TRUNCATE` would delete the rows another subtest is still reading. Leave `t.Parallel()` out of subtests
that share a fixture this way.

To run database tests in parallel, give each subtest its own state:
- its own transaction that it rolls back,
- or its own schema.

Brandur Leach describes both approaches thoroughly in ["On t.Parallel"](https://brandur.org/t-parallel).

## Other tools in the testing package

Additionally, the `testing` package includes several other tools worth mentioning, alongside the ones listed above:

- [`t.Context()`](https://pkg.go.dev/testing#T.Context) returns a context that the framework cancels just before the
  cleanup functions run. Don't use it inside a cleanup function, because it is already canceled there.
- [`t.Setenv()`](https://pkg.go.dev/testing#T.Setenv) sets an environment variable and restores the old value through
  `t.Cleanup()`. Because it changes the whole process, it fails in a parallel test or under a parallel parent.
- [`t.TempDir()`](https://pkg.go.dev/testing#T.TempDir) returns a temporary directory that the framework removes when
  the test and its subtests complete.
- [`t.ArtifactDir()`](https://pkg.go.dev/testing#T.ArtifactDir), new in
  [Go 1.26](https://go.dev/doc/go1.26#testingpkgtesting), returns a directory for test output files. With
  `go test -artifacts`, the files stay under the output directory. Without the flag, the framework deletes the directory
  after the test.
