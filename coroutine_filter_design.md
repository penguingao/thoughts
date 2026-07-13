# Design: C++20 Coroutine Wrapper for Envoy HTTP Filters

This document outlines the design for an optional wrapper that allows writing Envoy HTTP filters using C++20 coroutines, aimed at improving readability and reducing callback complexity.

## 1. Overview

Envoy's current HTTP filter API is callback-based, which can lead to complex state management and re-entrancy issues (callback hell) for filters with complex asynchronous logic. This design proposes a wrapper filter that bridges Envoy's callback interface to a sequential-looking coroutine API.

## 2. Key Design Decisions

### 2.1. Implementation Strategy
*   **Wrapper Filter**: Implemented as an upstream Envoy extension (wrapper) that implements the standard `Http::PassThroughFilter` (or similar) and delegates to the coroutine. This avoids modifying the core Envoy filter interfaces.
*   **Optional**: Developers can choose to write traditional callback filters or use this coroutine wrapper.

### 2.2. Threading and Safety
*   **Single-Threaded**: Coroutines will execute strictly on the Envoy worker thread assigned to the stream.
*   **No Multi-Threading**: Offloading to background threads and resuming from them is disallowed to prevent thread-safety issues with Envoy's core structures. Any asynchronous resumption must be scheduled back to the worker thread's dispatcher.

### 2.3. Performance
*   **Allocation Elision (HALO)**: Rely on compiler Heap Allocation Elision (HALO) initially to minimize coroutine frame allocation overhead.
*   **Future Optimization**: If profiling shows allocation overhead is a bottleneck, investigate custom allocators (via `promise_type::operator new`) using thread-local pools or arenas.

### 2.4. Flow Control (Watermarks)
*   **Automatic Suspension**: The wrapper filter implements Envoy's watermark callbacks.
*   **Watermark Aware Awaitables**: Awaitables like `forward_data` will check the watermark status. If the high watermark is reached, the awaitable suspends the coroutine. The wrapper resumes it when the low watermark callback is received.

### 2.5. Teardown and Cancellation
*   **Resumption with Error**: When a stream is prematurely destroyed (e.g., reset by peer), the wrapper resumes the suspended coroutine, passing a cancellation error (e.g., `absl::StatusCode::kCancelled`) to the pending awaitable.
*   **Shared State Safety**: To prevent use-after-free, a ref-counted `SharedState` is used between the wrapper and coroutine helpers. The wrapper invalidates this state (nulls Envoy pointers) in its destructor. Helpers check this state and safely error out or noop if the stream is gone.
*   **Graceful Exit**: The coroutine is expected to handle the error and exit via `co_return`.

### 2.6. Timeouts
*   **Global Wrapper**: A composable wrapper `co_await with_timeout(awaitable, duration)` will be provided. It uses the Envoy dispatcher (accessed via the shared state) to schedule a timer and race it against the awaitable.

### 2.7. State Sharing (Decode/Encode)
*   **Shared Context**: A `std::shared_ptr<SharedContext>` is passed to both `decode` and `encode` coroutines, allowing safe state sharing across request and response paths, even during async overlap or cleanup.

### 2.8. Error Handling
*   **Status-Based**: Use `absl::Status` and `absl::StatusOr` for all error propagation.
*   **No Exceptions**: C++ exceptions are forbidden. The coroutine `promise_type::unhandled_exception()` will call `std::terminate()`.

## 3. Proposed API Sketch

```c++
namespace Envoy {
namespace Http {

struct SharedContext {
  // User-defined shared state between decode and encode paths
};

enum class DecodeResult {
  kSuccess,
  kTerminate,
  kReset,
  kRecreateStream,
};

// Awaitable that returns the next chunk of data
class DataGenerator {
public:
  // Suspends until data is available or stream is cancelled
  coroutine_::Awaitable<absl::StatusOr<Buffer::InstancePtr>> next();
  bool end_stream() const;
};

// Callbacks provided to the coroutine
using HeaderGetter = std::function<coroutine_::Awaitable<std::pair<absl::StatusOr<RequestHeaderMapPtr>, DataGenerator>>()>;
using HeaderForwarder = std::function<absl::StatusOr<DataForwarder>(RequestHeaderMapPtr)>;
using DataForwarder = std::function<coroutine_::Awaitable<absl::Status>(Buffer::InstancePtr)>;
using LocalReplier = std::function<void(Http::Code, absl::string_view)>;

// The coroutine entry point for decoding
HttpDecoder decode(HeaderGetter get_headers,
                   HeaderForwarder forward_headers,
                   LocalReplier reply_locally,
                   std::shared_ptr<SharedContext> context) {
  absl::StatusOr<RequestHeaderMapPtr> headers_or;
  DataGenerator data_generator;

  // 1. Wait for headers
  std::tie(headers_or, data_generator) = co_await get_headers();
  if (!headers_or.ok()) {
    co_return DecodeResult::kTerminate;
  }
  auto headers = std::move(*headers_or);

  // Example Custom Logic: Buffer data until we have enough to parse
  DataParser parser;
  while (!parser.hasEnoughData() && !data_generator.end_stream()) {
    absl::StatusOr<Buffer::InstancePtr> data = co_await data_generator.next();
    if (!data.ok()) {
      // Handle cancellation or stream error
      co_return DecodeResult::kTerminate;
    }
    
    if (absl::Status status = parser.feed(*data); !status.ok()) {
      reply_locally(Http::Code::BadRequest, "Invalid payload");
      co_return DecodeResult::kTerminate;
    }
  }

  // 2. Forward headers
  auto forward_data_or = forward_headers(std::move(headers));
  if (!forward_data_or.ok()) {
    co_return DecodeResult::kReset;
  }
  auto forward_data = std::move(*forward_data_or);

  // 3. Forward buffered data (respecting flow control via co_await)
  Buffer::InstancePtr buffered_data = parser.takeBufferedData();
  if (buffered_data && buffered_data->length() > 0) {
    absl::Status status = co_await forward_data(std::move(buffered_data));
    if (!status.ok()) {
      co_return DecodeResult::kReset;
    }
  }

  co_return DecodeResult::kSuccess;
}

} // namespace Http
} // namespace Envoy
```

## 4. Summary of Key Decisions

*   **Wrapper Pattern**: Implemented as an optional wrapper filter, avoiding changes to core Envoy APIs.
*   **Single-Threaded**: Runs strictly on the Envoy worker thread for the stream; no background thread resumption.
*   **Flow Control**: Integrates with Envoy's watermark callbacks to automatically suspend/resume data forwarding.
*   **Safe Cancellation**: Uses a ref-counted shared state. When a stream is destroyed, the wrapper invalidates Envoy pointers, resumes the coroutine with a cancellation error, and lets it exit gracefully via `co_return`.
*   **Timeouts**: Supported via a global `with_timeout` helper that interacts with the Envoy dispatcher.
*   **State Sharing**: `std::shared_ptr<SharedContext>` passed to both decode and encode paths.
*   **Error Handling**: Strictly uses `absl::Status` / `absl::StatusOr`; exceptions trigger `std::terminate`.
