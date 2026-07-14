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
                   TerminatingActions terminating_actions, Context) {
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
                   TerminatingActions, Context) {
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
    `HeaderGetter`. This makes some incorrect orderings unrepresentable.
-   We use an rvalue ref-qualifier on `operator()` (`operator() (...) &&`) to
    signal that an action is consumed by the call, e.g.
    `std::move(forward_header)()`.

These rule out some misorderings, but they are not a full linear-type system: a
ref-qualifier does not stop `std::move(x)()` from being called twice, so
double-use is caught at runtime (subsequent calls return an error), not at
compile time. Clang's consumable-type attributes (`[[clang::consumable]]`,
`[[clang::callable_when]]`, `[[clang::set_typestate]]`) could push single-use
enforcement to compile time under Envoy's warnings-as-errors; this is worth
validating.

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
by `Context`.

## (Somewhat) symmetric response handling and coordination

We can write response handling in a similar way, which brings the necessity of
sharing state between `decode()` and `encode()`. This should be handled by
`Context` providing some async message passing primitives.

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
the decode path (request), we take use of the Context object. In the wrapper
filter, we provide a base class that provides `StreamInfo` and the likes. A
filter can choose to implement a subclass of Context that provides additional
methods or awaitable coroutines to pass information from decode path to encode
path. A typical filter should be content with synchronous setter and getter
methods. In the rare case async behavior is needed, coroutines can be implemented.

## Coroutine life cycle

The coroutine owns its own frame: it is only ever destroyed at `final_suspend`,
never torn down mid-flight by the wrapper filter. This keeps memory management
safe — no frame is freed while the coroutine is suspended or running — and it
lets a coroutine outlive the HTTP stream when it needs to, to finish background
work after the stream is gone.

Two signals drive the life cycle, and keeping them distinct matters:

-   **Stream liveness** — whether the coroutine can still make progress on the
    stream. It dies when the wrapper filter / stream is torn down.
-   **Frame ownership** — who is responsible for destroying the frame once the
    coroutine completes.

### Reset: fail fast on stream operations, leave everything else

When the stream is reset, only the awaits the coroutine issued to the stream
infrastructure (`HeaderGetter` / `DataGenerator` / `DataForwarder` /
`TrailerGetter` / `TerminatingActions`) are aborted. Once the stream is gone
these awaitables fail fast — `await_ready()` returns `true` and the await resumes
immediately with an abort status — so the coroutine never re-suspends on a stream
operation. The next time its logic touches any stream op it gets the abort and
unwinds through a normal `co_return`, running all destructors (and
`absl::Cleanup`s), and awaiting further cleanup first if it wants to.

We deliberately do **not** chase down other pending awaits (a file read, a
timer). The coroutine rides those to their normal completion and then reaches a
stream op, which aborts it. This is exactly what lets the same mechanism serve an
intentional detach — a coroutine that wants to keep running after the stream is
gone simply keeps awaiting non-stream operations and never touches the dead
stream again — without any explicit `detach()` call.

This rests on one contract: **every non-stream awaitable the coroutine can park
on must eventually complete or be cancellable.** The default `Task` timeout (see
Timeouts) normally satisfies it — an awaited operation aborts on its own after
the timeout. The trap is an awaitable that escapes that default and whose
producer died with the stream (for example, a decode↔encode `Context` primitive
whose peer is gone) — it must supply its own completion/abort, or the coroutine
parks forever.

### Ownership and destruction

Whether the wrapper filter (the parent) will destroy the frame is decided at
`final_suspend`. The parent holds a `shared_ptr` ownership token; the coroutine
holds a `weak_ptr` to it, and checks it at `final_suspend`:

-   token still owned → `suspend_always`: the owner reads the `PostDecodeAction`
    and calls `destroy()`.
-   token gone → `suspend_never`: the frame destroys itself, because nobody is
    left to read the result.

When the wrapper filter is destroyed with the coroutine still pending, it does
not destroy the coroutine — it **hands the task to the pending-task registry**,
moving the ownership token with it. Because ownership is *transferred* rather than
dropped, the `weak_ptr` stays valid throughout, so there is no window in which a
coroutine reaches `final_suspend` with no one to free it. The registry is now the
owner and destroys the frame when the coroutine completes.

### The registry is a watchdog; the default is to crash

The pending-task registry is primarily a **watchdog**, not a garbage collector.
Every task it holds has a deadline. A task that overstays is a bug — a coroutine
that forgot to terminate, or one parked on a non-completing await — and the
**default action is to crash the process**, so newly introduced lifecycle bugs
surface immediately in tests and canaries instead of leaking silently in
production. (A softer action — force-`destroy()` plus a stat — can be selected
where a crash is unacceptable; force-destroy runs only RAII cleanup, since the
frame is destroyed while suspended.)

The registry is also the observability and drain hook: it exports the count of
in-flight detached tasks, e2e tests assert it drains to zero, and worker drain
enumerates it to force-terminate anything still alive.

### What a coroutine may touch after reset

`Context` is always safe to call, though it may return an error. Every other
action (getters, forwarders, terminating actions) returns an error once the
stream is reset. Cleanup that must run on reset should therefore be expressed as
RAII (destructors / `absl::Cleanup`), which run on the normal `co_return` unwind
and may safely use `Context`.

## Re-entrancy

The wrapper filter's implementation will need to be carefully written to avoid
re-entrancy to itself while it interacts with HCM / filter manager. Where
possible, tail call should be used to make sure the code is reentrant safe. This
is the biggest risk of this design.
