# C++ Concurrency & Multithreading â€” Machine Coding Solutions

> **Language:** C++17/20 | **Compiler flags:** `g++ -std=c++17 -pthread`
> **Coverage:** Threads, Mutex, Deadlock, Condition Variables, Semaphores, Thread Pools, Atomics, Lock-free structures, Concurrent request handling

---

# PATTERN 1 â€” Thread Basics & Mutex

---

## Q1. Thread Creation, Join, Detach

### Problem Statement
Spawn N worker threads, each performing a task, and wait for all to finish before the main thread exits.

**Real scenario:** Flipkart's bulk email service spawns one thread per email batch. Report generation spawns threads per section.

- **Key concepts:** `std::thread`, `join()`, `detach()`, thread ID
- **Asked at:** Amazon, Goldman Sachs

---

### C++ Solution

```cpp
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

// Task each worker thread will run
void workerTask(int threadId, int durationMs) {
    std::cout << "[Thread " << threadId << "] started\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(durationMs));
    std::cout << "[Thread " << threadId << "] done\n";
}

class ThreadLauncher {
public:
    void runAll(int numThreads) {
        std::vector<std::thread> threads;
        threads.reserve(numThreads);

        for (int i = 0; i < numThreads; ++i) {
            // Emplace constructs the thread in-place
            threads.emplace_back(workerTask, i, 100 * (i + 1));
        }

        // join() â€” main thread blocks until each worker finishes
        for (auto& t : threads) {
            if (t.joinable()) t.join();
        }

        std::cout << "All threads completed\n";
    }

    void runDetached() {
        std::thread bg([] {
            std::this_thread::sleep_for(std::chrono::milliseconds(500));
            std::cout << "Background thread done\n";
        });
        bg.detach(); // main thread does NOT wait
        std::cout << "Main continues immediately after detach\n";
    }
};

int main() {
    ThreadLauncher launcher;
    launcher.runAll(4);
    return 0;
}
```

### Key Points
| | `join()` | `detach()` |
|--|----------|------------|
| Main waits? | Yes | No |
| Thread lifetime | Tied to joining thread | Independent |
| Use when | You need the result | Fire-and-forget background work |

### Tradeoffs
- **join() before destructor:** If a `std::thread` object is destroyed without calling `join()` or `detach()`, it calls `std::terminate()`. Always guard with RAII wrapper (`std::jthread` in C++20 auto-joins on destruction).
- **std::jthread (C++20):** Preferred â€” automatically joins on destruction and supports cooperative cancellation via `stop_token`.

---

## Q2. Mutex â€” Protecting Shared State

### Problem Statement
Multiple threads increment a shared counter. Without synchronization, the result is non-deterministic (data race). Fix it using a mutex.

**Real scenario:** Paytm's transaction counter, order ID generator â€” shared state accessed by concurrent request threads.

- **Key concepts:** `std::mutex`, `std::lock_guard`, data race, critical section
- **Asked at:** Paytm, Zerodha

---

### C++ Solution

```cpp
#include <mutex>
#include <thread>
#include <vector>
#include <iostream>

// WRONG: Data race â€” undefined behaviour
class UnsafeCounter {
    int count_ = 0;
public:
    void increment() { ++count_; } // NOT atomic â€” read-modify-write is 3 instructions
    int get() const  { return count_; }
};

// CORRECT: Mutex-protected counter
class SafeCounter {
    int count_ = 0;
    mutable std::mutex mtx_; // mutable so const methods can lock

public:
    void increment() {
        std::lock_guard<std::mutex> lock(mtx_); // RAII â€” unlocks when lock goes out of scope
        ++count_;
    }

    int get() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return count_;
    }

    void reset() {
        std::lock_guard<std::mutex> lock(mtx_);
        count_ = 0;
    }
};

void demo() {
    SafeCounter counter;
    constexpr int NUM_THREADS = 10;
    constexpr int INCREMENTS  = 100000;

    std::vector<std::thread> threads;
    for (int i = 0; i < NUM_THREADS; ++i) {
        threads.emplace_back([&counter] {
            for (int j = 0; j < INCREMENTS; ++j) counter.increment();
        });
    }
    for (auto& t : threads) t.join();

    std::cout << "Expected: " << NUM_THREADS * INCREMENTS
              << " Got: "     << counter.get() << "\n";
    // Always prints: Expected: 1000000 Got: 1000000
}
```

### What is a Data Race?
A data race occurs when:
1. Two or more threads access the same memory location concurrently
2. At least one access is a write
3. There is no synchronization between them

Result: **Undefined Behaviour** (UB) in C++. The compiler and CPU may reorder, elide, or cache the operations.

### Tradeoffs
- **`std::lock_guard`:** Simplest RAII lock. No manual unlock. Cannot be unlocked early.
- **`std::unique_lock`:** More flexible â€” can unlock/relock, transfer ownership, use with condition variables.
- **`std::scoped_lock` (C++17):** Locks multiple mutexes simultaneously without deadlock (uses deadlock avoidance algorithm internally).

---

## Q3. Deadlock â€” What It Is and How It Happens

### Problem Statement
Two threads each hold one lock and try to acquire the other â€” neither can proceed. Demonstrate deadlock and explain detection.

**Real scenario:** Two database transactions each holding a row lock and waiting for the other's row. Classic bank transfer deadlock.

- **Key concepts:** Circular wait, hold-and-wait, mutual exclusion, no preemption
- **Asked at:** Morgan Stanley, Goldman Sachs, Oracle

---

### C++ Solution

```cpp
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

std::mutex mtxA, mtxB;

// DEADLOCK scenario
void threadOne_DEADLOCK() {
    std::lock_guard<std::mutex> lockA(mtxA); // Thread 1 acquires A
    std::cout << "Thread 1: locked A\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(50)); // context switch window
    std::lock_guard<std::mutex> lockB(mtxB); // Thread 1 waits for B (Thread 2 holds it)
    std::cout << "Thread 1: locked B\n";     // NEVER REACHED
}

void threadTwo_DEADLOCK() {
    std::lock_guard<std::mutex> lockB(mtxB); // Thread 2 acquires B
    std::cout << "Thread 2: locked B\n";
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::lock_guard<std::mutex> lockA(mtxA); // Thread 2 waits for A (Thread 1 holds it)
    std::cout << "Thread 2: locked A\n";     // NEVER REACHED
}

// FOUR CONDITIONS for deadlock (Coffman conditions â€” ALL must hold):
// 1. Mutual Exclusion  â€” resource held exclusively
// 2. Hold and Wait     â€” holding one resource while waiting for another
// 3. No Preemption     â€” resource cannot be forcibly taken
// 4. Circular Wait     â€” T1 waits for T2, T2 waits for T1

// FIX 1: Lock ordering â€” always acquire locks in the same global order
void threadOne_FIXED() {
    std::lock_guard<std::mutex> lockA(mtxA); // both threads: A first, then B
    std::lock_guard<std::mutex> lockB(mtxB);
    std::cout << "Thread 1 (fixed): locked A then B\n";
}

void threadTwo_FIXED() {
    std::lock_guard<std::mutex> lockA(mtxA); // same order as Thread 1
    std::lock_guard<std::mutex> lockB(mtxB);
    std::cout << "Thread 2 (fixed): locked A then B\n";
}

// FIX 2: std::scoped_lock (C++17) â€” acquires both atomically, no deadlock
void transfer_SCOPED_LOCK(double& from, double& to, double amount,
                           std::mutex& fromMtx, std::mutex& toMtx) {
    std::scoped_lock lock(fromMtx, toMtx); // deadlock-free multi-lock
    from -= amount;
    to   += amount;
}

// FIX 3: std::lock() + std::adopt_lock
void transfer_STD_LOCK(std::mutex& m1, std::mutex& m2) {
    std::lock(m1, m2);                              // atomically locks both
    std::lock_guard<std::mutex> lg1(m1, std::adopt_lock); // RAII takes ownership
    std::lock_guard<std::mutex> lg2(m2, std::adopt_lock);
    // ... critical section
}

int main() {
    // Run fixed version
    std::thread t1(threadOne_FIXED);
    std::thread t2(threadTwo_FIXED);
    t1.join(); t2.join();
    std::cout << "No deadlock!\n";
    return 0;
}
```

### Deadlock Detection Diagram
```
Thread 1 holds A, wants B  ---+
                               +--> Circular wait = DEADLOCK
Thread 2 holds B, wants A  ---+
```

### Prevention Strategies
| Strategy | How | Eliminates Condition |
|----------|-----|---------------------|
| Lock ordering | Global order on mutexes | Circular wait |
| `scoped_lock` | All-or-nothing acquisition | Hold and wait |
| Try-lock with timeout | `try_lock_for()` + rollback | No preemption |
| Lock hierarchy | Enforce lower-to-higher order | Circular wait |

### Tradeoffs
- **Lock ordering** is simple but requires knowing all locks upfront. Hard to maintain in large codebases.
- **`scoped_lock` (C++17)** is the modern preferred solution for multi-mutex scenarios.
- **Timeouts:** `std::timed_mutex::try_lock_for()` â€” if lock not acquired in time, release held locks and retry. Adds complexity but prevents permanent deadlock.

---

## Q4. Lock Types â€” `lock_guard` vs `unique_lock` vs `scoped_lock`

### Problem Statement
Understand when to use each lock type. Implement a bank account with deposit/withdraw requiring flexible locking.

- **Key concepts:** RAII locks, deferred locking, try-lock, condition variables
- **Asked at:** All system design rounds

---

### C++ Solution

```cpp
#include <mutex>
#include <condition_variable>
#include <stdexcept>
#include <iostream>

class BankAccount {
    double balance_;
    mutable std::mutex mtx_;
    std::condition_variable cv_;

public:
    explicit BankAccount(double initial) : balance_(initial) {}

    // lock_guard: simplest, cannot unlock early, cannot be used with condition_variable
    void deposit_simple(double amount) {
        std::lock_guard<std::mutex> lock(mtx_); // locks on construction, unlocks on destruction
        balance_ += amount;
        cv_.notify_one();
    }

    // unique_lock: flexible â€” deferred lock, early unlock, movable, works with CV
    bool withdraw(double amount) {
        std::unique_lock<std::mutex> lock(mtx_);

        // Wait until balance is sufficient (condition variable requires unique_lock)
        cv_.wait(lock, [this, amount] { return balance_ >= amount; });
        // lock is re-acquired after wait returns

        balance_ -= amount;
        return true;
    }

    // unique_lock with deferred: lock only when needed
    bool tryWithdraw(double amount) {
        std::unique_lock<std::mutex> lock(mtx_, std::defer_lock); // not yet locked
        // ... do some work without the lock ...
        lock.lock(); // lock explicitly when needed

        if (balance_ < amount) return false;
        balance_ -= amount;
        return true;
    }

    // unique_lock: try-lock (non-blocking)
    bool tryDeposit(double amount) {
        std::unique_lock<std::mutex> lock(mtx_, std::try_to_lock);
        if (!lock.owns_lock()) {
            std::cout << "Mutex busy, skipping deposit\n";
            return false;
        }
        balance_ += amount;
        return true;
    }

    double getBalance() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return balance_;
    }
};

// scoped_lock: lock MULTIPLE mutexes atomically (C++17)
void transfer(BankAccount& from, BankAccount& to, double amount) {
    // Problem: can't use lock_guard for two mutexes (deadlock risk)
    // Solution: scoped_lock â€” acquires all or none, in a deadlock-free order
    // scoped_lock<std::mutex, std::mutex> lock(from.getMtx(), to.getMtx());
    // Then do transfer...
    // (Simplified: in real impl, expose mutex or make transfer a friend)
    from.withdraw(amount);
    to.deposit_simple(amount);
}
```

