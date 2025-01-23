---
title: Interrupting scanning in Go
description: Description
date: 2022-10-15
author: leon h
draft: false
favorite: false
# tags:
#   - tag1
---

`bufio.Scanner` is "a convenient interface for reading data such as a file of newline-delimited lines of text". It will stop scanning either by reaching the end of the input or an error. The usual pattern goes.

<!--more-->

https://henvic.dev/posts/signal-notify-context/

https://pkg.go.dev/os/signal

https://medium.com/golangspec/in-depth-introduction-to-bufio-scanner-in-golang-55483bb689b4

https://stackoverflow.com/questions/37079639/how-we-can-stop-bufio-scanning-at-golang

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	ctx = context.WithValue(ctx, cmd.NameKey, name)
	ctx = context.WithValue(ctx, cmd.VersionKey, version)
	ctx = context.WithValue(ctx, cmd.AuthorKey, author)

	if err := cmd.Execute(ctx); err != nil {
		log.Fatalf("honeybadger: %v", err)
	}
}

		uciOut := os.Stdout
		uciIn, pipe := io.Pipe()

		// graceful shutdown when context canceled
		// sending EOF to the UCI scanner by closing the pipe
		go func() { _, _ = io.Copy(pipe, os.Stdin) }()
		go func() { <-ctx.Done(); pipe.Close() }()

		e := engine.New(
			engine.WithName(fmt.Sprintf("%s v%s", name, version)),
			engine.WithAuthor(author),
			engine.WithLogger(uci.Logger(uciOut)),
		)

		uci.Run(ctx, e, uciIn, uciOut)

// uci
func Run(ctx context.Context, e Engine, r io.Reader, w io.Writer) {
	respond := newResponder(w)

	for scanner := bufio.NewScanner(r); scanner.Scan(); {
		c := parse(strings.Fields(scanner.Text()))
		if c == nil {
			continue
		}
		c.run(ctx, e, respond)
		if _, ok := c.(commandQuit); ok {
			break
		}
	}
}
```
