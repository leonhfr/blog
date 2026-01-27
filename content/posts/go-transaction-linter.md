---
title: I shipped a transaction bug, so I built a linter
description: Database operations leaked outside a transaction and broke prod. Here's how I built a Go linter to prevent it.
date: 2026-01-27
draft: false
showTableOfContents: true
tags:
  - go
  - databases
  - static analysis
  - developer tools
---

Some bugs compile cleanly, pass all tests, and slip through code reviews. I shipped one of those at work: a database transaction that silently leaked operations outside its boundary. I don't like being stressed with a broken prod, so I built a custom linter to catch them at compile time. Here's how.

<!--more-->

## The Bug: Leaking Transactions

Database transactions are essential for data integrity. Operations wrapped in a transaction are expected to display all-or-nothing behavior: either every operation succeeds, or everything rolls back.

One pattern for managing transactions is callbacks. This style of transaction is common when using ORMs such as [Gorm](https://gorm.io/docs/transactions.html). However, this approach makes it easier to accidentally bypass transaction boundaries. Operations can leak outside the intended scope, leading to data corruption and race conditions. Let's look at some code.

At [my current workplace](https://www.re-cap.com), the Go backend uses a repository pattern with explicit transactions:

```go
func (s *Service) UpdateUser(ctx context.Context, userID string) error {
    return s.repo.Transaction(ctx, func(tx models.Repo) error {
        user, err := tx.GetUser(ctx, userID)
        if err != nil {
            return err
        }
        user.Name = "Updated"
        return tx.SaveUser(ctx, user)
    })
}
```

The callback receives `tx`, a transaction-scoped repository. All database operations inside must use `tx` to participate in the transaction.

This pattern works, but it's easy to mix up the two scopes:

```go
func (s *Service) UpdateUser(ctx context.Context, userID string) error {
    return s.repo.Transaction(ctx, func(tx models.Repo) error {
        user, err := s.repo.GetUser(ctx, userID) // Bug: uses s.repo, not tx.
        if err != nil {
            return err
        }
        user.Name = "Updated"
        return tx.SaveUser(ctx, user) // Ok: inside the transaction.
    })
}
```

The `GetUser` call uses `s.repo` (the component's field) instead of `tx` (the transaction callback parameter). This operation executes outside the transaction boundary.

This mistake is easy to miss because the code often appears to work correctly. It typically happens when wrapping existing code in a transaction and forgetting to update one reference.

## It's Hard to Catch

What's more, this bug is insidious and hard to catch. The code compiles without error. Tests pass because they run in isolation with no contention. Failures are unpredictable and usually only happen under load. Worst of all, the failure mode is often silent data corruption, not loud and easy-to-diagnose crashes.

It's also easy to miss in code reviews, especially when the bug is nested a few functions deep. AI review tools only caught it when it was obvious, so they were not reliable enough. After some of these slipped through and resulted in ~me banging my head against the wall~ long debugging sessions, I decided to find a more efficient way to catch them.

Static analysis is well-suited here. The bug is structural, not behavioral. It's all about which variable the code references (`s.repo` vs `tx`), not about runtime values. The pattern is detectable from source code alone.

That's why I decided to write a linter. While I use them every day, I wasn't exactly sure how they worked nor where to start. However, writing one seemed like an interesting challenge, so I'm sharing what I learned.

## The `go/analysis` Framework

The current standard for Go linters is the [`go/analysis` framework](https://pkg.go.dev/golang.org/x/tools/go/analysis). It makes static analysis surprisingly accessible. The framework lets you focus on the linter logic while it handles all the complexity of parsing, type-checking, and running analyses.

The package provides a standardized structure for building analyzers with the `analysis.Analyzer` type:

```go
func NewAnalyzer() *analysis.Analyzer {
    return &analysis.Analyzer{
        Name:     "transactioncheck",
        Doc:      "checks that Transaction callbacks use the transaction-scoped repo parameter",
        Run:      runTransactionCheck,
        Requires: []*analysis.Analyzer{inspect.Analyzer},
    }
}
```

The `Run` field contains a function that executes the analysis on a single package. It receives an `*analysis.Pass` struct containing everything needed for analysis: parsed AST, type information, and a `Reportf` method for flagging violations.

The `Requires` field specifies a list of other analyzers this one depends on. Here, we depend on `inspect.Analyzer`, which provides an optimized AST traversal mechanism.

Following Go convention, where linters typically have a `check` suffix (like `staticcheck` or `nilcheck`), I called mine `transactioncheck`.

## Implementation Deep Dive

{{< alert "circle-info" >}}
The source code is provided as a [standalone repository](https://github.com/leonhfr/transactioncheck).
{{< /alert >}}

### Walking the AST

Here's the entry point of the analyzer:

```go
func runTransactionCheck(pass *analysis.Pass) (any, error) {
    inspect := pass.ResultOf[inspect.Analyzer].(*inspector.Inspector)

    nodeFilter := []ast.Node{(*ast.CallExpr)(nil)}

    inspect.Preorder(nodeFilter, func(n ast.Node) {
        callExpression := n.(*ast.CallExpr)
        if !isTransactionCall(pass, callExpression) {
            return
        }

        // Analyze the transaction callback...
    })

    return nil, nil
}
```

We filter for call expressions (`*ast.CallExpr`). We want to inspect every function or method call, and nothing else. This avoids visiting every node in the AST. Then, if it's a `Transaction` call, we analyze its callback for violations.

### Identifying Transaction Calls

Before we can detect misused repositories, we need to determine whether a call is a transaction call. The implementation of this is specific to each codebase.

First, we need to detect repository interfaces. In our example, we have a unique `models.Repo` interface. We match it by name (`Repo`) and package location (`example.com/models`).

Now that we can identify repositories, we need to detect transaction calls. In the AST, method calls like `repo.DoSomething()` are represented as selector expressions (`*ast.SelectorExpr`), while direct calls like `doSomething()` are not. We check that the node is a selector expression, the method name is `Transaction`, and the receiver is a repository interface.

### Tracking the Transaction Parameter

When we find a transaction call, we extract the first parameter of the callback. This is the transaction-scoped parameter, `tx`. We need to verify that all operations within the callback use it, which means tracking it while we traverse the callback's AST.

We capture two things:
- the parameter name (`txParameterName`) for error messages like "should use transaction parameter `tx`",
- the type object (`txParameterObject`) for identity comparison.

In Go's type system, two identifiers referring to the same variable share the same `types.Object`, which we can compare with a simple equality check `==`. This allows us to track the transaction parameter without relying on name matching, which would fail anyway if the variable name is shadowed.

```go
func isTransactionParameter(
    pass *analysis.Pass,
    expression ast.Expr,
    txParameterObject types.Object,
) bool {
    identifier, ok := expression.(*ast.Ident)
    if !ok {
        return false
    }

    object := pass.TypesInfo.Uses[identifier]
    if object == nil {
        object = pass.TypesInfo.Defs[identifier]
    }

    return object == txParameterObject
}
```

### Traversing the Callback

For each transaction callback we find, we traverse its AST to look for violations. We use `ast.Inspect` to walk every node:

```go
func checkTransactionCallback(
    pass *analysis.Pass,
    functionBody *ast.BlockStmt,
) {
    ast.Inspect(functionBody, func(n ast.Node) bool {
        callExpr, ok := n.(*ast.CallExpr)
        if !ok {
            return true // The node is not a function call, we keep traversing.
        }

        if isTransactionCall(pass, callExpr) {
            return false // Stop traversing: nested transactions have their own scope.
        }

        // Check for violations...

        return true
    })
}
```

The traversal stops when it encounters a nested transaction. A nested `Transaction` call creates its own transaction scope with its own `tx` parameter. Any code inside that nested callback is outside our current analysis scope and will be analyzed separately when we encounter that transaction call.

### Detecting Outer Repository Method Calls

One violation type happens when we call a method on the outer repository instead of the transaction parameter:

```go
s.repo.Transaction(ctx, func(tx models.Repo) error {
    user := s.repo.GetUser(ctx, "123") // Violation: should use tx.GetUser.
    return tx.SaveUser(ctx, user)
})
```

We detect this by checking if the receiver of a method call is a repository interface that isn't our transaction parameter:

```go
func checkRepoMethodCall(
    pass *analysis.Pass,
    selectorExpression *ast.SelectorExpr,
    txParameterName string,
    txParameterObject types.Object,
) {
    receiverType := pass.TypesInfo.TypeOf(selectorExpression.X)
    if !isRepoInterface(receiverType) {
        return // Receiver is not a repository interface.
    }

    if isTransactionParameter(pass, selectorExpression.X, txParameterObject) {
        return // Using the transaction parameter is correct.
    }

    pass.Reportf(
        selectorExpression.X.Pos(),
        "using non-transaction repo; should use %q instead",
        txParameterName,
    )
}
```

### Detecting Outer Repositories Passed to Helper Functions

The second violation we want to detect is more subtle: it happens when passing the outer repository to a helper function instead of the transaction parameter.

```go
func (s *Service) CreateUser(ctx context.Context, userID string) error {
    return s.repo.Transaction(ctx, func(tx models.Repo) error {
        return helperFunction(ctx, s.repo, userID) // Violation: should pass tx.
    })
}

func helperFunction(ctx context.Context, repo models.Repo, userID string) error {
    return repo.SaveUser(ctx, &User{ID: userID})
}
```

Here, `s.repo` is passed as an argument, and the helper uses it for database operations outside the transaction. The detection logic is similar to before: check if any argument is a repository interface that isn't the transaction parameter:

```go
func checkCallArguments(
    pass *analysis.Pass,
    callExpression *ast.CallExpr,
    txParameterName string,
    txParameterObject types.Object,
) {
    for _, argument := range callExpression.Args {
        argumentType := pass.TypesInfo.TypeOf(argument)
        if !isRepoInterface(argumentType) {
            continue // Argument is not a repository interface.
        }

        if isTransactionParameter(pass, argument, txParameterObject) {
            continue // Passing the transaction parameter is correct.
        }

        pass.Reportf(
            argument.Pos(),
            "passing non-transaction repo to function; should pass %q instead",
            txParameterName,
        )
    }
}
```

### Recursive Analysis of Helper Functions

Detecting violations at the call site isn't always enough. Consider a chain of helper functions:

```go
func (s *Service) CreateOrder(ctx context.Context, userID string) error {
    return s.repo.Transaction(ctx, func(tx models.Repo) error {
        // Ok: passing the transaction callback parameter.
        return s.processOrder(ctx, tx, userID)
    })
}

func (s *Service) processOrder(ctx context.Context, repo models.Repo, userID string) error {
    // Ok: using the repository parameter.
    user, err := repo.GetUser(ctx, userID)
    if err != nil {
        return err
    }
    // Ok: passing the transaction callback parameter.
    return s.finalizeOrder(ctx, repo, user)
}

func (s *Service) finalizeOrder(ctx context.Context, repo models.Repo, user *User) error {
    // Bug: uses s.repo instead of the repo parameter.
    return s.repo.SaveOrder(ctx, &Order{UserID: user.ID})
}
```

Taken separately, none of these functions would be flagged. `finalizeOrder` is not a violation in isolation. It only becomes a bug when called as part of a transaction. To catch this, we need to recursively analyze helper functions.

When the linter sees `s.processOrder(ctx, tx, userID)` passing the transaction parameter, it recurses into `processOrder`, now tracking `repo` as the transaction parameter. When `processOrder` calls `s.finalizeOrder(ctx, repo, user)`, the linter recurses again. Finally, inside `finalizeOrder`, it detects that `s.repo.SaveOrder` uses `s.repo` instead of the `repo` parameter.

To prevent infinite loops when helpers call each other, the linter tracks visited functions. This ensures each function is analyzed at most once per transaction.

## Testing with `analysistest`

The `go/analysis` framework provides [`analysistest`](https://pkg.go.dev/golang.org/x/tools/go/analysis/analysistest), which makes testing Analyzers simple and elegant:

```go
func TestAnalyzer(t *testing.T) {
    testdata := analysistest.TestData()
    analysistest.Run(t, testdata, NewAnalyzer(), "transactioncheck")
}
```

Test cases then live in `testdata/src/transactioncheck` and use special `// want` comments for assertions:

```go
func (s *Service) IncorrectUsage(ctx context.Context) error {
    return s.repo.Transaction(ctx, func(tx models.Repo) error {
        _ = s.repo.GetUser( // want "using non-transaction repo; should use \"tx\" instead"
            ctx,
            "123",
        )
        return nil
    })
}

func (s *Service) CorrectUsage(ctx context.Context) error {
    return s.repo.Transaction(ctx, func(tx models.Repo) error {
        _ = tx.GetUser(ctx, "123")  // No "want" comment, should not report.
        return nil
    })
}
```

The test framework verifies that lines with `// want` comments produce matching diagnostics and that lines without them produce no diagnostics. This catches both false negatives (missed bugs) and false positives (false reports).

Code in `/testdata/src` is isolated from the rest of the codebase, so we need to mock dependencies such as the `models.Repo` interface. The subdirectories below `/testdata/src` have to mirror the import path.

## Running the Linter

With `go/analysis`, running the linter is straightforward. Because we only have a single analyzer, we use the `singlechecker` package. The main function is as simple as:

```go
func main() {
    singlechecker.Main(transactioncheck.NewAnalyzer())
}
```

This generates a complete CLI with flags for output format, verbosity, and more.

It's possible to run an analyzer [as part of `golangci-lint`](https://golangci-lint.run/docs/contributing/new-linters) (in fact, they only accept linters written with this framework). Since `transactioncheck` is not generic to all codebases, proposing it as a public linter didn't make sense.

`golangci-lint` also supports private linters, but that would require building a custom binary and configuring every developer's editor to use it. This was not practical.

For this reason, a custom linter is best run as a standalone tool. For example, at work we use [mise](https://mise.jdx.dev/) as a task runner, so it's easy to integrate it:

```toml
[tasks.lint]
depends_post = "transactioncheck"
run = 'golangci-lint run -v'

[tasks.build-transactioncheck]
sources = ["./tools/transactioncheck/**/*.go"]
run = 'go build -o bin/transactioncheck tools/transactioncheck/cmd/main.go'

[tasks.transactioncheck]
depends = "build-transactioncheck"
run = 'bin/transactioncheck ./...'
```

The `depends` key ensures the linter is built before it runs. The `depends_post` key runs `transactioncheck` after `golangci-lint`.

Finally, I added the linter to CI. New violations now break the build, preventing these bugs from reaching production.

## You Should Write Your Own Linter

When I first ran `transactioncheck`, it found multiple violations across the codebase. These were not hypothetical bugs. Thankfully, none were in services handling financial data, as those get more diligent review, but the risk was real.

What started as frustration with transaction bugs became a two-day project that now protects the entire codebase. The linter runs in seconds, catches a whole class of bugs at compile time, and has required almost no maintenance since.

If your team has code patterns that are easy to get wrong, consider building a custom linter!