### Quick Reference
```
lock_guard    â€” simple RAII, cannot unlock early, no CV support
unique_lock   â€” flexible: defer, try, timed, CV support, movable
scoped_lock   â€” multiple mutexes atomically (C++17), replaces std::lock()
shared_lock   â€” reader lock for shared_mutex (multiple readers OK)
```

---

## Q5. Recursive Mutex â€” Re-entrant Locking

### Problem Statement
A function that holds a lock calls another function that also tries to acquire the same lock. With a regular mutex this deadlocks â€” use `std::recursive_mutex`.

- **Key concepts:** Recursive mutex, re-entrant code, lock count
- **Asked at:** Bloomberg, DE Shaw

---

### C++ Solution

```cpp
#include <mutex>
#include <iostream>

class RecursiveProcessor {
    std::recursive_mutex mtx_; // allows same thread to lock multiple times
    int data_ = 0;

public:
    void process() {
        std::lock_guard<std::recursive_mutex> lock(mtx_); // lock count: 1
        data_ += 10;
        processInternal(); // calls a function that also locks â€” OK with recursive_mutex
    }

private:
    void processInternal() {
        std::lock_guard<std::recursive_mutex> lock(mtx_); // lock count: 2 â€” same thread, OK
        data_ *= 2;
        // lock released: count -> 1
    }
    // outer lock released: count -> 0, mutex actually unlocked
};

// PROBLEM with regular mutex (would deadlock):
class RegularProcessor {
    std::mutex mtx_;
    int data_ = 0;

public:
    void process() {
        std::lock_guard<std::mutex> lock(mtx_); // acquired
        data_ += 10;
        // processInternal(); // DEADLOCK â€” tries to acquire mtx_ again on same thread
    }
};

// BETTER DESIGN: separate locked/unlocked versions
class BetterDesign {
    std::mutex mtx_;
    int data_ = 0;

    void processInternal_nolock() { data_ *= 2; } // assumes lock held by caller

public:
    void process() {
        std::lock_guard<std::mutex> lock(mtx_);
        data_ += 10;
        processInternal_nolock(); // no locking inside
    }
};
```

### Tradeoffs
- **`recursive_mutex` pros:** Allows re-entrant locking; useful for recursive algorithms.
- **`recursive_mutex` cons:** Slower than `mutex`. Often signals a design problem â€” prefer extracting `_nolock` versions.
- **Rule of thumb:** Prefer redesigning with `_nolock` internal functions over using `recursive_mutex`.

---

## Q6. Read-Write Lock â€” `shared_mutex`

### Problem Statement
Multiple threads read a shared cache simultaneously, but writes must be exclusive. Using a plain mutex forces sequential reads â€” use `shared_mutex` for concurrent reads.

**Real scenario:** In-memory config store (many readers, rare writes). DNS cache. Feature flag store.

- **Key concepts:** `shared_mutex`, `shared_lock` (readers), `unique_lock` (writers), reader-writer problem
- **Asked at:** Google, Uber, Flipkart

---

### C++ Solution

```cpp
#include <shared_mutex>
#include <unordered_map>
#include <string>
#include <optional>
#include <iostream>
#include <thread>
#include <vector>

class ThreadSafeConfigStore {
    std::unordered_map<std::string, std::string> config_;
    mutable std::shared_mutex rwMtx_; // shared_mutex: C++17

public:
    // READ: multiple threads can read simultaneously
    std::optional<std::string> get(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(rwMtx_); // shared (reader) lock
        auto it = config_.find(key);
        if (it == config_.end()) return std::nullopt;
        return it->second;
    }

    // WRITE: exclusive access â€” all readers blocked during write
    void set(const std::string& key, const std::string& value) {
        std::unique_lock<std::shared_mutex> lock(rwMtx_); // exclusive (writer) lock
        config_[key] = value;
    }

    void remove(const std::string& key) {
        std::unique_lock<std::shared_mutex> lock(rwMtx_);
        config_.erase(key);
    }

    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(rwMtx_);
        return config_.size();
    }
};

void demo() {
    ThreadSafeConfigStore store;
    store.set("feature_dark_mode", "true");
    store.set("max_connections", "100");

    std::vector<std::thread> readers;
    for (int i = 0; i < 5; ++i) {
        readers.emplace_back([&store, i] {
            auto val = store.get("feature_dark_mode");
            std::cout << "Reader " << i << ": " << (val ? *val : "not found") << "\n";
        });
    }

    std::thread writer([&store] {
        store.set("feature_dark_mode", "false"); // blocks all readers
    });

    for (auto& r : readers) r.join();
    writer.join();
}
```

### Reader-Writer Lock Semantics
```
Readers: multiple can hold shared_lock simultaneously
Writer:  must wait for ALL readers to release, then gets exclusive access
New readers: blocked while writer is waiting (to prevent writer starvation)
```

### Tradeoffs
- **Writer starvation:** If readers continuously hold the lock, a writer may wait indefinitely. C++'s `shared_mutex` does NOT guarantee fairness â€” implementation-defined.
- **Read-heavy workloads:** `shared_mutex` shines when reads >> writes. For balanced read/write, plain `mutex` may be faster (less overhead).
- **`shared_timed_mutex`:** Adds `try_lock_shared_for()` â€” useful when readers need timeout behaviour.

---

## Q7. Timed Mutex â€” Avoiding Indefinite Blocking

### Problem Statement
Acquire a lock but give up after a timeout instead of blocking forever. Essential for liveness guarantees.

- **Key concepts:** `timed_mutex`, `try_lock_for()`, `try_lock_until()`
- **Asked at:** High-frequency trading firms

---

### C++ Solution

```cpp
#include <mutex>
#include <thread>
#include <chrono>
#include <iostream>

class TimedLockDemo {
    std::timed_mutex mtx_;
    int criticalData_ = 0;

public:
    bool tryUpdate(int value, int timeoutMs) {
        // Try to acquire lock; give up after timeoutMs
        if (mtx_.try_lock_for(std::chrono::milliseconds(timeoutMs))) {
            // Lock acquired
            criticalData_ = value;
            std::this_thread::sleep_for(std::chrono::milliseconds(200)); // simulate work
            mtx_.unlock();
            std::cout << "Update succeeded: " << value << "\n";
            return true;
        } else {
            // Timeout â€” could not acquire lock
            std::cout << "Timeout acquiring lock for value " << value << "\n";
            return false;
        }
    }

    // try_lock_until: absolute time point
    bool tryUpdateUntil(int value) {
        auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(100);
        std::unique_lock<std::timed_mutex> lock(mtx_, deadline);
        if (!lock.owns_lock()) {
            std::cout << "Deadline exceeded\n";
            return false;
        }
        criticalData_ = value;
        return true;
    }
};

void demo() {
    TimedLockDemo demo;
    std::thread t1([&demo] { demo.tryUpdate(42, 500); });  // holds lock for 200ms
    std::this_thread::sleep_for(std::chrono::milliseconds(50));
    std::thread t2([&demo] { demo.tryUpdate(99, 100); });  // times out (lock held for 200ms)
    t1.join(); t2.join();
}
```

### Tradeoffs
- **Timeout prevents livelock/deadlock** but adds complexity: what to do on timeout? Log? Retry? Alert?
- **Exponential backoff on retry:** If `try_lock_for` fails, wait with jitter before retrying â€” reduces contention.

---

## Pattern 1 â€” Summary

| Concept | Tool | Use When |
|---------|------|----------|
| Basic protection | `mutex` + `lock_guard` | Single mutex, no CV needed |
| Flexible lock | `unique_lock` | Deferred lock, CV, try-lock |
| Multi-mutex | `scoped_lock` | Locking 2+ mutexes without deadlock |
| Concurrent reads | `shared_mutex` + `shared_lock` | Many readers, few writers |
| Re-entrant | `recursive_mutex` | Same thread acquires lock multiple times |
| Timeout | `timed_mutex` + `try_lock_for` | Avoid indefinite blocking |
| Deadlock prevention | Lock ordering / scoped_lock | Always acquire locks in same order |

---

# PATTERN 2 â€” Condition Variables & Semaphores

> **When you hear:** "wait for a condition", "notify/signal", "producer-consumer", "bounded buffer"
> **Reach for:** `std::condition_variable` with `std::unique_lock`

---

## Q8. Condition Variable â€” Wait and Notify

### Problem Statement
Thread A must wait until Thread B sets a flag. Without a condition variable, Thread A would busy-spin (wasting CPU). Use `condition_variable` to sleep and wake efficiently.

- **Key concepts:** `wait()`, `notify_one()`, `notify_all()`, spurious wakeup
- **Asked at:** Amazon, Microsoft

---

### C++ Solution

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

class EventFlag {
    std::mutex mtx_;
    std::condition_variable cv_;
    bool ready_ = false;

public:
    // Called by the producer/signaller
    void signal() {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            ready_ = true;
        } // unlock before notify to avoid waking waiter while still holding lock
        cv_.notify_one();
    }

    // Called by the consumer/waiter
    void wait() {
        std::unique_lock<std::mutex> lock(mtx_);
        // ALWAYS use predicate form â€” protects against spurious wakeups
        // Spurious wakeup: thread wakes up even though notify was not called
        cv_.wait(lock, [this] { return ready_; });
        // Equivalent to:
        // while (!ready_) cv_.wait(lock);
        std::cout << "Event received! Processing...\n";
    }

    // Wait with timeout
    bool waitFor(std::chrono::milliseconds timeout) {
        std::unique_lock<std::mutex> lock(mtx_);
        return cv_.wait_for(lock, timeout, [this] { return ready_; });
        // Returns true if condition met, false if timed out
    }

    void reset() {
        std::lock_guard<std::mutex> lock(mtx_);
        ready_ = false;
    }
};

