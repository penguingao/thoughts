# C++20 coroutine support

C++20 introduced compiler primitives to support
[coroutines](https://en.cppreference.com/cpp/language/coroutines). Coroutines
look like functions, but they do not reside on the stack.

If a function contains `co_await`, `co_yield` or `co_return`, it is a coroutine.

One of the main reasons to write asynchronous code in C++ is to handle external
events, typically I/O, because blocking I/O would otherwise require a dedicated
thread for each logical sequence of operations.

In network proxies like Envoy, or networking protocol code like
QUICHE, the predominant paradigm is event-driven programming, which naturally
comes with many callbacks.

While performant, this approach inevitably leads to "callback hell" as the
software grows in complexity. Callbacks are difficult to understand for several
reasons:

-   They fragment a sequence of events into many functions, making it harder to
    trace and see the big picture.
-   They make debugging difficult because the call stack doesn't naturally carry the
    "why" of an event. We only know "how" a callback is delivered, but by the
    time it is delivered, it is usually hard to know what
    sequence of events led to it.
-   They are prone to re-entrancy bugs. While an entire
    sequence is often handled on a single thread, calling a function
    carries the risk that the function eventually re-enters the originating
    class. At the point of re-entry, the state of the class might be
    inconsistent. After returning from the function call, the internal state
    might also have changed.
-   Object lifecycle management is complex. When an object registers a callback with another
    object, it must cancel the callback if it is destructed.
    This is usually documented in the contract between the
    objects, but it is hard to test or enforce.

Coroutines make it possible to write blocking I/O style procedures. They avoid
callback hell if used properly:

-   The code is more streamlined.
-   Each time a coroutine is created and awaited, the caller's handler
    is given to the coroutine, making it possible to trace back to the originator
    of the call.
-   It allows the use of function-local variables to keep track of
    state between async events, which reduces re-entrancy bugs. Even if
    the same function is called again, the variables are local to a new
    instance of the coroutine.
-   Local variables are destroyed only when the coroutine finishes,
    making resource management safer and easier.

# Envoy's HTTP filters

Envoy's HTTP filters make Envoy's functionality modular and composable.
Over time, many callbacks and features have been added. Each
HTTP filter might have several lifecycle callbacks, plus a few HTTP event
callbacks. The interfaces have grown
[big enough](https://source.corp.google.com/piper///depot/google3/third_party/envoy/src/envoy/http/filter.h;rcl=945155946;l=1)
that they are not very friendly to new contributors, while also
increasing the cognitive load for reviewers who must track "safe"
versus "dangerous" callbacks.

# Sequential HTTP Filters with Coroutines
To implement a filter that buffers and parses a bunch of data before it decides
to forward headers to the next filters, we write something like
[this](https://github.com/envoyproxy/envoy/blob/main/source/extensions/filters/http/ai_protocol_manager/filter.cc).

Wouldn't it be nice if we can write the following code instead?

```c++
enum class DecodeResult {
  // the filter has done its job. any further header, body, trailer events skip
  // this filter.
  kSuccess,
  // the filter wants to reset and fail the stream.
  kReset,
  // similar to the current recreateStream callback, it restarts the filter
  // chain.
  kRecreateStream,
};
HttpDecoder decode(HeaderGetter get_headers, HeaderForwarder forward_headers) {
  // status: to signal stream reset or any other failures of the request
  // headers: a shared_ptr of the header map
  // header_action_token: can only be used once during forward_headers
  auto [header_status, headers, header_action_token, data_generator] = co_await get_headers();

  DataParser parser;
  for co_await (InstancePtr data : data_generator) {
    if (absl::Status status = co_await parser.feed(data); !status.ok()) {
      co_return DecodeResult::kReset;
    }
    if (parser.hasEnoughData()) {
      break;
    }
  }
  if (!parser.hasEnoughData()) {
    co_return DecodeResult::kReset;
  }
  auto [status, forward_data] = forward_headers(std::move(header_action_token));
  if (!status.ok()) {
    co_return DecodeResult::kReset;
  }
  for co_await (InstancePtr data : parser.bufferedData()) {
    if (absl::Status status = co_await forward_data(data); !status.ok()) {
      co_return DecodeResult::kReset;
    }
  }
  // we don't care about future data and trailers (if any), so they just pass through
  co_return DecodeResult::kSuccess;
}
```

We can also imagine simple filters really just read the headers and `co_return`:

```c++
HttpDecoder decode(HeaderGetter get_headers, HeaderForwarder forward_headers) {
  auto [header_status, headers, header_action_token, data_generator] = co_await get_headers();
  // do things with headers

  if (auto [status, forward_data] = forward_headers(std::move(header_action_token)); !status.ok()) {
    co_return DecodeResult::kReset;
  }
  co_return DecodeResult::kSuccess;
}
```

We can define a similar coroutine on the response path: `HttpEncoder
encode(...)`.

Writing filters this way has several advantages:

- It makes it explicit that header processing occurs before data, which occurs before trailers.
- It is a compile-time error to forward headers twice.
- It gives the filter control over how much to buffer per stream.
- The filter can freely inject data.
- It does not suffer from re-entrancy and lifecycle problems. State is local to
  this coroutine and is safely freed as soon as `co_return` occurs.

Obviously, we need access to `StreamInfo`, configuration, route tables, etc.
These functionalities should be provided via synchronous function calls that do
not suffer from re-entrancy issues.

We also need asynchronous functionality like local replies and state sharing between
the decode and encode paths.

Furthermore, the API needs refinement to support these features,
along with timeout support. We could allow each `co_await` call to accept a timeout.

But hopefully, this gives a rough idea of how things would look.

# More Details

