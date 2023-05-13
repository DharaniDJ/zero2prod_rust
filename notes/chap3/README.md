# Chapter3 - Sign up a New Subscriber

We are starting a new project from scratch - there is a fair amount of upfront heavy-lifting we need to take
care of:
- choose a web framework and get familiar with it;
- define our testing strategy;
- choose a crate to interact with our database (we will have to save those emails somewhere!);
- define how we want to manage changes to our database schemas over time (a.k.a. migrations);
- actually write some queries.

## Choosing a Web Framework

actix-web is one of Rust’s oldest frameworks. It has seen extensive production usage, and it has built a large
community and plugin ecosystem; last but not least, it runs on tokio, therefore minimising the likelihood
of having to deal with incompatibilities and interop between different async runtimes.
actix-web will therefore be our choice for Zero To Production.

## Our First Endpoint: A Basic Health Check

Let’s try to get off the ground by implementing a health-check endpoint: when we receive a GET request for
/health_check we want to return a 200 OK response with no body.

We can use /health_check to verify that the application is up and ready to accept incoming requests.
Combine it with a SaaS service like pingdom.com and you can bealerted when your API goes dark - quite a
good baseline for an email newsletter that you are running on the side.

A health-check endpoint can also be handy if you are using a container orchestrator to juggle your application (e.g. Kubernetes or Nomad): the orchestrator can call /health_check to detect if the API has become
unresponsive and trigger a restart

### Anatomy of an actix-web Application

#### **Server - HttpServer**
HttpServer is the backbone supporting our application. It takes care of things like:
- where should the application be listening for incoming requests? A TCP socket
(e.g. 127.0.0.1:8000)? A Unix domain socket?
- what is the maximum number of concurrent connections that we should allow? How many new
connections per unit of time?
- should we enable transport layer security (TLS)?
- etc.

HttpServer, in other words, handles all transport level concerns.

What happens afterwards? What does HttpServer do when it has established a new connection with a client
of our API and we need to start handling their requests?

That is where App comes into play!

#### **Application - App**
App is where all your application logic lives: routing, middlewares, request handlers, etc.

App is the component whose job is to take an incoming request as input and spit out a response.

App is a practical example of the builder pattern: new() gives us a clean slate to which we can add, one bit at
a time, new behaviour using a fluent API (i.e. chaining method calls one after the other).

#### **Endpoint - Route**
route takes 2 parameters:
- path, a string, possibly templated (e.g. "/{name}") to accommodate dynamic path segments;
- route, an instance of the Route struct.

Route combine a handler with a set of guards.

Guards specify conditions that a request must satisfy in order to “match” and be passed over to the handler.

App iterates over all registered endpoints
until it finds a matching one (both path template and guards are satisfied) and passes over the request object
to the handler.

A type implements the Responder trait if it can be converted into a HttpResponse - it is implemented off the
shelf for a variety of common types (e.g. strings, status codes, bytes, HttpResponse, etc.).

#### **Runtime - tokio**
We need main to be asynchronous because HttpServer::run is an asynchronous method
but main, the entrypoint of our binary, cannot be an asynchronous function. Why is that?

Asynchronous programming in Rust is built on top of the Future trait: a future stands
for a value that may not be there yet. All futures expose a poll method which has to be called
to allow the future to make prograss and eventually resolve to its final value.
You can think of Rust's futures as lazy: unless polled, there is no guarantee that they
will execute to completion. This has often been described as a pull model compared to the
push model adopted by other languages.

Rust's standard library,by design, does not include an asynchronous runtime: you are supposed
to bring one into your project as a dependency, one more crate under [dependencies] in your
Cargo.toml. This approach is extremely versatile: you are free to implement your own runtime,
optimised to cater for the specific requirements of your usecase.

Then who is in charge to call poll on it? There is no special configuration syntax that tells the
Rust compiler that one of your dependencies is an asynchronous runtime and, to be fair,
there is not even a standardised defination of what a runtime is.

You are therefore expected to launch your asynchronous runtime at the top of your main function and then
use it to drive your futures to completion.

tokio::main is a procedural macro and this is a great opportunity to introduce cargo expand, an awesome
addition to our Swiss army knife for Rust development.

Rust macros operate at the token level: they take in a stream of symbols (e.g. in our case, the whole main
function) and output a stream of new symbols which then gets passed to the compiler. In other words, the
main purpose of Rust macros is **code generation**.

cargo expand: it expands all macros in your code without passing the output to the compiler, allowing
you to step through it and understand what is going on.

cargo-expand relies on the nightly compiler to expand our macros.

We are starting tokio’s async runtime and we are using it to drive the future returned by HttpServer::run
to completion.

In other words, the job of #[tokio::main] is to give us the illusion of being able to define an asynchronous
main while, under the hood, it just takes our main asynchronous body and writes the necessary boilerplate
to make it run on top of tokio’s runtime.

## Integration Test
/health_check was our first endpoint and we verified everything was working as expected by launching the
application and testing it manually via curl.

Manual testing though is time-consuming: as our application gets bigger, it gets more and more expensive to manually check that all our assumptions on its behaviour are still valid every time we perform some changes. We’d like to automate as much as possible: those checks should be run in our CI pipeline every time we are committing a change in order to prevent regressions.

### How do we test an Endpoint?
An API is a means to an end: a tool exposed to the outside world to perform some kind of task (e.g. store a document, publish an email, etc.).

The endpoints we expose in our API define the contract between us and our clients: a shared agreement about the inputs and the outputs of the system, its interface.

The contract might evolve over time and we can roughly picture two scenarios: - backwards-compatible changes (e.g. adding a new endpoint); - breaking changes (e.g. removing an endpoint or dropping a field from the schema of its output).

In the first case, existing API clients will keep working as they are. In the second case, existing integrations are likely to break if they relied on the violated portion of the contract.

While we might intentionally deploy breaking changes to our API contract, it is critical that we do not break it accidentally.

What is the most reliable way to check that we have not introduced a user-visible regression?
Testing the API by interacting with itin the same exact way a user would: performing HTTP requests against it and verifying our assumptions on the responses we receive.

This is often referred to as black box testing: we verify the behaviour of a system by examining its output given a set of inputs without having access to the details of its internal implementation.

actix-web providessome conveniencesto interact with an Appwithout skipping the routing logic, but there are severe shortcomings to its approach.

- migrating to another web framework would force us to rewrite our whole integration test suite. As much as possible, we’d like our integration tests to be highly decoupled from the technology underpinning our API implementation (e.g. having framework-agnostic integration tests is life-saving when you are going through a large rewrite or refactoring!);

-  due to some actix-web’s limitations4, we wouldn’t be able to share our App startup logic between our production code and our testing code, therefore undermining our trust in the guarantees provided by our test suite due to the risk of divergence over time.

We will opt for a fully black-box solution: we will launch our application at the beginning of each test and interact with it using an off-the-shelf HTTP client (e.g. reqwest).