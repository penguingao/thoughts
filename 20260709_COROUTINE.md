**tl;dr;** C++20 provides primitives to implement coroutine. Coroutine enables
writing asynchronous logic in sequential style, which is easier to reason about.
This document outlines a common Envoy wrapper HTTP filter that makes it possible
to write HTTP filter logic in coroutines. It makes use of compiler semantics to
enforce correct sequence of actions that a filter can perform. The goal is to
lower cognitive load for code authors and reviewers so that they can focus on
higher level goals than Envoy internals.

# C++20 coroutine support

C++20 introduced compiler primitives to support
[coroutines](https://en.cppreference.com/cpp/language/coroutines). Coroutines
look like functions, but they do not reside on the stack. It allows writing
asynchronous code as sequential-looking functions, making it easier to reason
about.

If a function contains `co_await`, `co_yield` or `co_return`, it is a coroutine.

# The callback hell

One of the main reasons to write asynchronous code in C++ is to handle external
events, typically I/O, because blocking I/O would otherwise require a dedicated
thread for each logical sequence of operations.

In network proxies like Envoy, or networking protocol libraries like QUICHE, the
predominant paradigm is event-driven programming, which naturally comes with
many callbacks.

While performant, this approach inevitably leads to "callback hell" as the
software grows in complexity. Callbacks are difficult to understand for several
reasons:

-   They fragment a sequence of event handling logic into many functions, making
    it harder to trace and see the big picture.
-   This naturally requires a lot of state stored in objects for functions to
    understand the context.
-   The many callbacks make debugging difficult because the call stack doesn't
    naturally carry the "why" of an event. We only know "how" a callback is
    delivered, but by the time it is delivered, it is usually hard to know what
    sequence of events led to it.
-   They are prone to re-entrancy bugs. While an entire sequence is often
    handled on a single thread, calling a function carries the risk that the
    function eventually re-enters the originating class. At the point of
    re-entry, the state of the class might be inconsistent. After returning from
    the function call, the internal state might also have changed.
-   Object lifecycle management is complex. When an object registers a callback
    with another object, it must cancel the callback if it is destructed. This
    is usually documented in the contract between the objects, but it is hard to
    test or enforce.

Coroutines make it possible to write blocking I/O style procedures. They avoid
the "callback hell" if used properly:

-   The code is more streamlined.
-   Each time a coroutine is created and awaited, the caller's handler is given
    to the coroutine, making it possible to trace back to the originator of the
    call.
-   It allows the use of function-local variables to keep track of state between
    async events, which reduces re-entrancy bugs. Even if the same function is
    called again, the variables are local to a new instance of the coroutine.
-   Local variables are destroyed only when the coroutine finishes, making
    resource management safer and easier.

# Envoy's HTTP filters

Envoy's HTTP filters make Envoy's functionality modular and composable. Over
time, many callbacks and features have been added. Each HTTP filter might have
several lifecycle callbacks, plus a few HTTP event callbacks. The interfaces
have grown
[big enough](https://github.com/envoyproxy/envoy/blob/main/envoy/http/filter.h)
that they are not very friendly to new contributors, while also increasing the
cognitive load for reviewers who must track "safe" versus "dangerous" callbacks.

# Sequential HTTP Filters with Coroutines

To implement a filter that buffers and parses a bunch of data before it decides
to forward headers to the next filters, we write something like
[this](https://github.com/envoyproxy/envoy/blob/main/source/extensions/filters/http/ai_protocol_manager/filter.cc).

Wouldn't it be nice if we can write the following code instead?

```c++
HttpDecoder decode(HeaderGetter get_headers, HeaderForwarder forward_headers,
                   TerminatingActions terminating_actions, RequestInfo) {
  RequestHeaders headers;
  DataGenerator data_generator;
  ASSIGN_OR_CO_RETURN(std::tie(headers, data_generator),
    co_await std::move(get_headers)());

  // parser uses a read-only view of the headers so it stays valid after the
  // headers are forwarded (moved) below.
  DataParser parser(headers.read_only_headers());
  while (!parser.hasEnoughData()) {
    ASSIGN_OR_CO_RETURN(
      Buffer::InstancePtr data, co_await data_generator.next());
    // end stream signalled via nullptr. parser hasn't seen enough. error out.
    if (data == nullptr) {
      co_return std::move(terminating_actions).replyLocally(Http::Code::BadRequest);
    }
    if (absl::Status status = co_await parser.feed(std::move(data));
        !status.ok()) {
      co_return std::move(terminating_actions).replyLocally(Http::Code::BadRequest);
    }
  }

  // start proxying / forwarding.
  DataForwarder forward_data;
  ASSIGN_OR_CO_RETURN(
    forward_data, std::move(forward_headers)(std::move(headers)));

  absl::StatusOr<Buffer::InstancePtr> buffered_data = parser.bufferedData();
  while (buffered_data.ok() && *buffered_data != nullptr) {
    if (absl::Status status = co_await forward_data(*std::move(buffered_data));
        !status.ok()) {
      co_return std::move(terminating_actions).replyLocally(Http::Code::InternalError);
    }
    buffered_data = parser.bufferedData();
  }
  if (!buffered_data.ok()) {
    co_return std::move(terminating_actions).replyLocally(Http::Code::InternalError);
  }

  // Finish body forwarding and transition to trailer mode. This does not end
  // the stream; the stream ends when forward_trailers is invoked below.
  ASSIGN_OR_CO_RETURN(
    TrailerForwarder forward_trailers, co_await std::move(forward_data)());

  // can be called before end stream.
  TrailerGetter get_trailers = std::move(data_generator).finalize();
  ASSIGN_OR_CO_RETURN(
    std::optional<RequestTrailers> trailers, co_await std::move(get_trailers)());
  if (trailers.has_value()) {
    std::move(forward_trailers)(std::move(trailers));
  }

  co_return PostDecodeAction::kSkip;
}
```

We can also imagine simple filters really just read the headers and `co_return`:

```c++
HttpDecoder decode(HeaderGetter get_headers, HeaderForwarder,
                   TerminatingActions, RequestInfo) {
  RequestHeaders headers;
  ASSIGN_OR_CO_RETURN(
    std::tie(headers, std::ignore), co_await std::move(get_headers)());

  // do things with headers
  co_return PostDecodeAction::kSkip;
}
```

Obviously, it can be broken into a few helper functions and coroutines to make
it more readable, but putting it all in one body signifies the sequential nature
of HTTP processing in this paradigm.

## Error handling

Fallible steps return `absl::Status` / `absl::StatusOr`. Two kinds of failure
show up, and they call for different handling:

-   Stream / infrastructure failures. Getters and forwarders return an error once
    the stream can no longer make progress (for example, after it is reset). The
    response is always the same — stop and return the filter's default action —
    so these are propagated with `ASSIGN_OR_CO_RETURN` and need no per-site
    handling.
-   The filter's own domain errors, such as a parse failure. These call for
    different responses (a `BadRequest` versus an `InternalError` local reply),
    so they are handled with an explicit branch.

Two conveniences keep the common case terse. The coroutine's promise carries a
default `PostDecodeAction` (`kReset`) that `ASSIGN_OR_CO_RETURN` returns on error,
so propagation sites do not repeat it. And `TerminatingActions::replyLocally`
returns the `PostDecodeAction` to `co_return`, collapsing the reply-then-return
pair into a single statement.

## Taking advantage of the type system

There are a couple of semantic tricks worth pointing out:

-   The next action a filter can take is obtained from acting on the previous
    one. e.g. `DataGenerator` can only be obtained by reading headers off
    `HeaderGetter`. This makes incorrect or out-of-sequence orderings unrepresentable in code.
-   We use an rvalue ref-qualifier on `operator()` (`operator() (...) &&`) to
    signal that an action is consumed by the call, e.g.
    `std::move(forward_header)()`.

By requiring explicit `std::move(...)` to yield or transition actions, we leverage
the C++ compiler to enforce affine/linear state transitions. This shifts control-flow
correctness verification from fallible human code reviews to automated compiler checks:
a developer cannot accidentally execute actions out of sequence or bypass required intermediate steps.

While rvalue ref-qualifiers enforce that lvalues cannot be reused directly without `std::move`,
a ref-qualifier alone does not prevent `std::move(x)()` from being called twice on a moved object;
in the runtime implementation, subsequent calls return an error. To elevate single-use
enforcement to compile-time errors, we combine Clang's consumable-type attributes
(`[[clang::consumable]]`, `[[clang::callable_when]]`, `[[clang::set_typestate]]`) with static analysis
(see [Static analysis & memory safety enforcements](#static-analysis--memory-safety-enforcements) in More Details).

## Buffering and flow control.

Aside from correctness, the use of coroutine between `DataGenerator` and
`DataForwarder` gives the filter explicit control of how much data it buffers.
Before forwarding, it actively reads more data if it has budget and more is
needed. Because `DataForwarder` is `co_await`ed upon, remote flow control push
back is implicitly implemented.

## Stream life cycle

Early termination of the HTTP stream is surfaced through the return status of
`HeaderGetter` and friends: after a reset they abort, and the coroutine unwinds
the next time it touches a stream operation (see Coroutine life cycle below). The
lack of a class object here makes it harder to use callbacks, which prevents
re-entrance bugs too.

If the filter no longer cares about the request, it simply `co_return
PostDecodeAction::kSkip`.

## Other useful information to the filter implementation

Obviously, we need access to `StreamInfo`, configuration, route tables, etc.
These functionalities should be provided via synchronous function calls provided
by `RequestInfo`.

## (Somewhat) symmetric response handling and coordination

We can write response handling in a similar way, which brings the necessity of
sharing state between `decode()` and `encode()`. This should be handled by
`RequestInfo` providing some async message passing primitives.

## Timeouts

Writing "blocking"-looking code requires timeout support. Rather than forcing an
explicit `with_timeout()` wrapper onto every call, each `Task` carries a default
timeout, so an await cannot hang indefinitely by accident. `with_timeout()`
remains available to override the default — tighten or extend it — for a specific
call. The default timeout is also what makes the life-cycle contract hold in
practice: it bounds "every non-stream awaitable must eventually complete or be
cancellable", so a coroutine parked on a stalled operation aborts on its own well
before the pending-task watchdog has to step in.

Hopefully, this gives a rough idea of how things would look.

# More Details

If the code snippets look appealing, it would be intriguing to see how the
interfaces are declared. To avoid boiling the ocean, we can implement this as a
common wrapper http filter that bridges the current callback semantics to the
proposed sequential coroutine paradigm.

```c++

// The coroutine can co_return before end_stream, so it must declare how any
// remaining stream events are treated after it returns.
enum class PostDecodeAction {
  // The filter is done and future events bypass it: any further unread header,
  // body, or trailer events skip this filter.
  //
  // Headers and trailers that were read but not forwarded are forwarded
  // automatically. Data that was read but not forwarded is dropped.
  //
  // This is the normal successful completion — the filter has finished and lets
  // the rest of the stream pass through untouched.
  kSkip,
  // The filter wants to reset the stream on any further event. If no local reply
  // has started, any further event triggers a stream reset (HTTP/1.1 connection
  // closure, HTTP/2 stream reset). If a local reply has started there will be no
  // further decode events anyway.
  //
  // kSkip and kReset differ only in how future events are handled; they coincide
  // when a local reply guarantees there are no future events.
  kReset,
};

#define FORWARD_INLINE_HEADER_FUNCS(name)   \
  const HeaderEntry* name() const override { return backing_map_->name(); } \
  size_t remove##name() override { return backing_map_->remove##name(); } \
  absl::string_view get##name##Value() const override { return backing_map_->get##name##Value(); } \
  void set##name(absl::string_view value) override { backing_map_->set##name(value); }

#define FORWARD_INLINE_HEADER_STRING_FUNCS(name) \
  FORWARD_INLINE_HEADER_FUNCS(name) \
  void append##name(absl::string_view data, absl::string_view delimiter) override { \
    backing_map_->append##name(data, delimiter); \
  } \
  void setReference##name(absl::string_view value) override { \
    backing_map_->setReference##name(value); \
  }

#define FORWARD_INLINE_HEADER_NUMERIC_FUNCS(name) \
  FORWARD_INLINE_HEADER_FUNCS(name) \
  void set##name(uint64_t value) override { \
    backing_map_->set##name(value); \
  }

class RequestHeaders : public RequestHeaderMap {
private:
  RequestHeaderMapSharedPtr backing_map_;
public:
  INLINE_REQ_RESP_STRING_HEADERS(FORWARD_INLINE_HEADER_STRING_FUNCS)
  INLINE_REQ_RESP_NUMERIC_HEADERS(FORWARD_INLINE_HEADER_NUMERIC_FUNCS)
  INLINE_REQ_STRING_HEADERS(FORWARD_INLINE_HEADER_STRING_FUNCS)
  INLINE_REQ_NUMERIC_HEADERS(FORWARD_INLINE_HEADER_NUMERIC_FUNCS)

  // Provides a read-only view into the header map even after headers are
  // forwarded.
  RequestHeaderMapConstSharedPtr read_only_headers() const;

  // Cannot copy.
  RequestHeaders(const RequestHeaders&) = delete;
  RequestHeaders& operator=(const RequestHeaders&) = delete;

  // Moveable.
  RequestHeaders(RequestHeaders&& other) {
    backing_map_ = other.backing_map_;
    other.backing_map_ = nullptr;
  }
  RequestHeaders& operator=(RequestHeaders&& other) {
    backing_map_ = other.backing_map_;
    other.backing_map_ = nullptr;
    return *this;
  }
};

class HeaderGetter {
public:
  Coroutine::Task<absl::StatusOr<std::tuple<RequestHeaders, DataGenerator>>> operator() ()  &&;
  // other methods to interact with the real http filter.
};

class DataGenerator {
public:
  // Awaits the next chunk.
  //
  // Status signals error condition, and the filter should start cleaning itself
  // up.
  //
  // Buffer is nullptr when there is no more data. Further calls might return
  // either error status or nullptr.
  Coroutine::Task<absl::StatusOr<Buffer::InstancePtr>> next();

  // Signals that the filter is done with data processing. Remaining data (if
  // any) will pass through this filter.
  TrailerGetter finalize() &&;
};

class TrailerGetter {
public:
  Coroutine::Task<absl::StatusOr<std::optional<RequestTrailers>>> operator() () &&;
  // other methods to interact with the real http filter.
};

class HeaderForwarder {
public:
  absl::StatusOr<DataForwarder> operator() (RequestHeaders headers)&&;
};

class DataForwarder {
public:
  // Forwards a body chunk. Forwarding nullptr forwards an empty chunk; it does
  // NOT signal end of data (unlike a nullptr from `DataGenerator::next()`).
  Coroutine::Task<absl::Status> operator() (Buffer::InstancePtr data);

  // Finishes body forwarding and transitions to trailer mode, handing back a
  // TrailerForwarder. Neither overload ends the stream; the stream ends when the
  // TrailerForwarder is invoked. An optional final body chunk may be passed.
  Coroutine::Task<TrailerForwarder> operator() (Buffer::InstancePtr data = nullptr) &&;
};

class TrailerForwarder {
public:
  // Ends a stream with or without trailers.
  absl::Status operator() (std::optional<RequestTrailers> trailers) &&;
};

class TerminatingActions {
public:
  // Starts the local reply. Once a local reply is sent, `HeaderGetter` /
  // `DataGenerator` / `TrailerGetter` return an error for a non-terminating
  // filter; the filter should clean itself up and `co_return` the returned
  // action. A terminating filter can keep receiving the remainder of the
  // messages. Returns `PostDecodeAction::kReset` so it can be `co_return`ed
  // directly.
  PostDecodeAction replyLocally(Http::Code code) &&;

  // Attempts to recreate the filter chain. Once it has started (ok status),
  // `HeaderGetter` / `DataGenerator` / `TrailerGetter` return an error and the
  // filter should clean itself up.
  absl::Status recreateStream() &&;

  // more overloads can be provided to provide ease of use.
};

```

## Decode encode communication

In the case where the encode path (response) needs to get some information from
the decode path (request), we take use of the RequestInfo object. In the wrapper
filter, we provide a base class that provides `StreamInfo` and the likes. A
filter can choose to implement a subclass of RequestInfo that provides additional
methods or awaitable coroutines to pass information from decode path to encode
path. A typical filter should be content with synchronous setter and getter
methods. In the rare case async behavior is needed, coroutines can be implemented.

## Coroutine life cycle

The coroutine owns its own frame: it is only ever destroyed at `final_suspend`,
never torn down mid-flight by the wrapper filter. This keeps memory management
safe — no frame is freed while the coroutine is suspended or running. This is
chosen over silent destroy at suspention points `co_await` or `co_yield`. The
reasoning aligns with the choice of not supporting exception in Envoy on the
data plane.

This does call for advisory cancellation, meaning the outer most caller of a
coroutine has to wait until the coroutine is done before it frees resources that
might be used by the coroutines. In this case, it is the wrapper filter's job to
kick off the cancellation during `onDestroy`. The current design makes it so
that all stream operations transfer memory ownership to the coroutine. Other
informational objects (stream info, etc) are passed as `shared_ptr`.

We want to have a watch dog to make sure the filters do eventually finish the
clean-up.

We propagate the cancellation all the way down to the leaf of the coroutine
call. This achieves fail fast.

### Reset: fail fast on stream operations, leave everything else

When the stream is reset, only the awaits the coroutine issued to the stream
infrastructure (`HeaderGetter` / `DataGenerator` / `DataForwarder` /
`TrailerGetter` / `TerminatingActions`) are aborted. Once the stream is gone
these awaitables fail fast — `await_ready()` returns `true` and the await resumes
immediately with an abort status — so the coroutine never re-suspends on a stream
operation. The next time its logic touches any stream op it gets the abort and
unwinds through a normal `co_return`, running all destructors (and
`absl::Cleanup`s), and awaiting further cleanup first if it wants to.

Normally, the cancellation is propagated for any new asynchronous call to make
sure the unwind happens quickly. However, we can add support to suppress the
cancellation for newly launched coroutines for clean-ups.

### Ownership and destruction

Caller always destruct a coroutine it calls.

### Watch dog 

We need a watch dog to export pending cancellation coroutine filters to make
sure they eventually finish. Normally, the filters finish fast by completion
token.

### What a coroutine may touch after reset

`RequestInfo` is always safe to call, though it may return an error. Every other
action (getters, forwarders, terminating actions) returns an error once the
stream is reset. Cleanup that must run on reset should therefore be expressed as
RAII (destructors / `absl::Cleanup`), which run on the normal `co_return` unwind
and may safely use `RequestInfo`.

## Static analysis & memory safety enforcements

To complement compile-time type-state checks (`&&` ref-qualifiers) and prevent common C++20 coroutine memory hazards (such as holding non-owning references across `co_await` suspend points), the coroutine filter framework relies on automated static analysis in Envoy's build and CI toolchain.

### 1. Preventing dangling local references across `co_await`

In C++20 coroutines, storing non-owning reference types (e.g., `absl::string_view`, `std::string_view`, `T&`, raw pointers) in local variables across `co_await` suspend points creates Use-After-Free hazards if the underlying HTTP stream or header map is destroyed while suspended.

To enforce local variable safety automatically, a custom **Clang-Tidy AST Matcher** rule is configured in CI:

```c++
// Clang-Tidy AST matcher rule: flag non-owning local variables that persist across co_await
varDecl(
  isLocalVarDecl(),
  hasType(isNonOwningType()), // matches absl::string_view, std::string_view, T&, T*, std::span
  hasAncestor(coroutineBodyStmt()),
  isLiveAcrossSuspendPoint()
).bind("dangling_coroutine_local");
```

When triggered under Envoy's `-Werror` build rules, this AST matcher fails the build if a developer attempts to keep a string view or reference view across a suspend point, forcing them to store an owned type (`std::string`, `HeaderMapConstSharedPtr`) or re-query after resuming.

### 2. Clang lifetime attributes (`[[clang::lifetimebound]]` & `-Wdangling-gsl`)

The header interfaces and getters are annotated with Clang lifetime bounds:

```c++
class RequestHeaders {
public:
  absl::string_view getPathValue() const [[clang::lifetimebound]];
};
```

Building with `-Wdangling-gsl` enables Clang's GSL (Guidelines Support Library) lifetime safety analysis. The compiler treats `absl::string_view` as a `[[gsl::Pointer]]` bound to the `RequestHeaders` `[[gsl::Owner]]`. If the `RequestHeaders` instance is moved or consumed prior to a `co_await`, Clang issues a dangling reference compiler warning.

### 3. Compile-time typestate checks (`[[clang::consumable]]`)

To complement the rvalue ref-qualifiers (`operator()() &&`) discussed in [Taking advantage of the type system](#taking-advantage-of-the-type-system), action classes (`HeaderForwarder`, `DataForwarder`, `TrailerForwarder`) are annotated with Clang's consumable-type attributes:

```c++
class [[clang::consumable(unconsumed)]] HeaderForwarder {
public:
  absl::StatusOr<DataForwarder> operator()(RequestHeaders headers) &&
      [[clang::callable_when(unconsumed)]] [[clang::set_typestate(consumed)]];
};
```

This causes Clang static analysis to issue a compile-time error/warning if `std::move(forward_headers)(...)` is called more than once on the same object instance.

## Re-entrancy

The wrapper filter's implementation will need to be carefully written to avoid
re-entrancy to itself while it interacts with HCM / filter manager. Where
possible, tail calls should be used to make sure the code is reentrant safe. This
is the biggest risk of this design.
