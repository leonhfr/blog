---
title: One mockgen flag can make your CI 5x faster
description: How switching mockgen from package mode to source mode gave us a 5x speedup in mock generation.
date: 2025-11-01
draft: false
tags:
  - go
  - testing
  - ci
  - developer tools
---

At work, mock generation was slow: 5 minutes locally, and 15 in CI. That's long enough that I'd start something else while waiting and forget to come back. Refusing to accept the harsh realities of life, I wondered if we could do better.

<!--more-->

We use [gomock](https://github.com/uber-go/mock) with `go:generate` directives scattered across the codebase:

```go
//go:generate mockgen -destination service_mock.go -package invoice . Service
```

This is package mode. You give it an import path and interfaces, and it generates mocks. Simple enough, right?

But package mode works by building the entire package and using reflection to inspect interfaces. When you have a large enough codebase with dozens of mocks, this adds up.

I discovered that mockgen has another mode: source mode.

```go
//go:generate mockgen -source service.go -destination service_mock.go -package invoice
```

Instead of building and reflecting, it parses the source file directly and analyzes the AST. No more compilation and no more reflection.

The migration was a simple search & replace. When testing locally, mock generation was 3 times faster. When I pushed the PR, I watched the CI job drop from 15 minutes to 3.

There's one quirk: source mode mocks every interface in the file. You can't specify just one as with package mode. However, you can use `-exclude_interfaces` to exclude some:

```go
//go:generate mockgen -source service.go -destination service_mock.go -package invoice -exclude_interfaces Repository
```

If mock generation is slow for you too, check your mockgen directives. One flag might be all you need.
