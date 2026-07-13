# C++20 coroutine support

C++20 introduced compiler primitives to support
[coroutines](https://en.cppreference.com/cpp/language/coroutines). Coroutines
look like functions, but they do not reside on the stack. It allows wrting
asynchronous code as sequential-looking functions, making it eader to reason
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
-   This naturally requires a lot of states stored in objects for functions to
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
    co_await std::move(get_headers)(), DecodeResult::kReset);

  // parser uses headers to help parsing.
  DataParser parser(headers);
  while (!parser.hasEnoughData()) {
    ASSIGN_OR_CO_RETURN(
      Buffer::InstancePtr data, co_await data_generator.next(),
      DecodeResult::kReset);
    // end stream signalled via nullptr. parser hasn't seen enough. error out.
    if (*data == nullptr) {
      std::move(terminating_actions).replyLocally(Http::Code::BadRequest);
      co_return DecodeResult::kReset;
    }
    if (absl::Status status = co_await parser.feed(*std::move(data));
        !status.ok()) {
      std::move(terminating_actions).replyLocally(Http::Code::BadRequest);
      co_return DecodeResult::kReset;
    }
  };

  // start proxying / forwarding.
  DataForwarder forward_data;
  ASSIGN_OR_CO_RETURN(
    forward_data, std::move(forward_headers)(std::move(headers)),
    DecodeResult::kReset);

  absl::StatusOr<Buffer::InstancePtr> buffered_data = parser.bufferedData();
  while (buffered_data.ok() && *buffered_data != nullptr) {
    if (absl::Status status = co_await forward_data(*std::move(buffered_data));
        status.ok()) {
      std::move(terminating_actions).replyLocally(Http::Code::InternalError);
      co_return DecodeResult::kReset;
    }
    buffered_data = parser.bufferedData();
  }
  if (!buffered_data.ok()) {
    std::move(terminating_actions).replyLocally(Http::Code::InternalError);
    co_return DecodeResult::kReset;
  }

  // std::move(forward_data)() signals that we no longer need forward_data, but
  // it doesn't signal end stream. std::move(forward_data)(nullptr) does.
  ASSIGN_OR_CO_RETURN(
    TrailerForwarder forward_trailers, co_await std::move(forward_data)(),
    DecodeResult::kReset);

  // can be called before end stream.
  TrailerGetter get_trailers = std::move(data_generator).finalize();
  ASSIGN_OR_RETURN(
    std::optional<RequestTrailers> trailers, co_await std::move(get_trailers)(),
    DecodeResult::kReset);
  if (tarilers.has_data()) {
    std::move(forward_trailers)(std::move(trailers));
  }

  co_return DecodeResult::kConinue;
}
```

We can also imagine simple filters really just read the headers and `co_return`:

```c++
HttpDecoder decode(HeaderGetter get_headers, HeaderForwarder,
                   TerminatingActions, Context) {
  RequestHeaders headers;
  ASSIGN_OR_CO_RETURN(
    std::tie(headers, std::ignore), co_await std::move(get_headers)(),
    DecodeResult::kReset);

  // do things with headers
  co_return DecodeResult::kSkip;
}
```

Obviously, it can be broken into a few helper function and coroutines to make it
more readable, but putting it all in one body signifies the sequential nature of
HTTP processing in this paradigm.

## Taking advantage of compiler static analysis
Aside from this, there are other semantic tricks worth pointing out:
-   the next action that the filter can take is obtained from acting on the
    previous action. e.g. `DataGenerator` can only be got by reading headers
    off `HeaderGetter`.
-   we use rvalue ref-qualifier on `operator()` (`operator() (...) &&`) to force
    revocation of an action. e.g. `std::move(forward_header)()`.

This makes it more likely that the control flow is correct when the code
compiles.

## Buffering and flow control.
Aside from correctness, the use of coroutine between `DataGenerator` and
`DataForwarder` gives the filter explicit control of how much data it buffers.
Before forwarding, it actively read more data if it has budget and more is
needed. Because `DataForwarder` is `co_await`ed upon, remote flow control push
back is implicitly implemented.

## Stream life cycle
Early termination of the HTTP stream is handled by the return status of
`HeaderGetter` and friends. A pure sequential filter would not need to handle
cancellation of other async operations because by the time it `co_await`s on any
of the `HeaderGetter` / `HeaderForwarder` coroutines, it must have finished the
async operations. The lack of a class object here makes it harder to use
callbacks, which prevents re-entrance bugs too.

If the filter no longer cares about the request, it simply `co_return
DecodeResult::kSkip`.

In the rare case where we need a few async events to race, we can still use
`co_await` to explicitly join the operation before `co_return`.

## Other useful information to the filter implementation
Obviously, we need access to `StreamInfo`, configuration, route tables, etc.
These functionalities should be provided via synchronous function calls provided
by `Context`.

## (Somewhat) symmetric response handling and coordination
We can write response handling in a similar way, which brings the necessity of
sharing state between `decode()` and `encode()`. This should be handled by
`Context` providing some async message passing primitives.

## Timeouts
Writing "blocking" looking code also requires proper timeout support. This
should be provided by a standard `with_timeout()` wrapper for each async call
that needs to be time boxed.

Hopefully, this gives a rough idea of how things would look.

# More Details

If the code snippets look appealing, it would be intriguing to see how the
interfaces are declared. To avoid boiling the ocean, we can implement this as a
common wrapper http filter that bridges the current callback semantics to the
proposed sequantial coroutine paradigm.

```c++