void demo() {
    EventFlag event;

    std::thread waiter([&event] {
        std::cout << "Waiter: waiting for event...\n";
        event.wait();
        std::cout << "Waiter: done\n";
    });

    std::this_thread::sleep_for(std::chrono::milliseconds(200));
    std::cout << "Signaller: sending event\n";
    event.signal();

    waiter.join();
}
```

### Spurious Wakeup
A condition variable can wake up without `notify_one()`/`notify_all()` being called (OS-level artifact). **Always** use the predicate form:
```cpp
cv.wait(lock, predicate);  // correct â€” re-checks predicate on every wakeup
// NOT: cv.wait(lock);     // wrong â€” may proceed without condition being true
```

### `notify_one` vs `notify_all`
| | `notify_one` | `notify_all` |
|--|---|---|
| Wakes up | 1 waiting thread | All waiting threads |
| Use when | One consumer per signal | Multiple consumers or broadcast |
| Cost | O(1) | O(n) waiting threads |

---

## Q9. Producer-Consumer â€” Bounded Buffer

### Problem Statement
A producer generates items and puts them in a bounded buffer (fixed capacity). A consumer takes items. If the buffer is full, the producer waits; if empty, the consumer waits.

**Real scenario:** Kafka's producer-broker-consumer pattern. Log pipeline (logger thread produces, writer thread consumes).

- **Key concepts:** Bounded buffer, two CVs (not-full, not-empty), condition_variable
- **Difficulty:** Medium | **Asked at:** Amazon, Uber, Flipkart

---

### C++ Solution

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <optional>

template<typename T>
class BoundedBuffer {
    std::queue<T> buffer_;
    const size_t capacity_;
    std::mutex mtx_;
    std::condition_variable notFull_;   // signalled when an item is consumed
    std::condition_variable notEmpty_;  // signalled when an item is produced
    bool closed_ = false;

public:
    explicit BoundedBuffer(size_t capacity) : capacity_(capacity) {}

    // Producer: blocks if full
    void produce(T item) {
        std::unique_lock<std::mutex> lock(mtx_);
        notFull_.wait(lock, [this] { return buffer_.size() < capacity_ || closed_; });
        if (closed_) return;
        buffer_.push(std::move(item));
        notEmpty_.notify_one(); // wake up one consumer
    }

    // Consumer: blocks if empty
    std::optional<T> consume() {
        std::unique_lock<std::mutex> lock(mtx_);
        notEmpty_.wait(lock, [this] { return !buffer_.empty() || closed_; });
        if (buffer_.empty()) return std::nullopt; // closed and empty
        T item = std::move(buffer_.front());
        buffer_.pop();
        notFull_.notify_one(); // wake up one producer
        return item;
    }

    // Graceful shutdown â€” wake up all waiters
    void close() {
        std::lock_guard<std::mutex> lock(mtx_);
        closed_ = true;
        notFull_.notify_all();
        notEmpty_.notify_all();
    }

    size_t size() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return buffer_.size();
    }
};

void demo() {
    BoundedBuffer<int> buf(5);

    // Producer thread
    std::thread producer([&buf] {
        for (int i = 0; i < 10; ++i) {
            buf.produce(i);
            std::cout << "Produced: " << i << "\n";
        }
        buf.close();
    });

    // Consumer thread
    std::thread consumer([&buf] {
        while (auto item = buf.consume()) {
            std::cout << "Consumed: " << *item << "\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(50));
        }
        std::cout << "Consumer: buffer closed\n";
    });

    producer.join();
    consumer.join();
}
```

### Trace (capacity=2)
```
Produce 0 -> buf=[0]     -> notify consumer
Produce 1 -> buf=[0,1]   -> notify consumer
Produce 2 -> FULL, wait
Consumer takes 0 -> buf=[1] -> notify producer
Produce 2 -> buf=[1,2]   -> producer resumes
```

### Complexity | produce: O(1) amortised | consume: O(1) | Space: O(capacity)

### Tradeoffs
- **Two CVs** (notFull + notEmpty) is the classic solution. Alternatively use one CV with `notify_all()` but that wakes both producers and consumers unnecessarily.
- **`std::queue<T>`:** Use a circular buffer array for better cache performance in high-throughput scenarios.
- **Multiple producers/consumers:** The same implementation handles M producers + N consumers correctly â€” each `notify_one()` wakes exactly one waiter.

---

## Q10. Semaphore â€” Counting and Binary

### Problem Statement
Implement a counting semaphore that limits concurrent access to a resource pool (e.g., max 3 threads can access a DB connection at once). Also implement a binary semaphore for signalling.

**Real scenario:** Database connection pool limits. File descriptor limits. Rate limiting concurrent requests.

- **Key concepts:** Counting semaphore, `acquire()`, `release()`, `std::counting_semaphore` (C++20)
- **Asked at:** Visa, Goldman Sachs

---

### C++ Solution

```cpp
#include <mutex>
#include <condition_variable>
#include <semaphore>   // C++20 â€” std::counting_semaphore, std::binary_semaphore
#include <thread>
#include <iostream>

// ---- C++17 manual implementation ----
class CountingSemaphore {
    int count_;
    int maxCount_;
    std::mutex mtx_;
    std::condition_variable cv_;

public:
    explicit CountingSemaphore(int count) : count_(count), maxCount_(count) {}

    void acquire() { // P() / wait() â€” decrements count, blocks if 0
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { return count_ > 0; });
        --count_;
    }

    void release() { // V() / signal() â€” increments count, wakes one waiter
        std::unique_lock<std::mutex> lock(mtx_);
        if (count_ < maxCount_) ++count_;
        cv_.notify_one();
    }

    bool tryAcquire() {
        std::lock_guard<std::mutex> lock(mtx_);
        if (count_ <= 0) return false;
        --count_;
        return true;
    }

    int available() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return count_;
    }
};

// ---- C++20 std::counting_semaphore ----
// std::counting_semaphore<MaxValue> sem{initialCount};
// sem.acquire();  // blocks if count == 0
// sem.release();  // increments count

// Use case: limit concurrent DB connections
class ConnectionLimiter {
    CountingSemaphore sem_; // max 3 concurrent connections

public:
    explicit ConnectionLimiter(int maxConcurrent) : sem_(maxConcurrent) {}

    void executeQuery(int threadId, const std::string& query) {
        sem_.acquire(); // wait for a slot
        std::cout << "Thread " << threadId << " executing: " << query << "\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        std::cout << "Thread " << threadId << " done\n";
        sem_.release(); // release slot
    }
};

// Binary semaphore â€” used for signalling (like a condition variable but without a mutex)
class BinarySemaphore {
    CountingSemaphore sem_{0}; // starts at 0 (blocked)
public:
    void wait()   { sem_.acquire(); }
    void signal() { sem_.release(); }
};

void demo() {
    ConnectionLimiter limiter(3); // max 3 concurrent
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i) {
        threads.emplace_back([&limiter, i] {
            limiter.executeQuery(i, "SELECT * FROM orders WHERE id=" + std::to_string(i));
        });
    }
    for (auto& t : threads) t.join();
}
```

### Semaphore vs Mutex
| | Mutex | Binary Semaphore |
|--|-------|-----------------|
| Ownership | Thread that locked must unlock | Any thread can release |
| Count | 0 or 1 | 0 or 1 |
| Used for | Mutual exclusion | Signalling between threads |
| Deadlock risk | Yes (if not unlocked) | Lower (any thread can signal) |

---

## Q11. Barrier â€” Wait for All Threads to Reach a Point

### Problem Statement
N threads perform parallel computation in phases. All threads must complete Phase 1 before any starts Phase 2. Implement a barrier.

**Real scenario:** MapReduce â€” all mappers must finish before reducers start. Parallel matrix algorithms.

- **Key concepts:** Barrier, `std::barrier` (C++20), phase synchronization
- **Asked at:** Google, Qualcomm

---

### C++ Solution

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <vector>
#include <iostream>
#include <barrier>  // C++20

// ---- C++17 manual barrier ----
class Barrier {
    int total_;
    int count_;
    int generation_; // generation counter prevents reuse issues
    std::mutex mtx_;
    std::condition_variable cv_;

public:
    explicit Barrier(int n) : total_(n), count_(n), generation_(0) {}

    void arrive_and_wait() {
        std::unique_lock<std::mutex> lock(mtx_);
        int gen = generation_;
        if (--count_ == 0) {
            // Last thread to arrive â€” wake all and reset
            ++generation_;
            count_ = total_;
            cv_.notify_all();
        } else {
            // Wait until generation changes (all others have arrived)
            cv_.wait(lock, [this, gen] { return generation_ != gen; });
        }
    }
};

// ---- C++20 std::barrier ----
// std::barrier bar(N, completionFn);
// bar.arrive_and_wait(); // blocks until all N threads arrive
// completionFn called once per phase by the last arriving thread

void parallelPhases(int numThreads) {
    Barrier barrier(numThreads);
    std::vector<std::thread> threads;

    for (int id = 0; id < numThreads; ++id) {
        threads.emplace_back([id, &barrier] {
            // Phase 1
            std::cout << "Thread " << id << ": Phase 1\n";
            std::this_thread::sleep_for(std::chrono::milliseconds(id * 50));
            barrier.arrive_and_wait(); // ALL must finish Phase 1 before Phase 2

            // Phase 2
            std::cout << "Thread " << id << ": Phase 2\n";
        });
    }

    for (auto& t : threads) t.join();
}

int main() {
    parallelPhases(4);
    return 0;
}
```

---

## Q12. `std::once_flag` â€” Thread-Safe Singleton

### Problem Statement
Ensure a singleton is initialised exactly once, even if multiple threads call `getInstance()` simultaneously.

**Real scenario:** Logger, DB connection, configuration singleton. Thread-safe lazy initialization.

- **Key concepts:** `call_once`, `once_flag`, double-checked locking pitfall
- **Asked at:** Amazon, Adobe, Morgan Stanley

---

### C++ Solution

```cpp
#include <mutex>
#include <memory>
#include <iostream>

// WRONG: Double-checked locking without memory fence (classic bug)
class BadSingleton {
    static BadSingleton* instance_;
    static std::mutex mtx_;

    BadSingleton() {}
public:
    static BadSingleton* getInstance() {
        if (!instance_) {                            // 1st check (no lock)
            std::lock_guard<std::mutex> lock(mtx_);
            if (!instance_) {                        // 2nd check (with lock)
                instance_ = new BadSingleton();      // PROBLEM: compiler can reorder:
                // 1. alloc memory  2. set instance_  3. construct object
                // Another thread sees instance_ != nullptr but object not fully constructed
            }
        }
        return instance_;
    }
};

// CORRECT 1: call_once â€” guaranteed one-time init
class SingletonCallOnce {
    static std::unique_ptr<SingletonCallOnce> instance_;
    static std::once_flag initFlag_;

