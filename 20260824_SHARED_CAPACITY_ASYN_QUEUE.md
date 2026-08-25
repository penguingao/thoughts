To implement AI filter manager, we need a way to efficiently passing buffers in
the filter chain. The natural way of modeling this is awaitable queues. Each
filter reads from an input queue, and writes to the output queue, which is the
input queue of the next filter.

# Total capacity
To control memory footprint, we'd like the chain of queues to hold no more than a
fixed number of bytes in total. Instead of dividing the capacity equally between
queues, we should allow individual queue to take as much as the capacity.

# Coroutine resumption and reentrancy
If the queue's `push()` and `pop()` are awaitables, a suspended caller can be
resumed when data is available. It's tempting to punt the resumption to the next
dispatcher / scheduler iteration from a clean stack to avoid reentrancy
problems and stack overflow.

However, we'd like to optimize for passing data through the chain of queues as
fast as possible, because the faster the data is handed through the chain of
queues, the lower the overall memory pressure we present to the system.

We should allow direct resumption. To avoid infinite loops and stack overflow, we
should only allow one operation to resume the other. Compared to `pop()`
resuming `push()`, it's better to allow `push()` to directly wake up `pop()` if
there is already a pending `pop()` caller, because:

1.  the data effectively skips the queue if there is already a pending `pop()`
1.  if the source is I/O bound, the data flows through the chain in a single
    dispatcher / scheduler iteration without any queueing
1.  if there exists a queue that's blocked, the chain of `push()` pushes the
    data all the way down to the said queue, giving earlier queue's consumer an
    early chance to see the data
1.  each blocked `push()` holds memory that's not accounted for in capacity, by
    allowing `push()` to wake up `pop()` synchronously, we minimize the number of
    pending `push()`, and make most of the coroutines in the chain blocked on
    `pop()` at the end of each iteration of the dispatcher / scheduler

Each `pop()` operation either blocks, or reads directly off the queue. In the
latter case, it releases capacity, which should cause blocked `push()` to wake
up in the next iteration.

When multiple chains of queues share the same capacity, the FIFO makes sure each
chain has a chance to progress.

# Queue ownership

To make sure the queue can be destructed safely, we need to define an ownership
model.

In a one-pusher-one-popper setup, either can be the owner. Since we typically
want the popper to see EOF, it's better to let the popper be the owner.

In a multiple-pusher-one-popper setup, the popper should be the owner for the
same reason.

It's unlikely to be useful to use the queue in a one-pusher-multiple-popper
setup, because it can be modeled by a pusher using multiple 1:1 queues. Neither
is multiple-pusher-multiple-popper a useful abstraction.

So we should make the queue move-only, with a single owner, which is always
the popper. To discourage using `shared_ptr`, we can have a push accessor that
can access the queue in a non-owning way. They can use a `weak_ptr` to know the
liveness of the underlying queue.

# Tear-down
Whenever caller / awaitable is woken up, we might cause destructor to be called.
During push(), the Queue can be destructed. Pending waiters can be cancelled
too.

In push(), when direct resumption is involved, there shouldn't be other push
waiters in this Queue to be cancelled, so as long as it can safely check whether
destructor has run after the synchronous resumption of pop(), it is memory safe.
Since there should be only one pending pop(), the destructor should be trivial
anyway. A destructor might also cause Capacity to be destructed.

During Capacity processing waiters, we might resume push()'s caller, which might
destruct Queue and Capacity itself. Queue's destructor must notify other pending
pushers, which should cancel the pending acquisition from capacity. It then
moves on to destruct capacity. Since capacity is held by `shared_ptr` by Queue,
by the time `Capacity` destructor is run, there shouldn't be any pending
acquisition anymore. When we finally return to processing waiters, we should
see that destructor has run, so we return.

# Cancellation
If push() handles direct resumption, there shouldn't be other push waiters that
can be cancelled. There is only one possible pop waiter, so there is nothing
that can be cancelled.

During capacity processing waiters, we might resume push()'s caller, which might
cancel other pending pushers. We need to make sure those cancellations don't
mess up the loop that processing waiters runs. The safest is to just tombstone
the cancelled waiters.

# C++ API Blueprint

```C++
// Weighted async semaphore managing global capacity.
class Semaphore : public std::enable_shared_from_this<Semaphore> {
public:
  explicit Semaphore(std::optional<uint64_t> max_permits = std::nullopt);

  // Synchronous non-blocking acquisition attempt.
  std::optional<SemaphoreReservation> tryAcquire(uint64_t permits = 1);

  // Asynchronous FIFO acquisition awaitable.
  SemaphoreAwaitable acquire(uint64_t permits = 1);

  std::optional<uint64_t> maxPermits() const;
  uint64_t currentPermits() const;

private:
  friend class SemaphoreReservation;
  void release(uint64_t permits); // RAII-only via SemaphoreReservation
};

using Capacity = Semaphore;
using CapacityReservation = SemaphoreReservation;

template <typename T, typename SizeFunc = DefaultItemSize<T>>
class AsyncQueue {
public:
  explicit AsyncQueue(CapacityPtr capacity = nullptr, SizeFunc size_func = SizeFunc());

  AsyncQueue(AsyncQueue&&) noexcept;
  AsyncQueue& operator=(AsyncQueue&&) noexcept;
  AsyncQueue(const AsyncQueue&) = delete;
  AsyncQueue& operator=(const AsyncQueue&) = delete;

  // Non-owning accessor for producers holding weak_ptr<Core>.
  PushAccessor pushAccessor() const;

  // Direct handoff to waiting popper if available; otherwise acquires capacity and queues item.
  Task<absl::Status> push(T item);
  template <typename U = T> bool tryPush(U&& item);

  // Directly reads off queue or suspends until data is pushed. Releases capacity on completion.
  // Returns std::nullopt when queue is closed (EOF).
  Task<absl::StatusOr<std::optional<T>>> pop();
  std::optional<T> tryPop();

  // Signals EOF to poppers and closes the queue (never blocks).
  void close();
  bool closed() const;
  bool empty() const;
  uint64_t itemCount() const;
  uint64_t currentSize() const;
  std::optional<uint64_t> maxSize() const;
  CapacityPtr capacity() const;
};
```
