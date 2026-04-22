## The Tenets

The Tenets are a set of design principles we've settled on for developing services:

- Hexagonal design
- Schema-based development
- Consumer-driven interfaces

They are not set in stone, either in their definitions or in their inclusion as tenets. We're always looking to improve, and can discuss the adjustment of Tenets and their inclusion and removal as our approach evolves.
### Hexagonal design

A great starting point is to [read this blog post by Herberto Graca](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/), but here's a quick summary.

Hexagonal design emphasizes designing software such that it has a "core" and everything outside of that core is loosely coupled through "ports" and "adapters".

The **Core** is the business logic. It contains the entities that allow the application to perform its function.

**Ports** are the defined public connection points that can be used to drive the application. An example of a Port could be a public method or function of a package's Core. It says nothing about _who_ consumes it, and is designed to make the most sense for the Core itself. Ports shouldn't contort themselves to be convenient for the consumer.

**Adapters** are the concrete implementations that allow specific consumers to use the application's Ports. We discuss two types of Adapters: **Generic** and **Specific**.

A Generic Adapter is an implementation of a standardized protocol, allowing any consumer to use that protocol. Examples of Generic Adapters may be HTTP-JSON or GRPC services that translates those types of requests into operations on the Ports. Specific Adapters are any that do not use a standardized protocol (including ABIs). We generally prefer Generic Adapters, as they produce less coupling.

It's important to note that Ports are generally defined in the application itself, while Adapters could be defined by the application or by consumers of the application. Generic Adapters are often defined by the application. Specific ones may be defined by the consumers, often by using the Core as a library. Critically, in both cases, the Ports are never "aware" of the Adapters.

If all of the applications in your ecosystem are defined this way, then each application can use or define Adapters to consume the Ports of its dependencies, and it enforces a layering of separated concerns.

One way to determine quickly if your application is correctly structured is to look at its imports. Imports should only ever go one direction – the Core should never import an Adapter for one of its own Ports. Likewise, Ports should never import Adapter modules. At a larger scale, providers of Ports and/or Adapters should never import directly from their consumers.
### Schema-based development

The hexagonal core of your service is very likely to need to manipulate some sort of data.

If it has a persistent data store, the service that provides it, whether it be the file-system or a database server, is external to your service itself. Therefore, it should be behind an adapter. The persistent data store service behind the adapter, or the adapter itself, internally, may impose a particular data structure on the data (*e.g.*, a go struct). Because we want to be fundamentally decoupled from that internal representation, what should the adapter provide to the core?

It's the persistent data store adapter's job to translate data from data store into an **internal representation** that the core manipulates. There are many ways to define the internal representation, but we've decided to leverage [Protocol Buffers](https://protobuf.dev/), due to their extensive tooling ecosystem, cross-language features, and versioning.

**Schema-based development** is the idea that you define your data schema early, and make it public, then build your application around that schema. It also means that the "internal" representation is not purely internal, it is part of the Ports of the application that other applications can consume.