    int data_ = 0;
    SingletonCallOnce() { std::cout << "Singleton created\n"; }

public:
    static SingletonCallOnce& getInstance() {
        std::call_once(initFlag_, [] {
            instance_.reset(new SingletonCallOnce());
        });
        return *instance_;
    }

    void doWork() { ++data_; }
    int getData() const { return data_; }
};
std::unique_ptr<SingletonCallOnce> SingletonCallOnce::instance_;
std::once_flag SingletonCallOnce::initFlag_;

// CORRECT 2: Meyers Singleton â€” simplest, guaranteed by C++11 standard
// Local static initialisation is thread-safe since C++11
class MeyersSingleton {
    MeyersSingleton() { std::cout << "Meyers singleton created\n"; }
public:
    static MeyersSingleton& getInstance() {
        static MeyersSingleton instance; // initialised exactly once, thread-safe
        return instance;
    }
    // Delete copy/move
    MeyersSingleton(const MeyersSingleton&) = delete;
    MeyersSingleton& operator=(const MeyersSingleton&) = delete;
};

void demo() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 5; ++i) {
        threads.emplace_back([] {
            MeyersSingleton::getInstance(); // safe from multiple threads
        });
    }
    for (auto& t : threads) t.join();
}
```

### Tradeoffs
- **Meyers Singleton** is the recommended approach in modern C++ â€” simplest and correct.
- **`call_once`** is useful when you need more control over the initialisation code.
- **Eager initialization** (static member initialized at program start): Always safe, no lazy overhead, but initializes even if never used.

---

## Pattern 2 â€” Summary

| Concept | Tool | Use When |
|---------|------|----------|
| Wait for event | `condition_variable` + predicate | Thread must pause until condition is true |
| Bounded buffer | Two CVs (notFull/notEmpty) | Producer-consumer with capacity limit |
| Limit concurrency | `counting_semaphore` | Max N threads in a section |
| Signalling | `binary_semaphore` | One thread signals another |
| Phase sync | `barrier` / `arrive_and_wait` | All threads must finish phase N before phase N+1 |
| One-time init | `call_once` / Meyers Singleton | Thread-safe lazy initialization |

---

# PATTERN 3 â€” Thread Pool & Task Queue

> **When you hear:** "worker threads", "task queue", "parallel execution", "thread reuse"
> **Reach for:** Thread pool â€” fixed set of threads pulling tasks from a shared queue.

---

## Q13. Thread Pool Implementation

### Problem Statement
Create a thread pool with N worker threads. Submit tasks (callables) to a queue. Workers pick up tasks and execute them. Graceful shutdown drains the queue.

**Real scenario:** Java's `ExecutorService`, Python's `ThreadPoolExecutor`, Nginx's worker pool. Flipkart's batch order processor, Zerodha's trade executor.

- **Key concepts:** Worker threads, task queue, `std::function`, `std::future`, shutdown
- **Difficulty:** Hard | **Asked at:** Amazon, Google, Flipkart

---

### C++ Solution

```cpp
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <functional>
#include <future>
#include <vector>
#include <iostream>
#include <stdexcept>

class ThreadPool {
public:
    explicit ThreadPool(size_t numThreads) : stopped_(false) {
        for (size_t i = 0; i < numThreads; ++i) {
            workers_.emplace_back([this] { workerLoop(); });
        }
    }

    // Submit a task and get a future for its result
    template<typename F, typename... Args>
    auto submit(F&& fn, Args&&... args)
        -> std::future<std::invoke_result_t<F, Args...>>
    {
        using ReturnType = std::invoke_result_t<F, Args...>;

        if (stopped_) throw std::runtime_error("ThreadPool is stopped");

        auto task = std::make_shared<std::packaged_task<ReturnType()>>(
            std::bind(std::forward<F>(fn), std::forward<Args>(args)...)
        );

        std::future<ReturnType> result = task->get_future();

        {
            std::lock_guard<std::mutex> lock(queueMtx_);
            tasks_.emplace([task]() { (*task)(); });
        }
        cv_.notify_one();
        return result;
    }

    // Shutdown: wait for all queued tasks to finish
    void shutdown() {
        {
            std::lock_guard<std::mutex> lock(queueMtx_);
            stopped_ = true;
        }
        cv_.notify_all(); // wake all workers so they can exit
        for (auto& w : workers_) {
            if (w.joinable()) w.join();
        }
    }

    size_t pendingTasks() const {
        std::lock_guard<std::mutex> lock(queueMtx_);
        return tasks_.size();
    }

    ~ThreadPool() {
        if (!stopped_) shutdown();
    }

private:
    void workerLoop() {
        while (true) {
            std::function<void()> task;
            {
                std::unique_lock<std::mutex> lock(queueMtx_);
                cv_.wait(lock, [this] { return stopped_ || !tasks_.empty(); });

                if (stopped_ && tasks_.empty()) return; // exit condition

                task = std::move(tasks_.front());
                tasks_.pop();
            }
            task(); // execute OUTSIDE the lock
        }
    }

    std::vector<std::thread>          workers_;
    std::queue<std::function<void()>> tasks_;
    mutable std::mutex                queueMtx_;
    std::condition_variable           cv_;
    bool                              stopped_;
};

// Demo
int main() {
    ThreadPool pool(4);

    // Submit tasks returning values
    std::vector<std::future<int>> results;
    for (int i = 0; i < 8; ++i) {
        results.push_back(pool.submit([i] {
            std::this_thread::sleep_for(std::chrono::milliseconds(50 * i));
            std::cout << "Task " << i << " on thread "
                      << std::this_thread::get_id() << "\n";
            return i * i;
        }));
    }

    // Collect results
    for (int i = 0; i < 8; ++i) {
        std::cout << "Result[" << i << "] = " << results[i].get() << "\n";
    }

    pool.shutdown();
    return 0;
}
```

### How It Works
```
submit(fn)                        workerLoop()
    â”‚                                  â”‚
    â”œâ”€â”€ wrap fn in packaged_task        â”œâ”€â”€ wait on CV until task available
    â”œâ”€â”€ push to tasks_ queue           â”œâ”€â”€ pop task from queue
    â”œâ”€â”€ notify_one()                   â”œâ”€â”€ execute task()
    â””â”€â”€ return future                  â””â”€â”€ loop
```

### Complexity | submit: O(1) | worker overhead: O(1) per task | Space: O(N workers + M queued tasks)

### Tradeoffs
| Design Choice | This impl | Alternative |
|---------------|-----------|-------------|
| Task storage | `std::queue` (FIFO) | Priority queue for task priorities |
| Worker count | Fixed at creation | Dynamic sizing (grow/shrink with load) |
| Task type | `std::function` (heap alloc) | `std::move_only_function` (C++23) / custom |
| Shutdown | Drain queue | Immediate (discard pending tasks) |

---

## Q14. Work-Stealing Thread Pool

### Problem Statement
In a standard thread pool, all threads compete for one shared queue (contention). In work-stealing, each thread has its own queue; idle threads steal from busy threads' queues.

**Real scenario:** Java's `ForkJoinPool`, Intel TBB, Go's goroutine scheduler.

- **Key concepts:** Per-thread deques, work stealing, load balancing
- **Difficulty:** Hard | **Asked at:** Google, Qualcomm

---

### C++ Solution

```cpp
#include <thread>
#include <deque>
#include <mutex>
#include <vector>
#include <functional>
#include <optional>
#include <iostream>
#include <atomic>

class WorkStealingQueue {
    std::deque<std::function<void()>> deque_;
    mutable std::mutex mtx_;

public:
    void push(std::function<void()> task) {
        std::lock_guard<std::mutex> lock(mtx_);
        deque_.push_back(std::move(task)); // own tasks pushed/popped from back (LIFO = cache-friendly)
    }

    std::optional<std::function<void()>> pop() {
        std::lock_guard<std::mutex> lock(mtx_);
        if (deque_.empty()) return std::nullopt;
        auto task = std::move(deque_.back());
        deque_.pop_back();
        return task;
    }

    std::optional<std::function<void()>> steal() { // other threads steal from front (FIFO)
        std::lock_guard<std::mutex> lock(mtx_);
        if (deque_.empty()) return std::nullopt;
        auto task = std::move(deque_.front());
        deque_.pop_front();
        return task;
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return deque_.empty();
    }
};

class WorkStealingPool {
    std::vector<std::unique_ptr<WorkStealingQueue>> queues_;
    std::vector<std::thread> workers_;
    std::atomic<bool> stopped_{false};
    std::atomic<size_t> submitIdx_{0};

public:
    explicit WorkStealingPool(size_t n) {
        for (size_t i = 0; i < n; ++i)
            queues_.push_back(std::make_unique<WorkStealingQueue>());

        for (size_t i = 0; i < n; ++i)
            workers_.emplace_back([this, i] { workerLoop(i); });
    }

    void submit(std::function<void()> task) {
        // Round-robin submission
        size_t idx = submitIdx_++ % queues_.size();
        queues_[idx]->push(std::move(task));
    }

    ~WorkStealingPool() {
        stopped_ = true;
        for (auto& w : workers_) if (w.joinable()) w.join();
    }

private:
    void workerLoop(size_t id) {
        while (!stopped_) {
            // Try own queue first
            if (auto task = queues_[id]->pop()) {
                (*task)();
                continue;
            }
            // Try stealing from other queues
            bool stole = false;
            for (size_t i = 0; i < queues_.size(); ++i) {
                if (i == id) continue;
                if (auto task = queues_[i]->steal()) {
                    (*task)();
                    stole = true;
                    break;
                }
            }
            if (!stole) std::this_thread::yield(); // nothing to steal, yield CPU
        }
    }
};
```

### Work Stealing Advantage
```
Standard pool:  all threads â†’ [shared queue] (hot lock, contention)
Work stealing:  T0â†’[Q0]  T1â†’[Q1]  T2â†’[Q2]  T3â†’[Q3]
                         ^         ^
                    T3 steals  T1 steals from Q0 when idle
```

### Tradeoffs
- **Work stealing** reduces contention by 1/N (each queue has 1/N the access frequency).
- **LIFO own queue, FIFO steal:** Own thread pops from back (LIFO = recently pushed tasks are cache-hot). Stolen tasks from front (oldest = less likely to create sub-tasks that the original thread would process).
- **Complexity:** More complex than single-queue pool. Overkill for most applications; shine in recursive/divide-and-conquer workloads.

---

## Q15. Thread-Safe Queue

### Problem Statement
Design a concurrent FIFO queue supporting `enqueue`, `dequeue`, and `tryDequeue` (non-blocking) for use across multiple producer and consumer threads.

- **Key concepts:** Mutex + CV, move semantics, `std::optional`
- **Asked at:** Amazon, Goldman Sachs

---

### C++ Solution

```cpp
#include <queue>
#include <mutex>
#include <condition_variable>
#include <optional>
#include <memory>

