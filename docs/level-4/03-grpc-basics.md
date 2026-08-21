# 03 · gRPC Basics

REST and JSON ([Level 3, Module 2](../level-3/02-building-rest-apis.md))
are the right default for public APIs. gRPC trades human-readable JSON for
a compact binary format (Protocol Buffers) and generates strongly-typed
client/server code from a single `.proto` schema — a common choice for
internal service-to-service calls where both ends are Go (or any of the
dozen other languages `protoc` supports) and performance or type safety
matters more than curl-ability.

Everything below was generated and run for real: `protoc` v35.1,
`protoc-gen-go`, and `protoc-gen-go-grpc` installed via `brew install
protobuf` and `go install .../protoc-gen-go@latest` /
`.../protoc-gen-go-grpc@latest`, then a real server and client processes
talked over a real TCP socket.

## The schema

```protobuf
syntax = "proto3";

package greet;

option go_package = "l4m03/greetpb";

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string name = 1;
}

message HelloResponse {
  string message = 1;
}
```

Generating Go code from it:

```console
$ protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greetpb/greet.proto

$ ls greetpb/
greet.pb.go       # message types: HelloRequest, HelloResponse
greet.proto
greet_grpc.pb.go  # GreeterClient interface, GreeterServer interface, registration
```

The numbers after each field (`name = 1`, `message = 1`) are wire tags, not
default values — they're how the binary encoding identifies fields without
sending field names, and they must never be reused for a different field
once a schema has shipped, or old and new clients will misinterpret each
other's messages.

## The server

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net"

	"google.golang.org/grpc"

	pb "l4m03/greetpb"
)

type server struct {
	pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, req *pb.HelloRequest) (*pb.HelloResponse, error) {
	return &pb.HelloResponse{Message: "Hello, " + req.GetName() + "!"}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	grpcServer := grpc.NewServer()
	pb.RegisterGreeterServer(grpcServer, &server{})
	fmt.Println("grpc server listening on :50051")
	log.Fatal(grpcServer.Serve(lis))
}
```

Embedding `pb.UnimplementedGreeterServer` is not boilerplate to skip — it's
what keeps `*server` satisfying the `GreeterServer` interface after a
`.proto` change adds a new RPC method: the generated "unimplemented"
version returns an `Unimplemented` gRPC status for the new method instead
of failing to compile, so a server can be rebuilt against a newer schema
before every method is implemented.

## The client

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	pb "l4m03/greetpb"
)

func main() {
	conn, err := grpc.NewClient("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	client := pb.NewGreeterClient(conn)
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	resp, err := client.SayHello(ctx, &pb.HelloRequest{Name: "Ada"})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println(resp.GetMessage())
}
```

Running server and client as two separate processes:

```console
$ go build -o bin/server ./server && go build -o bin/client ./client
$ ./bin/server &
grpc server listening on :50051

$ ./bin/client
Hello, Ada!
```

`insecure.NewCredentials()` explicitly opts out of TLS — required as of
recent `grpc-go` versions rather than silently defaulting to plaintext, so
that shipping an insecure connection to production is a visible,
deliberate choice in the code, not an accident of omission. Every RPC call
takes a `context.Context` exactly like an HTTP handler does
([Level 3, Module 4](../level-3/04-context-package.md)) — the 2-second
timeout here bounds `SayHello` the same way `http.Client.Timeout` bounds
a REST call in [Module 2](02-microservices-patterns.md).

## Go-specific traps

- **Regenerating without `paths=source_relative`** places generated files
  under a directory tree matching the full Go module path derived from
  `go_package`, which surprises people expecting the file next to the
  `.proto` — always pass it for a single-module layout like this one.
- **Field access via generated getters (`req.GetName()`), not direct field
  access (`req.Name`)**, is the safer habit — `GetName()` is nil-safe (a
  `nil *HelloRequest` returns the zero value instead of panicking), which
  matters because a `nil` message is a valid, if useless, value in
  protobuf-generated Go.
- **Forgetting to regenerate after editing a `.proto`** is invisible until
  runtime — the compiled Go code and the schema silently drift apart, and
  Go's compiler cannot tell you your `.proto` file changed.
- **Reusing or renumbering a field tag** breaks wire compatibility for any
  client/server pair not upgraded in lockstep — always add new fields with
  new numbers, and mark removed ones `reserved` in the `.proto` rather than
  deleting the number outright.
- **A gRPC server returning a plain Go `error`** (as `SayHello` could)
  loses status-code information — use `status.Errorf(codes.InvalidArgument,
  "...")` from `google.golang.org/grpc/status` so clients can distinguish
  "you sent bad input" from "the server broke" instead of getting an opaque
  `Unknown` status.

## Cheat sheet

| Task | Command / API |
|---|---|
| Install tooling | `brew install protobuf`; `go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`; `go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest` |
| Generate Go code | `protoc --go_out=. --go_opt=paths=source_relative --go-grpc_out=. --go-grpc_opt=paths=source_relative x.proto` |
| Start a server | `grpc.NewServer()`; `pb.RegisterXServer(s, impl)`; `s.Serve(lis)` |
| Connect a client | `grpc.NewClient(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))` |
| Bound an RPC call | `context.WithTimeout(ctx, d)`, pass `ctx` as the call's first argument |
| Nil-safe field access | Generated getters: `msg.GetField()`, never `msg.Field` on a possibly-nil message |
| Typed error with status code | `status.Errorf(codes.NotFound, "...")` |

## Related lessons

- HTTP/REST as the alternative transport, and its own timeout/context
  discipline: [Level 3, Module 2](../level-3/02-building-rest-apis.md) and
  [Level 3, Module 4](../level-3/04-context-package.md).
- Circuit breakers and client timeouts applied to any RPC transport, gRPC
  included: [Module 2](02-microservices-patterns.md).

## Exercise

Add a `SayHelloStream` server-streaming RPC to `greet.proto` that takes a
`HelloRequest` and returns a `stream HelloResponse`, sending three
greetings ("Hi", "Hello", "Hey" — each followed by the name) with a short
`time.Sleep` between each. Regenerate the code, implement it on the server
using `stream.Send(...)`, and write a client that calls
`stream.Recv()` in a loop until it gets `io.EOF`, printing each message as
it arrives.
