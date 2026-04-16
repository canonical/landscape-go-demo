## Hexagonal design

A great starting point is to [read this blog post by Herberto Graca](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/), but here's a quick summary.

Hexagonal design emphasizes designing software such that it has a "core" and everything outside of that core is loosely coupled through "ports" and "adapters".

The **Core** is the business logic. It contains the entities that allow the application to perform its function.

**Ports** are the defined public connection points that can be used to drive the application. An example of a Port could be a public method or function of a package's Core. It says nothing about _who_ consumes it, and is designed to make the most sense for the Core itself. Ports shouldn't contort themselves to be convenient for the consumer.

**Adapters** are the concrete implementations that allow specific consumers to use the application's Ports. An example may be an HTTP or GRPC service that translates those types of requests into operations on the Ports.

It's important to note that Ports are generally defined in the application itself, while Adapters could be defined by the application or by consumers of the application. In the case of very common or "obvious" ones, the Adapters are often defined by the application. More niche ones may be defined by the consumers, often by using the Core as a library. Critically, in both cases, the Ports are never "aware" of the Adapters.

If all of the applications in your ecosystem are defined this way, then each application can use or define Adapters to consume the Ports of its dependencies, and it enforces a layering of separated concerns.
## Schema-based development

The hexagonal core of your service is very likely to need to manipulate some sort of data.

If it has a persistent data store, the service that provides it, whether it be the file-system or a database server, is external to your service itself. Therefore, it should be behind an adapter. The persistent data store service behind the adapter, or the adapter itself, internally, may impose a particular data structure on the data (*e.g.*, a go struct). Because we want to be fundamentally decoupled from that internal representation, what should the adapter provide to the core?

It's the persistent data store adapter's job to translate data from data store into an **internal representation** that the core manipulates. There are many ways to define the internal representation, but we've decided to leverage [Protocol Buffers](https://protobuf.dev/), due to their extensive tooling ecosystem, cross-language features, and versioning.

**Schema-based development** is the idea that you define your data schema early, and make it public, then build your application around that schema. It also means that the "internal" representation is not purely internal, it is part of the Ports of the application that other applications can consume.
## An example service

Let's use an example: a small username/password authentication service. Its core logic can be defined as:

- given a username and password, validate that the password is correct for the username, then provide an authentication token

This sounds like a pretty simple service! In fact, a very basic implementation could probably be a single function. However, if we follow hexagonal design and schema-based development, the service will be designed in such a way that it will be easy to integrate with, and easy to swap out the "concrete" aspects of its implementation.
### The schema

The first step is to do the schema design, establishing the data representation. From the core logic definition above, we can extract three pieces of data:

1. username - a string, probably subject to some formatting rules
2. password - a string, also likely subject to some formatting rules
3. authentication token - likely also a string, an encoded representation of some binary data

Because the username and password need to be provided together, we'll need a data structure that contains them both. It's also a good idea to have a data structure for the authentication token, as there may be metadata fields that go along with it, but also because having a structure makes it easier to manipulate as a custom type.

Here's our protocol buffer definition:

```protobuf
message Login {
	string username = 1;
	string password = 2;
}

message AuthToken {
	string data = 1;
}
```
## Resources

- [Landscape's protocol buffer repository](https://github.com/canonical/landscape-proto)

### Early feedback notes

- the protos repository - are the services de-coupled from their schema?
	- should mention the rationale(s) for having all of the protos in a single repository
	- mentioning how public API is exhaustively defined by the protos and is therefore back-end language agnostic
- hexagonal - include some write-up about using module/package imports as an indicator of whether you are violating the domain layering of the application
- are we going to standardize the definition of ports within the service layer
	- could be part of the protocol buffer definition (via the service definitions)
- consumer-driven interfaces