template<typename T>
class ConcurrentQueue {
    mutable std::mutex mtx_;
    std::queue<T> queue_;
    std::condition_variable cv_;

public:
    void enqueue(T value) {
        {
            std::lock_guard<std::mutex> lock(mtx_);
            queue_.push(std::move(value));
        }
        cv_.notify_one();
    }

    // Blocking dequeue â€” waits until item available
    T dequeue() {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    // Non-blocking â€” returns nullopt if empty
    std::optional<T> tryDequeue() {
        std::lock_guard<std::mutex> lock(mtx_);
        if (queue_.empty()) return std::nullopt;
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    // Dequeue with timeout
    std::optional<T> dequeueFor(std::chrono::milliseconds timeout) {
        std::unique_lock<std::mutex> lock(mtx_);
        if (!cv_.wait_for(lock, timeout, [this] { return !queue_.empty(); }))
            return std::nullopt;
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return queue_.empty();
    }

    size_t size() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return queue_.size();
    }
};
```

---

## Q16. Thread-Safe Stack

### Problem Statement
Implement a concurrent stack with `push`, `pop`, and `top`. Handle the race condition inherent in separate `top()` + `pop()` calls (check-then-act race).

- **Key concepts:** Combined top-and-pop, exception safety
- **LeetCode:** â€” | **Asked at:** Amazon, Bloomberg

---

### C++ Solution

```cpp
#include <stack>
#include <mutex>
#include <memory>
#include <optional>
#include <stdexcept>

template<typename T>
class ThreadSafeStack {
    std::stack<T> stack_;
    mutable std::mutex mtx_;

public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mtx_);
        stack_.push(std::move(value));
    }

    // Combined pop: returns value + removes in one atomic operation
    // Avoids TOCTOU race between top() and pop()
    std::optional<T> pop() {
        std::lock_guard<std::mutex> lock(mtx_);
        if (stack_.empty()) return std::nullopt;
        T value = std::move(stack_.top());
        stack_.pop();
        return value;
    }

    // Peek without removing
    std::optional<T> top() const {
        std::lock_guard<std::mutex> lock(mtx_);
        if (stack_.empty()) return std::nullopt;
        return stack_.top();
    }

    bool empty() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return stack_.empty();
    }

    size_t size() const {
        std::lock_guard<std::mutex> lock(mtx_);
        return stack_.size();
    }
};

// Why combined pop matters:
// RACE CONDITION with separate top() + pop():
// Thread A calls top() -> gets value X
// Thread B calls pop() -> removes X
// Thread A calls pop() -> now gets something else!
// Solution: combine into one locked operation
```

---

## Q17. Thread-Safe HashMap (Striped Locking)

### Problem Statement
Design a concurrent hash map. Using one global lock makes all operations sequential. Use **striped locking** (one lock per bucket group) to allow concurrent access to different buckets.

**Real scenario:** Java's `ConcurrentHashMap` uses 16 lock stripes by default. Redis uses per-slot locking for clusters.

- **Key concepts:** Striped locking, lock striping, reduced contention
- **Difficulty:** Hard | **Asked at:** Google, Amazon

---

### C++ Solution

```cpp
#include <vector>
#include <list>
#include <mutex>
#include <shared_mutex>
#include <optional>
#include <functional>

template<typename K, typename V, size_t NUM_STRIPES = 16>
class StripedHashMap {
    struct Entry { K key; V value; };
    using Bucket = std::list<Entry>;

    std::vector<Bucket>            buckets_;
    // Each stripe protects BUCKET_COUNT/NUM_STRIPES buckets
    mutable std::array<std::shared_mutex, NUM_STRIPES> stripeLocks_;
    size_t bucketCount_;

    size_t getBucketIdx(const K& key) const {
        return std::hash<K>{}(key) % bucketCount_;
    }

    size_t getStripeIdx(size_t bucketIdx) const {
        return bucketIdx % NUM_STRIPES;
    }

public:
    explicit StripedHashMap(size_t bucketCount = 256)
        : buckets_(bucketCount), bucketCount_(bucketCount) {}

    void put(const K& key, V value) {
        size_t bidx  = getBucketIdx(key);
        size_t sidx  = getStripeIdx(bidx);

        std::unique_lock<std::shared_mutex> lock(stripeLocks_[sidx]);
        auto& bucket = buckets_[bidx];
        for (auto& entry : bucket) {
            if (entry.key == key) { entry.value = std::move(value); return; }
        }
        bucket.push_back({key, std::move(value)});
    }

    std::optional<V> get(const K& key) const {
        size_t bidx = getBucketIdx(key);
        size_t sidx = getStripeIdx(bidx);

        std::shared_lock<std::shared_mutex> lock(stripeLocks_[sidx]); // reader lock
        for (const auto& entry : buckets_[bidx]) {
            if (entry.key == key) return entry.value;
        }
        return std::nullopt;
    }

    bool remove(const K& key) {
        size_t bidx = getBucketIdx(key);
        size_t sidx = getStripeIdx(bidx);

        std::unique_lock<std::shared_mutex> lock(stripeLocks_[sidx]);
        auto& bucket = buckets_[bidx];
        for (auto it = bucket.begin(); it != bucket.end(); ++it) {
            if (it->key == key) { bucket.erase(it); return true; }
        }
        return false;
    }
};

// Usage
void demo() {
    StripedHashMap<std::string, int> map;
    map.put("rahul", 100);
    map.put("priya", 200);

    auto val = map.get("rahul");
    if (val) std::cout << "rahul = " << *val << "\n";

    map.remove("rahul");
}
```

### Striped Locking Diagram
```
Bucket:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
Stripe:  [--- stripe 0 ---]  [--- stripe 1 ---]  ...

Thread A accesses bucket 2 (stripe 0) â€” locks stripe 0
Thread B accesses bucket 6 (stripe 1) â€” locks stripe 1
-> No contention between A and B!
```

### Tradeoffs
| Approach | Throughput | Complexity |
|----------|-----------|------------|
| Global mutex | Low (fully sequential) | Simple |
| Striped locks (N stripes) | High (N-way parallel) | Medium |
| Per-bucket locks | Very high (fully parallel) | High (memory for N locks) |
| Lock-free (CAS) | Highest | Very high |

---

## Pattern 3 â€” Summary

| Concept | Tool | Use When |
|---------|------|----------|
| Parallel task execution | Thread pool | Reuse threads for many short tasks |
| Load balancing | Work-stealing pool | Recursive/uneven workloads |
| Concurrent FIFO | `ConcurrentQueue` | Producer-consumer without fixed capacity |
| Concurrent LIFO | `ThreadSafeStack` | DFS, undo stacks across threads |
| Concurrent map | Striped HashMap | High-throughput key-value with low contention |

---

# PATTERN 4 â€” Atomic Operations & Memory Order

> **When you hear:** "lock-free", "CAS", "compare-and-swap", "memory fence", "atomic counter"
> **Reach for:** `std::atomic<T>` with appropriate memory ordering.

---

## Q18. Atomic Counter & Spinlock

### Problem Statement
Implement a lock-free counter using `std::atomic`. Then implement a spinlock using atomic compare-and-swap (CAS).

**Real scenario:** Nginx's request counter, CPU performance counters, reference counting in shared_ptr.

- **Key concepts:** `atomic<T>`, `fetch_add`, `compare_exchange_weak/strong`, memory ordering
- **Asked at:** Qualcomm, Samsung, HFT firms

---

### C++ Solution

```cpp
#include <atomic>
#include <thread>
#include <vector>
#include <iostream>

// Lock-free atomic counter
class AtomicCounter {
    std::atomic<int> count_{0};

public:
    void increment() { count_.fetch_add(1, std::memory_order_relaxed); }
    void decrement() { count_.fetch_sub(1, std::memory_order_relaxed); }
    int  get()  const { return count_.load(std::memory_order_relaxed); }
    void reset()      { count_.store(0, std::memory_order_relaxed); }
};

// Spinlock using CAS (Compare-And-Swap)
class Spinlock {
    std::atomic_flag flag_ = ATOMIC_FLAG_INIT; // starts as false (unlocked)

public:
    void lock() {
        // Spin until we successfully set flag from false to true
        while (flag_.test_and_set(std::memory_order_acquire)) {
            // Busy-wait â€” on x86 use PAUSE hint to reduce power/improve performance
#if defined(__x86_64__) || defined(__i386__)
            __builtin_ia32_pause();
#else
            std::this_thread::yield();
#endif
        }
    }

    void unlock() {
        flag_.clear(std::memory_order_release);
    }

    bool tryLock() {
        return !flag_.test_and_set(std::memory_order_acquire);
    }
};

// CAS-based spinlock with exponential backoff
class BackoffSpinlock {
    std::atomic<bool> locked_{false};

public:
    void lock() {
        int backoff = 1;
        bool expected = false;
        while (!locked_.compare_exchange_weak(
                    expected, true,
                    std::memory_order_acquire,    // success memory order
                    std::memory_order_relaxed))   // failure memory order
        {
            expected = false; // CAS sets expected to actual value on failure â€” reset
            // Exponential backoff: spin for 2^backoff iterations
            for (int i = 0; i < backoff; ++i) std::this_thread::yield();
            backoff = std::min(backoff * 2, 1024);
        }
    }

    void unlock() { locked_.store(false, std::memory_order_release); }
};

void demo() {
    AtomicCounter counter;
    constexpr int N = 1000000;
    std::vector<std::thread> threads;

    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&counter] {
            for (int j = 0; j < N/4; ++j) counter.increment();
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Count: " << counter.get() << " (expected " << N << ")\n";
}
```

### Memory Ordering Quick Reference
| Order | Guarantee | Use When |
|-------|-----------|----------|
| `relaxed` | No ordering â€” just atomicity | Counter, stats (don't need sync) |
| `acquire` | No loads/stores moved before this load | Lock acquisition â€” "see all prior writes" |
| `release` | No loads/stores moved after this store | Lock release â€” "publish all my writes" |
| `acq_rel` | Both acquire + release | RMW operations (fetch_add on mutex) |
| `seq_cst` | Global total order | Default, safest, most expensive |

### Spinlock vs Mutex
| | Spinlock | Mutex |
|--|----------|-------|
| Waiting | Busy-spins (wastes CPU) | OS puts thread to sleep |
| Latency | Very low (no syscall) | Higher (context switch) |
| Use when | Lock held for < ~50ns | Lock held for longer |
| Starvation | Possible | OS scheduler prevents |

---

## Q19. Memory Ordering â€” Acquire/Release

### Problem Statement
Demonstrate why incorrect memory ordering causes bugs â€” and how `acquire`/`release` fence semantics fix them.

- **Key concepts:** Memory reordering, happens-before, store-load reordering
- **Asked at:** Intel, Qualcomm, ARM-based HFT systems

---

### C++ Solution

```cpp
#include <atomic>
#include <thread>
#include <cassert>
#include <iostream>