// since the coroutine filter can co_return before end_stream, we need to decide
// what to do after that.
enum class DecodeResult {
  // the filter has done its job. any further unread header, body, trailer
  // events skip this filter.
  //
  // for headers and trailers that has been read, but not forwarded, they are
  // forwarded automatically.
  //
  // for data that has been read but not forwarded, it is dropped.
  //
  // since local reply inhibits filter chain iteration, kSkip is the same as
  // kReset.
  kSkip,
  // the filter wants to reset the stream if any more event is seen. if no local
  // reply has started, this results in a stream reset (HTTP/1.1 connection
  // closure, and HTTP/2 stream reset), on any further event. if local reply has
  // started, there shouldn't be any decode event any way.
  kReset,
};

class RequestHeaders : public RequestHeaderMap {
private:
  RequestHeaderMapSharedPtr backing_map_;
public:
  // DEFINE_INLINE_STRING_HEADER delegates methods to `backing_map_`
  INLINE_REQ_RESP_STRING_HEADERS(DEFINE_INLINE_STRING_HEADER)
  INLINE_REQ_RESP_NUMERIC_HEADERS(DEFINE_INLINE_NUMERIC_HEADER)
  
  // Provides a read-only view into the header map even after headers are
  // forwarded.
  RequestHeaderMapConstSharedPtr read_only_headers() const;

  // Cannot copy.
  RequestHeaders(const RequestHeaders&) = delete;
  RequestHeaders& operator=(const RequestHeaders&) = delete;

  // Moveable.
  RequestHeaders(const RequestHeaders&& other) {
    backing_map_ = other.backing_map_;
    other.backing_map_ = nullptr;
  }
  RequestHeaders& operator=(const RequestHeaders&&) {
    backing_map_ = other.backing_map_;
    other.backing_map_ = nullptr;
    return *this;
  }
};

class HeaderGetter {
public:
  Coroutine::Task<absl::StatusOr<RequestHeaders, DataGenerator>> operator() &&;
  // other methods to interact with the real http filter.
};

class DataGenerator {
public:
  // Awaits the next chunk.
  // 
  // Status signals error condition, and the filter should start clean itself
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
  Coroutine::Task<absl::StatusOr<std::optional<RequestTrailers>>> operator() &&;
  // other methods to interact with the real http filter.
};

class HeaderForwarder {
public:
  absl::StatusOr<DataForwarder> operator() &&; 
};

class DataForwarder {
public:
  // forwarding nullptr doesn't mean end of data. this is different from getting
  // nullptr from `DataGeneratora::next()`.
  Coroutine::Task<absl::Status> operator() (Buffer::InstancePtr data);
  
  // Receives the trailer forwarder
  Coroutine::Task<TrailerForwarder> operator() (Buffer::InstancePtr data = nullptr) &&;
};

class TrailerForwarder {
public:
  // Ends a stream with or without trailers.
  absl::Status operator() (std::optional<RequestTrailers> trailers) &&;
};

class TerminatingActions {
public:
  // kicks off the local reply and encoder should be called. once a local reply
  // is sent, `HeaderGetter` / `DataGenerator` / `TrailerGetter` will return
  // error for a non-terminating filter. the filter should generally clean
  // itself up and `co_return`. a terminating filter can keep receiving
  // remainder of the messages.
  void replyLocally(Http::Code code) &&;

  // attempts to recreate the filter chain. once started (ok status)
  // `HeaderGetter` / `DataGenerator` / `TrailerGetter` will return error. the
  // filter should clean itself up.
  absl::StaturOr recreateStream() &&;

  // more overloads can be provided to provide ease of use.
};

```

## Decode encode communication

In the case where the encode path (response) needs to get some infromation
from the decode path (request), we take use of the Context object. In the
wrapper filter, we provide a base class that provides `StreamInfo` and the
likes. A filter can choose to implement a subclass of Context that provides
additional methods or awaitable coroutines to pass information from decode path
to encode path. A typical filter should content with synchronous setter and
getter methods. In the rare case async behavior is needed, coroutine be be
implemented.
