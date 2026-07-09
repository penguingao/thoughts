# C++20 coroutine support

C++20 introduced compiler primitives to use
[coroutine](https://en.cppreference.com/cpp/language/coroutines). Coroutine
looks like functions, but they do not reside on the stack.

If a function contains `co_await`, `co_yield` or `co_return`, it is a coroutine.

One of the reasons to write asynchronous code in C++ is to deal with external
events, typically I/O, because blocking I/O would otherwise require dedicated
thread for each logical sequence operations.

In the case of network proxies like Envoy, or networking protocol code like
QUICHE, the predominant paradigm is event driven programming, which natually
comes with a lot of callbacks.

While being performant, it comes with the inevitable "callback hell" as the
software grows in complexity. Callbacks are hard to understand for a few
reasons:

-   it fragments a sequence of events into many functions, making it harder to
    trace and get the full picture
-   it makes debugging hard because the call stack doesn't natually carry the
    "why" of a event - we only know "how" a callback is delivered, but by the
    time the callback is delivered, it's usually hard to know what's the
    sequence of events that lead to this callback
-   it is prone re-entrance bugs. While it is often the case that an entire
    sequence is handled on a single thread, each time a function is called
    carries the risk of that funciton eventually re-enters the the orginating
    class. At the point of re-entering, the state of the class might be
    inconsisent. After returning from the function call, the internal state
    might also have changed
-   object life cycle management. When an object registers a callback to another
    object, if the orginating object needs to be destructed, it needs to cancel
    the callback. This is usually documented in the contract between the
    objects, but hard to test or enforce

Coroutine makes it possible to write blocking I/O style procedures. It gets
around the callback hell if used properly:

-   the code is more streamlined
-   each time a coroutine is created, and awaited upon, the caller's "handler"
    is given to the coroutine, making it possible to trace back the originator
    of the call
-   it makes it easier to just use function local variables to keep track of
    state between async events, so that there is less re-entrance bug - even if
    the same function is called again, the variables are all local to a new
    instance of the coroutine
-   all the local variables are deleted only when all the coroutines it calls
    return, so they are safe and easier to deallocate

# Envoy's HTTP filters

Envoy's HTTP filters make Envoy's functionalities modular and composable.
Overtime, a lot of callbacks and functionalities have been added to it. Each
HTTP filter might have a few life cycle callbacks, plus a few HTTP event
callbacks. The interfaces have grown
[big enough](https://source.corp.google.com/piper///depot/google3/third_party/envoy/src/envoy/http/filter.h;rcl=945155946;l=1)
that it makes it not very friendly to new developers to contribute, while making
it also hard for reviewers because of the cognitive load of tracking the "safe"
callbacks and the "dangerous" ones.

# Functional programming of HTTP filters

Wouldn't it be nice if we can write an HTTP filter in the following way, to
implement a filter that buffers and parses a bunch of data before it decides to
forward headers to the next filters?

```c++
enum class DecoderResult {
  // the filter has done its job. any further header, body, trailer events skip
  // this filter.
  kSuccess,
  // the filter wants to reset and fail the stream.
  kReset,
  // similar to the current recreateStream callbac, it restarts the filter
  // chain.
  kRecreateStream,
};
HttpDecoder decode(HeaderGetter get_headers, ForwardHeaders forward_headers) {
  // status: to signal stream reset or any other failures of the request
  // headers: a shared_ptr of the header map
  // header_action_token: can only be used once during forward_headers
  auto [header_status, headers, header_action_token, data_generator] = co_await get_headers();

  DataParser parser;
  for co_await(InstancePtr data : data_generator) {
    if (absl::Status status = coawait parser.feed(data); !status.ok()) {
      co_return DecodeResult::kReset;
    }
    if (parser.hasEnoughData()) {
      break;
    }
  }
  if (!parser.hasEnoughData()) {
    return DecodeResult::kReset;
  }
  auto [status, forward_data] = forward_headers(std::move(header_action_token));
  if (!status.ok()) {
    co_return DecodeResult::kReset;
  }
  for co_await(InstancePtr data : parser.bufferedData()) {
    if (absl::Status status = co_await forward_data(parser.bufferedData()); !status.ok()) {
      co_return DecodeResult::kReset;
    }
  }
  // we don't care about future data and trailers (if any), so they just pass through
  co_return DecodeResult::kSuccess;
}
```

We can also imagine simple filters really just read the headers and `co_return`:

```c++
HttpDecoder decode(HeaderGetter get_headers, ForwardHeaders forward_headers) {
  auto [header_status, headers, header_action_token, data_generator] = co_await get_headers();
  // do things with headers

  if (auto [status, forward_data] = forward_headers(std::move(header_action_token)); !status.ok()) {
    co_return DecodeResult::kReset;
  }
  co_return DecodeResult::kSuccess;
}
```

We can define a similar coroutine on response path and `HttpEncoder
encode(...)`.

This way of writing a filter has a few nice properties:

- it's explicit that headers processing is before data, which is before trailers
- it's a compile time error to forward header twice
- it gives the filter the control over how much to buffer per stream
- the filter can also freely inject data
- it does not suffer reentrance and life cycle problems. states are local to
  this coroutine and they are safe to free as soon as `co_return` happens.

Obviously, we need access to streamInfo, configuration, route table, and so on.
These functionality should be provided as synchronous function call, and they
should not suffer re-entrce problem.

We also need async functionalities like local reply, state sharing between
decode and encode path.

Furthermore the API needs some other fine tuning
to provide those functionalities, plus the support of timeout. We can make each
of the `co_await` calls take a timeout.

But hopefully, it gives a rough idea on how things would look like.

# More Details