// WRONG: Relaxed ordering â€” may see x=0 even after flag is set
namespace WrongOrdering {
    std::atomic<int>  x{0};
    std::atomic<bool> ready{false};

    void producer() {
        x.store(42, std::memory_order_relaxed);     // (A)
        ready.store(true, std::memory_order_relaxed); // (B)
        // CPU/compiler may reorder (B) before (A)!
    }

    void consumer() {
        while (!ready.load(std::memory_order_relaxed)); // spin until ready
        // May see x == 0 even though producer set it to 42!
        // assert(x.load(std::memory_order_relaxed) == 42); // MAY FAIL!
    }
}

// CORRECT: Release/Acquire synchronisation
namespace CorrectOrdering {
    std::atomic<int>  x{0};
    std::atomic<bool> ready{false};

    void producer() {
        x.store(42, std::memory_order_relaxed);      // (A): non-atomic store is fine
        // release: everything before this store is visible to any thread
        // that subsequently does an acquire-load on 'ready'
        ready.store(true, std::memory_order_release); // (B) â€” RELEASE fence
    }

    void consumer() {
        // acquire: once this load sees true, we can see everything that
        // happened-before the corresponding release store
        while (!ready.load(std::memory_order_acquire)); // ACQUIRE fence
        // GUARANTEED: x == 42 here
        assert(x.load(std::memory_order_relaxed) == 42); // ALWAYS passes
    }
}

// Common pattern: mutex implementation via atomic
namespace CustomMutex {
    std::atomic<bool> locked{false};
    int sharedData = 0;

    void lock() {
        bool expected = false;
        while (!locked.compare_exchange_weak(expected, true,
               std::memory_order_acquire,   // on success: acquire
               std::memory_order_relaxed)) { // on failure: relaxed
            expected = false;
            std::this_thread::yield();
        }
    }

    void unlock() {
        locked.store(false, std::memory_order_release); // release
    }

    void criticalSection(int value) {
        lock();
        sharedData = value; // protected
        unlock();
    }
}
```

### Happens-Before Relationship
```
Producer thread:           Consumer thread:
x.store(42, relaxed)  â”€â”
                        â”‚  release/acquire sync
ready.store(true,       â”‚ â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€â”€> ready.load(acquire) == true
            release) â”€â”€â”€â”˜                         x.load(relaxed) == 42  (GUARANTEED)
```

---

## Q20. Lock-Free Stack (Treiber Stack)

### Problem Statement
Implement a lock-free stack using CAS on the head pointer. Multiple threads push/pop concurrently without any mutex.

- **Key concepts:** CAS loop, ABA problem, hazard pointers
- **Difficulty:** Hard | **Asked at:** Bloomberg, HFT firms, Intel

---

### C++ Solution

```cpp
#include <atomic>
#include <optional>
#include <memory>
#include <iostream>

template<typename T>
class LockFreeStack {
    struct Node {
        T value;
        Node* next;
        explicit Node(T val) : value(std::move(val)), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};

public:
    void push(T value) {
        Node* newNode = new Node(std::move(value));
        // CAS loop: atomically set head to newNode if it still equals old head
        newNode->next = head_.load(std::memory_order_relaxed);
        while (!head_.compare_exchange_weak(
                    newNode->next, newNode,
                    std::memory_order_release,
                    std::memory_order_relaxed)) {
            // CAS failed: head changed since we read it
            // CAS automatically updates newNode->next to current head
            // so we just retry
        }
    }

    std::optional<T> pop() {
        Node* oldHead = head_.load(std::memory_order_relaxed);
        while (oldHead) {
            // Try to swing head from oldHead to oldHead->next
            if (head_.compare_exchange_weak(
                    oldHead, oldHead->next,
                    std::memory_order_acquire,
                    std::memory_order_relaxed)) {
                T value = std::move(oldHead->value);
                // WARNING: Memory leak + ABA problem â€” proper impl needs hazard pointers
                // or deferred deletion (epoch-based reclamation)
                delete oldHead;
                return value;
            }
            // CAS failed: oldHead updated to actual current head â€” retry
        }
        return std::nullopt; // stack was empty
    }

    bool empty() const {
        return head_.load(std::memory_order_relaxed) == nullptr;
    }

    ~LockFreeStack() {
        while (pop()); // drain
    }
};

void demo() {
    LockFreeStack<int> stack;
    std::vector<std::thread> threads;

    for (int i = 0; i < 4; ++i) {
        threads.emplace_back([&stack, i] {
            for (int j = 0; j < 10; ++j) stack.push(i * 10 + j);
        });
    }
    for (auto& t : threads) t.join();

    int count = 0;
    while (stack.pop()) ++count;
    std::cout << "Popped " << count << " elements\n"; // 40
}
```

### ABA Problem
```
Thread A: reads head = ptr_X (value A)
Thread B: pops X, pushes Y, pushes X back (same address, NEW value B)
Thread A: CAS(head, ptr_X, X->next) â€” SUCCEEDS (ptr_X matches)
           but head->next now points to a deleted node!
```

**Solutions:**
- **Tagged pointer:** Pack a version counter into the pointer's unused bits. CAS checks pointer + version.
- **Hazard pointers:** Threads publish pointers they're reading; reclamation deferred until no hazard pointer points to the node.
- **Epoch-based reclamation (EBR):** Used in `folly::LockFreeQueue`, RCU in Linux kernel.

---

# PATTERN 5 â€” Concurrent Request Handling

---

## Q21. Rate Limiter â€” Thread-Safe Token Bucket

### Problem Statement
Implement a thread-safe rate limiter using the token bucket algorithm. Multiple threads check if their request is allowed concurrently.

**Real scenario:** Razorpay API gateway rate limiting. Swiggy restaurant order throttling.

- **Key concepts:** Thread-safe rate limiter, `shared_mutex` for reads, `unique_lock` for token consumption
- **Asked at:** Razorpay, Swiggy, Amazon

---

### C++ Solution

```cpp
#include <mutex>
#include <chrono>
#include <string>
#include <unordered_map>
#include <iostream>

class TokenBucketRateLimiter {
    struct Bucket {
        double tokens;
        std::chrono::steady_clock::time_point lastRefill;
        std::mutex mtx;

        Bucket(double capacity, std::chrono::steady_clock::time_point now)
            : tokens(capacity), lastRefill(now) {}
    };

    const double capacity_;        // max tokens
    const double refillPerSecond_; // tokens added per second
    std::unordered_map<std::string, std::unique_ptr<Bucket>> buckets_;
    std::shared_mutex bucketMapMtx_; // protects the map itself

    Bucket& getOrCreateBucket(const std::string& key) {
        // First try with a shared (read) lock
        {
            std::shared_lock<std::shared_mutex> rlock(bucketMapMtx_);
            auto it = buckets_.find(key);
            if (it != buckets_.end()) return *it->second;
        }
        // Not found â€” upgrade to exclusive (write) lock
        std::unique_lock<std::shared_mutex> wlock(bucketMapMtx_);
        // Re-check (another thread may have inserted while we were upgrading)
        auto [it, inserted] = buckets_.emplace(
            key,
            std::make_unique<Bucket>(capacity_, std::chrono::steady_clock::now())
        );
        return *it->second;
    }

    void refill(Bucket& b) {
        auto now = std::chrono::steady_clock::now();
        double elapsed = std::chrono::duration<double>(now - b.lastRefill).count();
        b.tokens = std::min(capacity_, b.tokens + elapsed * refillPerSecond_);
        b.lastRefill = now;
    }

public:
    TokenBucketRateLimiter(double capacity, double refillPerSecond)
        : capacity_(capacity), refillPerSecond_(refillPerSecond) {}

    bool isAllowed(const std::string& userId) {
        Bucket& bucket = getOrCreateBucket(userId);
        std::lock_guard<std::mutex> lock(bucket.mtx); // per-bucket lock
        refill(bucket);
        if (bucket.tokens >= 1.0) {
            bucket.tokens -= 1.0;
            return true;
        }
        return false;
    }
};

void demo() {
    TokenBucketRateLimiter limiter(5.0, 2.0); // 5 capacity, 2/sec refill
    std::vector<std::thread> threads;
    std::atomic<int> allowed{0}, rejected{0};

    for (int i = 0; i < 20; ++i) {
        threads.emplace_back([&, i] {
            std::string userId = "user_" + std::to_string(i % 3);
            if (limiter.isAllowed(userId)) ++allowed;
            else                           ++rejected;
        });
    }
    for (auto& t : threads) t.join();
    std::cout << "Allowed: " << allowed << " Rejected: " << rejected << "\n";
}
```

---

## Q22. Thread-Safe LRU Cache

### Problem Statement
Implement a thread-safe LRU cache supporting concurrent `get` and `put` from multiple threads.

**Real scenario:** CDN edge cache, in-memory session store (e.g., Redis-like), DNS resolver cache.

- **Key concepts:** `shared_mutex` for concurrent reads, `unique_lock` for writes+eviction
- **Difficulty:** Hard | **Asked at:** Amazon, Uber, Flipkart

---

### C++ Solution

```cpp
#include <list>
#include <unordered_map>
#include <shared_mutex>
#include <optional>

template<typename K, typename V>
class ThreadSafeLRU {
    struct Node { K key; V value; };
    using ListIt = typename std::list<Node>::iterator;

    std::list<Node>               lru_;      // front = MRU, back = LRU
    std::unordered_map<K, ListIt> map_;
    const size_t                  capacity_;
    mutable std::shared_mutex     rwMtx_;

public:
    explicit ThreadSafeLRU(size_t capacity) : capacity_(capacity) {}

    std::optional<V> get(const K& key) {
        // Upgrade pattern: first check with read lock, then write lock for MRU update
        std::unique_lock<std::shared_mutex> lock(rwMtx_); // write lock â€” must move to front
        auto it = map_.find(key);
        if (it == map_.end()) return std::nullopt;
        // Move to front (MRU)
        lru_.splice(lru_.begin(), lru_, it->second);
        return it->second->value;
    }

    void put(const K& key, V value) {
        std::unique_lock<std::shared_mutex> lock(rwMtx_);
        auto it = map_.find(key);
        if (it != map_.end()) {
            it->second->value = std::move(value);
            lru_.splice(lru_.begin(), lru_, it->second);
            return;
        }
        if (map_.size() == capacity_) {
            // Evict LRU (back of list)
            map_.erase(lru_.back().key);
            lru_.pop_back();
        }
        lru_.push_front({key, std::move(value)});
        map_[key] = lru_.begin();
    }

