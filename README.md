# websocket

[![GoDoc](https://godoc.org/nhooyr.io/websocket?status.svg)](https://godoc.org/nhooyr.io/websocket)
[![Codecov](https://img.shields.io/codecov/c/github/nhooyr/websocket.svg?color=brightgreen)](https://codecov.io/gh/nhooyr/websocket)

websocket is a minimal and idiomatic WebSocket library for Go.

This library is not final and the API is subject to change.

If you have any feedback, please feel free to open an issue.

## Install

```bash
go get nhooyr.io/websocket@v0.2.0
```

## Features

- Minimal yet pragmatic API
- First class context.Context support
- Thoroughly tested, fully passes the [autobahn-testsuite](https://github.com/crossbario/autobahn-testsuite)
- Concurrent writes
- Zero dependencies outside of the stdlib for the core library
- JSON and ProtoBuf helpers in the wsjson and wspb subpackages

## Roadmap

- [ ] WebSockets over HTTP/2 [#4](https://github.com/nhooyr/websocket/issues/4)
- [ ] Deflate extension support [#5](https://github.com/nhooyr/websocket/issues/5)

## Examples

For a production quality example that shows off the full API, see the [echo example on the godoc](https://godoc.org/nhooyr.io/websocket#example-package--Echo). On github, the example is at [example_echo_test.go](./example_echo_test.go).

### Server

```go
http.HandlerFunc(func (w http.ResponseWriter, r *http.Request) {
	c, err := websocket.Accept(w, r, websocket.AcceptOptions{})
	if err != nil {
		// ...
	}
	defer c.Close(websocket.StatusInternalError, "the sky is falling")

	ctx, cancel := context.WithTimeout(r.Context(), time.Second*10)
	defer cancel()
	
	var v interface{}
	err = wsjson.Read(ctx, c, &v)
	if err != nil {
		// ...
	}
	
	log.Printf("received: %v", v)
	
	c.Close(websocket.StatusNormalClosure, "")
})
```

### Client

```go
ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
defer cancel()

c, _, err := websocket.Dial(ctx, "ws://localhost:8080", websocket.DialOptions{})
if err != nil {
	// ...
}
defer c.Close(websocket.StatusInternalError, "the sky is falling")

err = wsjson.Write(ctx, c, "hi")
if err != nil {
	// ...
}

c.Close(websocket.StatusNormalClosure, "")
```

## Design considerations

- Minimal API is easier to maintain and learn
- Context based cancellation is more ergonomic and robust than setting deadlines
- No ping support because TCP keep alives work fine for HTTP/1.1 and they do not make
  sense with HTTP/2 (see [#1](https://github.com/nhooyr/websocket/issues/1))
- net.Conn is never exposed as WebSocket over HTTP/2 will not have a net.Conn.
- Using net/http's Client for dialing means we do not have to reinvent dialing hooks
  and configurations like other WebSocket libraries

## Comparison

While I believe nhooyr/websocket has a better API than existing libraries, 
both gorilla/websocket and gobwas/ws were extremely useful in implementing the
WebSocket protocol correctly so *big thanks* to the authors of both. In particular,
I made sure to go through the issue tracker of gorilla/websocket to make sure
I implemented details correctly and understood how people were using the package
in production.

### gorilla/websocket

https://github.com/gorilla/websocket

This package is the community standard but it is 6 years old and over time
has accumulated cruft. Using is not clear as there are many ways to do things
and there are some rough edges. Just compare the godoc of
[nhooyr/websocket](https://godoc.org/github.com/nhooyr/websocket) side by side with
[gorilla/websocket](https://godoc.org/github.com/gorilla/websocket).

The API for nhooyr/websocket has been designed such that there is only one way to do things
which makes it easy to use correctly.

Furthermore, nhooyr/websocket has support for newer Go idioms such as context.Context and
also uses net/http's Client and ResponseWriter directly for WebSocket handshakes.
gorilla/websocket writes its handshakes to the underlying net.Conn which means
it has to reinvent hooks for TLS and proxying and prevents support of HTTP/2.

Another advantage of nhooyr/websocket is that it supports concurrent writers out of the box.

### x/net/websocket

https://godoc.org/golang.org/x/net/websocket

Unmaintained and the API does not reflect WebSocket semantics. Should never be used.

See https://github.com/golang/go/issues/18152

### gobwas/ws

https://github.com/gobwas/ws

This library has an extremely flexible API but that comes at the cost of usability
and clarity. 

This library is fantastic in terms of performance. The author put in significant
effort to ensure its speed and I have applied as many of its optimizations as
I could into nhooyr/websocket. Definitely check out his fantastic [blog post](https://medium.freecodecamp.org/million-websockets-and-go-cc58418460bb) 
about performant WebSocket servers.

If you want a library that gives you absolute control over everything, this is the library,
but for most users, the API provided by nhooyr/websocket will fit better as it is nearly just
as performant but much easier to use correctly and idiomatic.
