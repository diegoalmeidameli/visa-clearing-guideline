# Visa Clearing Style Guide

## Table of Contents

- [Introduction](#introduction)
- [Style](#style)
  - [Prefix Unexported Consts with _](#prefix-unexported-consts-with-_)
- [Patterns](#patterns)
  - [Rebase as team policy](#rebase-as-team-policy)
  - [Test Tables](#test-tables)
  - [Name pattern for branches](#name-pattern-for-branches)

## Introduction

Styles are the conventions that guides our code. The term style is a bit of a misnomer, since these conventions cover far more than just source file formatting—gofmt handles that for us.

The goal of this guide is to manage this complexity by describing in detail the Dos and Don'ts of writing Go code at Visa Clearing Team. These rules exist to keep the code base manageable while still allowing engineers to use Go language features productively.

This guide was created by [Diego Almeida] as a way to bring new team members up to speed with using Go and can be changed based on feedback from others.

  [Diego Almeida]: https://github.com/diegoalmeidameli

This documents idiomatic conventions in Go code that we follow at Visa Clearing Team. A lot of these are general guidelines for Go, while others extend upon external
resources:

1. [Effective Go](https://golang.org/doc/effective_go.html)
2. [Go Common Mistakes](https://github.com/golang/go/wiki/CommonMistakes)
3. [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)

All code should be error-free when run through `golint` and `go vet`. We recommend setting up your editor to:

- Run `goimports` on save
- Run `golint` and `go vet` to check for errors

You can find information in editor support for Go tools here: <https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins>

## Style

### Prefix Unexported Consts with _

Prefix unexported top-level `const`s with `_` to make it clear when they are used that they are global symbols.

Exception: Unexported error values, which should be prefixed with `err`.

Rationale: Top-level variables and constants have a package scope. Using a generic name makes it easy to accidentally use the wrong value in a different file.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

</td><td>

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

</td></tr>
</tbody></table>

## Patterns

### Rebase as Team Policy

It's hard to generalize since every team is different, but we have to start from somewhere. When a feature branch's development is complete, rebase/squash all the work down to the minimum number of meaningful commits and avoid creating a merge commit – either making sure the changes fast-forward (or simply cherry-pick those commits into the target branch).

While the work is still in progress and a feature branch needs to be brought up to date with the upstream target branch, use rebase – as opposed to pull or merge – not to pollute the history with spurious merges.

**Pros:**
- Code history remains flat and readable. Clean, clear commit messages are as much part of the documentation of your code base as code comments, comments on your issue tracker etc. For this reason, it's important not to pollute history with 31 single-line commits that partially cancel each other out for a single feature or bug fix. Going back through history to figure out when a bug or feature was introduced, and why it was done, is going to be tough in a situation like this.

- Manipulating a single commit is easy (e.g. reverting them).

**Cons:**
- Squashing the feature down to a handful of commits can hide context, unless you keep around the historical branch with the entire development history.

- Rebasing doesn't play well with pull requests, because you can't see what minor changes someone made if they rebased.

- Rebasing can be dangerous! Rewriting history of shared branches is prone to team work breakage. This can be mitigated by doing the rebase/squash on a copy of the feature branch, but rebase carries the implication that competence and carefulness must be employed.

- It's more work: Using rebase to keep your feature branch updated requires that you resolve similar conflicts again and again. Yes, you can reuse recorded resolutions (rerere) sometimes, but merges win here: Just solve the conflicts one time, and you're set.

- Another side effect of rebasing with remote branches is that you need to force push at some point. The biggest problem we've seen at Atlassian is that people force push – which is fine – but haven't set git push.default. This results in updates to all branches having the same name, both locally and remotely, and that is dreadful to deal with.

Here is a [link](https://drive.google.com/file/d/14Q8wUVXm9T_ouFm3U_U2bQH8vpNb4uPv/view) to a video describing the use of rebase in our context.

### Test Tables

Use table-driven tests with [subtests] to avoid duplicating code when the core test logic is repetitive.

  [subtests]: https://blog.golang.org/subtests

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

</td><td>

```go
// func TestSplitHostPort(t *testing.T)

tt := []struct{
  input     string
  expectedHost string
  expectedPort string
}{
  {
    input:     "192.0.2.0:8000",
    expectedHost: "192.0.2.0",
    expectedPort: "8000",
  },
  {
    input:     "192.0.2.0:http",
    expectedHost: "192.0.2.0",
    expectedPort: "http",
  },
  {
    input:     ":8000",
    expectedHost: "",
    expectedPort: "8000",
  },
  {
    input:     "1:8",
    expectedHost: "1",
    expectedPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.input, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.input)
    require.NoError(t, err)
    assert.Equal(t, tt.expectedHost, host)
    assert.Equal(t, tt.expectedPort, port)
  })
}
```

</td></tr>
</tbody></table>

Test tables make it easier to add context to error messages, reduce duplicate logic, and add new test cases.

We follow the convention that the slice of structs is referred to as `tt` and each test case `tc`. Further, we encourage explicating the input and output values for each test case with with `input` and `expected` prefixes.

```go
tests := []struct{
  input     string
  expectedHost string
  expectedPort string
}{
  // ...
}

for _, tc := range tt {
  // ...
}
```

### Name pattern for branches

Name your branch with the prefix of the card generated by Jira + keywords that make sense for the context of the activity. Examples:
```
feature/gen-9870-auth-syncronization
bugfix/gen-12508-prevents-wrong-patch
```