    size_t size() const {
        std::shared_lock<std::shared_mutex> lock(rwMtx_);
        return map_.size();
    }
};
```

### Optimisation Note
A read-modify-write `get()` (must move to front = write op) means we can't use `shared_lock` for it. For truly concurrent reads, consider a version that only promotes on every K-th access, or use a lock-free approach with a `std::atomic` timestamp per entry.

---

## Q23. Parallel Request Dispatcher

### Problem Statement
Dispatch N incoming requests to a pool of handlers. Each request has a timeout. If a handler doesn't respond in time, the request fails with a timeout error.

**Real scenario:** Load balancer dispatching HTTP requests. Microservice client with timeout enforcement.

- **Key concepts:** `std::async`/`std::future`, `wait_for` on future, timeout
- **Asked at:** Uber, Amazon

---

### C++ Solution

```cpp
#include <future>
#include <chrono>
#include <string>
#include <vector>
#include <iostream>
#include <random>

struct Request  { int id; std::string payload; };
struct Response { int requestId; std::string result; bool timedOut; };

class RequestDispatcher {
    ThreadPool pool_; // reuse ThreadPool from Q13

public:
    explicit RequestDispatcher(size_t workers) : pool_(workers) {}

    Response dispatch(const Request& req, std::chrono::milliseconds timeout) {
        auto future = pool_.submit([&req] {
            // Simulate variable processing time
            int delay = (req.id % 5) * 80; // 0, 80, 160, 240, 320 ms
            std::this_thread::sleep_for(std::chrono::milliseconds(delay));
            return "Response for request " + std::to_string(req.id);
        });

        if (future.wait_for(timeout) == std::future_status::ready) {
            return { req.id, future.get(), false };
        }
        return { req.id, "TIMEOUT", true };
    }

    std::vector<Response> dispatchBatch(
        const std::vector<Request>& requests,
        std::chrono::milliseconds timeout)
    {
        // Submit all in parallel
        std::vector<std::future<std::string>> futures;
        for (const auto& req : requests) {
            futures.push_back(pool_.submit([req] {
                int delay = (req.id % 5) * 80;
                std::this_thread::sleep_for(std::chrono::milliseconds(delay));
                return "OK:" + std::to_string(req.id);
            }));
        }

        // Collect results with timeout
        std::vector<Response> responses;
        for (size_t i = 0; i < futures.size(); ++i) {
            if (futures[i].wait_for(timeout) == std::future_status::ready) {
                responses.push_back({ requests[i].id, futures[i].get(), false });
            } else {
                responses.push_back({ requests[i].id, "TIMEOUT", true });
            }
        }
        return responses;
    }
};
```

---

## Q24. Thread-Safe Event Bus (Pub-Sub)

### Problem Statement
Multiple threads publish events; multiple subscriber threads receive them. Subscriptions and publications are concurrent.

- **Key concepts:** `shared_mutex` for subscribe map, per-subscriber queues, move semantics
- **Asked at:** Uber, Swiggy

---

### C++ Solution

```cpp
#include <functional>
#include <unordered_map>
#include <vector>
#include <shared_mutex>
#include <mutex>
#include <string>
#include <any>

class EventBus {
    using Handler = std::function<void(const std::any&)>;
    std::unordered_map<std::string, std::vector<Handler>> handlers_;
    mutable std::shared_mutex mtx_;

public:
    // Subscribe: takes a typed handler
    template<typename T>
    void subscribe(const std::string& topic, std::function<void(const T&)> handler) {
        std::unique_lock<std::shared_mutex> lock(mtx_);
        handlers_[topic].emplace_back([h = std::move(handler)](const std::any& evt) {
            if (auto* ptr = std::any_cast<T>(&evt)) h(*ptr);
        });
    }

    // Publish: calls all handlers for topic â€” concurrent-safe
    template<typename T>
    void publish(const std::string& topic, T event) {
        std::vector<Handler> localHandlers;
        {
            std::shared_lock<std::shared_mutex> lock(mtx_); // read lock for lookup
            auto it = handlers_.find(topic);
            if (it == handlers_.end()) return;
            localHandlers = it->second; // copy handler list (allows unlock before calling)
        }
        // Call handlers outside the lock â€” avoids deadlock if handler calls publish()
        for (auto& h : localHandlers) h(std::make_any<T>(std::move(event)));
    }
};

void demo() {
    EventBus bus;

    bus.subscribe<std::string>("order.created",
        [](const std::string& orderId) {
            std::cout << "Payment service: processing order " << orderId << "\n";
        });

    bus.subscribe<std::string>("order.created",
        [](const std::string& orderId) {
            std::cout << "Inventory service: reserving for order " << orderId << "\n";
        });

    std::vector<std::thread> publishers;
    for (int i = 0; i < 3; ++i) {
        publishers.emplace_back([&bus, i] {
            bus.publish<std::string>("order.created", "ORD_" + std::to_string(i));
        });
    }
    for (auto& t : publishers) t.join();
}
```

---

## Q25. Circuit Breaker â€” Thread-Safe State Machine

### Problem Statement
Implement a thread-safe Circuit Breaker. Concurrent threads call services through it. State transitions (CLOSED -> OPEN -> HALF_OPEN) must be atomic.

**Real scenario:** Every microservice call in Flipkart, Swiggy, or Uber goes through a circuit breaker.

- **Key concepts:** Atomic state transitions, `compare_exchange`, thread-safe state machine
- **Asked at:** Uber, Amazon, Flipkart

---

### C++ Solution

```cpp
#include <atomic>
#include <functional>
#include <stdexcept>
#include <chrono>
#include <iostream>

class CircuitBreaker {
public:
    enum class State { CLOSED, OPEN, HALF_OPEN };

private:
    std::atomic<State> state_{State::CLOSED};
    std::atomic<int>   failureCount_{0};
    std::atomic<int>   successCount_{0};
    std::atomic<std::chrono::steady_clock::time_point::rep> openedAt_{0};

    const int    failureThreshold_;
    const int    successThreshold_;
    const long   cooldownMs_;

    void transitionToOpen() {
        State expected = State::CLOSED;
        if (state_.compare_exchange_strong(expected, State::OPEN,
                std::memory_order_acq_rel)) {
            auto now = std::chrono::steady_clock::now().time_since_epoch().count();
            openedAt_.store(now, std::memory_order_release);
            failureCount_.store(0);
            std::cout << "[CB] CLOSED -> OPEN\n";
        }
        // If CAS fails: another thread already transitioned â€” that's fine
    }

    void transitionToHalfOpen() {
        State expected = State::OPEN;
        if (state_.compare_exchange_strong(expected, State::HALF_OPEN,
                std::memory_order_acq_rel)) {
            successCount_.store(0);
            std::cout << "[CB] OPEN -> HALF_OPEN\n";
        }
    }

    void transitionToClosed() {
        State expected = State::HALF_OPEN;
        if (state_.compare_exchange_strong(expected, State::CLOSED,
                std::memory_order_acq_rel)) {
            failureCount_.store(0);
            std::cout << "[CB] HALF_OPEN -> CLOSED (recovered)\n";
        }
    }

    bool cooldownExpired() const {
        auto openedAt = std::chrono::steady_clock::time_point(
            std::chrono::steady_clock::time_point::duration(
                openedAt_.load(std::memory_order_acquire)));
        auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - openedAt).count();
        return elapsed >= cooldownMs_;
    }

public:
    CircuitBreaker(int failureThreshold, int successThreshold, long cooldownMs)
        : failureThreshold_(failureThreshold)
        , successThreshold_(successThreshold)
        , cooldownMs_(cooldownMs) {}

    template<typename F>
    auto call(F&& fn) -> decltype(fn()) {
        // Check for cooldown expiry (OPEN -> HALF_OPEN)
        if (state_.load(std::memory_order_acquire) == State::OPEN) {
            if (cooldownExpired()) {
                transitionToHalfOpen();
            } else {
                throw std::runtime_error("Circuit OPEN â€” request rejected");
            }
        }

        // HALF_OPEN: only one probe request allowed at a time
        if (state_.load(std::memory_order_acquire) == State::HALF_OPEN) {
            // Allow through â€” on success: close; on failure: reopen
        }

        try {
            auto result = fn();
            // Success path
            if (state_.load(std::memory_order_acquire) == State::HALF_OPEN) {
                if (successCount_.fetch_add(1) + 1 >= successThreshold_) {
                    transitionToClosed();
                }
            } else {
                failureCount_.store(0, std::memory_order_relaxed);
            }
            return result;
        } catch (...) {
            // Failure path
            if (state_.load(std::memory_order_acquire) == State::HALF_OPEN) {
                transitionToOpen(); // probe failed, back to OPEN
            } else {
                if (failureCount_.fetch_add(1) + 1 >= failureThreshold_) {
                    transitionToOpen();
                }
            }
            throw;
        }
    }

    State getState() const { return state_.load(std::memory_order_acquire); }
};
```

---

## Q26. Read-Copy-Update (RCU) â€” Lock-Free Reads

### Problem Statement
Implement a simple RCU-like pattern for a config store where reads are completely lock-free and writes copy-on-write.

**Real scenario:** Linux kernel uses RCU for routing tables. Feature flags in production services.

- **Key concepts:** `atomic<shared_ptr>`, copy-on-write, hazard-free reads
- **Asked at:** Google, Linux kernel contributors

---

### C++ Solution

```cpp
#include <atomic>
#include <memory>
#include <string>
#include <unordered_map>
#include <thread>
#include <iostream>

using Config = std::unordered_map<std::string, std::string>;

class RCUConfigStore {
    // atomic<shared_ptr> â€” C++20 atomic shared_ptr support
    // In C++17 use std::atomic<std::shared_ptr<Config>> with std::atomic_load/store
    std::shared_ptr<const Config> current_;
    std::mutex writeMtx_; // serialise concurrent writers

public:
    explicit RCUConfigStore(Config initial)
        : current_(std::make_shared<Config>(std::move(initial))) {}

    // READ: completely lock-free â€” just atomic load of shared_ptr
    std::shared_ptr<const Config> read() const {
        return std::atomic_load(&current_); // atomic load of shared_ptr
    }

    // WRITE: copy the config, modify the copy, swap atomically
    void update(const std::string& key, const std::string& value) {
        std::lock_guard<std::mutex> lock(writeMtx_); // only one writer at a time
        // Copy-on-write: create a new config based on the current one
        auto newConfig = std::make_shared<Config>(*current_);
        (*newConfig)[key] = value;
        std::atomic_store(&current_, newConfig); // atomic swap
        // Old config is kept alive by any readers holding a shared_ptr to it
        // It will be destroyed when the last reader releases it
    }
};