Protocol buffer [messages types](https://protobuf.dev/programming-guides/proto3/#simple) define the data. Protocol buffer [services](https://protobuf.dev/programming-guides/proto3/#services) _can_ be used to define Ports, but they are certainly not the only way.

Landscape's protocol buffer definitions are [all in one repository](https://github.com/canonical/landscape-proto). This allows for various applications possibly implemented in different programming languages to share the same definitions (remember, they define the Ports) and for the holistic generation of things like OpenAPI specifications and documentation. It also allows for the schemas to be versioned together.
### Consumer-driven interfaces

This one is a little bit Go-specific, because it is mainly possible due to the nature of interfaces in Go. The general concept is that any given Core defines its dependencies as interfaces, and the Core itself owns those interfaces.

A small example:

```go
package demo

type Worker struct {
	logger Logger
}

type Logger interface {
	Error(msg string, args ...any)
	Info(msg string, args ...any)
}

func NewWorker(logger Logger) *Worker {
    return &Worker{logger: logger}
}
```

In this example, `Worker` has a dependency, `logger`, with type `Logger`, an interface. Instead of defining `Logger` somewhere else, say, where one or more implementers are defined, it is defined alongside its consumer, `Worker`.

There are a few reasons to do things this way:

- It makes it clear when looking at module's code how to implement providers for its dependencies.
- It ensures that the interface is only as large as it needs to be. The module only defines the methods it uses, rather than depending on an interface that has "extras".
- The module itself can easily define mock or default implementations without needing to rely on another module

Just because an interface is defined by the consumer doesn't mean it can't match an existing interface. The `Logger` example above is a subset of the interface of Go's [slog](https://pkg.go.dev/log/slog). This implies a default implementation, and saves us from "inventing" new interfaces unnecessarily.
## An example service

*This section is a work-in-progress*

Let's use an example: a small username/password authentication service. Its core logic can be defined as:

- given a username and password, validate that the password is correct for the username, then provide an authentication token

This sounds like a pretty simple service! In fact, a very basic implementation could probably be a single function. However, if we follow hexagonal design and schema-based development, the service will be designed in such a way that it will be easy to integrate with, and easy to swap out the "concrete" aspects of its implementation.
### The schema

The first step is to do the schema design, establishing the data representation. From the core logic definition above, we can extract three pieces of data:

1. username - a string, probably subject to some formatting rules
2. password - a string, also likely subject to some formatting rules
3. authentication token - likely also a string, an encoded representation of some binary data

Because the username and password need to be provided together, we'll need a data structure that contains them both. It's also a good idea to have a data structure for the authentication token, as there may be metadata fields that go along with it, but also because having a structure makes it easier to manipulate as a custom type.

Here's the protocol buffer definition for our data:

```protobuf
message Login {
	string username = 1;
	string password = 2;
}

message AuthToken {
	string data = 1;
}
```

Our Core will use the code generated from these definitions as the types it receives and sends on its Ports. To translate this to Go, they will be the parameter and return types of the Core struct's methods.
### The Core public API

As this is a Go service, the public API of our Core should be defined as methods that implement an interface:

```go
// internal/domain/service.go - the interface.
package domain

import (
	"context"

	pb "github.com/canonical/landscape-go-demo/gen/authservicepb"
)

// AuthServicePort is the inbound port interface for AuthService.
type AuthServicePort interface {
	Authenticate(ctx context.Context, login *pb.Login) (*pb.AuthToken, error)
}

// Consumer-driven interface definitions for Adapters.
type TokenAdapter interface {
	CreateToken(ctx context.Context, username string) (*pb.AuthToken, error)
}
```

The implementation implements the interface and depends upon any Adapters:

```go
// internal/domain/impl.go - the implementation.
package domain

import (
	"context"

	pb "github.com/canonical/landscape-go-demo/gen/authservicepb"
)

// AuthServiceImpl implements AuthServicePort.
type AuthServiceImpl struct {
	tokenAdapter TokenAdapter
}

// NewAuthService creates a new AuthServiceImpl.
func NewAuthService(tokenAdapter TokenAdapter) *AuthServiceImpl {
	return &AuthServiceImpl{
		tokenAdapter: tokenAdapter,
	}
}

// The implementation of Authenticate on AuthServiceImpl.
func (s *AuthServiceImpl) Authenticate(ctx context.Context, login *pb.Login) (*pb.AuthToken, error) {
	// Authentication logic here, then...
	
	return s.tokenAdapter.CreateToken(ctx, login.GetUsername())
}
```

In the `main` function for the application, the implementations are stitched together:

```go
// cmd/server/main.go
package main

import (
	"github.com/canonical/landscape-go-demo/internal/domain"
	"github.com/canonical/landscape-go-demo/internal/adapters/tokenadapter"
)

func main() {
	// Whatever config-collecting code is required, e.g., reading env vars.

	tokenAdapter, err := tokenadapter.New("SOME CONFIG VALUE")
	if err != nil {
		log.Fatalf("adapter configuration error are often fatal: %v", err)
	}
	
	svc := domain.NewAuthService(tokenAdapter)
	
	// Whatever service-running code is required, e.g., running a gPRC service.
}
```
## Resources

- [Landscape's protocol buffer repository](https://github.com/canonical/landscape-proto)