void demo() {
    RCUConfigStore store({{"rate_limit", "100"}, {"feature_x", "on"}});

    // Readers â€” no locks needed
    std::vector<std::thread> readers;
    for (int i = 0; i < 5; ++i) {
        readers.emplace_back([&store] {
            auto config = store.read(); // lock-free
            std::cout << "rate_limit=" << config->at("rate_limit") << "\n";
        });
    }

    // Writer
    std::thread writer([&store] {
        store.update("rate_limit", "200");
        std::cout << "Updated rate_limit to 200\n";
    });

    for (auto& r : readers) r.join();
    writer.join();
}
```

### RCU Pattern
```
Old config: [rate_limit=100]  <-- readers hold shared_ptr, keeps it alive
                |
Writer copies: [rate_limit=200]
Atomically swaps current_ to point to new config
Old config destroyed when last reader releases its shared_ptr
```

---

# PATTERN 6 â€” Classic Concurrency Problems

---

## Q27. Dining Philosophers

### Problem Statement
5 philosophers sit at a table. Each needs 2 forks (shared with neighbours) to eat. Design a solution that prevents deadlock and starvation.

- **Key concepts:** Resource allocation, deadlock, starvation prevention
- **Asked at:** Classic OS interview question, Goldman Sachs, Uber

---

### C++ Solution

```cpp
#include <mutex>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

class DiningPhilosophers {
    static constexpr int N = 5;
    std::mutex forks_[N];

    void thinkAndEat(int id) {
        // Deadlock fix: always pick up lower-indexed fork first
        int left  = id;
        int right = (id + 1) % N;

        if (left > right) std::swap(left, right); // ensure lower index first

        std::lock_guard<std::mutex> forkL(forks_[left]);
        std::lock_guard<std::mutex> forkR(forks_[right]);

        std::cout << "Philosopher " << id << " is EATING\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "Philosopher " << id << " finished eating\n";
    }

    // Alternative: scoped_lock (C++17) â€” acquires both forks atomically
    void thinkAndEatScopedLock(int id) {
        std::scoped_lock lock(forks_[id], forks_[(id + 1) % N]);
        std::cout << "Philosopher " << id << " eating (scoped_lock)\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(50));
    }

public:
    void run() {
        std::vector<std::thread> philosophers;
        for (int i = 0; i < N; ++i) {
            philosophers.emplace_back([this, i] {
                for (int round = 0; round < 3; ++round) {
                    thinkAndEatScopedLock(i); // no deadlock
                }
            });
        }
        for (auto& p : philosophers) p.join();
    }
};

int main() {
    DiningPhilosophers dp;
    dp.run();
    std::cout << "All philosophers done\n";
    return 0;
}
```

---

## Q28. Readers-Writers Problem (No Starvation)

### Problem Statement
Multiple readers and writers access shared data. Writers need exclusive access. Readers can read concurrently. Prevent writer starvation.

- **Key concepts:** Reader starvation vs writer starvation, fair R/W lock
- **Asked at:** Oracle, Microsoft

---

### C++ Solution

```cpp
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>

// Writer-preference: once a writer is waiting, no new readers start
class FairRWLock {
    std::mutex mtx_;
    std::condition_variable cvReaders_, cvWriters_;
    int activeReaders_  = 0;
    int activeWriters_  = 0;
    int waitingWriters_ = 0;

public:
    void readLock() {
        std::unique_lock<std::mutex> lock(mtx_);
        // Wait if there's an active writer OR waiting writers (writer preference)
        cvReaders_.wait(lock, [this] {
            return activeWriters_ == 0 && waitingWriters_ == 0;
        });
        ++activeReaders_;
    }

    void readUnlock() {
        std::unique_lock<std::mutex> lock(mtx_);
        if (--activeReaders_ == 0 && waitingWriters_ > 0) {
            cvWriters_.notify_one();
        }
    }

    void writeLock() {
        std::unique_lock<std::mutex> lock(mtx_);
        ++waitingWriters_;
        cvWriters_.wait(lock, [this] {
            return activeReaders_ == 0 && activeWriters_ == 0;
        });
        --waitingWriters_;
        ++activeWriters_;
    }

    void writeUnlock() {
        std::unique_lock<std::mutex> lock(mtx_);
        --activeWriters_;
        if (waitingWriters_ > 0) cvWriters_.notify_one(); // prefer writers
        else                     cvReaders_.notify_all(); // wake all readers
    }
};
```

---

## Q29. Print in Order (Thread Synchronisation)

### Problem Statement
Three threads print "first", "second", "third" in order regardless of execution order. Classic sync problem.

- **LeetCode:** 1114 | **Asked at:** Amazon, Microsoft

---

### C++ Solution

```cpp
#include <mutex>
#include <condition_variable>
#include <functional>
#include <thread>
#include <iostream>

class Foo {
    std::mutex mtx_;
    std::condition_variable cv_;
    int turn_ = 1; // which function is allowed to run

public:
    void first(std::function<void()> printFirst) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { return turn_ == 1; });
        printFirst();
        turn_ = 2;
        cv_.notify_all();
    }

    void second(std::function<void()> printSecond) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { return turn_ == 2; });
        printSecond();
        turn_ = 3;
        cv_.notify_all();
    }

    void third(std::function<void()> printThird) {
        std::unique_lock<std::mutex> lock(mtx_);
        cv_.wait(lock, [this] { return turn_ == 3; });
        printThird();
    }
};

// LeetCode 1115: FooBar alternate printing
class FooBar {
    std::mutex mtx_;
    std::condition_variable cv_;
    bool fooTurn_ = true;
    int n_;

public:
    explicit FooBar(int n) : n_(n) {}

    void foo(std::function<void()> printFoo) {
        for (int i = 0; i < n_; ++i) {
            std::unique_lock<std::mutex> lock(mtx_);
            cv_.wait(lock, [this] { return fooTurn_; });
            printFoo();
            fooTurn_ = false;
            cv_.notify_one();
        }
    }

    void bar(std::function<void()> printBar) {
        for (int i = 0; i < n_; ++i) {
            std::unique_lock<std::mutex> lock(mtx_);
            cv_.wait(lock, [this] { return !fooTurn_; });
            printBar();
            fooTurn_ = true;
            cv_.notify_one();
        }
    }
};

int main() {
    Foo foo;
    std::thread t3([&foo] { foo.third([] { std::cout << "third\n"; }); });
    std::thread t1([&foo] { foo.first([] { std::cout << "first\n"; }); });
    std::thread t2([&foo] { foo.second([] { std::cout << "second\n"; }); });
    t1.join(); t2.join(); t3.join(); // always prints: first second third
    return 0;
}
```

---

## Q30. H2O Molecule Builder (Semaphore + Barrier)

### Problem Statement
Two types of threads produce H and O atoms. Group them into H2O molecules: exactly 2 H threads and 1 O thread must be released together.

- **LeetCode:** 1117 | **Asked at:** Google

---

### C++ Solution

```cpp
#include <semaphore>  // C++20
#include <mutex>
#include <condition_variable>
#include <thread>
#include <functional>
#include <iostream>

class H2O {
    std::counting_semaphore<2> hSem_{0}; // need 2 H
    std::counting_semaphore<1> oSem_{0}; // need 1 O
    std::mutex mtx_;
    std::condition_variable cv_;
    int hCount_ = 0;
    int oCount_ = 0;

public:
    void hydrogen(std::function<void()> releaseHydrogen) {
        {
            std::unique_lock<std::mutex> lock(mtx_);
            ++hCount_;
        }
        hSem_.acquire(); // wait for molecule to be ready
        releaseHydrogen();
        hSem_.release(); // signal: done releasing
    }

    void oxygen(std::function<void()> releaseOxygen) {
        {
            std::unique_lock<std::mutex> lock(mtx_);
            ++oCount_;
        }
        oSem_.acquire(); // wait for molecule
        releaseOxygen();
        oSem_.release();
    }

    void buildMolecules() {
        while (true) {
            std::unique_lock<std::mutex> lock(mtx_);
            if (hCount_ >= 2 && oCount_ >= 1) {
                hCount_ -= 2; oCount_ -= 1;
                lock.unlock();
                hSem_.release(2); // signal 2 H threads
                oSem_.release(1); // signal 1 O thread
                // wait for them to be done before next molecule
                hSem_.acquire(2);
                oSem_.acquire(1);
            } else {
                lock.unlock();
                std::this_thread::yield();
            }
        }
    }
};
```

---

# MASTER CHEATSHEET â€” C++ Concurrency

## By Concept

| Concept | Tool(s) | Key Rule |
|---------|---------|----------|
| Basic protection | `mutex` + `lock_guard` | Always use RAII locks |
| Flexible lock | `unique_lock` | Required for condition_variable |
| Multi-mutex | `scoped_lock` | Acquires all atomically, no deadlock |
| Concurrent reads | `shared_mutex` + `shared_lock` | Multiple readers, one writer |
| Deadlock prevention | Lock ordering OR `scoped_lock` | Same lock order everywhere |
| Wait for event | `condition_variable` + predicate | ALWAYS use predicate to guard spurious wakeups |
| Bounded buffer | CV (notFull + notEmpty) | Signal after every produce/consume |
| Limit concurrency | `counting_semaphore` | acquire() blocks when count==0 |
| One-time init | `call_once` / Meyers Singleton | Static local = thread-safe in C++11+ |
| Parallel tasks | Thread pool | Submit returns `future<T>` |
| Phase sync | `barrier::arrive_and_wait()` | All threads must arrive before phase advances |
| Lock-free | `atomic<T>` + CAS | Use acquire/release pairs |
| Lock-free stack | CAS on head pointer | Beware ABA problem |
| RCU reads | `atomic<shared_ptr>` | Lock-free reads, CoW writes |

## Memory Order Decision Tree
```
Do you need ordering guarantees?
    NO  -> memory_order_relaxed  (counters, stats)
    YES ->
        Reading a lock/flag? -> memory_order_acquire
        Writing a lock/flag? -> memory_order_release
        Read-modify-write?   -> memory_order_acq_rel
        Need global order?   -> memory_order_seq_cst (default, safest)
```

## Common Bugs & Fixes
| Bug | Description | Fix |
|-----|-------------|-----|
| Data race | Two threads write/read same var without sync | Add mutex or use `atomic<T>` |
| Deadlock | Circular lock acquisition | Lock ordering / `scoped_lock` |
| Spurious wakeup | CV wakes without notify | Always use predicate in `wait()` |
| ABA problem | Pointer value matches but object changed | Tagged pointers / hazard pointers |
| Double-checked locking | Unsafe singleton in C++03 | Meyers singleton or `call_once` |
| Stale shared_ptr | Reader sees partially constructed object | Use `atomic<shared_ptr>` (C++20) |
| TOCTOU race | `top()` + `pop()` separately | Combine into one locked operation |
