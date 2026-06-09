# /accMachine Coding Interview Solutions

> **100 real-world machine coding problems** asked at Flipkart, Razorpay, Uber, Swiggy, Dream11, and more.
> All code is in **TypeScript** with strict **OOP** — interfaces, classes, encapsulation, single responsibility.

---

## Pattern Index

| Pattern                  | Questions |
| ------------------------ | --------- |
| Sliding Window / Time    | Q1–Q10   |
| Cache Design             | Q11–Q15  |
| Trie / Prefix            | Q16–Q22  |
| Heap / Priority Queue    | Q23–Q34  |
| HashMap / O(1) Design    | Q35–Q46  |
| Stack / Queue            | Q47–Q58  |
| Graph / Topological Sort | Q59–Q72  |
| Interval / Sweep Line    | Q73–Q79  |
| Binary Search            | Q80–Q84  |
| Backtracking / Matrix    | Q85–Q90  |
| DP / Greedy              | Q91–Q95  |
| Misc / Design            | Q96–Q100 |

---

# PATTERN 1 — Sliding Window / Time

> **When you hear:** "per user", "per second", "within last X seconds", "rate limit", "quota"
> **Reach for:** A sliding window over timestamps — keep a deque/queue of recent event times and evict anything outside the window.

---

## Q1. Rate Limiter — Sliding Window Log

### Problem Statement

Design a rate limiter that allows at most **N requests per user per time window** (e.g., 5 requests in 60 seconds). Every time a user makes a request, return `true` if allowed or `false` if the limit is exceeded. The window slides with real time — not fixed to a clock boundary.

**Real scenario:** Razorpay's API gateway allows 100 payment requests per merchant per minute. Beyond that it returns HTTP 429.

- **Pattern:** Sliding Window Log
- **Data Structures:** `Map<userId, number[]>` (user → deque of timestamps)
- **Difficulty:** Medium
- **Asked at:** Flipkart, Razorpay, Stripe, CRED

---

### OOP Design

```
IRateLimiter          ← interface (contract)
    └── SlidingWindowRateLimiter  ← concrete class
            ├── isAllowed(userId, currentTime): boolean
            └── remaining(userId, currentTime): number
```

---

### Approach

1. Maintain a `Map` where each `userId` maps to an array (acting as a deque) of timestamps.
2. On every request, compute `cutoff = currentTime - windowSeconds`.
3. **Evict** all timestamps from the front of the array that are ≤ cutoff (they are outside the window).
4. If `array.length < limit` → append current timestamp → return `true`.
5. Otherwise the window is full → return `false` (do NOT append — request was rejected).

---

### TypeScript Solution

```typescript
// ── Interfaces ─────────────────────────────────────────────────────────────

interface IRateLimiter {
  isAllowed(userId: string, currentTime: number): boolean;
  remaining(userId: string, currentTime: number): number;
}

// ── Implementation ──────────────────────────────────────────────────────────

class SlidingWindowRateLimiter implements IRateLimiter {
  private readonly limit: number;
  private readonly windowSeconds: number;
  // userId → timestamps of requests inside the current window
  private readonly requestLog: Map<string, number[]>;

  constructor(limit: number, windowSeconds: number) {
    this.limit = limit;
    this.windowSeconds = windowSeconds;
    this.requestLog = new Map();
  }

  // Evict expired timestamps and return the live window for a user
  private getWindow(userId: string, currentTime: number): number[] {
    if (!this.requestLog.has(userId)) {
      this.requestLog.set(userId, []);
    }
    const timestamps = this.requestLog.get(userId)!;
    const cutoff = currentTime - this.windowSeconds;

    // Remove all timestamps older than the window
    while (timestamps.length > 0 && timestamps[0] <= cutoff) {
      timestamps.shift();
    }
    return timestamps;
  }

  isAllowed(userId: string, currentTime: number): boolean {
    const window = this.getWindow(userId, currentTime);

    if (window.length < this.limit) {
      window.push(currentTime);
      return true;
    }
    return false;
  }

  remaining(userId: string, currentTime: number): number {
    const window = this.getWindow(userId, currentTime);
    return Math.max(0, this.limit - window.length);
  }
}

// ── Demo ────────────────────────────────────────────────────────────────────

const limiter = new SlidingWindowRateLimiter(3, 10); // 3 requests per 10s

const events: [string, number][] = [
  ["user_A", 1],
  ["user_A", 4],
  ["user_A", 8],
  ["user_A", 9],   // 3 still in window → BLOCKED
  ["user_A", 12],  // t=1 expired (12-10=2 > 1) → ALLOWED
];

for (const [userId, t] of events) {
  const allowed = limiter.isAllowed(userId, t);
  const rem = limiter.remaining(userId, t);
  console.log(`t=${t}s  ${allowed ? "✓ ALLOWED" : "✗ BLOCKED"}  remaining=${rem}`);
}
```

**Output:**

```
t=1s  ✓ ALLOWED  remaining=2
t=4s  ✓ ALLOWED  remaining=1
t=8s  ✓ ALLOWED  remaining=0
t=9s  ✗ BLOCKED  remaining=0
t=12s ✓ ALLOWED  remaining=1
```

---

### Worked Example Trace

```
limit=3, window=10s

t= 1: cutoff=-9  → evict nothing → len=0 < 3 → ALLOW  → log=[1]
t= 4: cutoff=-6  → evict nothing → len=1 < 3 → ALLOW  → log=[1,4]
t= 8: cutoff=-2  → evict nothing → len=2 < 3 → ALLOW  → log=[1,4,8]
t= 9: cutoff=-1  → evict nothing → len=3 = 3 → BLOCK
t=12: cutoff= 2  → evict t=1    → log=[4,8]
                 → len=2 < 3    → ALLOW  → log=[4,8,12]
```

### Complexity

|       | Value                                                               |
| ----- | ------------------------------------------------------------------- |
| Time  | O(N) worst case; O(1) amortised (each timestamp added/removed once) |
| Space | O(N × users) — N = limit per window                               |

---

---

## Q2. Token Bucket Rate Limiter

### Problem Statement

Design a **token bucket** rate limiter. A bucket holds up to `capacity` tokens. Tokens refill at `refillRate` tokens/second. Each request consumes 1 token. If the bucket is empty, the request is rejected. Unlike the sliding window, burst requests are **explicitly allowed** up to the bucket capacity.

**Real scenario:** PhonePe allows merchants to burst 10 rapid payment requests if they've accumulated tokens, but the sustained throughput is capped at 2/second.

- **Pattern:** Token Bucket (lazy refill on demand)
- **Data Structures:** `Map<userId, TokenBucketState>`
- **Difficulty:** Medium
- **Asked at:** PhonePe, Razorpay, Uber, Stripe

---

### OOP Design

```
IBucket               ← interface
TokenBucketState      ← value object (tokens, lastRefillTime)
TokenBucketRateLimiter ← service class
    ├── isAllowed(userId, currentTime): boolean
    └── getTokens(userId, currentTime): number
```

---

### Approach

1. Each user has a `TokenBucketState`: `{ tokens: number, lastRefillTime: number }`.
2. On every request, compute `elapsed = currentTime - lastRefillTime`.
3. **Refill:** `tokens = Math.min(capacity, tokens + elapsed * refillRate)`.
4. Update `lastRefillTime = currentTime`.
5. If `tokens >= 1` → consume one → return `true`. Else → return `false`.

---

### TypeScript Solution

```typescript
// ── Value Object ────────────────────────────────────────────────────────────

interface TokenBucketState {
  tokens: number;
  lastRefillTime: number;
}

// ── Interface ───────────────────────────────────────────────────────────────

interface IRateLimiter {
  isAllowed(userId: string, currentTime: number): boolean;
}

// ── Implementation ──────────────────────────────────────────────────────────

class TokenBucketRateLimiter implements IRateLimiter {
  private readonly capacity: number;
  private readonly refillRate: number; // tokens per second
  private readonly buckets: Map<string, TokenBucketState>;

  constructor(capacity: number, refillRate: number) {
    this.capacity = capacity;
    this.refillRate = refillRate;
    this.buckets = new Map();
  }

  private getOrCreateBucket(userId: string): TokenBucketState {
    if (!this.buckets.has(userId)) {
      // New user starts with a full bucket
      this.buckets.set(userId, { tokens: this.capacity, lastRefillTime: 0 });
    }
    return this.buckets.get(userId)!;
  }

  private refill(state: TokenBucketState, currentTime: number): void {
    const elapsed = currentTime - state.lastRefillTime;
    const newTokens = elapsed * this.refillRate;
    state.tokens = Math.min(this.capacity, state.tokens + newTokens);
    state.lastRefillTime = currentTime;
  }

  isAllowed(userId: string, currentTime: number): boolean {
    const state = this.getOrCreateBucket(userId);

    // Step 1: Refill tokens based on elapsed time
    this.refill(state, currentTime);

    // Step 2: Try to consume a token
    if (state.tokens >= 1) {
      state.tokens -= 1;
      return true;
    }
    return false;
  }

  getTokens(userId: string, currentTime: number): number {
    const state = this.getOrCreateBucket(userId);
    this.refill(state, currentTime);
    return Math.floor(state.tokens * 100) / 100; // round to 2dp
  }
}

// ── Demo ────────────────────────────────────────────────────────────────────

const limiter2 = new TokenBucketRateLimiter(5, 1); // capacity=5, refill 1/sec

const events2: [string, number][] = [
  ["merchant", 0], // token: 4
  ["merchant", 0], // token: 3
  ["merchant", 0], // token: 2
  ["merchant", 0], // token: 1
  ["merchant", 0], // token: 0
  ["merchant", 0], // BLOCKED — bucket empty
  ["merchant", 3], // 3s elapsed → refill 3 → token: 3 → consume → 2
];

for (const [userId, t] of events2) {
  const allowed = limiter2.isAllowed(userId, t);
  const tokens = limiter2.getTokens(userId, t);
  console.log(`t=${t}s  ${allowed ? "✓ ALLOWED" : "✗ BLOCKED"}  tokens=${tokens}`);
}
```

**Output:**

```
t=0s  ✓ ALLOWED  tokens=4
t=0s  ✓ ALLOWED  tokens=3
t=0s  ✓ ALLOWED  tokens=2
t=0s  ✓ ALLOWED  tokens=1
t=0s  ✓ ALLOWED  tokens=0
t=0s  ✗ BLOCKED  tokens=0
t=3s  ✓ ALLOWED  tokens=2
```

---

### Worked Example Trace

```
capacity=5, refillRate=1 token/sec

t=0: elapsed=0 → refill=0  → tokens=5 → consume → tokens=4  ALLOW
t=0: elapsed=0 → refill=0  → tokens=4 → consume → tokens=3  ALLOW
t=0: elapsed=0 → refill=0  → tokens=3 → consume → tokens=2  ALLOW
t=0: elapsed=0 → refill=0  → tokens=2 → consume → tokens=1  ALLOW
t=0: elapsed=0 → refill=0  → tokens=1 → consume → tokens=0  ALLOW
t=0: elapsed=0 → refill=0  → tokens=0 → 0 < 1   → BLOCK
t=3: elapsed=3 → refill=3  → tokens=min(5, 0+3)=3 → consume → tokens=2  ALLOW
```

### Comparison

|               | Sliding Window Log | Token Bucket                |
| ------------- | ------------------ | --------------------------- |
| Burst allowed | No (exact count)   | Yes (up to capacity)        |
| Memory        | O(N per user)      | O(1) per user               |
| Best for      | Strict API quotas  | Network I/O, payment bursts |

### Complexity

|       | Value            |
| ----- | ---------------- |
| Time  | O(1) per request |
| Space | O(users)         |

---

---

## Q3. Leaky Bucket Rate Limiter

### Problem Statement

Design a **leaky bucket** rate limiter. Incoming requests queue up in a fixed-capacity bucket. The bucket drains at a constant rate — one request processed every `1/drainRate` seconds. If the bucket is full when a new request arrives, it is **dropped**. This enforces a perfectly smooth output rate regardless of bursty input.

**Real scenario:** Uber's dispatch service processes ride requests at a fixed throughput to avoid overwhelming the matching engine — even during surge events, the system drains at constant speed.

- **Pattern:** Leaky Bucket (fixed-rate queue)
- **Data Structures:** `Queue<Request>` with fixed capacity
- **Difficulty:** Medium
- **Asked at:** Uber, Paytm, Ola

---

### OOP Design

```
IRequest              ← interface (id, timestamp)
LeakyBucket           ← encapsulates queue + drain logic
    ├── add(request, currentTime): boolean
    └── queueLength(): number
LeakyBucketRateLimiter ← top-level service (one bucket per user)
```

---

### Approach

1. Each user has a `LeakyBucket` — a queue with a max `capacity`.
2. On every request, **simulate draining** first: `drained = Math.floor(elapsed * drainRate)` slots freed.
3. Remove up to `drained` entries from the front of the queue.
4. If `queue.length < capacity` → enqueue the request → **accepted**.
5. If `queue.length === capacity` → **drop** the request.

---

### TypeScript Solution

```typescript
// ── Interfaces ──────────────────────────────────────────────────────────────

interface IRequest {
  id: string;
  timestamp: number;
}

interface IBucket {
  add(request: IRequest, currentTime: number): boolean;
  queueLength(): number;
}

// ── LeakyBucket ─────────────────────────────────────────────────────────────

class LeakyBucket implements IBucket {
  private readonly capacity: number;
  private readonly drainRate: number;   // requests per second
  private readonly queue: IRequest[];
  private lastDrainTime: number;

  constructor(capacity: number, drainRate: number) {
    this.capacity = capacity;
    this.drainRate = drainRate;
    this.queue = [];
    this.lastDrainTime = 0;
  }

  private drain(currentTime: number): void {
    const elapsed = currentTime - this.lastDrainTime;
    const slots = Math.floor(elapsed * this.drainRate);

    if (slots > 0) {
      // Remove processed requests from the front
      const toDrain = Math.min(slots, this.queue.length);
      this.queue.splice(0, toDrain);
      this.lastDrainTime = currentTime;
    }
  }

  add(request: IRequest, currentTime: number): boolean {
    // Step 1: Simulate drain
    this.drain(currentTime);

    // Step 2: Accept if there is capacity
    if (this.queue.length < this.capacity) {
      this.queue.push(request);
      return true;
    }
    return false; // overflow — drop
  }

  queueLength(): number {
    return this.queue.length;
  }
}

// ── RateLimiter service ─────────────────────────────────────────────────────

class LeakyBucketRateLimiter {
  private readonly capacity: number;
  private readonly drainRate: number;
  private readonly buckets: Map<string, LeakyBucket>;

  constructor(capacity: number, drainRate: number) {
    this.capacity = capacity;
    this.drainRate = drainRate;
    this.buckets = new Map();
  }

  private getBucket(userId: string): LeakyBucket {
    if (!this.buckets.has(userId)) {
      this.buckets.set(userId, new LeakyBucket(this.capacity, this.drainRate));
    }
    return this.buckets.get(userId)!;
  }

  isAllowed(userId: string, requestId: string, currentTime: number): boolean {
    const bucket = this.getBucket(userId);
    return bucket.add({ id: requestId, timestamp: currentTime }, currentTime);
  }

  queueLength(userId: string): number {
    return this.buckets.get(userId)?.queueLength() ?? 0;
  }
}

// ── Demo ────────────────────────────────────────────────────────────────────

const limiter3 = new LeakyBucketRateLimiter(3, 1); // capacity=3, drain 1/sec

const events3: [string, string, number][] = [
  ["user1", "req_1", 0],
  ["user1", "req_2", 0],
  ["user1", "req_3", 0],
  ["user1", "req_4", 0],  // bucket full → DROPPED
  ["user1", "req_5", 2],  // 2s → drain 2 → queue has 1 → ACCEPTED
  ["user1", "req_6", 2],
  ["user1", "req_7", 2],
  ["user1", "req_8", 2],  // bucket full again → DROPPED
];

for (const [uid, rid, t] of events3) {
  const allowed = limiter3.isAllowed(uid, rid, t);
  const ql = limiter3.queueLength(uid);
  console.log(`t=${t}s  ${rid}  ${allowed ? "✓ QUEUED " : "✗ DROPPED"}  queue=${ql}`);
}
```

**Output:**

```
t=0s  req_1  ✓ QUEUED   queue=1
t=0s  req_2  ✓ QUEUED   queue=2
t=0s  req_3  ✓ QUEUED   queue=3
t=0s  req_4  ✗ DROPPED  queue=3
t=2s  req_5  ✓ QUEUED   queue=2
t=2s  req_6  ✓ QUEUED   queue=3
t=2s  req_7  ✗ DROPPED  queue=3
t=2s  req_8  ✗ DROPPED  queue=3
```

---

### Worked Example Trace

```
capacity=3, drainRate=1/sec

t=0: drain=0  queue=[]          → len=0 < 3 → QUEUE  → [req_1]
t=0: drain=0  queue=[r1]        → len=1 < 3 → QUEUE  → [r1,r2]
t=0: drain=0  queue=[r1,r2]     → len=2 < 3 → QUEUE  → [r1,r2,r3]
t=0: drain=0  queue=[r1,r2,r3]  → len=3 = 3 → DROP
t=2: elapsed=2 → drained=floor(2×1)=2 → remove r1,r2 → queue=[r3]
     len=1 < 3 → QUEUE → [r3,r5]
```

### Complexity

|       | Value                                 |
| ----- | ------------------------------------- |
| Time  | O(drained) per call — O(1) amortised |
| Space | O(capacity) per bucket                |

---

---

## Q4. API Hit Counter — Hits in the Last 5 Minutes

### Problem Statement

Design a hit counter that records web page hits and answers: **"How many hits occurred in the past 5 minutes?"**. Implement `hit(timestamp)` and `getHits(timestamp)`. Timestamps are in seconds and arrive in non-decreasing order.

**Real scenario:** Amazon CloudWatch tracks "requests in last 5 minutes" per API endpoint on monitoring dashboards.

- **Pattern:** Sliding Window (single queue)
- **Data Structures:** Array of `[timestamp, count]` pairs
- **Difficulty:** Medium
- **Asked at:** Amazon, Atlassian, LinkedIn
- **LeetCode:** 362

---

### OOP Design

```
IHitCounter           ← interface
HitCounter            ← basic (one entry per hit)
HitCounterOptimised   ← groups same-timestamp hits into (ts, count) pairs
```

---

### Approach

1. Maintain a queue of `[timestamp, count]` pairs and a running `total`.
2. `hit(ts)`: If the last entry has the same timestamp, increment its count. Otherwise push `[ts, 1]`. Increment `total`.
3. `getHits(ts)`: Evict entries from the front where `ts - entry.ts >= 300`. Subtract their counts from `total`. Return `total`.

---

### TypeScript Solution

```typescript
// ── Interface ───────────────────────────────────────────────────────────────

interface IHitCounter {
  hit(timestamp: number): void;
  getHits(timestamp: number): number;
}

// ── Optimised Implementation ────────────────────────────────────────────────

class HitCounter implements IHitCounter {
  private static readonly WINDOW = 300; // 5 minutes in seconds

  // Each entry: [timestamp, count]
  private readonly log: [number, number][];
  private total: number;

  constructor() {
    this.log = [];
    this.total = 0;
  }

  hit(timestamp: number): void {
    // Group hits with the same timestamp
    if (this.log.length > 0 && this.log[this.log.length - 1][0] === timestamp) {
      this.log[this.log.length - 1][1]++;
    } else {
      this.log.push([timestamp, 1]);
    }
    this.total++;
  }

  getHits(timestamp: number): number {
    const cutoff = timestamp - HitCounter.WINDOW;

    // Evict entries outside the window
    while (this.log.length > 0 && this.log[0][0] <= cutoff) {
      const [, count] = this.log.shift()!;
      this.total -= count;
    }
    return this.total;
  }
}

// ── Demo ────────────────────────────────────────────────────────────────────

const counter = new HitCounter();

counter.hit(1);
counter.hit(1);   // 2 hits at t=1
counter.hit(300);
counter.hit(300);
counter.hit(300); // 3 hits at t=300

console.log(counter.getHits(300)); // window=[1..300] → 5
console.log(counter.getHits(301)); // t=1 expires (cutoff=1, 1<=1) → 3
console.log(counter.getHits(600)); // t=300 expires → 0
```

**Output:**

```
5
3
0
```

---

### Worked Example Trace

```
WINDOW = 300s

hit(1)   → log=[[1,1]]         total=1
hit(1)   → log=[[1,2]]         total=2
hit(300) → log=[[1,2],[300,1]] total=3
hit(300) → log=[[1,2],[300,2]] total=4
hit(300) → log=[[1,2],[300,3]] total=5

getHits(300): cutoff=0  → nothing evicted → return 5
getHits(301): cutoff=1  → [1,2] evicted (1 ≤ 1) → total=3 → return 3
getHits(600): cutoff=300 → [300,3] evicted (300 ≤ 300) → total=0 → return 0
```

### Complexity

| Method    | Time                         | Space                            |
| --------- | ---------------------------- | -------------------------------- |
| hit()     | O(1)                         | —                               |
| getHits() | O(evicted) — O(1) amortised | O(distinct timestamps in window) |

---

---

## Q5. Per-User Quota Across Multiple Endpoints

### Problem Statement

Design a quota service for an API gateway where **each endpoint has its own rate limit per user**. For example: `GET /search` allows 5 calls/minute, `POST /checkout` allows 2 calls/minute. The system independently tracks and enforces limits per `(userId, endpoint)` pair.

**Real scenario:** CRED enforces separate limits for its OCR-based bill-scan endpoint (expensive) vs its rewards-read endpoint (cheap). Razorpay similarly limits payment initiation more strictly than balance checks.

- **Pattern:** Sliding Window per composite key
- **Data Structures:** `Map<compositeKey, number[]>` + `Map<endpoint, EndpointPolicy>`
- **Difficulty:** Medium
- **Asked at:** CRED, Razorpay, Amazon, Flipkart

---

### OOP Design

```
EndpointPolicy        ← value object  (limit, windowSeconds)
QuotaResult           ← value object  (allowed, message, retryAfter?)
IQuotaService         ← interface
MultiEndpointRateLimiter ← service class
    ├── registerEndpoint(endpoint, policy): void
    ├── isAllowed(userId, endpoint, time): QuotaResult
    └── getUsage(userId, endpoint, time): UsageSummary
```

---

### Approach

1. Each `EndpointPolicy` holds `{ limit, windowSeconds }`.
2. Composite key = `"${userId}::${endpoint}"`.
3. Each composite key maps to a timestamp array (sliding window log).
4. On `isAllowed`: evict expired timestamps → check count vs limit.
5. Return a rich `QuotaResult` with `retryAfter` so the caller knows when to retry.

---

### TypeScript Solution

```typescript
// ── Value Objects ───────────────────────────────────────────────────────────

interface EndpointPolicy {
  limit: number;
  windowSeconds: number;
}

interface QuotaResult {
  allowed: boolean;
  used: number;
  limit: number;
  remaining: number;
  retryAfterSeconds?: number; // only set when blocked
}

interface UsageSummary {
  userId: string;
  endpoint: string;
  used: number;
  limit: number;
  windowSeconds: number;
  remaining: number;
}

// ── Interface ───────────────────────────────────────────────────────────────

interface IQuotaService {
  registerEndpoint(endpoint: string, policy: EndpointPolicy): void;
  isAllowed(userId: string, endpoint: string, currentTime: number): QuotaResult;
  getUsage(userId: string, endpoint: string, currentTime: number): UsageSummary | null;
}

// ── Implementation ──────────────────────────────────────────────────────────

class MultiEndpointRateLimiter implements IQuotaService {
  private readonly policies: Map<string, EndpointPolicy>;
  // compositeKey → timestamps within current window
  private readonly windows: Map<string, number[]>;

  constructor() {
    this.policies = new Map();
    this.windows = new Map();
  }

  registerEndpoint(endpoint: string, policy: EndpointPolicy): void {
    this.policies.set(endpoint, policy);
  }

  private compositeKey(userId: string, endpoint: string): string {
    return `${userId}::${endpoint}`;
  }

  private evictExpired(timestamps: number[], cutoff: number): void {
    while (timestamps.length > 0 && timestamps[0] <= cutoff) {
      timestamps.shift();
    }
  }

  private getWindow(userId: string, endpoint: string, currentTime: number): number[] {
    const key = this.compositeKey(userId, endpoint);
    if (!this.windows.has(key)) {
      this.windows.set(key, []);
    }
    const timestamps = this.windows.get(key)!;
    const policy = this.policies.get(endpoint)!;
    this.evictExpired(timestamps, currentTime - policy.windowSeconds);
    return timestamps;
  }

  isAllowed(userId: string, endpoint: string, currentTime: number): QuotaResult {
    const policy = this.policies.get(endpoint);

    // Unknown endpoint → fail open
    if (!policy) {
      return { allowed: true, used: 0, limit: Infinity, remaining: Infinity };
    }

    const window = this.getWindow(userId, endpoint, currentTime);
    const used = window.length;

    if (used < policy.limit) {
      window.push(currentTime);
      return {
        allowed: true,
        used: used + 1,
        limit: policy.limit,
        remaining: policy.limit - used - 1,
      };
    }

    // Blocked — compute retry window
    const retryAfter = window[0] + policy.windowSeconds - currentTime;
    return {
      allowed: false,
      used,
      limit: policy.limit,
      remaining: 0,
      retryAfterSeconds: Math.ceil(retryAfter),
    };
  }

  getUsage(userId: string, endpoint: string, currentTime: number): UsageSummary | null {
    const policy = this.policies.get(endpoint);
    if (!policy) return null;

    const window = this.getWindow(userId, endpoint, currentTime);
    return {
      userId,
      endpoint,
      used: window.length,
      limit: policy.limit,
      windowSeconds: policy.windowSeconds,
      remaining: Math.max(0, policy.limit - window.length),
    };
  }
}

// ── Demo ────────────────────────────────────────────────────────────────────

const quotaService = new MultiEndpointRateLimiter();

quotaService.registerEndpoint("/search",   { limit: 5, windowSeconds: 60 });
quotaService.registerEndpoint("/checkout", { limit: 2, windowSeconds: 60 });
quotaService.registerEndpoint("/balance",  { limit: 10, windowSeconds: 60 });

const userId = "user_42";

const testCases: [string, string, number][] = [
  [userId, "/search",   1],
  [userId, "/search",   2],
  [userId, "/checkout", 3],
  [userId, "/checkout", 4],
  [userId, "/checkout", 5],  // 3rd checkout → BLOCKED
  [userId, "/search",   6],
  [userId, "/search",   7],
  [userId, "/search",   8],
  [userId, "/search",   9],  // 6th search → BLOCKED
  [userId, "/balance",  10],
];

console.log(`${"t".padEnd(4)}  ${"Endpoint".padEnd(12)}  Status`);
console.log("─".repeat(55));

for (const [uid, ep, t] of testCases) {
  const result = quotaService.isAllowed(uid, ep, t);
  const status = result.allowed ? "✓ ALLOWED" : "✗ BLOCKED";
  const detail = result.allowed
    ? `used=${result.used}/${result.limit}`
    : `retry in ${result.retryAfterSeconds}s`;
  console.log(`t=${String(t).padEnd(2)}  ${ep.padEnd(12)}  ${status}  (${detail})`);
}

// Usage summary
console.log("\n── Usage Summary at t=10 ──");
for (const ep of ["/search", "/checkout", "/balance"]) {
  const usage = quotaService.getUsage(userId, ep, 10);
  if (usage) {
    console.log(`${ep}: ${usage.used}/${usage.limit} used, ${usage.remaining} remaining`);
  }
}
```

**Output:**

```
t     Endpoint      Status
───────────────────────────────────────────────────────
t=1   /search       ✓ ALLOWED  (used=1/5)
t=2   /search       ✓ ALLOWED  (used=2/5)
t=3   /checkout     ✓ ALLOWED  (used=1/2)
t=4   /checkout     ✓ ALLOWED  (used=2/2)
t=5   /checkout     ✗ BLOCKED  (retry in 58s)
t=6   /search       ✓ ALLOWED  (used=3/5)
t=7   /search       ✓ ALLOWED  (used=4/5)
t=8   /search       ✓ ALLOWED  (used=5/5)
t=9   /search       ✗ BLOCKED  (retry in 52s)
t=10  /balance      ✓ ALLOWED  (used=1/10)

── Usage Summary at t=10 ──
/search: 5/5 used, 0 remaining
/checkout: 2/2 used, 0 remaining
/balance: 1/10 used, 9 remaining
```

---

### Worked Example Trace

```
Endpoint /checkout → limit=2, window=60s

t=3: key="user_42::/checkout"  window=[]    → 0 < 2 → ALLOW → [3]
t=4: key="user_42::/checkout"  window=[3]   → 1 < 2 → ALLOW → [3,4]
t=5: key="user_42::/checkout"  window=[3,4] → 2 = 2 → BLOCK
     retryAfter = window[0] + 60 - currentTime = 3 + 60 - 5 = 58s
```

---

### Complexity

|       | Value                                 |
| ----- | ------------------------------------- |
| Time  | O(evicted) amortised O(1) per request |
| Space | O(users × endpoints × limit)        |

---

## Pattern 1 — Summary

| Q  | Algorithm            | Key Insight                                            | Space                      |
| -- | -------------------- | ------------------------------------------------------ | -------------------------- |
| Q1 | Sliding Window Log   | Exact timestamp deque per user                         | O(N per user)              |
| Q2 | Token Bucket         | Lazy refill;`tokens += elapsed × rate`              | O(1) per user              |
| Q3 | Leaky Bucket         | Fixed drain; smooth output; overflow = drop            | O(capacity)                |
| Q4 | Hit Counter          | `(timestamp, count)` pairs to batch same-second hits | O(distinct ts in window)   |
| Q5 | Multi-Endpoint Quota | Composite key `userId::endpoint` per window          | O(users × endpoints × N) |

> **Interview tip:** Always ask: *"Is bursting allowed?"*
>
> - Yes → Token Bucket
> - Exact count needed → Sliding Window Log
> - Smooth output rate needed → Leaky Bucket

---

## Q6. Max Value in a Sliding Window (Monotonic Deque)

### Problem Statement

Given a stream of numbers and a window of size `k`, return the **maximum value** in the window as it slides forward one step at a time. The window moves right, dropping the leftmost element and adding a new one each step.

**Real scenario:** Uber's surge-pricing engine tracks the maximum demand signal in the last K time-ticks to decide multiplier. Zerodha shows the highest stock price seen in the last K ticks.

- **Pattern:** Monotonic Deque
- **Data Structures:** `Deque` storing indices in decreasing order of values
- **Difficulty:** Hard
- **Asked at:** Uber, Google, Zerodha, Groww
- **LeetCode:** 239

---

### OOP Design

```
ISlidingWindowMax          ← interface
MonotonicDeque             ← internal helper (encapsulates deque ops)
SlidingWindowMax           ← service class
```

### Approach

1. Maintain a **deque of indices** whose corresponding values are in **decreasing order** (monotone decreasing).
2. For each new element at index `i`:
   - Pop from the **back** any index whose value is ≤ current value (they can never be the max while `i` is in the window).
   - Pop from the **front** any index that is outside the current window (`i - k + 1`).
3. Push index `i` to the back.
4. The **front** of the deque is always the index of the window's maximum.

---

### TypeScript Solution

```typescript
interface ISlidingWindowMax {
  maxSlidingWindow(nums: number[], k: number): number[];
}

class MonotonicDeque {
  private readonly dq: number[] = []; // stores indices

  pushBack(index: number, nums: number[]): void {
    // Remove indices whose values are smaller — they'll never be the max
    while (this.dq.length > 0 && nums[this.dq[this.dq.length - 1]] <= nums[index]) {
      this.dq.pop();
    }
    this.dq.push(index);
  }

  popFront(expiredIndex: number): void {
    if (this.dq[0] === expiredIndex) this.dq.shift();
  }

  getMax(nums: number[]): number {
    return nums[this.dq[0]];
  }
}

class SlidingWindowMax implements ISlidingWindowMax {
  maxSlidingWindow(nums: number[], k: number): number[] {
    const result: number[] = [];
    const dq = new MonotonicDeque();

    for (let i = 0; i < nums.length; i++) {
      // Remove out-of-window index from front
      dq.popFront(i - k);
      // Maintain decreasing order
      dq.pushBack(i, nums);
      // Window is full — record max
      if (i >= k - 1) result.push(dq.getMax(nums));
    }
    return result;
  }
}

// Demo
const swm = new SlidingWindowMax();
console.log(swm.maxSlidingWindow([1,3,-1,-3,5,3,6,7], 3));
// Output: [3, 3, 5, 5, 6, 7]
```

---

### Worked Example Trace

```
nums=[1,3,-1,-3,5,3,6,7], k=3

i=0: dq=[0(1)]             window not full yet
i=1: 3>1 → pop 0 → dq=[1(3)]   not full yet
i=2: -1<3 → push → dq=[1(3),2(-1)]  → max=nums[1]=3
i=3: -3<-1 → push → dq=[1(3),2(-1),3(-3)]  → pop front: 1 still in [1..3] → max=3
i=4: 5>all → pop 3,2,1 → dq=[4(5)] → pop front: 1 expired → max=5
i=5: 3<5 → push → dq=[4(5),5(3)] → max=5
i=6: 6>5,3 → pop 5,4 → dq=[6(6)] → max=6
i=7: 7>6 → pop 6 → dq=[7(7)] → max=7
Result: [3,3,5,5,6,7]
```

### Complexity

|       | Value                                           |
| ----- | ----------------------------------------------- |
| Time  | O(n) — each element pushed/popped at most once |
| Space | O(k) — deque holds at most k indices           |

### Tradeoffs

| Approach                  | Time           | Space          | Notes                                          |
| ------------------------- | -------------- | -------------- | ---------------------------------------------- |
| Brute force scan          | O(n×k)        | O(1)           | Simple but slow for large k                    |
| **Monotonic deque** | **O(n)** | **O(k)** | **Optimal — standard interview answer** |
| Segment tree              | O(n log n)     | O(n)           | Overkill here; better for arbitrary queries    |

---

---

## Q7. Moving Average of Last N Readings

### Problem Statement

Design a `MovingAverage` class that computes the **average of the last N values** in a stream. On each new value, the oldest value in the window is dropped.

**Real scenario:** Swiggy's ETA engine maintains a moving average of delivery times per zone to display "estimated delivery: X min". Monitoring dashboards show rolling P50 latency.

- **Pattern:** Fixed-size Sliding Window (circular buffer)
- **Data Structures:** Fixed-size array + head pointer (circular)
- **Difficulty:** Easy
- **Asked at:** Swiggy, Amazon, Monitoring teams
- **LeetCode:** 346

---

### TypeScript Solution

```typescript
interface IMovingAverage {
  next(value: number): number;
}

class MovingAverage implements IMovingAverage {
  private readonly size: number;
  private readonly buffer: number[];
  private head: number = 0;      // next write position
  private count: number = 0;     // elements currently in window
  private sum: number = 0;

  constructor(size: number) {
    this.size = size;
    this.buffer = new Array(size).fill(0);
  }

  next(value: number): number {
    // Subtract outgoing value if window is full
    this.sum -= this.buffer[this.head];
    // Write new value at current head
    this.buffer[this.head] = value;
    this.sum += value;
    // Advance head (circular)
    this.head = (this.head + 1) % this.size;
    this.count = Math.min(this.count + 1, this.size);
    return this.sum / this.count;
  }

  windowSize(): number { return this.count; }
}

// Demo
const ma = new MovingAverage(3);
console.log(ma.next(1));  // 1.0
console.log(ma.next(10)); // 5.5
console.log(ma.next(3));  // 4.67
console.log(ma.next(5));  // 6.0  (1 dropped)
```

### Worked Example Trace

```
size=3, buffer=[0,0,0], sum=0

next(1):  subtract buffer[0]=0 → write 1 → buffer=[1,0,0] sum=1  → 1/1=1.0
next(10): subtract buffer[1]=0 → write 10→ buffer=[1,10,0] sum=11 → 11/2=5.5
next(3):  subtract buffer[2]=0 → write 3 → buffer=[1,10,3] sum=14 → 14/3=4.67
next(5):  subtract buffer[0]=1 → write 5 → buffer=[5,10,3] sum=18 → 18/3=6.0
```

### Complexity

|       | Value                |
| ----- | -------------------- |
| Time  | O(1) per call        |
| Space | O(N) — fixed window |

### Tradeoffs

- **Circular array** beats a Queue because it avoids shift()/dequeue overhead — O(1) with no GC churn.
- **Weighted moving average**: multiply each element by a weight decaying with age — more complex but more accurate for ETA prediction.
- **Limitation**: plain average is sensitive to outliers; use median or P90 for latency monitoring.

---

---

## Q8. Debounce / Throttle a Rapidly Firing Event

### Problem Statement

Design a **Throttler** that ensures a handler fires at most once per `cooldownMs` milliseconds, and a **Debouncer** that fires the handler only after `delayMs` of silence (no new events). Both are used to prevent flooding downstream systems.

**Real scenario:** Postman's autocomplete fires only after 300ms of typing silence (debounce). BrowserStack's window-resize handler recalculates layout at most once every 100ms (throttle).

- **Pattern:** Timestamp comparison
- **Data Structures:** Simple instance variables
- **Difficulty:** Easy
- **Asked at:** Postman, BrowserStack, any frontend-heavy company

---

### TypeScript Solution

```typescript
// ── Throttle ────────────────────────────────────────────────────────────────
class Throttler {
  private lastFired: number = -Infinity;

  constructor(private readonly cooldownMs: number) {}

  shouldFire(currentTime: number): boolean {
    if (currentTime - this.lastFired >= this.cooldownMs) {
      this.lastFired = currentTime;
      return true;
    }
    return false;
  }
}

// ── Debounce ─────────────────────────────────────────────────────────────────
class Debouncer {
  private lastEventTime: number = -Infinity;

  constructor(private readonly delayMs: number) {}

  onEvent(currentTime: number): void {
    this.lastEventTime = currentTime; // reset silence timer
  }

  shouldFire(currentTime: number): boolean {
    // Fire only if silence >= delayMs
    return currentTime - this.lastEventTime >= this.delayMs;
  }
}

// Demo — Throttle
const throttle = new Throttler(100);
const times = [0, 50, 100, 120, 200];
for (const t of times) {
  console.log(`t=${t}: ${throttle.shouldFire(t) ? "FIRE" : "skip"}`);
}
// Output: FIRE, skip, FIRE, skip, FIRE

// Demo — Debounce (search input)
const deb = new Debouncer(300);
[0, 100, 200, 250].forEach(t => deb.onEvent(t)); // rapid keystrokes
console.log(deb.shouldFire(549)); // 549-250=299 < 300 → false
console.log(deb.shouldFire(550)); // 550-250=300 >= 300 → true (FIRE)
```

### Complexity

|       | Value |
| ----- | ----- |
| Time  | O(1)  |
| Space | O(1)  |

### Tradeoffs

|          | Throttle                  | Debounce                      |
| -------- | ------------------------- | ----------------------------- |
| Fires    | At most once per interval | After N ms of silence         |
| Good for | Scroll, resize, mousemove | Search input, form validation |
| Risk     | May miss the final event  | Adds latency                  |

---

---

## Q9. Logger — Skip Same Message Within 10 Seconds

### Problem Statement

Design a logger that receives `(message, timestamp)` pairs and prints the message only if the same message hasn't been printed in the last **10 seconds**. Return `true` if printed, `false` if suppressed.

**Real scenario:** Amazon's distributed logger deduplicates repeated error messages — "DB connection failed" flooding the log every millisecond is suppressed to once per 10 seconds per node.

- **Pattern:** Last-seen timestamp lookup
- **Data Structures:** `Map<message, lastPrintedTime>`
- **Difficulty:** Easy
- **Asked at:** Amazon, Google, logging infrastructure teams
- **LeetCode:** 359

---

### TypeScript Solution

```typescript
interface ILogger {
  shouldPrintMessage(timestamp: number, message: string): boolean;
}

class Logger implements ILogger {
  private static readonly COOLDOWN = 10; // seconds
  // message → timestamp when it was last printed
  private readonly lastPrinted: Map<string, number>;

  constructor() {
    this.lastPrinted = new Map();
  }

  shouldPrintMessage(timestamp: number, message: string): boolean {
    const last = this.lastPrinted.get(message) ?? -Infinity;
    if (timestamp - last >= Logger.COOLDOWN) {
      this.lastPrinted.set(message, timestamp);
      return true;
    }
    return false;
  }
}

// Demo
const logger = new Logger();
const calls: [number, string][] = [
  [1,  "foo"],
  [2,  "bar"],
  [3,  "foo"],  // 3-1=2 < 10 → suppress
  [11, "foo"],  // 11-1=10 >= 10 → print
  [12, "bar"],  // 12-2=10 >= 10 → print
  [14, "foo"],  // 14-11=3 < 10 → suppress
];
for (const [ts, msg] of calls) {
  console.log(`t=${ts} "${msg}" → ${logger.shouldPrintMessage(ts, msg)}`);
}
```

**Output:**

```
t=1  "foo" → true
t=2  "bar" → true
t=3  "foo" → false
t=11 "foo" → true
t=12 "bar" → true
t=14 "foo" → false
```

### Complexity

|       | Value              |
| ----- | ------------------ |
| Time  | O(1) per call      |
| Space | O(unique messages) |

### Tradeoffs

- **Map grows unboundedly** with unique messages. In production, use a TTL-based eviction or Redis SETNX with expiry.
- **Alternative:** A sliding window deque per message — more accurate if you need count-based suppression (suppress if N occurrences in last T seconds).
- **Memory optimisation:** At scale, use a Bloom filter with TTL to probabilistically check recency at O(1) with fixed memory.

---

---

## Q10. Stock Ticker — Max Price in Last K Ticks

### Problem Statement

Design a `StockTracker` that accepts tick prices one by one and answers: **"What is the maximum price seen in the last K ticks?"** at any point.

**Real scenario:** Zerodha's chart engine highlights the candle high for the visible window. Groww shows the "52-week high" for a rolling 252-tick window.

- **Pattern:** Monotonic Deque (same as Q6, applied to a streaming API)
- **Data Structures:** `Deque<price>` (monotone decreasing, stores values not indices since window is always last K)
- **Difficulty:** Medium
- **Asked at:** Zerodha, Groww, DE Shaw

---

### TypeScript Solution

```typescript
interface IStockTracker {
  addPrice(price: number): void;
  getMaxInWindow(): number;
}

class StockTracker implements IStockTracker {
  private readonly k: number;
  private readonly window: number[]; // circular buffer of last k prices
  private readonly maxDeque: number[]; // monotone decreasing values
  private head: number = 0;
  private count: number = 0;

  constructor(k: number) {
    this.k = k;
    this.window = new Array(k).fill(0);
    this.maxDeque = [];
  }

  addPrice(price: number): void {
    // Remove outgoing price from front of maxDeque if window is full
    if (this.count === this.k) {
      const outgoing = this.window[this.head];
      if (this.maxDeque[0] === outgoing) this.maxDeque.shift();
    }

    // Remove smaller prices from back of maxDeque
    while (this.maxDeque.length > 0 && this.maxDeque[this.maxDeque.length - 1] <= price) {
      this.maxDeque.pop();
    }
    this.maxDeque.push(price);

    // Store price in circular buffer
    this.window[this.head] = price;
    this.head = (this.head + 1) % this.k;
    this.count = Math.min(this.count + 1, this.k);
  }

  getMaxInWindow(): number {
    if (this.count === 0) throw new Error("No prices recorded");
    return this.maxDeque[0];
  }
}

// Demo
const tracker = new StockTracker(3);
const prices = [100, 105, 98, 110, 95, 112];
for (const p of prices) {
  tracker.addPrice(p);
  console.log(`price=${p}  maxInLastK=${tracker.getMaxInWindow()}`);
}
```

**Output:**

```
price=100  maxInLastK=100
price=105  maxInLastK=105
price=98   maxInLastK=105
price=110  maxInLastK=110
price=95   maxInLastK=110
price=112  maxInLastK=112
```

### Complexity

|       | Value                   |
| ----- | ----------------------- |
| Time  | O(1) amortised per tick |
| Space | O(k)                    |

### Tradeoffs

- **Monotonic deque is O(1) amortised** vs O(k) for brute-force window scan — critical for high-frequency tick feeds.
- **Tradeoff:** Deque stores only the max candidate; you cannot query min or arbitrary percentiles from it. Use a sorted structure (e.g., balanced BST) if multi-stat queries are needed — O(log k) per op.
- **Alternative for min+max:** Maintain two monotonic deques (one increasing, one decreasing) simultaneously.

---

## Pattern 1 — Summary

| Q   | Algorithm            | Key Insight                                | Space                      |
| --- | -------------------- | ------------------------------------------ | -------------------------- |
| Q1  | Sliding Window Log   | Exact timestamp deque per user             | O(N per user)              |
| Q2  | Token Bucket         | Lazy refill; tokens += elapsed × rate     | O(1) per user              |
| Q3  | Leaky Bucket         | Fixed drain; overflow = drop               | O(capacity)                |
| Q4  | Hit Counter          | (timestamp, count) pairs                   | O(distinct ts)             |
| Q5  | Multi-Endpoint Quota | Composite key userId::endpoint             | O(users × endpoints × N) |
| Q6  | Monotonic Deque Max  | Back-pop smaller values; front-pop expired | O(k)                       |
| Q7  | Moving Average       | Circular buffer; subtract outgoing         | O(N)                       |
| Q8  | Throttle/Debounce    | Single timestamp comparison                | O(1)                       |
| Q9  | Logger Dedup         | Map<msg, lastTime>; COOLDOWN check         | O(unique msgs)             |
| Q10 | Stock Ticker Max     | Monotonic deque on circular buffer         | O(k)                       |

> **Interview tip:** Q6 and Q10 are the same pattern. Monotonic deque is the go-to whenever you need min/max of a sliding window in O(1) amortised time.

---

# PATTERN 2 — Cache Design

> **When you hear:** "most recently used", "evict", "capacity limit", "cache with expiry"
> **Reach for:** HashMap for O(1) lookup + a doubly linked list (or priority structure) for O(1) eviction order.

---

## Q11. LRU Cache — O(1) get/put

### Problem Statement

Design a **Least Recently Used (LRU) cache** with fixed capacity. On `get(key)`, return the value or -1. On `put(key, value)`, insert/update the key; if capacity is exceeded, evict the least recently used key. Both operations must be **O(1)**.

**Real scenario:** Every company uses LRU — Flipkart caches product data, Swiggy caches restaurant menus, browsers cache DOM nodes.

- **Pattern:** HashMap + Doubly Linked List
- **Data Structures:** `Map<key, ListNode>` + doubly linked list (DLL)
- **Difficulty:** Medium
- **LeetCode:** 146

---

### OOP Design

```
ICache<K,V>          ← interface
DoublyLinkedList     ← manages MRU/LRU order
ListNode             ← value object (key, val, prev, next)
LRUCache             ← composes Map + DLL
```

### Approach

1. Maintain a **doubly linked list** where head = most recent, tail = least recent.
2. `Map<key, ListNode>` gives O(1) lookup by key.
3. On `get`: move the node to the head (mark as recently used), return its value.
4. On `put`: if key exists, update value + move to head. If new: add at head. If over capacity, remove the tail node and its map entry.
5. Head/tail are **sentinel nodes** — no null checks needed.

---

### TypeScript Solution

```typescript
interface ICache<K, V> {
  get(key: K): V | -1;
  put(key: K, value: V): void;
}

class ListNode<K, V> {
  constructor(
    public key: K,
    public val: V,
    public prev: ListNode<K, V> | null = null,
    public next: ListNode<K, V> | null = null
  ) {}
}

class DoublyLinkedList<K, V> {
  private readonly head: ListNode<K, V>;  // sentinel (MRU side)
  private readonly tail: ListNode<K, V>;  // sentinel (LRU side)

  constructor() {
    this.head = new ListNode<K, V>(null as any, null as any);
    this.tail = new ListNode<K, V>(null as any, null as any);
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  addToFront(node: ListNode<K, V>): void {
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next!.prev = node;
    this.head.next = node;
  }

  remove(node: ListNode<K, V>): void {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
  }

  removeLast(): ListNode<K, V> {
    const lru = this.tail.prev!;
    this.remove(lru);
    return lru;
  }
}

class LRUCache<K = number, V = number> implements ICache<K, V> {
  private readonly capacity: number;
  private readonly map: Map<K, ListNode<K, V>>;
  private readonly list: DoublyLinkedList<K, V>;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.map = new Map();
    this.list = new DoublyLinkedList();
  }

  get(key: K): V | -1 {
    const node = this.map.get(key);
    if (!node) return -1;
    // Move to front (most recently used)
    this.list.remove(node);
    this.list.addToFront(node);
    return node.val;
  }

  put(key: K, value: V): void {
    if (this.map.has(key)) {
      const node = this.map.get(key)!;
      node.val = value;
      this.list.remove(node);
      this.list.addToFront(node);
    } else {
      if (this.map.size === this.capacity) {
        const lru = this.list.removeLast();
        this.map.delete(lru.key);
      }
      const node = new ListNode(key, value);
      this.list.addToFront(node);
      this.map.set(key, node);
    }
  }
}

// Demo
const lru = new LRUCache(2);
lru.put(1, 1);   // cache: {1=1}
lru.put(2, 2);   // cache: {1=1, 2=2}
console.log(lru.get(1));   // 1  → moves 1 to front
lru.put(3, 3);   // evicts 2 (LRU), cache: {1=1, 3=3}
console.log(lru.get(2));   // -1  (evicted)
console.log(lru.get(3));   // 3
```

### Trace

```
put(1,1): list=[1]         map={1}
put(2,2): list=[2,1]       map={1,2}
get(1):   move 1 to front  list=[1,2]  → return 1
put(3,3): cap=2, evict LRU=2  list=[3,1]  map={1,3}
get(2):   not in map → -1
get(3):   move 3 to front  list=[3,3]  → return 3
```

### Complexity

|       | Value       |
| ----- | ----------- |
| get   | O(1)        |
| put   | O(1)        |
| Space | O(capacity) |

### Tradeoffs

| Approach                             | Time           | Drawback                                                        |
| ------------------------------------ | -------------- | --------------------------------------------------------------- |
| **HashMap + DLL**              | **O(1)** | **Standard answer**                                       |
| Sorted array                         | O(n)           | Shifting on access                                              |
| JavaScript `Map` (insertion order) | O(1) trick     | Delete + re-insert maintains order — cleaner but less explicit |

> **JS trick:** `Map` preserves insertion order. Delete + re-insert on get = O(1) LRU without a custom DLL. Valid in interviews if you explain it.

---

---

## Q12. LFU Cache — Least Frequently Used

### Problem Statement

Design a cache where the **least frequently used** key is evicted when capacity is exceeded. Ties in frequency are broken by LRU order (least recently among equally frequent).

**Real scenario:** Flipkart's image CDN caches product photos — hot images (accessed thousands of times) should never be evicted. Rarely accessed images should go first.

- **Pattern:** Frequency buckets + HashMap
- **Data Structures:** `Map<key, {val, freq}>` + `Map<freq, DoublyLinkedList>` + `minFreq` pointer
- **Difficulty:** Hard
- **LeetCode:** 460

---

### TypeScript Solution

```typescript
class LFUNode {
  constructor(
    public key: number, public val: number,
    public freq: number = 1,
    public prev: LFUNode | null = null,
    public next: LFUNode | null = null
  ) {}
}

class FreqList {
  public head: LFUNode;
  public tail: LFUNode;
  public size: number = 0;

  constructor() {
    this.head = new LFUNode(-1, -1);
    this.tail = new LFUNode(-1, -1);
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  addToFront(node: LFUNode): void {
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next!.prev = node;
    this.head.next = node;
    this.size++;
  }

  remove(node: LFUNode): void {
    node.prev!.next = node.next;
    node.next!.prev = node.prev;
    this.size--;
  }

  removeLast(): LFUNode {
    const node = this.tail.prev!;
    this.remove(node);
    return node;
  }
}

class LFUCache {
  private readonly capacity: number;
  private minFreq: number = 0;
  private readonly keyMap: Map<number, LFUNode>;   // key → node
  private readonly freqMap: Map<number, FreqList>; // freq → list of nodes

  constructor(capacity: number) {
    this.capacity = capacity;
    this.keyMap = new Map();
    this.freqMap = new Map();
  }

  private getOrCreateList(freq: number): FreqList {
    if (!this.freqMap.has(freq)) this.freqMap.set(freq, new FreqList());
    return this.freqMap.get(freq)!;
  }

  private promote(node: LFUNode): void {
    const oldList = this.freqMap.get(node.freq)!;
    oldList.remove(node);
    // If the old freq was the minimum and its list is now empty, increment minFreq
    if (node.freq === this.minFreq && oldList.size === 0) this.minFreq++;
    node.freq++;
    this.getOrCreateList(node.freq).addToFront(node);
  }

  get(key: number): number {
    const node = this.keyMap.get(key);
    if (!node) return -1;
    this.promote(node);
    return node.val;
  }

  put(key: number, value: number): void {
    if (this.capacity === 0) return;

    if (this.keyMap.has(key)) {
      const node = this.keyMap.get(key)!;
      node.val = value;
      this.promote(node);
      return;
    }

    if (this.keyMap.size >= this.capacity) {
      // Evict LRU from the minimum-frequency list
      const lruList = this.freqMap.get(this.minFreq)!;
      const evicted = lruList.removeLast();
      this.keyMap.delete(evicted.key);
    }

    const newNode = new LFUNode(key, value, 1);
    this.keyMap.set(key, newNode);
    this.getOrCreateList(1).addToFront(newNode);
    this.minFreq = 1;
  }
}

// Demo
const lfu = new LFUCache(2);
lfu.put(1, 1);
lfu.put(2, 2);
console.log(lfu.get(1)); // 1  (freq[1]=2, freq[2]=1)
lfu.put(3, 3);  // evicts key=2 (freq=1, LRU), cache={1,3}
console.log(lfu.get(2)); // -1
console.log(lfu.get(3)); // 3
```

### Complexity

|       | Value       |
| ----- | ----------- |
| get   | O(1)        |
| put   | O(1)        |
| Space | O(capacity) |

### Tradeoffs

|                    | LRU                     | LFU                                      |
| ------------------ | ----------------------- | ---------------------------------------- |
| Eviction criterion | Least recently accessed | Least frequently accessed                |
| Warm-up problem    | None                    | New keys start at freq=1, easily evicted |
| Complexity         | Simple HashMap + DLL    | Two HashMaps + multiple DLLs             |
| Best for           | General caching         | Stable hot-set (CDN, DNS)                |

---

---

## Q13. In-Memory Cache with TTL / Expiry

### Problem Statement

Design a key-value store where each key has a **time-to-live (TTL)**. After the TTL expires, `get` returns `null`. Expired keys should also be evicted to free memory (lazy + periodic cleanup).

**Real scenario:** Swiggy caches restaurant availability with a 30-second TTL — stale data is auto-expired. Zomato caches search results for 60 seconds.

- **Pattern:** HashMap + Min-Heap (ordered by expiry time)
- **Data Structures:** `Map<key, {value, expiresAt}>` + `MinHeap<{expiresAt, key}>`

---

### TypeScript Solution

```typescript
interface CacheEntry<V> {
  value: V;
  expiresAt: number;
}

class TTLCache<K, V> {
  private readonly store: Map<K, CacheEntry<V>>;
  // Min-heap ordered by expiry (simulated with sorted array for interview)
  private readonly expiryHeap: { expiresAt: number; key: K }[];

  constructor() {
    this.store = new Map();
    this.expiryHeap = [];
  }

  put(key: K, value: V, ttlMs: number, now: number): void {
    const expiresAt = now + ttlMs;
    this.store.set(key, { value, expiresAt });
    this.heapPush({ expiresAt, key });
  }

  get(key: K, now: number): V | null {
    this.evictExpired(now);
    const entry = this.store.get(key);
    if (!entry || now >= entry.expiresAt) {
      this.store.delete(key);
      return null;
    }
    return entry.value;
  }

  // Lazy eviction: remove expired entries from heap + map
  private evictExpired(now: number): void {
    while (this.expiryHeap.length > 0 && this.expiryHeap[0].expiresAt <= now) {
      const { key } = this.heapPop()!;
      const entry = this.store.get(key);
      // Only delete if it hasn't been refreshed (check expiresAt matches)
      if (entry && entry.expiresAt <= now) this.store.delete(key);
    }
  }

  // Simple min-heap operations
  private heapPush(item: { expiresAt: number; key: K }): void {
    this.expiryHeap.push(item);
    this.expiryHeap.sort((a, b) => a.expiresAt - b.expiresAt);
  }

  private heapPop(): { expiresAt: number; key: K } | undefined {
    return this.expiryHeap.shift();
  }
}

// Demo
const cache = new TTLCache<string, string>();
cache.put("menu:restaurant_1", "Pizza, Pasta", 5000, 1000); // expires at 6000
cache.put("menu:restaurant_2", "Burger, Fries", 3000, 1000); // expires at 4000

console.log(cache.get("menu:restaurant_1", 3000)); // "Pizza, Pasta"
console.log(cache.get("menu:restaurant_2", 5000)); // null (expired at 4000)
console.log(cache.get("menu:restaurant_1", 7000)); // null (expired at 6000)
```

### Complexity

|       | Value                                     |
| ----- | ----------------------------------------- |
| put   | O(log n) heap insert                      |
| get   | O(k log n) where k = expired keys evicted |
| Space | O(n)                                      |

### Tradeoffs

- **Lazy eviction** (on `get`) is simple but memory isn't freed until next access. Add a background sweep for production.
- **Min-heap** ensures the earliest-expiring key is always O(1) accessible, making cleanup efficient.
- **Redis approach:** Each key has an expiry stored separately; lazy delete on access + active random sampling every 100ms.

---

---

## Q14. Cache with Pluggable Eviction Policy

### Problem Statement

Design a cache where the **eviction strategy is configurable** — you can plug in LRU, LFU, FIFO, or Random eviction without changing the cache's core code. This follows the **Strategy design pattern**.

**Real scenario:** Meesho's caching layer needs to support different eviction policies per use case — LRU for session data, LFU for product images, FIFO for event logs.

- **Pattern:** Strategy Pattern
- **OOP Principle:** Open/Closed — open for extension (new eviction policies), closed for modification.

---

### TypeScript Solution

```typescript
// ── Strategy interface ───────────────────────────────────────────────────────
interface IEvictionPolicy<K> {
  onAccess(key: K): void;
  onInsert(key: K): void;
  evict(): K;
}

// ── LRU Strategy ────────────────────────────────────────────────────────────
class LRUEvictionPolicy<K> implements IEvictionPolicy<K> {
  private readonly order: Map<K, number>; // key → last access time
  private time: number = 0;

  constructor() { this.order = new Map(); }

  onAccess(key: K): void { this.order.set(key, this.time++); }
  onInsert(key: K): void { this.order.set(key, this.time++); }

  evict(): K {
    let minTime = Infinity, lruKey!: K;
    for (const [k, t] of this.order) {
      if (t < minTime) { minTime = t; lruKey = k; }
    }
    this.order.delete(lruKey);
    return lruKey;
  }
}

// ── FIFO Strategy ────────────────────────────────────────────────────────────
class FIFOEvictionPolicy<K> implements IEvictionPolicy<K> {
  private readonly queue: K[] = [];

  onAccess(_key: K): void {} // FIFO ignores access
  onInsert(key: K): void { this.queue.push(key); }
  evict(): K { return this.queue.shift()!; }
}

// ── Cache with pluggable policy ──────────────────────────────────────────────
class PluggableCache<K, V> {
  private readonly store: Map<K, V>;
  private readonly policy: IEvictionPolicy<K>;

  constructor(private readonly capacity: number, policy: IEvictionPolicy<K>) {
    this.store = new Map();
    this.policy = policy;
  }

  get(key: K): V | undefined {
    const val = this.store.get(key);
    if (val !== undefined) this.policy.onAccess(key);
    return val;
  }

  put(key: K, value: V): void {
    if (!this.store.has(key) && this.store.size >= this.capacity) {
      const evicted = this.policy.evict();
      this.store.delete(evicted);
    }
    this.store.set(key, value);
    this.policy.onInsert(key);
  }
}

// Demo
const lruCache = new PluggableCache<number, string>(2, new LRUEvictionPolicy());
lruCache.put(1, "a"); lruCache.put(2, "b");
lruCache.get(1);       // access 1 → most recent
lruCache.put(3, "c");  // evicts 2 (LRU)
console.log(lruCache.get(2)); // undefined

const fifoCache = new PluggableCache<number, string>(2, new FIFOEvictionPolicy());
fifoCache.put(1, "a"); fifoCache.put(2, "b");
fifoCache.get(1);       // access ignored by FIFO
fifoCache.put(3, "c");  // evicts 1 (first inserted)
console.log(fifoCache.get(1)); // undefined
```

### Tradeoffs

| Policy | Use case                             | Weakness                                             |
| ------ | ------------------------------------ | ---------------------------------------------------- |
| LRU    | General purpose — temporal locality | Scan resistance: sequential access pollutes cache    |
| LFU    | Stable hot-set (CDN, DNS)            | New keys unfairly evicted (cold-start problem)       |
| FIFO   | Event queues, logs                   | Ignores access frequency and recency                 |
| Random | Simple, low overhead                 | Unpredictable — not suitable for hot-data workloads |

---

---

## Q15. All O(1): inc, dec, getMax, getMin

### Problem Statement

Design a data structure with O(1) operations: `inc(key)` — increment a key's count, `dec(key)` — decrement (remove if count=0), `getMaxKey()` — return any key with max count, `getMinKey()` — return any key with min count.

- **Pattern:** Bucket DLL — each node in the DLL represents one frequency level and holds all keys with that count.
- **LeetCode:** 432
- **Asked at:** Uber, Google

---

### TypeScript Solution

```typescript
class Bucket {
  public readonly keys: Set<string>;
  public prev: Bucket | null = null;
  public next: Bucket | null = null;

  constructor(public freq: number) {
    this.keys = new Set();
  }
}

class AllOne {
  private readonly keyFreq: Map<string, number>;  // key → frequency
  private readonly freqBucket: Map<number, Bucket>; // freq → bucket
  private readonly head: Bucket; // sentinel (min side)
  private readonly tail: Bucket; // sentinel (max side)

  constructor() {
    this.keyFreq = new Map();
    this.freqBucket = new Map();
    this.head = new Bucket(0);
    this.tail = new Bucket(0);
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  private addBucketAfter(freq: number, prev: Bucket): Bucket {
    const b = new Bucket(freq);
    b.prev = prev;
    b.next = prev.next;
    prev.next!.prev = b;
    prev.next = b;
    this.freqBucket.set(freq, b);
    return b;
  }

  private removeBucket(b: Bucket): void {
    b.prev!.next = b.next;
    b.next!.prev = b.prev;
    this.freqBucket.delete(b.freq);
  }

  inc(key: string): void {
    const oldFreq = this.keyFreq.get(key) ?? 0;
    const newFreq = oldFreq + 1;
    this.keyFreq.set(key, newFreq);

    const oldBucket = this.freqBucket.get(oldFreq);
    // Get or create bucket for newFreq, immediately after oldBucket (or head)
    const prevBucket = oldBucket ?? this.head;
    let newBucket = this.freqBucket.get(newFreq);
    if (!newBucket || newBucket.prev !== prevBucket) {
      newBucket = this.addBucketAfter(newFreq, prevBucket);
    }
    newBucket.keys.add(key);

    if (oldBucket) {
      oldBucket.keys.delete(key);
      if (oldBucket.keys.size === 0) this.removeBucket(oldBucket);
    }
  }

  dec(key: string): void {
    const oldFreq = this.keyFreq.get(key);
    if (!oldFreq) return;
    const newFreq = oldFreq - 1;

    if (newFreq === 0) {
      this.keyFreq.delete(key);
    } else {
      this.keyFreq.set(key, newFreq);
      let newBucket = this.freqBucket.get(newFreq);
      const oldBucket = this.freqBucket.get(oldFreq)!;
      if (!newBucket) newBucket = this.addBucketAfter(newFreq, oldBucket.prev!);
      newBucket.keys.add(key);
    }
    const oldBucket = this.freqBucket.get(oldFreq)!;
    oldBucket.keys.delete(key);
    if (oldBucket.keys.size === 0) this.removeBucket(oldBucket);
  }

  getMaxKey(): string {
    if (this.tail.prev === this.head) return "";
    return this.tail.prev!.keys.values().next().value;
  }

  getMinKey(): string {
    if (this.head.next === this.tail) return "";
    return this.head.next!.keys.values().next().value;
  }
}

// Demo
const ao = new AllOne();
ao.inc("a"); ao.inc("b"); ao.inc("b"); ao.inc("c"); ao.inc("c"); ao.inc("c");
console.log(ao.getMaxKey()); // "c" (freq=3)
console.log(ao.getMinKey()); // "a" (freq=1)
ao.dec("c"); ao.dec("c");
console.log(ao.getMaxKey()); // "b" or "c" (both freq=1 now... wait: b=2,c=1,a=1)
```

### Complexity

|         | Value     |
| ------- | --------- |
| All ops | O(1)      |
| Space   | O(n) keys |

### Tradeoffs

- **Why not HashMap + heap?** Heap gives O(log n) for inc/dec due to re-heapify.
- **Bucket DLL** keeps O(1) because adjacent frequencies are directly linked — no searching needed.
- **Complexity:** The hardest cache question. Key insight: buckets are ordered by frequency and you always move to an adjacent bucket (freq±1).

---

## Pattern 2 — Summary

| Q             | Structure                 | Eviction Trigger               | Time              |
| ------------- | ------------------------- | ------------------------------ | ----------------- |
| Q11 LRU       | HashMap + DLL             | Least recently accessed        | O(1)              |
| Q12 LFU       | 2× HashMap + DLL/bucket  | Least frequently accessed      | O(1)              |
| Q13 TTL       | HashMap + Min-Heap        | Time-based expiry              | O(log n) put      |
| Q14 Pluggable | Strategy + HashMap        | Policy-injected                | Depends on policy |
| Q15 AllOne    | HashMap + Freq-bucket DLL | — (no eviction, O(1) min/max) | O(1) all ops      |

---

# PATTERN 3 â€” Trie / Prefix

> **When you hear:** "autocomplete", "prefix", "starts with", "suggest words"
> **Reach for:** A Trie (prefix tree) â€” each node represents one character; paths from root to leaf spell out words.

---

## Q16. Search Autocomplete / Typeahead

### Problem Statement

Design a typeahead system: given a list of words inserted upfront, for any prefix query return all words that start with that prefix.

**Real scenario:** Flipkart's search bar suggests products as you type. Myntra shows matching brand/category names on each keystroke.

- **Pattern:** Trie + DFS to collect all words with given prefix
- **Difficulty:** Medium | **LeetCode:** 208 / 642
- **Asked at:** Flipkart, Myntra, Amazon

---

### TypeScript Solution

```typescript
interface IAutocomplete {
  insert(word: string): void;
  search(prefix: string): string[];
}

class TrieNode {
  children: Map<string, TrieNode> = new Map();
  isEnd: boolean = false;
}

class AutocompleteSystem implements IAutocomplete {
  private readonly root: TrieNode = new TrieNode();

  insert(word: string): void {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
      node = node.children.get(ch)!;
    }
    node.isEnd = true;
  }

  search(prefix: string): string[] {
    let node = this.root;
    for (const ch of prefix) {
      if (!node.children.has(ch)) return [];
      node = node.children.get(ch)!;
    }
    const results: string[] = [];
    this.dfs(node, prefix, results);
    return results;
  }

  private dfs(node: TrieNode, current: string, results: string[]): void {
    if (node.isEnd) results.push(current);
    for (const [ch, child] of node.children) {
      this.dfs(child, current + ch, results);
    }
  }
}

// Demo
const ac = new AutocompleteSystem();
["mobile","monitor","mouse","keyboard","macbook"].forEach(w => ac.insert(w));
console.log(ac.search("mo")); // ["mobile","monitor","mouse"]
console.log(ac.search("ma")); // ["macbook"]
```

### Complexity

|        | Value                              |
| ------ | ---------------------------------- |
| insert | O(L) â€” L = word length        |
| search | O(L + N) â€” N = matching words |
| Space  | O(total characters in all words)   |

### Tradeoffs

- **Pure DFS** returns all matches but is unbounded. **Limit results** with a `topK` parameter using a heap during DFS.
- **Alternative:** Sorted array + binary search for immutable word sets â€” simpler but O(log n + k) per query.
- **Real-world:** Elasticsearch and Solr use inverted indexes, not tries, for multi-word search.

---

## Q17. Autocomplete Ranked by Popularity

### Problem Statement

Extend autocomplete so each word has a **popularity score**. For a prefix query, return the **top 3 most popular** matching words.

**Real scenario:** Swiggy/Zomato rank dish suggestions by order count. "butter" autocompletes to "butter chicken" (most ordered) first.

- **Pattern:** Trie + sort top-K during collection
- **Difficulty:** Hard | **Asked at:** Swiggy, Zomato, Google

---

### TypeScript Solution

```typescript
class RankedTrieNode {
  children: Map<string, RankedTrieNode> = new Map();
  word: string | null = null;
  score: number = 0;
}

class RankedAutocomplete {
  private readonly root = new RankedTrieNode();
  private readonly K: number;

  constructor(topK: number = 3) { this.K = topK; }

  insert(word: string, score: number): void {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new RankedTrieNode());
      node = node.children.get(ch)!;
    }
    node.word = word;
    node.score = score;
  }

  search(prefix: string): { word: string; score: number }[] {
    let node = this.root;
    for (const ch of prefix) {
      if (!node.children.has(ch)) return [];
      node = node.children.get(ch)!;
    }
    const candidates: { word: string; score: number }[] = [];
    this.dfs(node, candidates);
    return candidates.sort((a, b) => b.score - a.score).slice(0, this.K);
  }

  private dfs(node: RankedTrieNode, out: { word: string; score: number }[]): void {
    if (node.word) out.push({ word: node.word, score: node.score });
    for (const child of node.children.values()) this.dfs(child, out);
  }
}

// Demo
const ranked = new RankedAutocomplete(3);
[["butter chicken", 900], ["butter naan", 400], ["butter paneer", 600],
 ["buttermilk", 200], ["butterscotch", 750]].forEach(([w, s]) =>
  ranked.insert(w as string, s as number));
console.log(ranked.search("butter"));
// [{word:"butter chicken",score:900},{word:"butterscotch",score:750},{word:"butter paneer",score:600}]
```

### Tradeoffs

- **Cache top-K at each Trie node** (update on insert) => O(K) per query instead of O(subtree size). Trades space for speed.
- **DFS + sort** is simpler for interviews. In production, pre-compute top-K per node.

---

## Q18. Contact Search by Name Prefix

### Problem Statement

Build a phone contact search: given a name prefix, return all contacts whose name starts with that prefix (case-insensitive).

- **Pattern:** Trie on lowercase names | **Difficulty:** Medium | **Asked at:** Truecaller-style

---

### TypeScript Solution

```typescript
interface Contact { name: string; phone: string; }

class ContactSearch {
  private readonly root = new TrieNode();
  private readonly contactMap: Map<string, Contact[]> = new Map();

  addContact(contact: Contact): void {
    const key = contact.name.toLowerCase();
    this.insertTrie(key);
    if (!this.contactMap.has(key)) this.contactMap.set(key, []);
    this.contactMap.get(key)!.push(contact);
  }

  findByPrefix(prefix: string): Contact[] {
    const matched = this.searchTrie(prefix.toLowerCase());
    return matched.flatMap(name => this.contactMap.get(name) ?? []);
  }

  private insertTrie(word: string): void {
    let node = this.root;
    for (const ch of word) {
      if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
      node = node.children.get(ch)!;
    }
    node.isEnd = true;
  }

  private searchTrie(prefix: string): string[] {
    let node = this.root;
    for (const ch of prefix) {
      if (!node.children.has(ch)) return [];
      node = node.children.get(ch)!;
    }
    const res: string[] = [];
    this.collectWords(node, prefix, res);
    return res;
  }

  private collectWords(node: TrieNode, cur: string, res: string[]): void {
    if (node.isEnd) res.push(cur);
    for (const [ch, child] of node.children) this.collectWords(child, cur + ch, res);
  }
}

// Demo
const cs = new ContactSearch();
cs.addContact({ name: "Rahul Sharma", phone: "9876543210" });
cs.addContact({ name: "Rahul Verma", phone: "9123456789" });
cs.addContact({ name: "Priya Singh", phone: "9988776655" });
console.log(cs.findByPrefix("rah")); // both Rahuls
```

### Tradeoffs

- **Case normalisation** (toLowerCase) at insert time is simpler than at query time â€” do it once.
- **Fuzzy matching:** Trie gives prefix-exact. For fuzzy (typos), combine with edit distance or a trigram index.

---

## Q19. URL Path Router

### Problem Statement

Build a URL router that matches parameterised paths like `/user/:id/orders`. A request to `/user/42/orders` should match and extract `{id: "42"}`.

- **Pattern:** Trie on path segments | **Difficulty:** Medium | **Asked at:** Postman, BrowserStack

---

### TypeScript Solution

```typescript
interface RouteMatch { handler: string; params: Record<string, string>; }

class RouterNode {
  children: Map<string, RouterNode> = new Map();
  paramChild: RouterNode | null = null;
  paramName: string | null = null;
  handler: string | null = null;
}

class URLRouter {
  private readonly root = new RouterNode();

  register(path: string, handler: string): void {
    let node = this.root;
    for (const seg of path.split("/").filter(Boolean)) {
      if (seg.startsWith(":")) {
        if (!node.paramChild) {
          node.paramChild = new RouterNode();
          node.paramChild.paramName = seg.slice(1);
        }
        node = node.paramChild;
      } else {
        if (!node.children.has(seg)) node.children.set(seg, new RouterNode());
        node = node.children.get(seg)!;
      }
    }
    node.handler = handler;
  }

  match(path: string): RouteMatch | null {
    const params: Record<string, string> = {};
    let node = this.root;
    for (const seg of path.split("/").filter(Boolean)) {
      if (node.children.has(seg)) {
        node = node.children.get(seg)!;
      } else if (node.paramChild) {
        params[node.paramChild.paramName!] = seg;
        node = node.paramChild;
      } else return null;
    }
    return node.handler ? { handler: node.handler, params } : null;
  }
}

// Demo
const router = new URLRouter();
router.register("/user/:id/orders", "getUserOrders");
router.register("/product/:pid", "getProduct");
console.log(router.match("/user/42/orders")); // {handler:"getUserOrders", params:{id:"42"}}
console.log(router.match("/product/789"));    // {handler:"getProduct", params:{pid:"789"}}
```

### Tradeoffs

- **Static segment wins over param** â€” exact match preferred over wildcard.
- **Wildcard `*`** can be added as another special child node type.
- **Express.js** uses a similar segment-based trie with regex support for complex patterns.

---

## Q20. Word Dictionary with `.` Wildcard

### Problem Statement

Design a `WordDictionary` supporting `addWord(word)` and `search(pattern)` where `.` matches any single character.

- **Pattern:** Trie + DFS | **Difficulty:** Medium | **LeetCode:** 211 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class WildcardTrieNode {
  children: (WildcardTrieNode | null)[] = new Array(26).fill(null);
  isEnd: boolean = false;
}

class WordDictionary {
  private readonly root = new WildcardTrieNode();

  addWord(word: string): void {
    let node = this.root;
    for (const ch of word) {
      const i = ch.charCodeAt(0) - 97;
      if (!node.children[i]) node.children[i] = new WildcardTrieNode();
      node = node.children[i]!;
    }
    node.isEnd = true;
  }

  search(pattern: string): boolean {
    return this.dfs(this.root, pattern, 0);
  }

  private dfs(node: WildcardTrieNode | null, pattern: string, idx: number): boolean {
    if (!node) return false;
    if (idx === pattern.length) return node.isEnd;
    const ch = pattern[idx];
    if (ch === ".") {
      return node.children.some(child => this.dfs(child, pattern, idx + 1));
    }
    return this.dfs(node.children[ch.charCodeAt(0) - 97], pattern, idx + 1);
  }
}

// Demo
const wd = new WordDictionary();
["bad","dad","mad"].forEach(w => wd.addWord(w));
console.log(wd.search(".ad")); // true
console.log(wd.search("b..")); // true
console.log(wd.search("pad")); // false
```

### Complexity

|                        | Value              |
| ---------------------- | ------------------ |
| addWord                | O(L)               |
| search (no wildcard)   | O(L)               |
| search (all wildcards) | O(26^L) worst case |

### Tradeoffs

- **Array[26]** children is faster than Map for lowercase-only alphabets â€” better cache locality.
- **Worst case** for wildcards is exponential â€” add depth limit for production use.

---

## Q21. Replace Words with Shortest Root

### Problem Statement

Given a dictionary of roots, replace every word in a sentence with its **shortest matching root**. If no root matches, keep the word.

- **Pattern:** Trie for prefix lookup | **Difficulty:** Medium | **LeetCode:** 648 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class RootReplacer {
  private readonly root = new TrieNode();

  buildDictionary(roots: string[]): void {
    for (const r of roots) {
      let node = this.root;
      for (const ch of r) {
        if (!node.children.has(ch)) node.children.set(ch, new TrieNode());
        node = node.children.get(ch)!;
      }
      node.isEnd = true;
    }
  }

  private findRoot(word: string): string {
    let node = this.root;
    for (let i = 0; i < word.length; i++) {
      const ch = word[i];
      if (!node.children.has(ch)) break;
      node = node.children.get(ch)!;
      if (node.isEnd) return word.slice(0, i + 1);
    }
    return word;
  }

  replaceWords(sentence: string): string {
    return sentence.split(" ").map(w => this.findRoot(w)).join(" ");
  }
}

// Demo
const rr = new RootReplacer();
rr.buildDictionary(["cat","bat","rat"]);
console.log(rr.replaceWords("the cattle was rattled by the battery"));
// "the cat was rat by the bat"
```

### Tradeoffs

- **Trie gives O(L) per word** vs O(R*L) brute force checking all roots.
- **Shortest root first:** Trie naturally returns the shortest because we stop at the first `isEnd` during traversal.

---

## Q22. Spell-Check â€” Suggest Nearest Valid Word

### Problem Statement

Given a misspelled word, suggest the **closest valid words** from a dictionary using edit distance. Return top-3 suggestions.

- **Pattern:** Edit distance DP | **Difficulty:** Hard | **Asked at:** Google

---

### TypeScript Solution

```typescript
class SpellChecker {
  private readonly dictionary: string[] = [];

  addWord(word: string): void { this.dictionary.push(word.toLowerCase()); }

  suggest(query: string, topK: number = 3): string[] {
    const q = query.toLowerCase();
    return this.dictionary
      .map(word => ({ word, dist: this.editDistance(q, word) }))
      .sort((a, b) => a.dist - b.dist)
      .slice(0, topK)
      .map(x => x.word);
  }

  private editDistance(a: string, b: string): number {
    const m = a.length, n = b.length;
    const dp: number[][] = Array.from({ length: m + 1 }, (_, i) =>
      Array.from({ length: n + 1 }, (_, j) => (i === 0 ? j : j === 0 ? i : 0))
    );
    for (let i = 1; i <= m; i++) {
      for (let j = 1; j <= n; j++) {
        dp[i][j] = a[i-1] === b[j-1]
          ? dp[i-1][j-1]
          : 1 + Math.min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1]);
      }
    }
    return dp[m][n];
  }
}

// Demo
const spchk = new SpellChecker();
["apple","application","apply","apt","ape","ability"].forEach(w => spchk.addWord(w));
console.log(spchk.suggest("appl")); // ["apple","apply","apt"]
```

### Complexity

|         | Value                                           |
| ------- | ----------------------------------------------- |
| suggest | O(D * L^2) where D=dict size, L=avg word length |
| Space   | O(L^2) for DP table                             |

### Tradeoffs

- **BK-Tree:** Organises words by edit distance, allowing O(log D) spell-check queries. Preferred for large dictionaries.
- **Trie + DP combination:** At each Trie node maintain a DP row â€” prune branches where min possible distance > threshold.

---

## Pattern 3 â€” Summary

| Q                | Technique        | Key Insight                                     |
| ---------------- | ---------------- | ----------------------------------------------- |
| Q16 Autocomplete | Trie + DFS       | Walk to prefix node, collect all descendants    |
| Q17 Ranked       | Trie + sort      | Cache top-K per node for O(K) queries           |
| Q18 Contact      | Trie + Map       | Store contacts at leaf nodes by normalised name |
| Q19 URL Router   | Segment Trie     | Param nodes (`:id`) as special children       |
| Q20 Wildcard     | Trie + DFS `.` | Try all 26 children on `.`                    |
| Q21 Root Replace | Trie early exit  | Return on first `isEnd` during traversal      |
| Q22 Spell Check  | Edit distance DP | BK-Tree for O(log n) in production              |

---

# PATTERN 4 â€” Heap / Priority Queue

> **When you hear:** "top K", "priority", "next job", "leaderboard", "median"
> **Reach for:** A min-heap (for top-K largest) or max-heap (for top-K smallest). Two heaps for median.

---

## Q23. Task Scheduler â€” Always Run Highest Priority

### Problem Statement

Design a task scheduler where tasks have priorities. Always execute the **highest-priority task** next.

**Real scenario:** Uber's dispatch assigns rides by priority. Ola's support queue processes urgent tickets first.

- **Pattern:** Max-Heap | **Difficulty:** Medium | **Asked at:** Uber, Ola

---

### TypeScript Solution

```typescript
interface Task { id: string; name: string; priority: number; }

class MaxHeap<T> {
  private readonly heap: T[] = [];
  constructor(private readonly compareFn: (a: T, b: T) => number) {}

  push(item: T): void {
    this.heap.push(item);
    this.bubbleUp(this.heap.length - 1);
  }

  pop(): T | undefined {
    if (this.heap.length === 0) return undefined;
    const top = this.heap[0];
    const last = this.heap.pop()!;
    if (this.heap.length > 0) { this.heap[0] = last; this.siftDown(0); }
    return top;
  }

  peek(): T | undefined { return this.heap[0]; }
  size(): number { return this.heap.length; }

  private bubbleUp(i: number): void {
    while (i > 0) {
      const parent = (i - 1) >> 1;
      if (this.compareFn(this.heap[i], this.heap[parent]) > 0) {
        [this.heap[i], this.heap[parent]] = [this.heap[parent], this.heap[i]];
        i = parent;
      } else break;
    }
  }

  private siftDown(i: number): void {
    const n = this.heap.length;
    while (true) {
      let largest = i;
      const l = 2*i+1, r = 2*i+2;
      if (l < n && this.compareFn(this.heap[l], this.heap[largest]) > 0) largest = l;
      if (r < n && this.compareFn(this.heap[r], this.heap[largest]) > 0) largest = r;
      if (largest === i) break;
      [this.heap[i], this.heap[largest]] = [this.heap[largest], this.heap[i]];
      i = largest;
    }
  }
}

class MinHeap<T> {
  private readonly heap: T[] = [];
  constructor(private readonly compareFn: (a: T, b: T) => number) {}

  push(item: T): void {
    this.heap.push(item);
    let i = this.heap.length - 1;
    while (i > 0) {
      const p = (i-1) >> 1;
      if (this.compareFn(this.heap[i], this.heap[p]) < 0) {
        [this.heap[i], this.heap[p]] = [this.heap[p], this.heap[i]]; i = p;
      } else break;
    }
  }

  pop(): T | undefined {
    if (!this.heap.length) return undefined;
    const top = this.heap[0];
    const last = this.heap.pop()!;
    if (this.heap.length) { this.heap[0] = last; this.siftDown(0); }
    return top;
  }

  peek(): T | undefined { return this.heap[0]; }
  size(): number { return this.heap.length; }

  private siftDown(i: number): void {
    const n = this.heap.length;
    while (true) {
      let min = i;
      const l = 2*i+1, r = 2*i+2;
      if (l < n && this.compareFn(this.heap[l], this.heap[min]) < 0) min = l;
      if (r < n && this.compareFn(this.heap[r], this.heap[min]) < 0) min = r;
      if (min === i) break;
      [this.heap[i], this.heap[min]] = [this.heap[min], this.heap[i]]; i = min;
    }
  }
}

class TaskScheduler {
  private readonly pq: MaxHeap<Task>;
  constructor() { this.pq = new MaxHeap<Task>((a, b) => a.priority - b.priority); }

  addTask(task: Task): void { this.pq.push(task); }

  runNext(): Task | null {
    const task = this.pq.pop();
    if (!task) { console.log("No tasks"); return null; }
    console.log(`Running: [P${task.priority}] ${task.name}`);
    return task;
  }

  hasTasks(): boolean { return this.pq.size() > 0; }
}

// Demo
const scheduler = new TaskScheduler();
scheduler.addTask({ id:"1", name:"Send invoice",  priority: 3 });
scheduler.addTask({ id:"2", name:"DB backup",     priority: 5 });
scheduler.addTask({ id:"3", name:"Send email",    priority: 1 });
scheduler.addTask({ id:"4", name:"Payment retry", priority: 4 });
while (scheduler.hasTasks()) scheduler.runNext();
// P5: DB backup -> P4: Payment retry -> P3: Send invoice -> P1: Send email
```

### Complexity | Time: O(log n) per add/run | Space: O(n)

### Tradeoffs

- **Same priority:** add a secondary sort key (insertion time) for FIFO among equal priorities.
- **Alternative:** `SortedList` (TreeMap) gives O(log n) but supports iteration over all priorities.

---

## Q24. Delay Queue â€” Run Job After T Seconds

### Problem Statement

Schedule a job to run at a specific future time. The queue always returns the **next job due** (earliest scheduled time).

**Real scenario:** Razorpay schedules payment retries after 5 minutes. Zeta runs settlement jobs at end-of-day.

- **Pattern:** Min-Heap keyed by scheduled time | **Difficulty:** Medium | **Asked at:** Razorpay, Zeta

---

### TypeScript Solution

```typescript
interface ScheduledJob { id: string; runAt: number; payload: string; }

class DelayQueue {
  private readonly pq: MinHeap<ScheduledJob>;
  constructor() { this.pq = new MinHeap<ScheduledJob>((a, b) => a.runAt - b.runAt); }

  schedule(job: ScheduledJob): void { this.pq.push(job); }

  poll(currentTime: number): ScheduledJob[] {
    const ready: ScheduledJob[] = [];
    while (this.pq.peek() && this.pq.peek()!.runAt <= currentTime) {
      ready.push(this.pq.pop()!);
    }
    return ready;
  }

  nextJobTime(): number | null { return this.pq.peek()?.runAt ?? null; }
}

// Demo
const dq = new DelayQueue();
dq.schedule({ id:"j1", runAt: 1000, payload: "Retry payment A" });
dq.schedule({ id:"j2", runAt:  500, payload: "Send OTP" });
dq.schedule({ id:"j3", runAt: 2000, payload: "Generate report" });
console.log(dq.poll(600));  // [{id:"j2"...}]
console.log(dq.poll(1500)); // [{id:"j1"...}]
console.log(dq.poll(3000)); // [{id:"j3"...}]
```

### Tradeoffs

- **Min-heap** gives O(log n) schedule, O(log n) poll. Constant-time peek to check if anything is due.
- **Time-wheel** (used in Linux kernel, Kafka): O(1) schedule + O(1) tick. Better for millions of timers.
- **Redis ZSET** with score=runAt: `ZRANGEBYSCORE 0 now` gets all due jobs atomically.

---

## Q25. Retry Queue with Exponential Backoff

### Problem Statement

Failed jobs are re-scheduled with exponentially increasing delays (1s, 2s, 4s, 8s...). The queue returns the next job that is due.

**Real scenario:** PhonePe retries failed payment webhook deliveries with exponential backoff.

- **Pattern:** Min-Heap by next retry time | **Difficulty:** Medium | **Asked at:** PhonePe

---

### TypeScript Solution

```typescript
interface RetryJob {
  id: string;
  payload: string;
  nextRetryAt: number;
  attempts: number;
  maxAttempts: number;
}

class RetryQueue {
  private readonly pq: MinHeap<RetryJob>;
  private readonly BASE_DELAY_MS = 1000;
  private readonly MAX_DELAY_MS  = 60000;

  constructor() { this.pq = new MinHeap<RetryJob>((a, b) => a.nextRetryAt - b.nextRetryAt); }

  enqueue(id: string, payload: string, maxAttempts: number, now: number): void {
    this.pq.push({ id, payload, nextRetryAt: now, attempts: 0, maxAttempts });
  }

  poll(currentTime: number): RetryJob[] {
    const due: RetryJob[] = [];
    while (this.pq.peek() && this.pq.peek()!.nextRetryAt <= currentTime) {
      due.push(this.pq.pop()!);
    }
    return due;
  }

  requeue(job: RetryJob, currentTime: number): boolean {
    if (job.attempts >= job.maxAttempts) {
      console.log(`Job ${job.id} exhausted retries`);
      return false;
    }
    const delay = Math.min(this.BASE_DELAY_MS * 2 ** job.attempts, this.MAX_DELAY_MS);
    job.attempts++;
    job.nextRetryAt = currentTime + delay;
    this.pq.push(job);
    return true;
  }
}

// Demo
const rq = new RetryQueue();
rq.enqueue("job_1", "POST /webhook payload", 4, 0);
let now = 0;
const job = rq.poll(0)[0];
rq.requeue(job, now); // retry at t=1000ms
now = 1000;
const job2 = rq.poll(now)[0];
rq.requeue(job2, now); // retry at t=3000ms
```

### Tradeoffs

- **Exponential backoff + jitter** (add random noise) prevents thundering herd.
- **Dead Letter Queue (DLQ):** Jobs exceeding maxAttempts go to a separate queue for manual investigation.
- **Idempotency:** Consumer must be idempotent â€” retries may deliver the same job multiple times.

---

## Q26. Top-K Trending Hashtags / Products

### Problem Statement

Given a stream of events, maintain the **top K most frequent** items at any time.

**Real scenario:** ShareChat shows "trending topics" â€” top-10 hashtags by post count.

- **Pattern:** HashMap + Min-Heap of size K | **Difficulty:** Medium | **LeetCode:** 347 | **Asked at:** ShareChat, Meesho

---

### TypeScript Solution

```typescript
class TopKTracker {
  private readonly freqMap: Map<string, number> = new Map();
  private readonly k: number;
  constructor(k: number) { this.k = k; }

  record(item: string): void {
    this.freqMap.set(item, (this.freqMap.get(item) ?? 0) + 1);
  }

  getTopK(): { item: string; count: number }[] {
    const minHeap = new MinHeap<{ item: string; count: number }>((a, b) => a.count - b.count);
    for (const [item, count] of this.freqMap) {
      minHeap.push({ item, count });
      if (minHeap.size() > this.k) minHeap.pop();
    }
    const result: { item: string; count: number }[] = [];
    while (minHeap.size()) result.push(minHeap.pop()!);
    return result.reverse();
  }
}

// Demo
const topK = new TopKTracker(3);
["cricket","cricket","cricket","bollywood","bollywood","ipl","ipl","ipl","ipl","food"]
  .forEach(t => topK.record(t));
console.log(topK.getTopK());
// [{item:"ipl",count:4},{item:"cricket",count:3},{item:"bollywood",count:2}]
```

### Complexity

|         | Value      |
| ------- | ---------- |
| record  | O(1)       |
| getTopK | O(n log k) |
| Space   | O(n)       |

### Tradeoffs

- **Min-heap of size K** is more efficient than sorting all n items: O(n log k) vs O(n log n).
- **Count-Min Sketch + Heap:** For massive streams, use probabilistic sketch to approximate counts in O(1) space.

---

## Q27. Live Leaderboard â€” Top N Players

### Problem Statement

Design a live leaderboard with `addScore(playerId, score)`, `getTopN(n)`, and `reset(playerId)`.

**Real scenario:** Dream11 shows the top-100 teams live during a cricket match.

- **Pattern:** HashMap + sort on query | **Difficulty:** Medium | **LeetCode:** 1244 | **Asked at:** Dream11, MPL

---

### TypeScript Solution

```typescript
class Leaderboard {
  private readonly scores: Map<string, number> = new Map();

  addScore(playerId: string, score: number): void {
    this.scores.set(playerId, (this.scores.get(playerId) ?? 0) + score);
  }

  getTopN(n: number): { playerId: string; score: number }[] {
    return Array.from(this.scores.entries())
      .map(([playerId, score]) => ({ playerId, score }))
      .sort((a, b) => b.score - a.score)
      .slice(0, n);
  }

  reset(playerId: string): void { this.scores.delete(playerId); }

  getRank(playerId: string): number {
    const playerScore = this.scores.get(playerId) ?? 0;
    let rank = 1;
    for (const score of this.scores.values()) if (score > playerScore) rank++;
    return rank;
  }
}

// Demo
const lb = new Leaderboard();
[["rahul",50],["priya",80],["amit",60],["sneha",80],["vikram",90]]
  .forEach(([p,s]) => lb.addScore(p as string, s as number));
lb.addScore("rahul", 40);
console.log(lb.getTopN(3));
```

### Tradeoffs

| Approach             | getTopN      | addScore | Use case                     |
| -------------------- | ------------ | -------- | ---------------------------- |
| Sort on query        | O(n log n)   | O(1)     | Few reads, many updates      |
| HashMap + sort       | O(n log n)   | O(1)     | Standard interview answer    |
| Sorted set (TreeMap) | O(k)         | O(log n) | Many reads, moderate updates |
| Redis ZSET           | O(log n + k) | O(log n) | Production real-time         |

---

## Q28. Median of a Number Stream

### Problem Statement

Support `addNum(n)` and `findMedian()` on a live number stream.

**Real scenario:** Uber's surge pricing computes the median trip time. Google Analytics computes rolling median session duration.

- **Pattern:** Two Heaps (max-heap lower half, min-heap upper half)
- **Difficulty:** Hard | **LeetCode:** 295 | **Asked at:** Uber, Google

---

### TypeScript Solution

```typescript
class MedianFinder {
  private readonly lowerHalf: MaxHeap<number>;
  private readonly upperHalf: MinHeap<number>;

  constructor() {
    this.lowerHalf = new MaxHeap<number>((a, b) => a - b);
    this.upperHalf = new MinHeap<number>((a, b) => a - b);
  }

  addNum(num: number): void {
    this.lowerHalf.push(num);
    if (this.upperHalf.size() > 0 && this.lowerHalf.peek()! > this.upperHalf.peek()!) {
      this.upperHalf.push(this.lowerHalf.pop()!);
    }
    if (this.lowerHalf.size() > this.upperHalf.size() + 1) {
      this.upperHalf.push(this.lowerHalf.pop()!);
    } else if (this.upperHalf.size() > this.lowerHalf.size()) {
      this.lowerHalf.push(this.upperHalf.pop()!);
    }
  }

  findMedian(): number {
    if (this.lowerHalf.size() > this.upperHalf.size()) return this.lowerHalf.peek()!;
    return (this.lowerHalf.peek()! + this.upperHalf.peek()!) / 2;
  }
}

// Demo
const mf = new MedianFinder();
[1, 2, 3, 4, 5].forEach(n => { mf.addNum(n); console.log(`added ${n} median=${mf.findMedian()}`); });
// 1->1, 2->1.5, 3->2, 4->2.5, 5->3
```

### Trace

```
add(1): lower=[1]    upper=[]     median=1
add(2): lower=[1]    upper=[2]    median=1.5
add(3): lower=[2]    upper=[3]    median=2 (after rebalance: lower=[1,2] -> lower has 2, upper=[3])
add(4): lower=[1,2]  upper=[3,4]  median=2.5
add(5): lower=[1,2,3] upper=[4,5] median=3
```

### Complexity | addNum: O(log n) | findMedian: O(1) | Space: O(n)

### Tradeoffs

- **Two heaps invariant:** lower.max <= upper.min always holds. Size diff <= 1.
- **Order statistics tree** gives O(log n) for any percentile, not just median.
- **Approximate:** For large streams, use reservoir sampling or CKMS sketch for O(1) space.

---

## Q29. Stock Price Tracker

### Problem Statement

`update(timestamp, price)`, `current()` (latest price), `maximum()`, `minimum()`.

- **Pattern:** Two Heaps (lazy deletion) + HashMap | **Difficulty:** Hard | **LeetCode:** 2034 | **Asked at:** Zerodha, Groww

---

### TypeScript Solution

```typescript
class StockPrice {
  private readonly prices: Map<number, number> = new Map();
  private latestTs: number = 0;
  private readonly maxHeap: MaxHeap<[number, number]>;
  private readonly minHeap: MinHeap<[number, number]>;

  constructor() {
    this.maxHeap = new MaxHeap<[number, number]>((a, b) => a[1] - b[1]);
    this.minHeap = new MinHeap<[number, number]>((a, b) => a[1] - b[1]);
  }

  update(timestamp: number, price: number): void {
    this.prices.set(timestamp, price);
    if (timestamp >= this.latestTs) this.latestTs = timestamp;
    this.maxHeap.push([timestamp, price]);
    this.minHeap.push([timestamp, price]);
  }

  current(): number { return this.prices.get(this.latestTs)!; }

  maximum(): number {
    while (this.maxHeap.peek() &&
           this.prices.get(this.maxHeap.peek()![0]) !== this.maxHeap.peek()![1]) {
      this.maxHeap.pop();
    }
    return this.maxHeap.peek()![1];
  }

  minimum(): number {
    while (this.minHeap.peek() &&
           this.prices.get(this.minHeap.peek()![0]) !== this.minHeap.peek()![1]) {
      this.minHeap.pop();
    }
    return this.minHeap.peek()![1];
  }
}

// Demo
const sp = new StockPrice();
sp.update(1, 100); sp.update(2, 150); sp.update(1, 130);
console.log(sp.current());  // 150
console.log(sp.maximum());  // 150
console.log(sp.minimum());  // 130
```

### Tradeoffs

- **Lazy deletion** avoids O(n) heap rebuilds. Stale entries are removed only when they reach the top.
- **Alternative:** A sorted map gives O(log n) for max/min without stale entry issues.

---

## Q30. Order Matching Engine

### Problem Statement

Buyers submit **bids** (max price to pay) and sellers submit **asks** (min price to accept). Match a bid with the lowest ask <= bid price.

**Real scenario:** Zerodha's order matching engine, NSE/BSE exchange systems.

- **Pattern:** Max-Heap (bids) + Min-Heap (asks) | **Difficulty:** Hard | **Asked at:** Zerodha, DE Shaw

---

### TypeScript Solution

```typescript
interface Order { id: string; price: number; qty: number; trader: string; }

class OrderBook {
  private readonly bids: MaxHeap<Order>;
  private readonly asks: MinHeap<Order>;
  private readonly trades: { buyId: string; sellId: string; price: number; qty: number }[] = [];

  constructor() {
    this.bids = new MaxHeap<Order>((a, b) => a.price - b.price);
    this.asks = new MinHeap<Order>((a, b) => a.price - b.price);
  }

  placeBid(order: Order): void { this.bids.push(order); this.match(); }
  placeAsk(order: Order): void { this.asks.push(order); this.match(); }

  private match(): void {
    while (this.bids.size() > 0 && this.asks.size() > 0) {
      const topBid = this.bids.peek()!;
      const topAsk = this.asks.peek()!;
      if (topBid.price < topAsk.price) break;

      const bid = this.bids.pop()!;
      const ask = this.asks.pop()!;
      const tradePrice = ask.price;
      const tradeQty   = Math.min(bid.qty, ask.qty);

      this.trades.push({ buyId: bid.id, sellId: ask.id, price: tradePrice, qty: tradeQty });
      console.log(`TRADE: ${bid.trader} buys ${tradeQty} @ ${tradePrice} from ${ask.trader}`);

      if (bid.qty > tradeQty) this.bids.push({ ...bid, qty: bid.qty - tradeQty });
      if (ask.qty > tradeQty) this.asks.push({ ...ask, qty: ask.qty - tradeQty });
    }
  }

  getTrades() { return this.trades; }
}

// Demo
const ob = new OrderBook();
ob.placeAsk({ id:"a1", price:100, qty:10, trader:"Seller_A" });
ob.placeAsk({ id:"a2", price: 98, qty: 5, trader:"Seller_B" });
ob.placeBid({ id:"b1", price:102, qty: 8, trader:"Buyer_X"  });
// TRADE: Buyer_X buys 5 @ 98 from Seller_B
// TRADE: Buyer_X buys 3 @ 100 from Seller_A
```

### Tradeoffs

- **Price-time priority:** Sort by price first, then by arrival time as tiebreaker.
- **FIFO within price level:** Use a queue per price level for realistic exchange behaviour.
- **HFT systems** use lock-free data structures and pre-allocated SLAB memory.

---

## Q31. Merge K Sorted Feeds

### Problem Statement

Merge K sorted arrays/streams into a single sorted stream efficiently.

**Real scenario:** Atlassian merges K sorted commit logs. Netflix merges K sorted event logs from different regions.

- **Pattern:** K-Way Merge using Min-Heap | **Difficulty:** Hard | **LeetCode:** 23 | **Asked at:** Atlassian, Google

---

### TypeScript Solution

```typescript
interface StreamEntry { value: number; streamIndex: number; elementIndex: number; }

class KWayMerge {
  merge(streams: number[][]): number[] {
    const minHeap = new MinHeap<StreamEntry>((a, b) => a.value - b.value);
    const result: number[] = [];

    for (let i = 0; i < streams.length; i++) {
      if (streams[i].length > 0) {
        minHeap.push({ value: streams[i][0], streamIndex: i, elementIndex: 0 });
      }
    }

    while (minHeap.size() > 0) {
      const { value, streamIndex, elementIndex } = minHeap.pop()!;
      result.push(value);
      const nextIdx = elementIndex + 1;
      if (nextIdx < streams[streamIndex].length) {
        minHeap.push({ value: streams[streamIndex][nextIdx], streamIndex, elementIndex: nextIdx });
      }
    }
    return result;
  }
}

// Demo
const merger = new KWayMerge();
console.log(merger.merge([[1,4,7],[2,5,8],[3,6,9]]));
// [1,2,3,4,5,6,7,8,9]
```

### Complexity | Time: O(N log K) â€” N total elements, K streams | Space: O(K)

### Tradeoffs

- **K-way merge** O(N log K) vs O(N log N) for flatten + sort â€” critical when K << N.
- **External sort:** Same algorithm applies to merging K sorted files that don't fit in memory.

---

## Q32. Load Balancer â€” Least-Loaded Server

### Problem Statement

Route incoming requests to the least-loaded server.

**Real scenario:** Uber routes trip requests to the least-loaded dispatch server.

- **Pattern:** HashMap of loads | **Difficulty:** Medium | **Asked at:** Uber

---

### TypeScript Solution

```typescript
class LoadBalancer {
  private readonly servers: Map<string, number> = new Map();
  private readonly requestServer: Map<string, string> = new Map();

  addServer(id: string): void { this.servers.set(id, 0); }
  removeServer(id: string): void { this.servers.delete(id); }

  route(requestId: string): string | null {
    if (this.servers.size === 0) return null;
    let minLoad = Infinity, chosen = "";
    for (const [id, load] of this.servers) {
      if (load < minLoad) { minLoad = load; chosen = id; }
    }
    this.servers.set(chosen, minLoad + 1);
    this.requestServer.set(requestId, chosen);
    return chosen;
  }

  complete(requestId: string): void {
    const serverId = this.requestServer.get(requestId);
    if (!serverId) return;
    this.servers.set(serverId, (this.servers.get(serverId) ?? 1) - 1);
    this.requestServer.delete(requestId);
  }
}

// Demo
const lb2 = new LoadBalancer();
["s1","s2","s3"].forEach(s => lb2.addServer(s));
["r1","r2","r3","r4"].forEach(r => console.log(`${r} -> ${lb2.route(r)}`));
```

### Tradeoffs

- **Round-robin** O(1) but ignores actual load. **Least connections** adapts but needs O(n) scan.
- **Heap-based O(log n):** Maintain a min-heap of servers for large fleets.

---

## Q33. Minimum Meeting Rooms Required

### Problem Statement

Given meeting intervals `[start, end]`, return the minimum number of rooms needed.

- **Pattern:** Sort + Min-Heap of end times | **Difficulty:** Medium | **LeetCode:** 253 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class MeetingRoomScheduler {
  minRooms(intervals: [number, number][]): number {
    if (!intervals.length) return 0;
    intervals.sort((a, b) => a[0] - b[0]);
    const endTimes = new MinHeap<number>((a, b) => a - b);
    for (const [start, end] of intervals) {
      if (endTimes.size() > 0 && endTimes.peek()! <= start) endTimes.pop();
      endTimes.push(end);
    }
    return endTimes.size();
  }
}

// Demo
const mrs = new MeetingRoomScheduler();
console.log(mrs.minRooms([[0,30],[5,10],[15,20]])); // 2
console.log(mrs.minRooms([[7,10],[2,4]]));           // 1
```

### Trace

```
Sort: [[0,30],[5,10],[15,20]]
[0,30]:  push 30   -> heap=[30]    rooms=1
[5,10]:  30>5 nofree -> push 10 -> heap=[10,30] rooms=2
[15,20]: 10<=15 free -> pop 10, push 20 -> heap=[20,30] rooms=2
```

### Tradeoffs

- **Sweep-line:** Create `(time, +1/-1)` events, sort, scan â€” same complexity, simpler code.
- **Two-pointer** on sorted start/end arrays: O(n log n) sort + O(n) scan â€” no heap needed.

---

## Q34. Find K Nearest Drivers / Restaurants

### Problem Statement

Given your location and a list of points, return the **K closest** points by Euclidean distance.

**Real scenario:** Uber finds nearest available drivers. Swiggy shows nearest K restaurants.

- **Pattern:** Max-Heap of size K | **Difficulty:** Medium | **LeetCode:** 973 | **Asked at:** Uber, Swiggy

---

### TypeScript Solution

```typescript
interface GeoPoint { id: string; x: number; y: number; }

class NearestFinder {
  findKNearest(origin: GeoPoint, points: GeoPoint[], k: number): GeoPoint[] {
    const dist = (p: GeoPoint) => (p.x - origin.x) ** 2 + (p.y - origin.y) ** 2;
    const maxHeap = new MaxHeap<{ point: GeoPoint; dist: number }>((a, b) => a.dist - b.dist);
    for (const p of points) {
      maxHeap.push({ point: p, dist: dist(p) });
      if (maxHeap.size() > k) maxHeap.pop();
    }
    const result: GeoPoint[] = [];
    while (maxHeap.size()) result.push(maxHeap.pop()!.point);
    return result.reverse();
  }
}

// Demo
const nf = new NearestFinder();
const origin: GeoPoint = { id: "me", x: 0, y: 0 };
const drivers: GeoPoint[] = [
  { id:"d1", x:1, y:1 }, { id:"d2", x:3, y:4 },
  { id:"d3", x:1, y:0 }, { id:"d4", x:5, y:2 },
];
console.log(nf.findKNearest(origin, drivers, 2).map(p => p.id));
// ["d3","d1"]
```

### Tradeoffs

| Approach        | Time           | Space | Notes                      |
| --------------- | -------------- | ----- | -------------------------- |
| Sort all        | O(n log n)     | O(n)  | Simple but scans all       |
| Max-heap size K | O(n log k)     | O(k)  | Best for large n, small k  |
| QuickSelect     | O(n) avg       | O(1)  | Best average, O(n^2) worst |
| KD-Tree         | O(log n) query | O(n)  | Best for repeated queries  |

---

## Pattern 4 â€” Summary

| Q                  | Structure             | Key Insight                                 |
| ------------------ | --------------------- | ------------------------------------------- |
| Q23 Task Scheduler | Max-Heap              | Highest priority always at top              |
| Q24 Delay Queue    | Min-Heap by time      | Earliest job always at top                  |
| Q25 Retry Queue    | Min-Heap + backoff    | Exponential delay = BASE * 2^attempts       |
| Q26 Top-K          | HashMap + Min-Heap(K) | Min-heap of size K â€” pop when overflow |
| Q27 Leaderboard    | HashMap + sort        | Redis ZSET in production                    |
| Q28 Median         | Two Heaps             | Max-heap lower half, min-heap upper half    |
| Q29 Stock Price    | Two Heaps + HashMap   | Lazy deletion for updates                   |
| Q30 Order Book     | Max-Heap + Min-Heap   | Match when top-bid >= top-ask               |
| Q31 K-Way Merge    | Min-Heap of heads     | O(N log K) vs O(N log N) sort               |
| Q32 Load Balancer  | HashMap + O(n) scan   | Use heap for large server fleets            |
| Q33 Meeting Rooms  | Sort + Min-Heap ends  | Reuse room if earliest end <= new start     |
| Q34 K Nearest      | Max-Heap size K       | Pop farthest when size > K                  |

---

# PATTERN 5 â€” HashMap O(1) Designs

> **When you hear:** "O(1) insert/delete/lookup", "random element", "snapshot", "time-based"
> **Reach for:** HashMap + clever secondary structure (array for random access, DLL for order, etc.)

---

## Q35. Design Twitter / Social Feed

### Problem Statement

Design a simplified Twitter: `postTweet(userId, tweetId)`, `getNewsFeed(userId)` (10 most recent tweets from the user and their followees), `follow(followerId, followeeId)`, `unfollow(followerId, followeeId)`.

**Real scenario:** ShareChat feed aggregation. Koo's home timeline construction.

- **Pattern:** HashMap + Min-Heap for K-way merge of feeds
- **Difficulty:** Medium | **LeetCode:** 355 | **Asked at:** Sprinklr, ShareChat

---

### TypeScript Solution

```typescript
interface Tweet { tweetId: number; timestamp: number; }

class Twitter {
  private readonly userTweets: Map<number, Tweet[]> = new Map();
  private readonly following: Map<number, Set<number>> = new Map();
  private clock: number = 0;

  postTweet(userId: number, tweetId: number): void {
    if (!this.userTweets.has(userId)) this.userTweets.set(userId, []);
    this.userTweets.get(userId)!.push({ tweetId, timestamp: this.clock++ });
  }

  getNewsFeed(userId: number): number[] {
    const followees = new Set([userId, ...(this.following.get(userId) ?? [])]);
    const allTweets: Tweet[] = [];
    for (const uid of followees) {
      const tweets = this.userTweets.get(uid) ?? [];
      allTweets.push(...tweets.slice(-10)); // last 10 per user
    }
    return allTweets
      .sort((a, b) => b.timestamp - a.timestamp)
      .slice(0, 10)
      .map(t => t.tweetId);
  }

  follow(followerId: number, followeeId: number): void {
    if (!this.following.has(followerId)) this.following.set(followerId, new Set());
    this.following.get(followerId)!.add(followeeId);
  }

  unfollow(followerId: number, followeeId: number): void {
    this.following.get(followerId)?.delete(followeeId);
  }
}

// Demo
const twitter = new Twitter();
twitter.postTweet(1, 5);
twitter.postTweet(1, 3);
twitter.follow(2, 1);
twitter.postTweet(2, 101);
console.log(twitter.getNewsFeed(2)); // [101, 3, 5]
twitter.unfollow(2, 1);
console.log(twitter.getNewsFeed(2)); // [101]
```

### Tradeoffs

- **Fan-out on read** (merge at query time): simple, reads are heavy â€” used here.
- **Fan-out on write** (push to all followers at post time): writes are heavy, reads are O(1). Twitter uses a hybrid (fan-out on write for < 10k followers, fan-out on read for celebrities).

---

## Q36. Insert Delete GetRandom O(1)

### Problem Statement

Design a set with `insert(val)`, `remove(val)`, `getRandom()` â€” all in O(1) average time.

- **Pattern:** HashMap + Dynamic Array | **Difficulty:** Medium | **LeetCode:** 380 | **Asked at:** Amazon, Flipkart

---

### TypeScript Solution

```typescript
class RandomizedSet {
  private readonly valToIdx: Map<number, number> = new Map();
  private readonly nums: number[] = [];

  insert(val: number): boolean {
    if (this.valToIdx.has(val)) return false;
    this.nums.push(val);
    this.valToIdx.set(val, this.nums.length - 1);
    return true;
  }

  remove(val: number): boolean {
    if (!this.valToIdx.has(val)) return false;
    const idx = this.valToIdx.get(val)!;
    const last = this.nums[this.nums.length - 1];
    // Swap val with last element
    this.nums[idx] = last;
    this.valToIdx.set(last, idx);
    this.nums.pop();
    this.valToIdx.delete(val);
    return true;
  }

  getRandom(): number {
    return this.nums[Math.floor(Math.random() * this.nums.length)];
  }
}

// Demo
const rs = new RandomizedSet();
rs.insert(1); rs.insert(2); rs.insert(3);
rs.remove(2);
console.log(rs.getRandom()); // 1 or 3 with equal probability
```

### Key Insight

- **Array** gives O(1) random access; **HashMap** maps value to index for O(1) lookup/delete.
- **Swap with last** trick enables O(1) deletion from array without shifting elements.

### Tradeoffs

- **Simple array** gives O(n) delete. **Linked list** gives O(1) delete but O(n) random access. This hybrid gets both.

---

## Q37. Random Weighted Sample

### Problem Statement

Given weights `[w1, w2, ..., wn]`, implement `pickIndex()` that returns index `i` with probability `wi / sum(w)`.

**Real scenario:** Meesho AB testing assigns users to experiments with weighted probabilities. Ad platforms sample ads by bid weight.

- **Pattern:** Prefix sum + Binary search | **Difficulty:** Medium | **LeetCode:** 528 | **Asked at:** Meesho

---

### TypeScript Solution

```typescript
class WeightedRandom {
  private readonly prefixSums: number[];
  private readonly total: number;

  constructor(weights: number[]) {
    this.prefixSums = [];
    let sum = 0;
    for (const w of weights) {
      sum += w;
      this.prefixSums.push(sum);
    }
    this.total = sum;
  }

  pickIndex(): number {
    const target = Math.random() * this.total;
    // Binary search for first prefix sum > target
    let lo = 0, hi = this.prefixSums.length - 1;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.prefixSums[mid] <= target) lo = mid + 1;
      else hi = mid;
    }
    return lo;
  }
}

// Demo
const wr = new WeightedRandom([1, 3, 1]); // P(0)=20%, P(1)=60%, P(2)=20%
const counts = [0, 0, 0];
for (let i = 0; i < 10000; i++) counts[wr.pickIndex()]++;
console.log(counts); // approx [2000, 6000, 2000]
```

### Tradeoffs

- **Binary search on prefix sums:** O(log n) per pick. Space O(n).
- **Alias method:** O(1) per pick after O(n) setup. Preferred when billions of picks needed.

---

## Q38. HashMap from Scratch

### Problem Statement

Implement a HashMap with `put(key, value)`, `get(key)`, `remove(key)` â€” handle collisions with chaining.

- **Pattern:** Array of buckets + Linked list chaining | **Difficulty:** Medium | **LeetCode:** 706 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class HashNode<K, V> {
  constructor(
    public key: K,
    public value: V,
    public next: HashNode<K, V> | null = null
  ) {}
}

class HashMap<K, V> {
  private readonly buckets: (HashNode<K, V> | null)[];
  private readonly CAPACITY: number;
  private size: number = 0;

  constructor(capacity: number = 1024) {
    this.CAPACITY = capacity;
    this.buckets = new Array(capacity).fill(null);
  }

  private hash(key: K): number {
    const str = String(key);
    let h = 0;
    for (let i = 0; i < str.length; i++) {
      h = (h * 31 + str.charCodeAt(i)) % this.CAPACITY;
    }
    return h;
  }

  put(key: K, value: V): void {
    const idx = this.hash(key);
    let node = this.buckets[idx];
    while (node) {
      if (node.key === key) { node.value = value; return; }
      node = node.next;
    }
    const newNode = new HashNode(key, value, this.buckets[idx]);
    this.buckets[idx] = newNode;
    this.size++;
  }

  get(key: K): V | undefined {
    let node = this.buckets[this.hash(key)];
    while (node) {
      if (node.key === key) return node.value;
      node = node.next;
    }
    return undefined;
  }

  remove(key: K): void {
    const idx = this.hash(key);
    let node = this.buckets[idx], prev: HashNode<K, V> | null = null;
    while (node) {
      if (node.key === key) {
        if (prev) prev.next = node.next;
        else this.buckets[idx] = node.next;
        this.size--;
        return;
      }
      prev = node; node = node.next;
    }
  }

  getSize(): number { return this.size; }
}

// Demo
const hm = new HashMap<string, number>();
hm.put("apple", 5); hm.put("banana", 3); hm.put("apple", 10);
console.log(hm.get("apple"));  // 10 (updated)
console.log(hm.get("cherry")); // undefined
hm.remove("banana");
console.log(hm.getSize()); // 1
```

### Tradeoffs

| Collision Strategy             | Pros                             | Cons                         |
| ------------------------------ | -------------------------------- | ---------------------------- |
| **Chaining (used)**      | Simple, handles high load factor | Extra memory per node        |
| Open addressing (linear probe) | Cache-friendly                   | Clustering, needs tombstones |
| Robin Hood hashing             | Low variance in probe length     | Complex implementation       |

---

## Q39. Transit Time / Path Sum Tracker

### Problem Statement

Given a list of direct travel times between stations, find the total travel time between any two stations (direct or via intermediate stops).

**Real scenario:** IRCTC/OLA maps calculate route time by summing segments. Delivery systems compute ETA through waypoints.

- **Pattern:** HashMap for adjacency + DFS/BFS for path | **Difficulty:** Medium

---

### TypeScript Solution

```typescript
class TransitGraph {
  private readonly graph: Map<string, Map<string, number>> = new Map();

  addRoute(from: string, to: string, time: number): void {
    if (!this.graph.has(from)) this.graph.set(from, new Map());
    if (!this.graph.has(to))   this.graph.set(to, new Map());
    this.graph.get(from)!.set(to, time);
    this.graph.get(to)!.set(from, time); // bidirectional
  }

  getTime(source: string, destination: string): number {
    if (source === destination) return 0;
    const dist = new Map<string, number>();
    const pq = new MinHeap<[number, string]>((a, b) => a[0] - b[0]);
    dist.set(source, 0);
    pq.push([0, source]);

    while (pq.size() > 0) {
      const [d, node] = pq.pop()!;
      if (node === destination) return d;
      if (d > (dist.get(node) ?? Infinity)) continue;
      for (const [neighbor, weight] of (this.graph.get(node) ?? new Map())) {
        const newDist = d + weight;
        if (newDist < (dist.get(neighbor) ?? Infinity)) {
          dist.set(neighbor, newDist);
          pq.push([newDist, neighbor]);
        }
      }
    }
    return -1; // unreachable
  }
}

// Demo
const tg = new TransitGraph();
tg.addRoute("Mumbai", "Pune", 180);
tg.addRoute("Pune", "Bangalore", 600);
tg.addRoute("Mumbai", "Bangalore", 900);
console.log(tg.getTime("Mumbai", "Bangalore")); // 780 (via Pune)
```

### Tradeoffs

- **Dijkstra (used)** for weighted graphs. O((E + V) log V).
- **BFS** works for unweighted graphs (equal costs) â€” simpler.

---

## Q40. Snapshot Array

### Problem Statement

Implement `SnapshotArray(n)`: `set(index, val)`, `snap()` returns snap_id, `get(index, snap_id)` returns value at that index when snap was taken.

- **Pattern:** HashMap per index storing (snap_id, value) pairs + binary search | **Difficulty:** Medium | **LeetCode:** 1146 | **Asked at:** Google

---

### TypeScript Solution

```typescript
class SnapshotArray {
  private readonly data: [number, number][][]; // data[i] = [(snapId, val), ...]
  private snapId: number = 0;

  constructor(n: number) {
    this.data = Array.from({ length: n }, () => [[0, 0]]);
  }

  set(index: number, val: number): void {
    const snapshots = this.data[index];
    if (snapshots[snapshots.length - 1][0] === this.snapId) {
      snapshots[snapshots.length - 1][1] = val; // update current snap
    } else {
      snapshots.push([this.snapId, val]);
    }
  }

  snap(): number { return this.snapId++; }

  get(index: number, snapId: number): number {
    const snapshots = this.data[index];
    // Binary search for largest snapId <= requested snapId
    let lo = 0, hi = snapshots.length - 1;
    while (lo < hi) {
      const mid = (lo + hi + 1) >> 1;
      if (snapshots[mid][0] <= snapId) lo = mid;
      else hi = mid - 1;
    }
    return snapshots[lo][1];
  }
}

// Demo
const sa = new SnapshotArray(3);
sa.set(0, 5);
const s0 = sa.snap(); // snap_id=0
sa.set(0, 6);
console.log(sa.get(0, s0)); // 5 (value at snap 0)
console.log(sa.get(0, sa.snap())); // 6
```

### Tradeoffs

- **Binary search on snapshots per index** gives O(log S) get where S = number of snaps.
- **Copy-on-write:** Copy entire array on snap â€” O(n) per snap, O(1) get. Better when sets are rare.
- **Persistent data structures:** Fully persistent arrays use path copying â€” O(log n) all ops.

---

## Q41. Time-Based Key-Value Store

### Problem Statement

`set(key, value, timestamp)` and `get(key, timestamp)` â€” get returns the value set at the largest timestamp <= given timestamp.

- **Pattern:** HashMap of sorted lists + binary search | **Difficulty:** Medium | **LeetCode:** 981 | **Asked at:** Amazon, Uber

---

### TypeScript Solution

```typescript
class TimeMap {
  private readonly store: Map<string, { timestamp: number; value: string }[]> = new Map();

  set(key: string, value: string, timestamp: number): void {
    if (!this.store.has(key)) this.store.set(key, []);
    this.store.get(key)!.push({ timestamp, value });
    // Assumes timestamps are inserted in increasing order
  }

  get(key: string, timestamp: number): string {
    const entries = this.store.get(key);
    if (!entries || entries.length === 0) return "";
    let lo = 0, hi = entries.length - 1;
    // Find last entry with timestamp <= given timestamp
    while (lo < hi) {
      const mid = (lo + hi + 1) >> 1;
      if (entries[mid].timestamp <= timestamp) lo = mid;
      else hi = mid - 1;
    }
    return entries[lo].timestamp <= timestamp ? entries[lo].value : "";
  }
}

// Demo
const tm = new TimeMap();
tm.set("foo", "bar", 1);
tm.set("foo", "bar2", 4);
console.log(tm.get("foo", 4)); // "bar2"
console.log(tm.get("foo", 3)); // "bar"
console.log(tm.get("foo", 0)); // ""
```

### Tradeoffs

- **Binary search on sorted timestamps** gives O(log T) get where T = number of versions.
- **Append-only** design makes this suitable for immutable audit logs (ledgers).

---

## Q42. Phone Directory

### Problem Statement

Design a phone directory with a pool of N numbers. `get()` returns an available number, `check(number)` returns availability, `release(number)` returns a number to the pool.

- **Pattern:** HashSet + Queue | **Difficulty:** Medium | **LeetCode:** 379 | **Asked at:** Microsoft

---

### TypeScript Solution

```typescript
class PhoneDirectory {
  private readonly available: Set<number>;
  private readonly pool: number[];

  constructor(maxNumbers: number) {
    this.available = new Set();
    this.pool = [];
    for (let i = 0; i < maxNumbers; i++) {
      this.available.add(i);
      this.pool.push(i);
    }
  }

  get(): number {
    if (this.pool.length === 0) return -1;
    const num = this.pool.shift()!;
    this.available.delete(num);
    return num;
  }

  check(number: number): boolean {
    return this.available.has(number);
  }

  release(number: number): void {
    if (!this.available.has(number)) {
      this.available.add(number);
      this.pool.push(number);
    }
  }
}

// Demo
const pd = new PhoneDirectory(3);
console.log(pd.get());      // 0
console.log(pd.get());      // 1
console.log(pd.check(2));   // true
pd.release(1);
console.log(pd.check(1));   // true
console.log(pd.get());      // 2
```

### Tradeoffs

- **Queue** for FIFO allocation. **Set** for O(1) availability check.
- Alternatively: `LinkedList<Integer> available` â€” O(1) get/release without O(n) queue shift. Use JavaScript `Deque` to avoid O(n) shift.

---

## Q43. Tic-Tac-Toe

### Problem Statement

Design a Tic-Tac-Toe board: `move(row, col, player)` returns `0` (no winner), `1` (player 1), or `2` (player 2).

- **Pattern:** Row/Col/Diagonal counters | **Difficulty:** Medium | **LeetCode:** 348 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class TicTacToe {
  private readonly rows: number[][];
  private readonly cols: number[][];
  private readonly diag: number[];
  private readonly antiDiag: number[];
  private readonly n: number;

  constructor(n: number) {
    this.n = n;
    this.rows     = Array.from({ length: n }, () => [0, 0]);
    this.cols     = Array.from({ length: n }, () => [0, 0]);
    this.diag     = [0, 0];
    this.antiDiag = [0, 0];
  }

  move(row: number, col: number, player: number): number {
    const p = player - 1; // 0-indexed
    this.rows[row][p]++;
    this.cols[col][p]++;
    if (row === col) this.diag[p]++;
    if (row + col === this.n - 1) this.antiDiag[p]++;

    if (this.rows[row][p] === this.n || this.cols[col][p] === this.n ||
        this.diag[p] === this.n || this.antiDiag[p] === this.n) {
      return player;
    }
    return 0;
  }
}

// Demo
const ttt = new TicTacToe(3);
console.log(ttt.move(0,0,1)); // 0
console.log(ttt.move(0,2,2)); // 0
console.log(ttt.move(1,1,1)); // 0
console.log(ttt.move(2,0,2)); // 0
console.log(ttt.move(2,2,1)); // 1 â€” player 1 wins diagonal
```

### Complexity | move: O(1) | Space: O(n)

### Tradeoffs

- **O(n) naive** checks each row/col/diag on every move.
- **Counting trick (used)** is O(1) per move by maintaining running sums.
- Extend to N-player or generalise N-in-a-row by changing the win condition.

---

## Q44. Snowflake ID Generator

### Problem Statement

Design a distributed unique ID generator. IDs should be: unique across machines, sortable by time, and generated without coordination.

**Real scenario:** Flipkart generates order IDs. Paytm generates transaction IDs. Twitter's Snowflake algorithm.

- **Pattern:** Bit-packing: timestamp + machine_id + sequence | **Difficulty:** Hard | **Asked at:** Flipkart, Paytm

---

### TypeScript Solution

```typescript
class SnowflakeIdGenerator {
  private readonly machineId: number;
  private readonly EPOCH = 1609459200000; // 2021-01-01
  private readonly MACHINE_BITS = 10;
  private readonly SEQUENCE_BITS = 12;
  private readonly MAX_SEQUENCE = (1 << this.SEQUENCE_BITS) - 1; // 4095

  private sequence: number = 0;
  private lastTimestamp: number = -1;

  constructor(machineId: number) {
    if (machineId < 0 || machineId >= (1 << this.MACHINE_BITS)) {
      throw new Error(`Machine ID must be between 0 and ${(1 << this.MACHINE_BITS) - 1}`);
    }
    this.machineId = machineId;
  }

  generateId(): string {
    let ts = Date.now() - this.EPOCH;
    if (ts === this.lastTimestamp) {
      this.sequence = (this.sequence + 1) & this.MAX_SEQUENCE;
      if (this.sequence === 0) {
        // Wait for next millisecond
        while (Date.now() - this.EPOCH <= this.lastTimestamp) {}
        ts = Date.now() - this.EPOCH;
      }
    } else {
      this.sequence = 0;
    }
    this.lastTimestamp = ts;
    // 64-bit ID: [41 bits ts][10 bits machine][12 bits sequence]
    // JS can't handle 64-bit int natively, so we return as string
    const id = BigInt(ts) << 22n | BigInt(this.machineId) << 12n | BigInt(this.sequence);
    return id.toString();
  }
}

// Demo
const gen = new SnowflakeIdGenerator(1);
console.log(gen.generateId()); // e.g. "6884732846849024"
console.log(gen.generateId()); // always larger â€” time-sortable
```

### Tradeoffs

- **Pros:** No coordination, time-sorted, embeds machine + sequence info.
- **Cons:** Clock skew causes issues â€” if system clock goes back, IDs may collide. Use `lastTimestamp` guard.
- **UUID v4** is simpler but random â€” not sortable. **UUID v7** adds timestamp prefix.

---

## Q45. URL Shortener

### Problem Statement

Design a URL shortener: `shorten(longUrl)` returns a 6-char code, `expand(code)` returns the original URL. Codes must be unique.

**Real scenario:** bit.ly, t.co, Paytm share links.

- **Pattern:** HashMap + Base62 encoding | **Difficulty:** Medium | **Asked at:** Paytm, Snapdeal

---

### TypeScript Solution

```typescript
class URLShortener {
  private readonly codeToUrl: Map<string, string> = new Map();
  private readonly urlToCode: Map<string, string> = new Map();
  private counter: number = 0;
  private readonly BASE62 = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz";
  private readonly BASE_URL = "https://sho.rt/";

  private encode(n: number): string {
    let result = "";
    while (n > 0) {
      result = this.BASE62[n % 62] + result;
      n = Math.floor(n / 62);
    }
    return result.padStart(6, "0");
  }

  shorten(longUrl: string): string {
    if (this.urlToCode.has(longUrl)) return this.BASE_URL + this.urlToCode.get(longUrl);
    const code = this.encode(++this.counter);
    this.codeToUrl.set(code, longUrl);
    this.urlToCode.set(longUrl, code);
    return this.BASE_URL + code;
  }

  expand(shortUrl: string): string | null {
    const code = shortUrl.replace(this.BASE_URL, "");
    return this.codeToUrl.get(code) ?? null;
  }
}

// Demo
const us = new URLShortener();
const short = us.shorten("https://www.flipkart.com/very/long/product/url?ref=xyz");
console.log(short);          // "https://sho.rt/000001"
console.log(us.expand(short)); // original URL
```

### Tradeoffs

- **Counter + Base62:** Predictable, sequential. Bad for security â€” easy to enumerate.
- **Random hash:** `MD5/SHA â†’ first 6 chars`. May collide â€” needs retry loop with collision check.
- **Distributed:** Use a counter per machine shard + machine prefix to avoid coordination.

---

## Q46. Dedup Service â€” Detect Duplicate Requests

### Problem Statement

Design a deduplication service: within a time window, if the same `(userId, requestId)` is received twice, the second request is a duplicate.

**Real scenario:** Paytm deduplicates payment requests to prevent double charges. Razorpay's idempotency layer uses request IDs.

- **Pattern:** HashMap + TTL (expiring entries) | **Difficulty:** Medium | **Asked at:** Paytm, Razorpay

---

### TypeScript Solution

```typescript
class DedupService {
  private readonly seen: Map<string, number> = new Map(); // key -> expiry time
  private readonly windowMs: number;

  constructor(windowMs: number) { this.windowMs = windowMs; }

  private makeKey(userId: string, requestId: string): string {
    return `${userId}:${requestId}`;
  }

  isDuplicate(userId: string, requestId: string, now: number): boolean {
    this.cleanup(now);
    const key = this.makeKey(userId, requestId);
    if (this.seen.has(key)) return true;
    this.seen.set(key, now + this.windowMs);
    return false;
  }

  private cleanup(now: number): void {
    for (const [key, expiry] of this.seen) {
      if (expiry <= now) this.seen.delete(key);
    }
  }
}

// Demo
const dedup = new DedupService(5000); // 5 second window
console.log(dedup.isDuplicate("user1", "req_abc", 1000)); // false (first time)
console.log(dedup.isDuplicate("user1", "req_abc", 2000)); // true  (duplicate)
console.log(dedup.isDuplicate("user1", "req_abc", 7000)); // false (window expired)
```

### Tradeoffs

- **In-memory HashMap** is fast but doesn't survive restarts. Use **Redis** with `SET key EX ttl NX` for distributed dedup.
- **Bloom Filter** for scale: probabilistic dedup with O(1) memory per entry, no false negatives (might have false positives).

---

## Pattern 5 â€” Summary

| Q                   | Structure                     | Key Insight                               |
| ------------------- | ----------------------------- | ----------------------------------------- |
| Q35 Twitter Feed    | HashMap + Heap                | Fan-out on read vs write tradeoff         |
| Q36 RandomizedSet   | HashMap + Array               | Swap-with-last for O(1) delete from array |
| Q37 Weighted Random | Prefix sum + BSearch          | O(log n) pick; Alias method for O(1)      |
| Q38 HashMap Scratch | Array + Chaining              | hash(key) % capacity for bucket index     |
| Q39 Transit Time    | Graph + Dijkstra              | Weighted shortest path                    |
| Q40 Snapshot Array  | Sorted (snapId,val) + BSearch | Binary search for floor snapshot          |
| Q41 Time-Based KV   | Sorted timestamps + BSearch   | Floor timestamp binary search             |
| Q42 Phone Directory | Set + Queue                   | O(1) check + O(1) get                     |
| Q43 Tic-Tac-Toe     | Row/Col counters              | O(1) per move with counter arrays         |
| Q44 Snowflake ID    | Bit-packing                   | ts[41] + machine[10] + seq[12]            |
| Q45 URL Shortener   | HashMap + Base62              | Counter -> Base62 code                    |
| Q46 Dedup Service   | HashMap + TTL                 | Expire entries outside window             |

---

# PATTERN 6 â€” Stack / Queue

> **When you hear:** "undo", "browser history", "balanced brackets", "min/max in O(1)", "next greater element"
> **Reach for:** Stack (LIFO) for reversal/history, monotonic stack for next-greater, two-stacks for queue.

---

## Q47. Browser History â€” Back / Forward

### Problem Statement

Design a browser history: `visit(url)`, `back(steps)` goes back up to `steps` pages, `forward(steps)` goes forward. Return current URL.

- **Pattern:** Two stacks (back-stack + forward-stack) | **Difficulty:** Medium | **LeetCode:** 1472 | **Asked at:** Juspay, Amazon

---

### TypeScript Solution

```typescript
class BrowserHistory {
  private readonly backStack: string[] = [];
  private readonly forwardStack: string[] = [];
  private current: string;

  constructor(homepage: string) { this.current = homepage; }

  visit(url: string): void {
    this.backStack.push(this.current);
    this.current = url;
    this.forwardStack.length = 0; // clear forward history
  }

  back(steps: number): string {
    while (steps-- > 0 && this.backStack.length > 0) {
      this.forwardStack.push(this.current);
      this.current = this.backStack.pop()!;
    }
    return this.current;
  }

  forward(steps: number): string {
    while (steps-- > 0 && this.forwardStack.length > 0) {
      this.backStack.push(this.current);
      this.current = this.forwardStack.pop()!;
    }
    return this.current;
  }
}

// Demo
const bh = new BrowserHistory("leetcode.com");
bh.visit("google.com"); bh.visit("facebook.com"); bh.visit("youtube.com");
console.log(bh.back(1));    // "facebook.com"
console.log(bh.back(1));    // "google.com"
console.log(bh.forward(1)); // "facebook.com"
bh.visit("linkedin.com");
console.log(bh.forward(2)); // "linkedin.com" (forward cleared by visit)
```

### Tradeoffs

- **Two stacks** = O(1) per operation. Clear forward stack on visit ensures history is linear.
- **Doubly-linked list** alternative: constant time navigation with a pointer; forward/back are pointer moves.

---

## Q48. Undo/Redo System

### Problem Statement

Design a text editor's undo/redo with `type(text)`, `delete(count)`, `undo()`, `redo()`. Return current document state.

- **Pattern:** Two stacks of commands | **Difficulty:** Medium | **Asked at:** Atlassian, Notion

---

### TypeScript Solution

```typescript
interface ICommand { execute(doc: string): string; undo(doc: string): string; }

class TypeCommand implements ICommand {
  constructor(private readonly text: string) {}
  execute(doc: string): string { return doc + this.text; }
  undo(doc: string): string { return doc.slice(0, -this.text.length); }
}

class DeleteCommand implements ICommand {
  private deleted: string = "";
  constructor(private readonly count: number) {}
  execute(doc: string): string {
    this.deleted = doc.slice(-this.count);
    return doc.slice(0, -this.count);
  }
  undo(doc: string): string { return doc + this.deleted; }
}

class TextEditor {
  private document: string = "";
  private readonly undoStack: { cmd: ICommand; before: string }[] = [];
  private readonly redoStack: { cmd: ICommand; before: string }[] = [];

  private apply(cmd: ICommand): void {
    const before = this.document;
    this.document = cmd.execute(this.document);
    this.undoStack.push({ cmd, before });
    this.redoStack.length = 0;
  }

  type(text: string): void { this.apply(new TypeCommand(text)); }
  delete(count: number): void { this.apply(new DeleteCommand(count)); }

  undo(): string {
    if (!this.undoStack.length) return this.document;
    const entry = this.undoStack.pop()!;
    this.redoStack.push(entry);
    this.document = entry.before;
    return this.document;
  }

  redo(): string {
    if (!this.redoStack.length) return this.document;
    const entry = this.redoStack.pop()!;
    this.document = entry.cmd.execute(entry.before);
    this.undoStack.push(entry);
    return this.document;
  }

  getDocument(): string { return this.document; }
}

// Demo
const editor = new TextEditor();
editor.type("Hello "); editor.type("World");
console.log(editor.getDocument()); // "Hello World"
editor.undo();
console.log(editor.getDocument()); // "Hello "
editor.redo();
console.log(editor.getDocument()); // "Hello World"
editor.delete(5);
console.log(editor.getDocument()); // "Hello "
```

### Tradeoffs

- **Command Pattern** allows extending to complex ops (bold, insert-at-cursor) without changing the editor core.
- **Snapshot approach** (store full doc state) is simpler but O(n) memory per command.
- **Operational Transformation (OT)** â€” used by Google Docs for collaborative editing where commands may arrive out of order.

---

## Q49. Min Stack â€” O(1) getMin

### Problem Statement

Design a stack with `push(val)`, `pop()`, `top()`, and `getMin()` â€” all O(1).

- **Pattern:** Auxiliary min-stack | **Difficulty:** Easy | **LeetCode:** 155 | **Asked at:** Amazon, Flipkart

---

### TypeScript Solution

```typescript
class MinStack {
  private readonly stack: number[] = [];
  private readonly minStack: number[] = [];

  push(val: number): void {
    this.stack.push(val);
    const currentMin = this.minStack.length
      ? Math.min(val, this.minStack[this.minStack.length - 1])
      : val;
    this.minStack.push(currentMin);
  }

  pop(): void {
    this.stack.pop();
    this.minStack.pop();
  }

  top(): number { return this.stack[this.stack.length - 1]; }

  getMin(): number { return this.minStack[this.minStack.length - 1]; }
}

// Demo
const ms = new MinStack();
ms.push(3); ms.push(1); ms.push(2);
console.log(ms.getMin()); // 1
ms.pop();
console.log(ms.getMin()); // 1
ms.pop();
console.log(ms.getMin()); // 3
```

### Key Insight: The minStack mirrors the main stack but stores the **running minimum** at each depth level.

---

## Q50. Max Stack

### Problem Statement

Design a stack with `push`, `pop`, `top`, `peekMax`, `popMax` â€” all O(log n). `popMax` removes the maximum element.

- **Pattern:** Stack + TreeMap | **Difficulty:** Hard | **LeetCode:** 716 | **Asked at:** Google

---

### TypeScript Solution

```typescript
class MaxStack {
  private readonly stack: number[] = [];
  // Map from value to list of stack indices where it appears
  private readonly maxMap: Map<number, number[]> = new Map();

  push(val: number): void {
    const idx = this.stack.length;
    this.stack.push(val);
    if (!this.maxMap.has(val)) this.maxMap.set(val, []);
    this.maxMap.get(val)!.push(idx);
  }

  pop(): number {
    const val = this.stack.pop()!;
    const indices = this.maxMap.get(val)!;
    indices.pop();
    if (indices.length === 0) this.maxMap.delete(val);
    return val;
  }

  top(): number { return this.stack[this.stack.length - 1]; }

  peekMax(): number { return Math.max(...this.maxMap.keys()); }

  popMax(): number {
    const max = this.peekMax();
    const indices = this.maxMap.get(max)!;
    const targetIdx = indices[indices.length - 1];

    // Remove elements above targetIdx, then re-push them
    const buffer: number[] = [];
    while (this.stack.length - 1 > targetIdx) buffer.push(this.pop());
    this.pop(); // remove max
    while (buffer.length) this.push(buffer.pop()!);
    return max;
  }
}

// Demo
const maxSt = new MaxStack();
maxSt.push(5); maxSt.push(1); maxSt.push(5);
console.log(maxSt.top());     // 5
console.log(maxSt.peekMax()); // 5
console.log(maxSt.popMax());  // 5 (removes topmost 5)
console.log(maxSt.top());     // 1
```

### Tradeoffs

- **True O(log n) popMax:** Use a sorted multiset (augmented BST) + DLL. Avoid pop/re-push approach.
- **Interview simplification:** The above pop/re-push approach is O(n) popMax but easier to explain.

---

## Q51. Basic Calculator â€” Evaluate `+`, `-`, `(`, `)`

### Problem Statement

Evaluate a string expression containing `+`, `-`, `(`, `)`, and spaces.

- **Pattern:** Stack for parentheses handling | **Difficulty:** Hard | **LeetCode:** 224 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class BasicCalculator {
  calculate(s: string): number {
    const stack: number[] = [];
    let result = 0, num = 0, sign = 1;

    for (const ch of s) {
      if (ch >= "0" && ch <= "9") {
        num = num * 10 + parseInt(ch);
      } else if (ch === "+") {
        result += sign * num; num = 0; sign = 1;
      } else if (ch === "-") {
        result += sign * num; num = 0; sign = -1;
      } else if (ch === "(") {
        // Push current result and sign onto stack
        stack.push(result, sign);
        result = 0; sign = 1;
      } else if (ch === ")") {
        result += sign * num; num = 0;
        result *= stack.pop()!;    // multiply by sign before paren
        result += stack.pop()!;    // add to result before paren
      }
    }
    return result + sign * num;
  }
}

// Demo
const calc = new BasicCalculator();
console.log(calc.calculate("1 + 1"));       // 2
console.log(calc.calculate("(1+(4+5+2)-3)+(6+8)")); // 23
console.log(calc.calculate("1-(     -2)")); // 3
```

### Tradeoffs

- **This approach** handles `+ - ()` in O(n). Extension to `* /` requires operator precedence â€” use two stacks (operands + operators) or Shunting Yard algorithm.

---

## Q52. JSON Parser (Bracket Matching)

### Problem Statement

Validate and parse a simplified JSON-like string. Check that all brackets `{`, `[`, `(` are balanced and properly nested.

- **Pattern:** Stack | **Difficulty:** Easy-Medium | **Asked at:** Freshworks, Postman

---

### TypeScript Solution

```typescript
class BracketValidator {
  private readonly PAIRS: Map<string, string> = new Map([
    [")", "("], ["]", "["], ["}", "{"]
  ]);

  isBalanced(s: string): boolean {
    const stack: string[] = [];
    for (const ch of s) {
      if ("([{".includes(ch)) {
        stack.push(ch);
      } else if (")]}".includes(ch)) {
        if (!stack.length || stack[stack.length - 1] !== this.PAIRS.get(ch)) {
          return false;
        }
        stack.pop();
      }
    }
    return stack.length === 0;
  }

  findUnmatched(s: string): number[] {
    const stack: number[] = []; // store indices
    const unmatched: number[] = [];
    const open = new Set("([{");
    const close = new Set(")]}");
    const match: Record<string, string> = { ")": "(", "]": "[", "}": "{" };

    for (let i = 0; i < s.length; i++) {
      if (open.has(s[i])) {
        stack.push(i);
      } else if (close.has(s[i])) {
        if (!stack.length || s[stack[stack.length-1]] !== match[s[i]]) {
          unmatched.push(i);
        } else {
          stack.pop();
        }
      }
    }
    return [...stack, ...unmatched].sort((a, b) => a - b);
  }
}

// Demo
const bv = new BracketValidator();
console.log(bv.isBalanced("({[]})")); // true
console.log(bv.isBalanced("([)]"));   // false
console.log(bv.findUnmatched("(]}")); // [0,1,2]
```

---

## Q53. Circular Queue

### Problem Statement

Design a circular queue with fixed capacity: `enqueue`, `dequeue`, `front`, `rear`, `isEmpty`, `isFull` â€” all O(1).

- **Pattern:** Ring buffer | **Difficulty:** Medium | **LeetCode:** 622 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class CircularQueue<T> {
  private readonly buffer: (T | undefined)[];
  private head: number = 0;
  private tail: number = 0;
  private count: number = 0;
  private readonly capacity: number;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.buffer = new Array(capacity);
  }

  enqueue(val: T): boolean {
    if (this.isFull()) return false;
    this.buffer[this.tail] = val;
    this.tail = (this.tail + 1) % this.capacity;
    this.count++;
    return true;
  }

  dequeue(): T | undefined {
    if (this.isEmpty()) return undefined;
    const val = this.buffer[this.head];
    this.buffer[this.head] = undefined;
    this.head = (this.head + 1) % this.capacity;
    this.count--;
    return val;
  }

  front(): T | undefined { return this.isEmpty() ? undefined : this.buffer[this.head]; }
  rear():  T | undefined { return this.isEmpty() ? undefined : this.buffer[(this.tail - 1 + this.capacity) % this.capacity]; }
  isEmpty(): boolean { return this.count === 0; }
  isFull():  boolean { return this.count === this.capacity; }
}

// Demo
const cq = new CircularQueue<number>(3);
cq.enqueue(1); cq.enqueue(2); cq.enqueue(3);
console.log(cq.isFull());  // true
console.log(cq.dequeue()); // 1
cq.enqueue(4);
console.log(cq.rear());    // 4
```

### Tradeoffs

- **Ring buffer** is cache-friendly and allocation-free after init. Used in Linux kernel I/O queues.
- **count variable** avoids the ambiguity between full and empty when head == tail.

---

## Q54. Queue Using Two Stacks

### Problem Statement

Implement a queue using only two stacks.

- **Pattern:** Stack pair (inbox/outbox) | **Difficulty:** Easy | **LeetCode:** 232 | **Asked at:** Amazon, Microsoft

---

### TypeScript Solution

```typescript
class MyQueue<T> {
  private readonly inbox:  T[] = [];
  private readonly outbox: T[] = [];

  push(val: T): void { this.inbox.push(val); }

  pop(): T | undefined {
    this.transfer();
    return this.outbox.pop();
  }

  peek(): T | undefined {
    this.transfer();
    return this.outbox[this.outbox.length - 1];
  }

  empty(): boolean { return this.inbox.length === 0 && this.outbox.length === 0; }

  private transfer(): void {
    if (this.outbox.length === 0) {
      while (this.inbox.length > 0) {
        this.outbox.push(this.inbox.pop()!);
      }
    }
  }
}

// Demo
const q = new MyQueue<number>();
q.push(1); q.push(2); q.push(3);
console.log(q.pop());  // 1
console.log(q.peek()); // 2
```

### Key Insight: Transfer from inbox to outbox is **amortised O(1)** â€” each element is moved at most once.

---

## Q55. Balanced Brackets Minimum Insertions

### Problem Statement

Given a string with `(` and `)`, return the minimum number of parentheses to insert to make it balanced.

- **Pattern:** Stack / counter | **Difficulty:** Medium | **LeetCode:** 1541 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class ParenthesesFixer {
  minAddToMakeValid(s: string): number {
    let open = 0, close = 0;
    for (const ch of s) {
      if (ch === "(") {
        open++;
      } else {
        if (open > 0) open--; // matched
        else close++;          // unmatched ')'
      }
    }
    return open + close;
  }
}

// Demo
const pf = new ParenthesesFixer();
console.log(pf.minAddToMakeValid("())")); // 1
console.log(pf.minAddToMakeValid("(((")); // 3
console.log(pf.minAddToMakeValid("()")); // 0
```

---

## Q56. Music Playlist â€” Repeat Prevention

### Problem Statement

Design a music player with `addSong(id)`, `next()` (plays next song, avoids repeating last K songs), `prev()`.

- **Pattern:** Circular buffer + HashSet for recently played | **Difficulty:** Medium | **Asked at:** Spotify/Gaana

---

### TypeScript Solution

```typescript
class MusicPlaylist {
  private readonly songs: number[];
  private readonly recentSet: Set<number>;
  private readonly recentQueue: number[];
  private readonly k: number;
  private currentIdx: number = 0;

  constructor(songs: number[], k: number) {
    this.songs = [...songs];
    this.k = k;
    this.recentSet = new Set();
    this.recentQueue = [];
  }

  private recordPlay(songId: number): void {
    if (this.recentSet.has(songId)) return;
    this.recentSet.add(songId);
    this.recentQueue.push(songId);
    if (this.recentQueue.length > this.k) {
      const oldest = this.recentQueue.shift()!;
      this.recentSet.delete(oldest);
    }
  }

  current(): number { return this.songs[this.currentIdx]; }

  next(): number {
    let nextIdx = (this.currentIdx + 1) % this.songs.length;
    // Skip recently played
    let tries = 0;
    while (this.recentSet.has(this.songs[nextIdx]) && tries < this.songs.length) {
      nextIdx = (nextIdx + 1) % this.songs.length;
      tries++;
    }
    this.recordPlay(this.songs[this.currentIdx]);
    this.currentIdx = nextIdx;
    return this.songs[this.currentIdx];
  }

  prev(): number {
    this.currentIdx = (this.currentIdx - 1 + this.songs.length) % this.songs.length;
    return this.songs[this.currentIdx];
  }
}

// Demo
const playlist = new MusicPlaylist([1,2,3,4,5], 3);
console.log(playlist.next()); // 2
console.log(playlist.next()); // 3
console.log(playlist.next()); // 4
console.log(playlist.next()); // 5 (skips 2,3,4 as recently played)
```

---

## Q57. Snake Game

### Problem Statement

Simulate a snake game: `move(direction)` returns the score (or -1 if game over â€” wall/self collision). The snake grows when it eats food.

- **Pattern:** Deque for snake body + Set for occupied cells | **Difficulty:** Medium | **LeetCode:** 353 | **Asked at:** Google

---

### TypeScript Solution

```typescript
type Direction = "U" | "D" | "L" | "R";

class SnakeGame {
  private readonly width: number;
  private readonly height: number;
  private readonly food: [number, number][];
  private foodIdx: number = 0;
  private readonly snake: [number, number][] = [[0, 0]];
  private readonly occupied: Set<string> = new Set(["0,0"]);
  private score: number = 0;

  constructor(width: number, height: number, food: [number, number][]) {
    this.width = width; this.height = height; this.food = food;
  }

  move(dir: Direction): number {
    const head = this.snake[0];
    const delta: Record<Direction, [number, number]> = {
      U:[-1,0], D:[1,0], L:[0,-1], R:[0,1]
    };
    const newHead: [number, number] = [head[0]+delta[dir][0], head[1]+delta[dir][1]];

    // Wall collision
    if (newHead[0] < 0 || newHead[0] >= this.height ||
        newHead[1] < 0 || newHead[1] >= this.width) return -1;

    // Check if eating food before checking self-collision (tail will move)
    const eatFood = this.foodIdx < this.food.length &&
                    this.food[this.foodIdx][0] === newHead[0] &&
                    this.food[this.foodIdx][1] === newHead[1];

    // Remove tail first (unless eating)
    if (!eatFood) {
      const tail = this.snake.pop()!;
      this.occupied.delete(`${tail[0]},${tail[1]}`);
    }

    // Self collision
    if (this.occupied.has(`${newHead[0]},${newHead[1]}`)) return -1;

    // Move
    this.snake.unshift(newHead);
    this.occupied.add(`${newHead[0]},${newHead[1]}`);
    if (eatFood) { this.score++; this.foodIdx++; }
    return this.score;
  }
}

// Demo
const snake = new SnakeGame(3, 2, [[0,1],[0,2]]);
console.log(snake.move("R")); // 1 (ate food)
console.log(snake.move("D")); // 1
console.log(snake.move("R")); // 1
console.log(snake.move("U")); // 2 (ate food)
```

### Tradeoffs

- **Deque + Set** gives O(1) head/tail operations. The Set avoids O(n) self-collision scan.
- **Array unshift** is O(n) â€” use a proper Deque (doubly-linked list) for O(1) head insertion in production.

---

## Q58. Circular Deque

### Problem Statement

Design a circular deque (double-ended queue) with `insertFront`, `insertLast`, `deleteFront`, `deleteLast`, `getFront`, `getRear`, `isEmpty`, `isFull` â€” all O(1).

- **Pattern:** Ring buffer with two pointers | **Difficulty:** Medium | **LeetCode:** 641 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class CircularDeque<T> {
  private readonly buffer: (T | undefined)[];
  private front: number;
  private rear: number;
  private size: number = 0;
  private readonly capacity: number;

  constructor(capacity: number) {
    this.capacity = capacity;
    this.buffer = new Array(capacity);
    this.front = 0;
    this.rear = capacity - 1;
  }

  insertFront(val: T): boolean {
    if (this.isFull()) return false;
    this.front = (this.front - 1 + this.capacity) % this.capacity;
    this.buffer[this.front] = val;
    this.size++;
    return true;
  }

  insertLast(val: T): boolean {
    if (this.isFull()) return false;
    this.rear = (this.rear + 1) % this.capacity;
    this.buffer[this.rear] = val;
    this.size++;
    return true;
  }

  deleteFront(): boolean {
    if (this.isEmpty()) return false;
    this.buffer[this.front] = undefined;
    this.front = (this.front + 1) % this.capacity;
    this.size--;
    return true;
  }

  deleteLast(): boolean {
    if (this.isEmpty()) return false;
    this.buffer[this.rear] = undefined;
    this.rear = (this.rear - 1 + this.capacity) % this.capacity;
    this.size--;
    return true;
  }

  getFront(): T | undefined { return this.isEmpty() ? undefined : this.buffer[this.front]; }
  getRear():  T | undefined { return this.isEmpty() ? undefined : this.buffer[this.rear]; }
  isEmpty(): boolean { return this.size === 0; }
  isFull():  boolean { return this.size === this.capacity; }
}

// Demo
const cd = new CircularDeque<number>(5);
cd.insertLast(1); cd.insertLast(2); cd.insertFront(0);
console.log(cd.getFront()); // 0
console.log(cd.getRear());  // 2
cd.deleteFront();
console.log(cd.getFront()); // 1
```

---

## Pattern 6 â€” Summary

| Q                     | Structure                    | Key Insight                                 |
| --------------------- | ---------------------------- | ------------------------------------------- |
| Q47 Browser History   | Two Stacks                   | Visit clears forward stack                  |
| Q48 Undo/Redo         | Command Pattern + Two Stacks | Each command knows its inverse              |
| Q49 Min Stack         | Auxiliary min-stack          | Mirror stack stores running minimum         |
| Q50 Max Stack         | Stack + Map                  | popMax: buffer and re-push                  |
| Q51 Calculator        | Stack for parens             | Push result+sign on `(`, restore on `)` |
| Q52 JSON Validator    | Stack                        | Match closing to last opening bracket       |
| Q53 Circular Queue    | Ring buffer                  | `(idx+1)%capacity` for wrap-around        |
| Q54 Queue from Stacks | Inbox/Outbox stacks          | Transfer amortised O(1)                     |
| Q55 Paren Fixer       | Counters                     | open+close = min insertions                 |
| Q56 Music Playlist    | Deque + HashSet              | Evict oldest from recent-set                |
| Q57 Snake Game        | Deque + Set                  | Remove tail before checking self-collision  |
| Q58 Circular Deque    | Ring buffer + two pointers   | front and rear wrap independently           |

---

# PATTERN 7 â€” Graph / Topological Sort

> **When you hear:** "dependencies", "cycle detection", "reachability", "connected components", "shortest path"
> **Reach for:** BFS/DFS for exploration, Topological Sort (Kahn's BFS or DFS) for dependency ordering, Dijkstra for weighted shortest path.

---

## Q59. Build System â€” Compile in Dependency Order

### Problem Statement

Given a list of build tasks and their dependencies, determine the order to build all tasks. Detect cycles (circular dependencies).

**Real scenario:** Bazel's build system orders compilation tasks. Maven resolves module build order.

- **Pattern:** Topological Sort (Kahn's algorithm) | **Difficulty:** Medium | **LeetCode:** 207 | **Asked at:** Atlassian, Google

---

### TypeScript Solution

```typescript
interface IBuildSystem {
  addTask(task: string): void;
  addDependency(task: string, dependsOn: string): void;
  getBuildOrder(): string[] | null; // null = cycle detected
}

class BuildSystem implements IBuildSystem {
  private readonly graph: Map<string, string[]> = new Map();   // task -> [dependents]
  private readonly inDegree: Map<string, number> = new Map(); // task -> dependency count
  private readonly tasks: Set<string> = new Set();

  addTask(task: string): void {
    this.tasks.add(task);
    if (!this.graph.has(task))    this.graph.set(task, []);
    if (!this.inDegree.has(task)) this.inDegree.set(task, 0);
  }

  addDependency(task: string, dependsOn: string): void {
    this.addTask(task); this.addTask(dependsOn);
    this.graph.get(dependsOn)!.push(task);
    this.inDegree.set(task, (this.inDegree.get(task) ?? 0) + 1);
  }

  getBuildOrder(): string[] | null {
    const queue: string[] = [];
    for (const [task, deg] of this.inDegree) {
      if (deg === 0) queue.push(task);
    }
    const order: string[] = [];
    while (queue.length > 0) {
      const task = queue.shift()!;
      order.push(task);
      for (const dependent of (this.graph.get(task) ?? [])) {
        const newDeg = this.inDegree.get(dependent)! - 1;
        this.inDegree.set(dependent, newDeg);
        if (newDeg === 0) queue.push(dependent);
      }
    }
    return order.length === this.tasks.size ? order : null; // null = cycle
  }
}

// Demo
const build = new BuildSystem();
build.addDependency("link",    "compile");
build.addDependency("compile", "parse");
build.addDependency("test",    "link");
console.log(build.getBuildOrder()); // ["parse","compile","link","test"]
```

### Complexity | Time: O(V + E) | Space: O(V + E)

### Tradeoffs

- **Kahn's BFS (used):** Iterative, easy to detect cycles (remaining tasks with non-zero in-degree).
- **DFS post-order:** Recursive, elegant, but stack overflow risk for deep graphs. Use iterative DFS with explicit stack.

---

## Q60. Spreadsheet Dependency Evaluator

### Problem Statement

In a spreadsheet, cells may reference other cells (`A1 = B1 + C1`). Evaluate all cells in correct order. Detect circular references.

- **Pattern:** Topological Sort on cell dependency graph | **Difficulty:** Medium | **Asked at:** Microsoft (Excel team), Notion

---

### TypeScript Solution

```typescript
class Spreadsheet {
  private readonly formulas: Map<string, string[]> = new Map(); // cell -> [depends on]
  private readonly values: Map<string, number> = new Map();

  setFormula(cell: string, dependsOn: string[], computeFn: (...vals: number[]) => number): void {
    this.formulas.set(cell, dependsOn);
    // Store compute function separately in a real impl; here we store a simple sum
    const vals = dependsOn.map(c => this.values.get(c) ?? 0);
    this.values.set(cell, computeFn(...vals));
  }

  setValue(cell: string, val: number): void {
    this.values.set(cell, val);
  }

  getEvalOrder(): string[] | null {
    const inDegree = new Map<string, number>();
    const graph = new Map<string, string[]>();

    for (const [cell, deps] of this.formulas) {
      if (!inDegree.has(cell)) inDegree.set(cell, 0);
      for (const dep of deps) {
        if (!graph.has(dep)) graph.set(dep, []);
        graph.get(dep)!.push(cell);
        inDegree.set(cell, (inDegree.get(cell) ?? 0) + 1);
      }
    }

    const queue = [...this.values.keys()].filter(c => !this.formulas.has(c) || (inDegree.get(c) ?? 0) === 0);
    const order: string[] = [];
    while (queue.length) {
      const cell = queue.shift()!;
      order.push(cell);
      for (const dep of (graph.get(cell) ?? [])) {
        const deg = (inDegree.get(dep) ?? 1) - 1;
        inDegree.set(dep, deg);
        if (deg === 0) queue.push(dep);
      }
    }
    return order;
  }
}

// Demo
const sheet = new Spreadsheet();
sheet.setValue("A1", 10);
sheet.setValue("B1", 20);
sheet.setFormula("C1", ["A1","B1"], (a, b) => a + b); // C1 = A1+B1
sheet.setFormula("D1", ["C1"], c => c * 2);            // D1 = C1*2
console.log(sheet.getEvalOrder()); // A1, B1, C1, D1
```

---

## Q61. Course Schedule â€” Can You Finish?

### Problem Statement

Given N courses and a list of `[course, prerequisite]` pairs, determine if it's possible to finish all courses (no cycle).

- **Pattern:** Cycle detection via Topological Sort | **Difficulty:** Medium | **LeetCode:** 207 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class CourseScheduler {
  canFinish(numCourses: number, prerequisites: [number, number][]): boolean {
    const inDegree = new Array(numCourses).fill(0);
    const graph: number[][] = Array.from({ length: numCourses }, () => []);

    for (const [course, pre] of prerequisites) {
      graph[pre].push(course);
      inDegree[course]++;
    }

    const queue = inDegree.map((d, i) => d === 0 ? i : -1).filter(i => i !== -1);
    let completed = 0;
    while (queue.length) {
      const course = queue.shift()!;
      completed++;
      for (const next of graph[course]) {
        if (--inDegree[next] === 0) queue.push(next);
      }
    }
    return completed === numCourses;
  }

  findOrder(numCourses: number, prerequisites: [number, number][]): number[] | null {
    const inDegree = new Array(numCourses).fill(0);
    const graph: number[][] = Array.from({ length: numCourses }, () => []);

    for (const [course, pre] of prerequisites) {
      graph[pre].push(course);
      inDegree[course]++;
    }

    const queue = inDegree.map((d, i) => d === 0 ? i : -1).filter(i => i !== -1);
    const order: number[] = [];
    while (queue.length) {
      const course = queue.shift()!;
      order.push(course);
      for (const next of graph[course]) {
        if (--inDegree[next] === 0) queue.push(next);
      }
    }
    return order.length === numCourses ? order : null;
  }
}

// Demo
const cs = new CourseScheduler();
console.log(cs.canFinish(2, [[1,0]]));       // true
console.log(cs.canFinish(2, [[1,0],[0,1]])); // false (cycle)
console.log(cs.findOrder(4, [[1,0],[2,0],[3,1],[3,2]])); // [0,1,2,3]
```

---

## Q62. Package Dependency Resolver

### Problem Statement

A package manager must install packages in the right order â€” all dependencies before a package.

- **Pattern:** Topological Sort | **Difficulty:** Medium | **Asked at:** npm/yarn-style system design

---

### TypeScript Solution

```typescript
class PackageResolver {
  private readonly deps: Map<string, string[]> = new Map();

  addPackage(pkg: string, dependencies: string[]): void {
    this.deps.set(pkg, dependencies);
    for (const dep of dependencies) {
      if (!this.deps.has(dep)) this.deps.set(dep, []);
    }
  }

  resolve(target: string): string[] | null {
    const visited = new Set<string>();
    const inStack = new Set<string>(); // cycle detection
    const order: string[] = [];

    const dfs = (pkg: string): boolean => {
      if (inStack.has(pkg)) return false; // cycle
      if (visited.has(pkg)) return true;
      inStack.add(pkg);
      for (const dep of (this.deps.get(pkg) ?? [])) {
        if (!dfs(dep)) return false;
      }
      inStack.delete(pkg);
      visited.add(pkg);
      order.push(pkg);
      return true;
    };

    if (!dfs(target)) return null;
    return order; // already in install order (post-order DFS)
  }
}

// Demo
const pm = new PackageResolver();
pm.addPackage("react-dom", ["react","scheduler"]);
pm.addPackage("react", ["loose-envify"]);
pm.addPackage("scheduler", []);
pm.addPackage("loose-envify", []);
console.log(pm.resolve("react-dom"));
// ["loose-envify","react","scheduler","react-dom"]
```

---

## Q63. People You May Know (Friend Suggestions)

### Problem Statement

Given a social graph, suggest friends for a user â€” people not directly connected who share the most mutual friends.

**Real scenario:** LinkedIn's "People You May Know". Facebook's friend suggestions.

- **Pattern:** BFS Level-2 + mutual count | **Difficulty:** Medium | **Asked at:** LinkedIn, Facebook

---

### TypeScript Solution

```typescript
class SocialGraph {
  private readonly friends: Map<number, Set<number>> = new Map();

  addFriendship(a: number, b: number): void {
    if (!this.friends.has(a)) this.friends.set(a, new Set());
    if (!this.friends.has(b)) this.friends.set(b, new Set());
    this.friends.get(a)!.add(b);
    this.friends.get(b)!.add(a);
  }

  suggest(userId: number, topK: number = 5): { userId: number; mutualCount: number }[] {
    const myFriends = this.friends.get(userId) ?? new Set<number>();
    const mutualCount = new Map<number, number>();

    for (const friend of myFriends) {
      for (const fof of (this.friends.get(friend) ?? new Set())) {
        if (fof !== userId && !myFriends.has(fof)) {
          mutualCount.set(fof, (mutualCount.get(fof) ?? 0) + 1);
        }
      }
    }

    return Array.from(mutualCount.entries())
      .map(([uid, count]) => ({ userId: uid, mutualCount: count }))
      .sort((a, b) => b.mutualCount - a.mutualCount)
      .slice(0, topK);
  }
}

// Demo
const sg = new SocialGraph();
[[1,2],[1,3],[2,4],[3,4],[4,5]].forEach(([a,b]) => sg.addFriendship(a,b));
console.log(sg.suggest(1)); // [{userId:4, mutualCount:2}]
```

### Tradeoffs

- **Two-hop BFS** gives O(F^2) where F = avg friends count. Feasible for sparse graphs.
- **Matrix multiplication approach:** `A^2` (adjacency matrix squared) gives mutual friend counts â€” O(V^3) but parallelisable.
- **At scale (LinkedIn):** Pre-compute mutual friend counts in batch Spark jobs, store in a key-value store.

---

## Q64. Friend Circles (Union-Find)

### Problem Statement

Given an NÃ—N matrix where `M[i][j]=1` means students i and j are direct friends, find the number of distinct friend circles.

- **Pattern:** Union-Find (Disjoint Set Union) | **Difficulty:** Medium | **LeetCode:** 547 | **Asked at:** Microsoft

---

### TypeScript Solution

```typescript
class UnionFind {
  private readonly parent: number[];
  private readonly rank: number[];
  private components: number;

  constructor(n: number) {
    this.parent = Array.from({ length: n }, (_, i) => i);
    this.rank = new Array(n).fill(0);
    this.components = n;
  }

  find(x: number): number {
    if (this.parent[x] !== x) this.parent[x] = this.find(this.parent[x]); // path compression
    return this.parent[x];
  }

  union(x: number, y: number): void {
    const px = this.find(x), py = this.find(y);
    if (px === py) return;
    if (this.rank[px] < this.rank[py]) this.parent[px] = py;
    else if (this.rank[px] > this.rank[py]) this.parent[py] = px;
    else { this.parent[py] = px; this.rank[px]++; }
    this.components--;
  }

  getComponents(): number { return this.components; }
}

function findCircleNum(M: number[][]): number {
  const n = M.length;
  const uf = new UnionFind(n);
  for (let i = 0; i < n; i++) {
    for (let j = i + 1; j < n; j++) {
      if (M[i][j] === 1) uf.union(i, j);
    }
  }
  return uf.getComponents();
}

// Demo
console.log(findCircleNum([[1,1,0],[1,1,0],[0,0,1]])); // 2
console.log(findCircleNum([[1,0,0],[0,1,0],[0,0,1]])); // 3
```

### Complexity | Time: O(N^2 * alpha(N)) near O(N^2) | Space: O(N)

### Tradeoffs

- **Union-Find with path compression + rank** is nearly O(1) per operation (inverse Ackermann alpha).
- **DFS/BFS alternative:** Same O(N^2) complexity, but Union-Find handles dynamic union queries better.

---

## Q65. Accounts Merge

### Problem Statement

Given accounts where each account has a name and list of emails, merge accounts that share any email.

- **Pattern:** Union-Find on emails | **Difficulty:** Medium | **LeetCode:** 721 | **Asked at:** Amazon, Google

---

### TypeScript Solution

```typescript
class AccountsMerger {
  merge(accounts: string[][]): string[][] {
    const emailParent = new Map<string, string>(); // email -> root email
    const emailName = new Map<string, string>();   // email -> account name

    const find = (x: string): string => {
      if (emailParent.get(x) !== x) emailParent.set(x, find(emailParent.get(x)!));
      return emailParent.get(x)!;
    };

    const union = (a: string, b: string): void => {
      const pa = find(a), pb = find(b);
      if (pa !== pb) emailParent.set(pa, pb);
    };

    // Initialize
    for (const account of accounts) {
      for (let i = 1; i < account.length; i++) {
        if (!emailParent.has(account[i])) emailParent.set(account[i], account[i]);
        emailName.set(account[i], account[0]);
        union(account[1], account[i]);
      }
    }

    // Group by root
    const groups = new Map<string, string[]>();
    for (const email of emailParent.keys()) {
      const root = find(email);
      if (!groups.has(root)) groups.set(root, []);
      groups.get(root)!.push(email);
    }

    return Array.from(groups.values()).map(emails => {
      emails.sort();
      return [emailName.get(emails[0])!, ...emails];
    });
  }
}

// Demo
const am = new AccountsMerger();
console.log(am.merge([
  ["Rahul","r@g.com","r@fb.com"],
  ["Priya","p@g.com"],
  ["Rahul","r@fb.com","r@tw.com"]
]));
// [["Rahul","r@fb.com","r@g.com","r@tw.com"],["Priya","p@g.com"]]
```

---

## Q66. Cheapest Flights Within K Stops

### Problem Statement

Find the cheapest price from source to destination with at most K stops.

**Real scenario:** MakeMyTrip/Cleartrip suggest cheapest flight routes with connection limits.

- **Pattern:** Bellman-Ford / modified BFS | **Difficulty:** Medium | **LeetCode:** 787 | **Asked at:** MakeMyTrip

---

### TypeScript Solution

```typescript
function findCheapestPrice(
  n: number,
  flights: [number, number, number][],
  src: number,
  dst: number,
  k: number
): number {
  // Bellman-Ford relaxed k+1 times
  let prices = new Array(n).fill(Infinity);
  prices[src] = 0;

  for (let i = 0; i <= k; i++) {
    const temp = [...prices];
    for (const [from, to, cost] of flights) {
      if (prices[from] !== Infinity && prices[from] + cost < temp[to]) {
        temp[to] = prices[from] + cost;
      }
    }
    prices = temp;
  }
  return prices[dst] === Infinity ? -1 : prices[dst];
}

// Demo
console.log(findCheapestPrice(4, [[0,1,100],[1,2,100],[2,0,100],[1,3,600],[2,3,200]], 0, 3, 1));
// 700 (0->1->3, 1 stop)
```

### Tradeoffs

- **Bellman-Ford with k iterations** is O(k * E). Simple and correct.
- **Dijkstra with state (node, stops_used):** O((V+E) log V) â€” faster for dense graphs.

---

## Q67. Delivery Route Optimization

### Problem Statement

Find the shortest route to deliver to multiple locations and return to the warehouse. (Simplified TSP for small n.)

- **Pattern:** DP on bitmask subsets | **Difficulty:** Hard | **Asked at:** Porter, Dunzo

---

### TypeScript Solution

```typescript
class DeliveryRouter {
  shortestRoute(dist: number[][]): number {
    const n = dist.length;
    const ALL_VISITED = (1 << n) - 1;
    // dp[mask][i] = min cost to visit all in mask, ending at i, starting from 0
    const dp: number[][] = Array.from({ length: 1 << n }, () => new Array(n).fill(Infinity));
    dp[1][0] = 0; // start at node 0, mask=1 (only node 0 visited)

    for (let mask = 1; mask < (1 << n); mask++) {
      for (let u = 0; u < n; u++) {
        if (!(mask & (1 << u)) || dp[mask][u] === Infinity) continue;
        for (let v = 0; v < n; v++) {
          if (mask & (1 << v)) continue; // already visited
          const newMask = mask | (1 << v);
          dp[newMask][v] = Math.min(dp[newMask][v], dp[mask][u] + dist[u][v]);
        }
      }
    }

    let ans = Infinity;
    for (let u = 1; u < n; u++) {
      ans = Math.min(ans, dp[ALL_VISITED][u] + dist[u][0]);
    }
    return ans;
  }
}

// Demo
const router = new DeliveryRouter();
const dist = [[0,10,15,20],[10,0,35,25],[15,35,0,30],[20,25,30,0]];
console.log(router.shortestRoute(dist)); // 80
```

### Complexity | Time: O(2^n * n^2) | Space: O(2^n * n)

### Tradeoffs

- **Bitmask DP** is exact but O(2^n * n^2) â€” feasible up to n~20.
- **Greedy nearest neighbor:** O(n^2), fast but not optimal. Used in practice for large n.
- **Christofides algorithm:** 1.5x optimal approximation. Used in Google Maps routing.

---

## Q68. Nearest Cab (BFS on Grid)

### Problem Statement

Given a grid where `0` = road, `1` = blocked, `C` = customer, `D` = driver, find the minimum time for the nearest driver to reach the customer using BFS.

- **Pattern:** Multi-source BFS | **Difficulty:** Medium | **Asked at:** Uber, Ola

---

### TypeScript Solution

```typescript
function nearestCab(grid: string[][]): number {
  const rows = grid.length, cols = grid[0].length;
  const queue: [number, number, number][] = []; // [row, col, dist]
  let customer: [number, number] | null = null;

  // Seed BFS from all driver positions simultaneously
  for (let r = 0; r < rows; r++) {
    for (let c = 0; c < cols; c++) {
      if (grid[r][c] === "D") queue.push([r, c, 0]);
      else if (grid[r][c] === "C") customer = [r, c];
    }
  }

  const visited = Array.from({ length: rows }, () => new Array(cols).fill(false));
  for (const [r, c] of queue) visited[r][c] = true;

  const dirs = [[-1,0],[1,0],[0,-1],[0,1]];
  while (queue.length) {
    const [r, c, dist] = queue.shift()!;
    if (customer && r === customer[0] && c === customer[1]) return dist;
    for (const [dr, dc] of dirs) {
      const nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
          !visited[nr][nc] && grid[nr][nc] !== "1") {
        visited[nr][nc] = true;
        queue.push([nr, nc, dist + 1]);
      }
    }
  }
  return -1;
}

// Demo
const grid = [
  ["D","0","0"],
  ["0","1","0"],
  ["0","0","C"]
];
console.log(nearestCab(grid)); // 4
```

---

## Q69. Word Ladder â€” Transform Word Step by Step

### Problem Statement

Transform `beginWord` to `endWord` by changing one letter at a time, where each intermediate word must be in the dictionary. Return the shortest transformation length.

- **Pattern:** BFS on word graph | **Difficulty:** Hard | **LeetCode:** 127 | **Asked at:** Amazon, Google

---

### TypeScript Solution

```typescript
function wordLadder(beginWord: string, endWord: string, wordList: string[]): number {
  const wordSet = new Set(wordList);
  if (!wordSet.has(endWord)) return 0;

  const queue: [string, number][] = [[beginWord, 1]];
  const visited = new Set<string>([beginWord]);

  while (queue.length) {
    const [word, steps] = queue.shift()!;
    for (let i = 0; i < word.length; i++) {
      for (let c = 0; c < 26; c++) {
        const newChar = String.fromCharCode(97 + c);
        const newWord = word.slice(0, i) + newChar + word.slice(i + 1);
        if (newWord === endWord) return steps + 1;
        if (wordSet.has(newWord) && !visited.has(newWord)) {
          visited.add(newWord);
          queue.push([newWord, steps + 1]);
        }
      }
    }
  }
  return 0;
}

// Demo
console.log(wordLadder("hit", "cog", ["hot","dot","dog","lot","log","cog"])); // 5
// hit -> hot -> dot -> dog -> cog
```

### Complexity | Time: O(M^2 * N) where M = word length, N = words | Space: O(M * N)

### Tradeoffs

- **Bidirectional BFS** reduces O(b^d) to O(2 * b^(d/2)) â€” significant for large word lists.
- **Pattern-based grouping:** Group words by pattern (`*ot`, `h*t`) for O(1) neighbor lookup.

---

## Q70. Flood Fill

### Problem Statement

Given an image (2D grid of pixel colors) and a starting pixel, flood fill: change the starting pixel and all connected same-color pixels to a new color.

- **Pattern:** BFS/DFS on grid | **Difficulty:** Easy | **LeetCode:** 733 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class FloodFill {
  fill(image: number[][], sr: number, sc: number, color: number): number[][] {
    const originalColor = image[sr][sc];
    if (originalColor === color) return image;
    const rows = image.length, cols = image[0].length;
    const queue: [number, number][] = [[sr, sc]];
    image[sr][sc] = color;

    const dirs = [[-1,0],[1,0],[0,-1],[0,1]];
    while (queue.length) {
      const [r, c] = queue.shift()!;
      for (const [dr, dc] of dirs) {
        const nr = r+dr, nc = c+dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && image[nr][nc] === originalColor) {
          image[nr][nc] = color;
          queue.push([nr, nc]);
        }
      }
    }
    return image;
  }
}

// Demo
const ff = new FloodFill();
console.log(ff.fill([[1,1,1],[1,1,0],[1,0,1]], 1, 1, 2));
// [[2,2,2],[2,2,0],[2,0,1]]
```

---

## Q71. Deadlock Detection

### Problem Statement

Given resource-allocation and request edges in a wait-for graph, detect if there is a deadlock (cycle).

**Real scenario:** Database lock managers detect deadlocks and abort the youngest transaction.

- **Pattern:** DFS cycle detection | **Difficulty:** Medium | **Asked at:** Oracle DB, system design interviews

---

### TypeScript Solution

```typescript
class DeadlockDetector {
  private readonly waitForGraph: Map<number, number[]> = new Map();

  addWait(process: number, waitingFor: number): void {
    if (!this.waitForGraph.has(process)) this.waitForGraph.set(process, []);
    this.waitForGraph.get(process)!.push(waitingFor);
  }

  hasDeadlock(): number[] | null {
    const visited = new Set<number>();
    const inStack = new Set<number>();
    const parent = new Map<number, number>();

    const dfs = (node: number): number | null => {
      visited.add(node); inStack.add(node);
      for (const neighbor of (this.waitForGraph.get(node) ?? [])) {
        if (!visited.has(neighbor)) {
          parent.set(neighbor, node);
          const cycle = dfs(neighbor);
          if (cycle !== null) return cycle;
        } else if (inStack.has(neighbor)) {
          return neighbor; // cycle found, return entry point
        }
      }
      inStack.delete(node);
      return null;
    };

    for (const node of this.waitForGraph.keys()) {
      if (!visited.has(node)) {
        const cycleEntry = dfs(node);
        if (cycleEntry !== null) return [cycleEntry]; // return deadlocked processes
      }
    }
    return null;
  }
}

// Demo
const dd = new DeadlockDetector();
dd.addWait(1, 2); dd.addWait(2, 3); dd.addWait(3, 1); // cycle: 1->2->3->1
console.log(dd.hasDeadlock()); // [1] (deadlock found)
```

---

## Q72. Web Crawler

### Problem Statement

Design a web crawler: given a seed URL, crawl all pages reachable from it (same domain only), collecting unique URLs.

**Real scenario:** Google's Googlebot, Bing's crawler, site auditors like Screaming Frog.

- **Pattern:** BFS + HashSet for visited | **Difficulty:** Medium | **LeetCode:** 1236 | **Asked at:** Google

---

### TypeScript Solution

```typescript
interface IHtmlParser {
  getUrls(url: string): string[];
}

class WebCrawler {
  crawl(startUrl: string, htmlParser: IHtmlParser): string[] {
    const hostname = startUrl.split("/")[2]; // extract domain
    const visited = new Set<string>([startUrl]);
    const queue: string[] = [startUrl];

    while (queue.length) {
      const url = queue.shift()!;
      for (const nextUrl of htmlParser.getUrls(url)) {
        const nextHostname = nextUrl.split("/")[2];
        if (nextHostname === hostname && !visited.has(nextUrl)) {
          visited.add(nextUrl);
          queue.push(nextUrl);
        }
      }
    }
    return Array.from(visited);
  }
}

// Demo
const mockParser: IHtmlParser = {
  getUrls(url: string): string[] {
    const graph: Record<string, string[]> = {
      "http://news.yahoo.com": ["http://news.yahoo.com/news","http://news.google.com"],
      "http://news.yahoo.com/news": ["http://news.yahoo.com/news/topics"],
      "http://news.google.com": []
    };
    return graph[url] ?? [];
  }
};
const crawler = new WebCrawler();
console.log(crawler.crawl("http://news.yahoo.com", mockParser));
// ["http://news.yahoo.com","http://news.yahoo.com/news","http://news.yahoo.com/news/topics"]
```

### Tradeoffs

- **BFS (used):** Breadth-first ensures shallow pages are discovered first (good for site maps).
- **Politeness:** Real crawlers add `robots.txt` checks and rate limiting (1 req/sec per domain).
- **Distributed crawling:** Use a URL frontier queue (Redis, Kafka) shared across workers to scale horizontally.

---

## Pattern 7 â€” Summary

| Q                    | Algorithm                 | Key Insight                               |
| -------------------- | ------------------------- | ----------------------------------------- |
| Q59 Build System     | Kahn's Topo Sort          | BFS from in-degree=0 nodes                |
| Q60 Spreadsheet      | Topo Sort                 | Detect circular cell references           |
| Q61 Course Schedule  | Kahn's + cycle            | completed count vs numCourses             |
| Q62 Package Resolver | DFS Topo Sort             | Post-order DFS gives install order        |
| Q63 Friend Suggest   | Two-hop BFS               | Count mutual friends via FOF              |
| Q64 Friend Circles   | Union-Find                | Path compression + rank                   |
| Q65 Accounts Merge   | Union-Find on emails      | Root email = group representative         |
| Q66 Cheapest Flights | Bellman-Ford k iterations | Copy prev prices to avoid same-level mix  |
| Q67 Delivery Route   | Bitmask DP                | TSP exact for n<=20                       |
| Q68 Nearest Cab      | Multi-source BFS          | Seed from all drivers simultaneously      |
| Q69 Word Ladder      | BFS on word graph         | Try all 26 chars at each position         |
| Q70 Flood Fill       | BFS/DFS grid              | Stop if same color to avoid infinite loop |
| Q71 Deadlock         | DFS cycle detection       | inStack set tracks current path           |
| Q72 Web Crawler      | BFS + same-domain filter  | Extract hostname for domain check         |

---

# PATTERN 8 â€” Interval / Sweep Line

> **When you hear:** "overlapping intervals", "merge ranges", "find free slots", "calendar conflict"
> **Reach for:** Sort by start time, then sweep. Use a min-heap of end times for active intervals.

---

## Q73. Calendar â€” Can I Schedule This Meeting?

### Problem Statement

Design a calendar where you can book a meeting `[start, end)`. Return `false` if the slot conflicts with an existing booking.

- **Pattern:** Sorted list + binary search for conflicts | **Difficulty:** Medium | **LeetCode:** 729 | **Asked at:** Zoho

---

### TypeScript Solution

```typescript
class Calendar {
  private readonly bookings: [number, number][] = [];

  book(start: number, end: number): boolean {
    // Check overlap: new interval [start,end) overlaps existing [s,e) if start<e && end>s
    for (const [s, e] of this.bookings) {
      if (start < e && end > s) return false;
    }
    this.bookings.push([start, end]);
    return true;
  }
}

// Demo
const cal = new Calendar();
console.log(cal.book(10, 20)); // true
console.log(cal.book(15, 25)); // false (overlaps)
console.log(cal.book(20, 30)); // true
```

### O(n) per booking. For O(log n): use a sorted TreeMap of intervals and check neighbors only.

---

## Q74. Calendar III â€” Max K Simultaneous Events

### Problem Statement

Return the maximum number of events that are active at any given time.

- **Pattern:** Sweep-line / difference array | **Difficulty:** Hard | **LeetCode:** 732 | **Asked at:** Google

---

### TypeScript Solution

```typescript
class CalendarMaxOverlap {
  private readonly events: Map<number, number> = new Map();

  book(start: number, end: number): number {
    this.events.set(start, (this.events.get(start) ?? 0) + 1);
    this.events.set(end,   (this.events.get(end)   ?? 0) - 1);

    let maxK = 0, running = 0;
    for (const time of Array.from(this.events.keys()).sort((a, b) => a - b)) {
      running += this.events.get(time)!;
      maxK = Math.max(maxK, running);
    }
    return maxK;
  }
}

// Demo
const cal3 = new CalendarMaxOverlap();
console.log(cal3.book(10, 20)); // 1
console.log(cal3.book(50, 60)); // 1
console.log(cal3.book(10, 40)); // 2
console.log(cal3.book(5, 15));  // 3
```

### Key Insight: Treat start as +1 event, end as -1 event. Sweep to find max concurrent.

---

## Q75. Merge Overlapping Schedules

### Problem Statement

Given a list of intervals, merge all overlapping ones.

- **Pattern:** Sort + merge | **Difficulty:** Easy-Medium | **LeetCode:** 56 | **Asked at:** Amazon, Google

---

### TypeScript Solution

```typescript
class IntervalMerger {
  merge(intervals: [number, number][]): [number, number][] {
    if (!intervals.length) return [];
    intervals.sort((a, b) => a[0] - b[0]);
    const merged: [number, number][] = [intervals[0]];

    for (let i = 1; i < intervals.length; i++) {
      const [start, end] = intervals[i];
      const last = merged[merged.length - 1];
      if (start <= last[1]) {
        last[1] = Math.max(last[1], end); // extend
      } else {
        merged.push([start, end]);
      }
    }
    return merged;
  }
}

// Demo
const im = new IntervalMerger();
console.log(im.merge([[1,3],[2,6],[8,10],[15,18]])); // [[1,6],[8,10],[15,18]]
```

---

## Q76. Find Free Time Slot for All Employees

### Problem Statement

Given each employee's busy schedule, find time slots where **all** employees are free (within a working window `[start, end]`).

- **Pattern:** Flatten + sort + find gaps | **Difficulty:** Medium | **LeetCode:** 759 | **Asked at:** Google

---

### TypeScript Solution

```typescript
class FreeTimeSlotFinder {
  findFreeTime(
    schedules: [number, number][][],
    dayStart: number,
    dayEnd: number,
    minDuration: number
  ): [number, number][] {
    // Flatten all busy intervals
    const busy: [number, number][] = schedules.flat();
    busy.sort((a, b) => a[0] - b[0]);

    // Merge overlapping busy slots
    const merged: [number, number][] = [];
    for (const interval of busy) {
      if (merged.length && interval[0] <= merged[merged.length-1][1]) {
        merged[merged.length-1][1] = Math.max(merged[merged.length-1][1], interval[1]);
      } else {
        merged.push([...interval] as [number, number]);
      }
    }

    // Find gaps
    const free: [number, number][] = [];
    let prev = dayStart;
    for (const [start, end] of merged) {
      if (start > prev && start - prev >= minDuration) {
        free.push([prev, start]);
      }
      prev = Math.max(prev, end);
    }
    if (dayEnd - prev >= minDuration) free.push([prev, dayEnd]);
    return free;
  }
}

// Demo
const fts = new FreeTimeSlotFinder();
// Rahul: 9-10, 12-13. Priya: 10-11, 14-15. Find free >= 60 min in 9-17
console.log(fts.findFreeTime([[[9,10],[12,13]],[[10,11],[14,15]]], 9, 17, 60));
// [[11,12],[13,14],[15,17]]
```

---

## Q77. Contiguous Seat Assignment

### Problem Statement

Given booked seats in a row, find if K contiguous seats are available.

- **Pattern:** Sweep / gap counting | **Difficulty:** Easy | **Asked at:** BookMyShow

---

### TypeScript Solution

```typescript
class SeatAssigner {
  private readonly booked: Set<number> = new Set();
  private readonly totalSeats: number;

  constructor(totalSeats: number) { this.totalSeats = totalSeats; }

  book(seat: number): void { this.booked.add(seat); }

  findContiguous(k: number): number[] | null {
    let start = 1, count = 0;
    let gapStart = 1;

    for (let seat = 1; seat <= this.totalSeats; seat++) {
      if (!this.booked.has(seat)) {
        count++;
        if (count === 1) gapStart = seat;
        if (count === k) {
          return Array.from({ length: k }, (_, i) => gapStart + i);
        }
      } else {
        count = 0;
      }
    }
    return null;
  }
}

// Demo
const sa2 = new SeatAssigner(10);
sa2.book(1); sa2.book(2); sa2.book(5);
console.log(sa2.findContiguous(3)); // [6,7,8] or [3,4,5]? Actually [3,4,?] â€” 3,4 then 5 is booked. So [6,7,8]
```

---

## Q78. Hotel Availability â€” Can We Book N Rooms for D Days?

### Problem Statement

Given hotel bookings `[checkIn, checkOut]`, for any new query check if `N` or more rooms are available for a given date range.

- **Pattern:** Sweep line on dates | **Difficulty:** Medium | **Asked at:** MakeMyTrip, OYO

---

### TypeScript Solution

```typescript
class HotelBooking {
  private readonly totalRooms: number;
  private readonly bookings: [number, number][] = [];

  constructor(totalRooms: number) { this.totalRooms = totalRooms; }

  book(checkIn: number, checkOut: number): boolean {
    // Check max occupancy in the requested window
    const maxOccupied = this.maxSimultaneous(checkIn, checkOut);
    if (maxOccupied >= this.totalRooms) return false;
    this.bookings.push([checkIn, checkOut]);
    return true;
  }

  private maxSimultaneous(queryIn: number, queryOut: number): number {
    const events: [number, number][] = [];
    for (const [ci, co] of this.bookings) {
      // Only consider bookings that overlap with query window
      if (ci < queryOut && co > queryIn) {
        events.push([ci, 1], [co, -1]);
      }
    }
    events.sort((a, b) => a[0] !== b[0] ? a[0] - b[0] : a[1] - b[1]);
    let max = 0, cur = 0;
    for (const [, delta] of events) {
      cur += delta;
      max = Math.max(max, cur);
    }
    return max;
  }
}

// Demo
const hotel = new HotelBooking(2);
console.log(hotel.book(1, 5));  // true (room 1)
console.log(hotel.book(2, 6));  // true (room 2)
console.log(hotel.book(3, 7));  // false (both rooms occupied during 3-5)
console.log(hotel.book(6, 9));  // true (room 2 free from 6)
```

---

## Q79. Car Pool â€” Can We Pick Up All Passengers?

### Problem Statement

A car has `capacity` seats. Given trips `[numPassengers, from, to]`, determine if all passengers can be transported simultaneously without exceeding capacity.

- **Pattern:** Difference array / sweep line | **Difficulty:** Medium | **LeetCode:** 1094 | **Asked at:** Ola, Uber

---

### TypeScript Solution

```typescript
function canCarPool(capacity: number, trips: [number, number, number][]): boolean {
  const stops: number[] = new Array(1001).fill(0); // max stop = 1000

  for (const [num, from, to] of trips) {
    stops[from] += num;
    stops[to]   -= num;
  }

  let current = 0;
  for (const delta of stops) {
    current += delta;
    if (current > capacity) return false;
  }
  return true;
}

// Demo
console.log(canCarPool(4, [[2,1,5],[3,3,7]])); // false (at stop 3: 2+3=5 > 4)
console.log(canCarPool(5, [[2,1,5],[3,3,7]])); // true
```

### Key Insight: Difference array is O(n + maxStop) â€” perfect for bounded stop indices.

---

## Pattern 8 â€” Summary

| Q                      | Technique         | Key Insight                               |
| ---------------------- | ----------------- | ----------------------------------------- |
| Q73 Calendar Book      | Linear scan / BST | Overlap: start<e && end>s                 |
| Q74 Max Overlap        | Sweep line events | +1 at start, -1 at end, scan max          |
| Q75 Merge Intervals    | Sort + merge      | Extend last interval if start <= last.end |
| Q76 Free Time          | Merge + gap scan  | Gap between merged busy blocks            |
| Q77 Contiguous Seats   | Gap counting      | Sliding window of empty seats             |
| Q78 Hotel Availability | Sweep on overlap  | Events only from overlapping bookings     |
| Q79 Car Pool           | Difference array  | +n at from, -n at to, scan                |

---

# PATTERN 9 â€” Binary Search

> **When you hear:** "find in sorted", "smallest/largest that satisfies", "minimize the maximum", "first bad"
> **Reach for:** Binary search on value range (answer space) or on a sorted structure.

---

## Q80. Typeahead Suggestion from Sorted Words

### Problem Statement

Given a sorted dictionary and a prefix, find all words with that prefix using binary search.

- **Pattern:** Binary search for left/right bounds | **Difficulty:** Medium | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class SortedTypeahead {
  private readonly words: string[];
  constructor(words: string[]) { this.words = [...words].sort(); }

  searchPrefix(prefix: string): string[] {
    const lo = this.lowerBound(prefix);
    const hi = this.upperBound(prefix);
    return this.words.slice(lo, hi);
  }

  private lowerBound(prefix: string): number {
    let lo = 0, hi = this.words.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.words[mid] < prefix) lo = mid + 1;
      else hi = mid;
    }
    return lo;
  }

  private upperBound(prefix: string): number {
    // Upper bound: prefix + char after 'z' = prefix + '{'
    const upper = prefix.slice(0, -1) + String.fromCharCode(prefix.charCodeAt(prefix.length-1)+1);
    let lo = 0, hi = this.words.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.words[mid] < upper) lo = mid + 1;
      else hi = mid;
    }
    return lo;
  }
}

// Demo
const st = new SortedTypeahead(["apple","application","apply","apt","banana"]);
console.log(st.searchPrefix("app")); // ["apple","application","apply"]
```

### Tradeoffs

- **Trie** is better for large dynamic dictionaries. Binary search requires pre-sorted array â€” ideal for static datasets.

---

## Q81. First Bad Commit (First True in Sorted Boolean)

### Problem Statement

Given n commits labeled `1..n`, and an API `isBad(n)` that returns `true` for all commits after the first bad one, find the first bad commit using minimum API calls.

- **Pattern:** Binary search on sorted space | **Difficulty:** Easy | **LeetCode:** 278 | **Asked at:** Amazon

---

### TypeScript Solution

```typescript
class FirstBadVersion {
  private readonly badFrom: number;
  constructor(badFrom: number) { this.badFrom = badFrom; }

  private isBadVersion(version: number): boolean {
    return version >= this.badFrom;
  }

  findFirstBad(n: number): number {
    let lo = 1, hi = n;
    while (lo < hi) {
      const mid = lo + ((hi - lo) >> 1);
      if (this.isBadVersion(mid)) hi = mid;
      else lo = mid + 1;
    }
    return lo;
  }
}

// Demo
const fbv = new FirstBadVersion(4);
console.log(fbv.findFirstBad(5)); // 4
```

### Key Insight: When `isBad(mid)` is true, mid could be the answer â†’ `hi = mid` (not mid-1). Classic left-bound binary search.

---

## Q82. Consistent Hashing Ring

### Problem Statement

Implement consistent hashing: distribute keys across N servers. When a server is added/removed, only K/N keys are remapped (not all K keys).

**Real scenario:** Redis Cluster, Cassandra, and CDN edge node selection use consistent hashing.

- **Pattern:** Sorted ring (sorted array) + binary search | **Difficulty:** Hard | **Asked at:** Flipkart, Amazon

---

### TypeScript Solution

```typescript
class ConsistentHashRing {
  private readonly ring: [number, string][] = []; // [hash, server]
  private readonly VIRTUAL_NODES: number;

  constructor(virtualNodes: number = 100) {
    this.VIRTUAL_NODES = virtualNodes;
  }

  private hash(key: string): number {
    let h = 0;
    for (let i = 0; i < key.length; i++) h = (h * 31 + key.charCodeAt(i)) >>> 0;
    return h % (2 ** 32);
  }

  addServer(server: string): void {
    for (let i = 0; i < this.VIRTUAL_NODES; i++) {
      const h = this.hash(`${server}#${i}`);
      const idx = this.ring.findIndex(([rh]) => rh > h);
      if (idx === -1) this.ring.push([h, server]);
      else this.ring.splice(idx, 0, [h, server]);
    }
  }

  removeServer(server: string): void {
    for (let i = this.ring.length - 1; i >= 0; i--) {
      if (this.ring[i][1] === server) this.ring.splice(i, 1);
    }
  }

  getServer(key: string): string | null {
    if (!this.ring.length) return null;
    const h = this.hash(key);
    // Find first ring entry >= h (clockwise)
    let lo = 0, hi = this.ring.length - 1;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.ring[mid][0] < h) lo = mid + 1;
      else hi = mid;
    }
    // Wrap around
    const idx = this.ring[lo][0] >= h ? lo : 0;
    return this.ring[idx][1];
  }
}

// Demo
const chr = new ConsistentHashRing(3);
chr.addServer("server1"); chr.addServer("server2"); chr.addServer("server3");
console.log(chr.getServer("user_rahul"));  // consistently maps to one server
console.log(chr.getServer("order_12345")); // maps to a server
chr.removeServer("server2");
console.log(chr.getServer("user_rahul")); // may or may not change â€” if it does, remaps to next server
```

### Tradeoffs

- **Virtual nodes** (100-200 per server) improve load distribution by creating multiple ring positions per server.
- **Key remapping on change:** Only keys between the removed node and its predecessor move. Expected K/N keys moved.

---

## Q83. Kth Rank / Percentile

### Problem Statement

Design a data structure that supports `addScore(score)` and `getRank(score)` â€” rank = number of scores > given score + 1.

- **Pattern:** Sorted array + binary search | **Difficulty:** Medium | **Asked at:** Dream11

---

### TypeScript Solution

```typescript
class RankCalculator {
  private readonly scores: number[] = [];

  addScore(score: number): void {
    // Insert in sorted order (binary search for position)
    let lo = 0, hi = this.scores.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.scores[mid] <= score) lo = mid + 1;
      else hi = mid;
    }
    this.scores.splice(lo, 0, score);
  }

  getRank(score: number): number {
    // Count scores strictly greater than this score
    let lo = 0, hi = this.scores.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.scores[mid] <= score) lo = mid + 1;
      else hi = mid;
    }
    return this.scores.length - lo + 1;
  }

  getPercentile(score: number): number {
    let lo = 0, hi = this.scores.length;
    while (lo < hi) {
      const mid = (lo + hi) >> 1;
      if (this.scores[mid] < score) lo = mid + 1;
      else hi = mid;
    }
    return Math.round((lo / this.scores.length) * 100);
  }
}

// Demo
const rc = new RankCalculator();
[50, 75, 90, 85, 60].forEach(s => rc.addScore(s));
console.log(rc.getRank(75));       // 3 (90, 85 are above)
console.log(rc.getPercentile(75)); // 40th percentile
```

---

## Q84. Range Sum with Point Updates (Fenwick Tree / BIT)

### Problem Statement

Design a data structure for `update(index, val)` and `rangeSum(left, right)` â€” both in O(log n).

**Real scenario:** Cricket scoreboards with live score updates, financial time-series with rolling sums.

- **Pattern:** Binary Indexed Tree (Fenwick Tree / BIT) | **Difficulty:** Medium | **LeetCode:** 307 | **Asked at:** Google, Zerodha

---

### TypeScript Solution

```typescript
class BinaryIndexedTree {
  private readonly tree: number[];
  private readonly n: number;

  constructor(nums: number[]) {
    this.n = nums.length;
    this.tree = new Array(this.n + 1).fill(0);
    for (let i = 0; i < this.n; i++) this.update(i, nums[i]);
  }

  update(index: number, delta: number): void {
    for (let i = index + 1; i <= this.n; i += i & (-i)) {
      this.tree[i] += delta;
    }
  }

  prefixSum(index: number): number {
    let sum = 0;
    for (let i = index + 1; i > 0; i -= i & (-i)) {
      sum += this.tree[i];
    }
    return sum;
  }

  rangeSum(left: number, right: number): number {
    return this.prefixSum(right) - (left > 0 ? this.prefixSum(left - 1) : 0);
  }
}

class NumArray {
  private readonly bit: BinaryIndexedTree;
  private readonly nums: number[];

  constructor(nums: number[]) {
    this.nums = [...nums];
    this.bit = new BinaryIndexedTree(nums);
  }

  update(index: number, val: number): void {
    this.bit.update(index, val - this.nums[index]);
    this.nums[index] = val;
  }

  sumRange(left: number, right: number): number {
    return this.bit.rangeSum(left, right);
  }
}

// Demo
const na = new NumArray([1,3,5]);
console.log(na.sumRange(0,2)); // 9
na.update(1, 2);
console.log(na.sumRange(0,2)); // 8
```

### Complexity | update: O(log n) | rangeSum: O(log n) | Space: O(n)

### Tradeoffs

| Structure             | Build      | Query    | Update   |
| --------------------- | ---------- | -------- | -------- |
| Prefix sum array      | O(n)       | O(1)     | O(n)     |
| **BIT/Fenwick** | O(n log n) | O(log n) | O(log n) |
| Segment tree          | O(n)       | O(log n) | O(log n) |

Segment tree supports range updates; BIT only supports point updates + range queries natively.

---

## Pattern 9 â€” Summary

| Q                    | Technique               | Key Insight                                 |
| -------------------- | ----------------------- | ------------------------------------------- |
| Q80 Typeahead        | Binary search bounds    | Lower/upper bound on sorted words           |
| Q81 First Bad Commit | Binary search           | hi=mid when condition true (left bound)     |
| Q82 Consistent Hash  | Sorted ring + BSearch   | Virtual nodes for even distribution         |
| Q83 Rank/Percentile  | Sorted insert + BSearch | Count elements > score                      |
| Q84 Range Sum        | BIT/Fenwick Tree        | i += i&(-i) to update; i -= i&(-i) to query |

---

# PATTERN 10 â€” Backtracking / Matrix

> **When you hear:** "all combinations", "permutations", "find all solutions", "matrix rotation", "grid simulation"
> **Reach for:** Backtracking (try-recurse-undo) for combinatorial search; in-place techniques for matrix transforms.

---

## Q85. Sudoku Solver

### Problem Statement

Solve a 9Ã—9 Sudoku puzzle. Fill in the empty cells (marked `'.'`) so each row, column, and 3Ã—3 box contains digits 1-9 exactly once.

- **Pattern:** Backtracking | **Difficulty:** Hard | **LeetCode:** 37 | **Asked at:** Google, Amazon

---

### TypeScript Solution

```typescript
class SudokuSolver {
  solve(board: string[][]): boolean {
    for (let r = 0; r < 9; r++) {
      for (let c = 0; c < 9; c++) {
        if (board[r][c] !== ".") continue;
        for (let d = 1; d <= 9; d++) {
          const ch = String(d);
          if (this.isValid(board, r, c, ch)) {
            board[r][c] = ch;
            if (this.solve(board)) return true;
            board[r][c] = "."; // backtrack
          }
        }
        return false; // no digit worked
      }
    }
    return true; // all cells filled
  }

  private isValid(board: string[][], row: number, col: number, ch: string): boolean {
    for (let i = 0; i < 9; i++) {
      if (board[row][i] === ch) return false;
      if (board[i][col] === ch) return false;
      const boxRow = 3 * Math.floor(row / 3) + Math.floor(i / 3);
      const boxCol = 3 * Math.floor(col / 3) + (i % 3);
      if (board[boxRow][boxCol] === ch) return false;
    }
    return true;
  }
}

// Demo
const board = [
  ["5","3",".",".","7",".",".",".","."],
  ["6",".",".","1","9","5",".",".","."],
  [".","9","8",".",".",".",".","6","."],
  ["8",".",".",".","6",".",".",".","3"],
  ["4",".",".","8",".","3",".",".","1"],
  ["7",".",".",".","2",".",".",".","6"],
  [".","6",".",".",".",".","2","8","."],
  [".",".",".","4","1","9",".",".","5"],
  [".",".",".",".","8",".",".","7","9"]
];
const solver = new SudokuSolver();
solver.solve(board);
console.log(board[0]); // ["5","3","4","6","7","8","9","1","2"]
```

### Complexity | Time: O(9^(empty cells)) worst case | Space: O(81) recursion depth

### Tradeoffs

- **Naive backtracking** with isValid check is O(9^81) worst case but prunes heavily in practice.
- **Constraint propagation (Arc Consistency):** Maintain sets of possible values per cell â€” reduces search space dramatically. Used in production Sudoku solvers.
- **Dancing Links (Algorithm X):** Exact Cover formulation â€” the fastest known exact Sudoku solver.

---

## Q86. Word Search / Boggle

### Problem Statement

Given an mÃ—n grid and a word, determine if the word exists in the grid by traversing adjacent cells (horizontal/vertical/diagonal). Each cell can only be used once.

- **Pattern:** DFS + backtracking on grid | **Difficulty:** Medium | **LeetCode:** 79 | **Asked at:** Amazon, Flipkart

---

### TypeScript Solution

```typescript
class WordSearch {
  exist(board: string[][], word: string): boolean {
    const rows = board.length, cols = board[0].length;

    const dfs = (r: number, c: number, idx: number): boolean => {
      if (idx === word.length) return true;
      if (r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] !== word[idx]) return false;

      const temp = board[r][c];
      board[r][c] = "#"; // mark visited

      const found =
        dfs(r+1,c,idx+1) || dfs(r-1,c,idx+1) ||
        dfs(r,c+1,idx+1) || dfs(r,c-1,idx+1);

      board[r][c] = temp; // restore (backtrack)
      return found;
    };

    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        if (dfs(r, c, 0)) return true;
      }
    }
    return false;
  }

  // Boggle: find all valid words from a Trie in the grid
  findAllWords(board: string[][], trie: TrieNode): string[] {
    const rows = board.length, cols = board[0].length;
    const results: string[] = [];

    const dfs = (r: number, c: number, node: TrieNode, path: string): void => {
      if (r < 0 || r >= rows || c < 0 || c >= cols || board[r][c] === "#") return;
      const ch = board[r][c];
      if (!node.children.has(ch)) return;

      const nextNode = node.children.get(ch)!;
      const newPath = path + ch;
      if (nextNode.isEnd) results.push(newPath);

      const temp = board[r][c];
      board[r][c] = "#";
      for (const [dr, dc] of [[-1,0],[1,0],[0,-1],[0,1],[-1,-1],[-1,1],[1,-1],[1,1]]) {
        dfs(r+dr, c+dc, nextNode, newPath);
      }
      board[r][c] = temp;
    };

    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        dfs(r, c, trie, "");
      }
    }
    return [...new Set(results)];
  }
}
```

### Tradeoffs

- **Mark cell with `#`** avoids a separate `visited` array â€” saves O(m*n) space.
- **Boggle with Trie prunes early:** If no word in the dictionary starts with current path, stop DFS. Critical for performance.

---

## Q87. N-Queens

### Problem Statement

Place N queens on an NÃ—N chessboard so no two queens threaten each other. Return all valid placements.

- **Pattern:** Backtracking with column/diagonal constraints | **Difficulty:** Hard | **LeetCode:** 51 | **Asked at:** Amazon, Google

---

### TypeScript Solution

```typescript
class NQueens {
  solveNQueens(n: number): string[][] {
    const results: string[][] = [];
    const cols = new Set<number>();
    const diag1 = new Set<number>(); // row - col
    const diag2 = new Set<number>(); // row + col
    const board: number[] = []; // board[row] = col of queen

    const backtrack = (row: number): void => {
      if (row === n) {
        results.push(this.buildBoard(board, n));
        return;
      }
      for (let col = 0; col < n; col++) {
        if (cols.has(col) || diag1.has(row-col) || diag2.has(row+col)) continue;
        cols.add(col); diag1.add(row-col); diag2.add(row+col);
        board.push(col);
        backtrack(row + 1);
        board.pop();
        cols.delete(col); diag1.delete(row-col); diag2.delete(row+col);
      }
    };

    backtrack(0);
    return results;
  }

  private buildBoard(board: number[], n: number): string[] {
    return board.map(col => ".".repeat(col) + "Q" + ".".repeat(n - col - 1));
  }
}

// Demo
const nq = new NQueens();
const solutions = nq.solveNQueens(4);
console.log(solutions.length); // 2
console.log(solutions[0]);     // [".Q..","...Q","Q...","..Q."]
```

### Trace (N=4)

```
Row 0: try col 0 -> Row 1: col 0 blocked, col 1 blocked (diag), col 2 -> Row 2: ...
```

### Tradeoffs

- **3 sets** (cols, diag1, diag2) give O(1) conflict checking per placement.
- **Bitmask** approach encodes all three in integers for faster bit operations.

---

## Q88. Maze Escape â€” Shortest Path

### Problem Statement

Given a maze (2D grid with walls), find the shortest path from start `(0,0)` to end `(m-1,n-1)`.

- **Pattern:** BFS on grid | **Difficulty:** Medium | **Asked at:** Dunzo, Amazon

---

### TypeScript Solution

```typescript
class MazeEscape {
  shortestPath(maze: number[][], start: [number,number], end: [number,number]): number {
    const rows = maze.length, cols = maze[0].length;
    if (maze[start[0]][start[1]] === 1 || maze[end[0]][end[1]] === 1) return -1;

    const queue: [number, number, number][] = [[start[0], start[1], 0]];
    const visited = Array.from({ length: rows }, () => new Array(cols).fill(false));
    visited[start[0]][start[1]] = true;

    const dirs = [[-1,0],[1,0],[0,-1],[0,1]];
    while (queue.length) {
      const [r, c, steps] = queue.shift()!;
      if (r === end[0] && c === end[1]) return steps;
      for (const [dr, dc] of dirs) {
        const nr = r+dr, nc = c+dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
            !visited[nr][nc] && maze[nr][nc] === 0) {
          visited[nr][nc] = true;
          queue.push([nr, nc, steps+1]);
        }
      }
    }
    return -1;
  }
}

// Demo
const me = new MazeEscape();
const maze = [
  [0,0,0,0],
  [1,1,0,1],
  [0,0,0,0],
  [0,1,1,0]
];
console.log(me.shortestPath(maze, [0,0], [3,3])); // 7
```

---

## Q89. Rotate Image 90 Degrees

### Problem Statement

Rotate an NÃ—N matrix 90 degrees clockwise **in-place**.

- **Pattern:** Transpose + reverse rows | **Difficulty:** Medium | **LeetCode:** 48 | **Asked at:** Amazon, Adobe

---

### TypeScript Solution

```typescript
class ImageRotator {
  rotate(matrix: number[][]): void {
    const n = matrix.length;
    // Step 1: Transpose (swap matrix[i][j] with matrix[j][i])
    for (let i = 0; i < n; i++) {
      for (let j = i + 1; j < n; j++) {
        [matrix[i][j], matrix[j][i]] = [matrix[j][i], matrix[i][j]];
      }
    }
    // Step 2: Reverse each row
    for (let i = 0; i < n; i++) {
      matrix[i].reverse();
    }
  }
}

// Demo
const img = [[1,2,3],[4,5,6],[7,8,9]];
new ImageRotator().rotate(img);
console.log(img);
// [[7,4,1],[8,5,2],[9,6,3]]
```

### Trace

```
Original:     Transposed:   Reversed rows:
1 2 3         1 4 7         7 4 1
4 5 6   ->    2 5 8    ->   8 5 2
7 8 9         3 6 9         9 6 3
```

### Tradeoffs

- **Transpose + reverse** is O(n^2) time, O(1) space.
- **Counterclockwise rotation:** Transpose + reverse columns (instead of rows).

---

## Q90. Game of Life

### Problem Statement

Simulate Conway's Game of Life: given the current state (0=dead, 1=alive), compute the next state. Do it **in-place** without extra space.

- **Pattern:** Encode transitions in-place using extra states | **Difficulty:** Medium | **LeetCode:** 289 | **Asked at:** Uber, Google

---

### TypeScript Solution

```typescript
class GameOfLife {
  nextState(board: number[][]): void {
    const rows = board.length, cols = board[0].length;
    const dirs = [[-1,-1],[-1,0],[-1,1],[0,-1],[0,1],[1,-1],[1,0],[1,1]];

    // Encoding: 1=alive->alive, -1=alive->dead, 2=dead->alive, 0=dead->dead
    const getLiveNeighbors = (r: number, c: number): number => {
      let count = 0;
      for (const [dr, dc] of dirs) {
        const nr = r+dr, nc = c+dc;
        if (nr >= 0 && nr < rows && nc >= 0 && nc < cols) {
          if (Math.abs(board[nr][nc]) === 1) count++;
        }
      }
      return count;
    };

    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        const live = getLiveNeighbors(r, c);
        if (board[r][c] === 1 && (live < 2 || live > 3)) board[r][c] = -1; // alive->dead
        if (board[r][c] === 0 && live === 3) board[r][c] = 2;               // dead->alive
      }
    }

    for (let r = 0; r < rows; r++) {
      for (let c = 0; c < cols; c++) {
        if (board[r][c] === -1) board[r][c] = 0;
        if (board[r][c] ===  2) board[r][c] = 1;
      }
    }
  }
}

// Demo
const gol = new GameOfLife();
const b = [[0,1,0],[0,0,1],[1,1,1],[0,0,0]];
gol.nextState(b);
console.log(b); // [[0,0,0],[1,0,1],[0,1,1],[0,1,0]]
```

### Key Insight: Use encoded values (`-1`, `2`) to represent transitions so the pass-1 reads original state and pass-2 finalises.

---

## Pattern 10 â€” Summary

| Q                | Algorithm             | Key Insight                       |
| ---------------- | --------------------- | --------------------------------- |
| Q85 Sudoku       | Backtracking          | isValid O(27); undo on fail       |
| Q86 Word Search  | DFS + mark '#'        | Restore cell on backtrack         |
| Q87 N-Queens     | Backtracking + 3 sets | O(1) conflict check via diag sets |
| Q88 Maze Escape  | BFS                   | Shortest = minimum steps BFS      |
| Q89 Rotate Image | Transpose + reverse   | In-place O(1) space               |
| Q90 Game of Life | Encoded states        | -1 = alive->dead, 2 = dead->alive |

---

# PATTERN 11 â€” Dynamic Programming / Greedy

> **When you hear:** "minimum cost", "maximum profit", "optimal substructure", "overlapping subproblems"
> **Reach for:** DP table (bottom-up) or memoisation. For greedy, prove the greedy choice property first.

---

## Q91. Text Justification

### Problem Statement

Given words and a page width, format them so each line (except the last) has exactly `width` characters with spaces distributed as evenly as possible between words.

- **Pattern:** Greedy (pack words) + space distribution | **Difficulty:** Hard | **LeetCode:** 68 | **Asked at:** Google, Notion

---

### TypeScript Solution

```typescript
class TextJustifier {
  fullJustify(words: string[], maxWidth: number): string[] {
    const lines: string[][] = [];
    let i = 0;

    // Greedy: pack as many words as possible into each line
    while (i < words.length) {
      let lineLen = words[i].length;
      let j = i + 1;
      while (j < words.length && lineLen + 1 + words[j].length <= maxWidth) {
        lineLen += 1 + words[j].length;
        j++;
      }
      lines.push(words.slice(i, j));
      i = j;
    }

    return lines.map((lineWords, idx) => {
      const isLast = idx === lines.length - 1;
      if (lineWords.length === 1 || isLast) {
        return lineWords.join(" ").padEnd(maxWidth, " ");
      }
      const totalChars = lineWords.reduce((s, w) => s + w.length, 0);
      const totalSpaces = maxWidth - totalChars;
      const gaps = lineWords.length - 1;
      const baseSpace = Math.floor(totalSpaces / gaps);
      const extraSpaces = totalSpaces % gaps;

      let result = lineWords[0];
      for (let k = 1; k < lineWords.length; k++) {
        const spaces = baseSpace + (k <= extraSpaces ? 1 : 0);
        result += " ".repeat(spaces) + lineWords[k];
      }
      return result;
    });
  }
}

// Demo
const tj = new TextJustifier();
console.log(tj.fullJustify(["This","is","an","example","of","text","justification"], 16));
// ["This    is    an","example  of text","justification   "]
```

### Tradeoffs

- **Greedy pack:** Always fill line to max â€” optimal for minimizing number of lines.
- **DP approach:** Minimizes total "badness" (squared empty spaces) â€” gives better visual balance. LaTeX uses this.

---

## Q92. Coin Change

### Problem Statement

Given coin denominations and an amount, find the **minimum number of coins** to make that amount. Return -1 if impossible.

**Real scenario:** Paytm Wallet/PhonePe returns exact change. ATMs dispense minimum bills.

- **Pattern:** DP (bottom-up, 1D) | **Difficulty:** Medium | **LeetCode:** 322 | **Asked at:** Amazon, Paytm

---

### TypeScript Solution

```typescript
class CoinChanger {
  minCoins(coins: number[], amount: number): number {
    const dp = new Array(amount + 1).fill(Infinity);
    dp[0] = 0;

    for (let amt = 1; amt <= amount; amt++) {
      for (const coin of coins) {
        if (coin <= amt && dp[amt - coin] !== Infinity) {
          dp[amt] = Math.min(dp[amt], dp[amt - coin] + 1);
        }
      }
    }
    return dp[amount] === Infinity ? -1 : dp[amount];
  }

  // Return the actual coins used
  minCoinsWithTrace(coins: number[], amount: number): number[] | null {
    const dp = new Array(amount + 1).fill(Infinity);
    const from = new Array(amount + 1).fill(-1);
    dp[0] = 0;

    for (let amt = 1; amt <= amount; amt++) {
      for (const coin of coins) {
        if (coin <= amt && dp[amt - coin] + 1 < dp[amt]) {
          dp[amt] = dp[amt - coin] + 1;
          from[amt] = coin;
        }
      }
    }
    if (dp[amount] === Infinity) return null;

    const result: number[] = [];
    let cur = amount;
    while (cur > 0) { result.push(from[cur]); cur -= from[cur]; }
    return result;
  }
}

// Demo
const cc = new CoinChanger();
console.log(cc.minCoins([1,5,6,9], 11)); // 2 (two 5s + one 1? No: 6+5=11, 2 coins)
console.log(cc.minCoinsWithTrace([1,5,6,9], 11)); // [6,5]
```

### Complexity | Time: O(amount * coins) | Space: O(amount)

### Tradeoffs

- **Greedy (largest coin first)** is wrong for arbitrary denominations. Only works for canonical systems like US coins.
- **DP** is correct for any denominations. O(amount * k) where k = number of coin types.

---

## Q93. Edit Distance

### Problem Statement

Find the minimum operations (insert, delete, replace) to transform string `word1` into `word2`.

**Real scenario:** Git diff, spell-checkers, DNA sequence alignment.

- **Pattern:** 2D DP | **Difficulty:** Hard | **LeetCode:** 72 | **Asked at:** Google, Amazon

---

### TypeScript Solution

```typescript
class EditDistanceCalculator {
  minDistance(word1: string, word2: string): number {
    const m = word1.length, n = word2.length;
    // dp[i][j] = min edits to convert word1[0..i-1] to word2[0..j-1]
    const dp: number[][] = Array.from({ length: m+1 }, (_, i) =>
      Array.from({ length: n+1 }, (_, j) => i === 0 ? j : j === 0 ? i : 0)
    );

    for (let i = 1; i <= m; i++) {
      for (let j = 1; j <= n; j++) {
        if (word1[i-1] === word2[j-1]) {
          dp[i][j] = dp[i-1][j-1];
        } else {
          dp[i][j] = 1 + Math.min(
            dp[i-1][j],   // delete from word1
            dp[i][j-1],   // insert into word1
            dp[i-1][j-1]  // replace
          );
        }
      }
    }
    return dp[m][n];
  }
}

// Demo
const ed = new EditDistanceCalculator();
console.log(ed.minDistance("horse", "ros")); // 3
console.log(ed.minDistance("intention", "execution")); // 5
```

### Trace

```
    ""  r  o  s
""   0  1  2  3
h    1  1  2  3
o    2  2  1  2
r    3  2  2  2
s    4  3  3  2
e    5  4  4  3
```

### Tradeoffs

- **Space optimization:** Only need the previous row â†’ reduce from O(mn) to O(n) space.
- **LCS approach:** Edit distance = m + n - 2*LCS(word1, word2).

---

## Q94. Best Discount Combination (0/1 Knapsack)

### Problem Statement

Given items with prices and discounts, and a budget, find the maximum total discount achievable without exceeding the budget.

**Real scenario:** Amazon's "Coupon combination" maximization. Meesho cashback maximization.

- **Pattern:** 0/1 Knapsack DP | **Difficulty:** Medium | **Asked at:** Amazon, Meesho

---

### TypeScript Solution

```typescript
interface Item { name: string; price: number; discount: number; }

class DiscountOptimizer {
  maxDiscount(items: Item[], budget: number): { discount: number; selected: string[] } {
    const n = items.length;
    // dp[i][b] = max discount using first i items with budget b
    const dp: number[][] = Array.from({ length: n+1 }, () => new Array(budget+1).fill(0));

    for (let i = 1; i <= n; i++) {
      const { price, discount } = items[i-1];
      for (let b = 0; b <= budget; b++) {
        dp[i][b] = dp[i-1][b]; // don't take item i
        if (price <= b) {
          dp[i][b] = Math.max(dp[i][b], dp[i-1][b-price] + discount);
        }
      }
    }

    // Traceback
    const selected: string[] = [];
    let b = budget;
    for (let i = n; i > 0; i--) {
      if (dp[i][b] !== dp[i-1][b]) {
        selected.push(items[i-1].name);
        b -= items[i-1].price;
      }
    }

    return { discount: dp[n][budget], selected };
  }
}

// Demo
const do2 = new DiscountOptimizer();
const items: Item[] = [
  { name: "iPhone case", price: 200, discount: 50 },
  { name: "Earphones",   price: 500, discount: 150 },
  { name: "Charger",     price: 300, discount: 80 },
  { name: "Power bank",  price: 600, discount: 200 },
];
console.log(do2.maxDiscount(items, 800));
// {discount:280, selected:["Charger","Earphones"]}
```

### Tradeoffs

- **Space optimised:** Use 1D dp array, iterate budget from high to low â€” O(budget) space.
- **Greedy (best discount/price ratio):** Not optimal for 0/1 knapsack. Only optimal for fractional knapsack.

---

## Q95. Wildcard Pattern Matcher

### Problem Statement

Implement wildcard matching where `?` matches exactly one character and `*` matches any sequence (including empty).

- **Pattern:** 2D DP | **Difficulty:** Hard | **LeetCode:** 44 | **Asked at:** Amazon, Google

---

### TypeScript Solution

```typescript
class WildcardMatcher {
  isMatch(s: string, p: string): boolean {
    const m = s.length, n = p.length;
    // dp[i][j] = p[0..j-1] matches s[0..i-1]
    const dp: boolean[][] = Array.from({ length: m+1 }, () => new Array(n+1).fill(false));
    dp[0][0] = true;

    // Pattern of stars can match empty string
    for (let j = 1; j <= n; j++) {
      if (p[j-1] === "*") dp[0][j] = dp[0][j-1];
    }

    for (let i = 1; i <= m; i++) {
      for (let j = 1; j <= n; j++) {
        if (p[j-1] === "*") {
          dp[i][j] = dp[i-1][j] || dp[i][j-1];
          // dp[i-1][j]: * matches one more char; dp[i][j-1]: * matches nothing
        } else if (p[j-1] === "?" || p[j-1] === s[i-1]) {
          dp[i][j] = dp[i-1][j-1];
        }
      }
    }
    return dp[m][n];
  }
}

// Demo
const wm = new WildcardMatcher();
console.log(wm.isMatch("aa", "a"));    // false
console.log(wm.isMatch("aa", "*"));    // true
console.log(wm.isMatch("cb", "?a"));   // false
console.log(wm.isMatch("adceb","*a*b")); // true
```

### Tradeoffs

- **Two-pointer greedy:** O(n) time but complex. DP is clearer for interviews.
- **Regex vs Wildcard:** Regex `.*` is equivalent to `*` here. Regex `a*` (zero or more `a`) is different from wildcard.

---

## Pattern 11 â€” Summary

| Q                 | Algorithm                  | Key Insight                                           |
| ----------------- | -------------------------- | ----------------------------------------------------- |
| Q91 Text Justify  | Greedy pack + even spacing | DP for "badness" minimization                         |
| Q92 Coin Change   | 1D DP                      | dp[amt] = min(dp[amt-coin]+1)                         |
| Q93 Edit Distance | 2D DP                      | 3 choices: insert, delete, replace                    |
| Q94 Knapsack      | 2D DP                      | dp[i][b] = max(skip, take)                            |
| Q95 Wildcard      | 2D DP                      | `*` matches empty (dp[i][j-1]) or one+ (dp[i-1][j]) |

---

# PATTERN 12 â€” Miscellaneous Design Patterns

> Event-driven systems, pub-sub, connection pooling, probabilistic structures, resilience patterns.

---

## Q96. Event Simulator (Priority Queue + Callbacks)

### Problem Statement

Design an event simulator: `schedule(delayMs, callback)` schedules a callback to run after `delayMs`. `tick(deltaMs)` advances time and fires all due callbacks in order.

**Real scenario:** Game engines process game events in timeline order. Simulation frameworks.

- **Pattern:** Min-Heap by event time | **Difficulty:** Medium | **Asked at:** Simulation system design

---

### TypeScript Solution

```typescript
interface SimulatorEvent { id: number; fireAt: number; callback: () => void; }

class EventSimulator {
  private readonly pq: MinHeap<SimulatorEvent>;
  private currentTime: number = 0;
  private eventCounter: number = 0;

  constructor() {
    this.pq = new MinHeap<SimulatorEvent>((a, b) => a.fireAt - b.fireAt);
  }

  schedule(delayMs: number, callback: () => void): number {
    const id = ++this.eventCounter;
    this.pq.push({ id, fireAt: this.currentTime + delayMs, callback });
    return id;
  }

  tick(deltaMs: number): void {
    this.currentTime += deltaMs;
    while (this.pq.peek() && this.pq.peek()!.fireAt <= this.currentTime) {
      const event = this.pq.pop()!;
      event.callback();
    }
  }

  getTime(): number { return this.currentTime; }
}

// Demo
const sim = new EventSimulator();
sim.schedule(100, () => console.log("Event A at 100ms"));
sim.schedule(50,  () => console.log("Event B at 50ms"));
sim.schedule(150, () => console.log("Event C at 150ms"));

sim.tick(60);  // fires B
sim.tick(60);  // fires A
sim.tick(60);  // fires C
```

### Tradeoffs

- **Deterministic simulation:** tick() is synchronous â€” perfect for testing and reproducibility.
- **Real timers (Node.js setTimeout):** Non-deterministic, affected by event loop lag. Use simulator for unit-testable game/simulation logic.

---

## Q97. Pub-Sub Message Bus

### Problem Statement

Design a publish-subscribe system: `subscribe(topic, handler)`, `unsubscribe(topic, handler)`, `publish(topic, message)` â€” all O(1) average.

**Real scenario:** Kafka consumers, Redis pub/sub, frontend event emitters (EventEmitter in Node.js).

- **Pattern:** HashMap of topic -> Set of handlers | **Difficulty:** Medium | **Asked at:** Uber, Swiggy

---

### TypeScript Solution

```typescript
type MessageHandler<T> = (message: T) => void;

interface IPubSub<T> {
  subscribe(topic: string, handler: MessageHandler<T>): void;
  unsubscribe(topic: string, handler: MessageHandler<T>): void;
  publish(topic: string, message: T): void;
}

class PubSubBus<T> implements IPubSub<T> {
  private readonly subscribers: Map<string, Set<MessageHandler<T>>> = new Map();

  subscribe(topic: string, handler: MessageHandler<T>): void {
    if (!this.subscribers.has(topic)) this.subscribers.set(topic, new Set());
    this.subscribers.get(topic)!.add(handler);
  }

  unsubscribe(topic: string, handler: MessageHandler<T>): void {
    this.subscribers.get(topic)?.delete(handler);
  }

  publish(topic: string, message: T): void {
    for (const handler of (this.subscribers.get(topic) ?? [])) {
      try { handler(message); }
      catch (e) { console.error(`Handler error on topic ${topic}:`, e); }
    }
  }

  getSubscriberCount(topic: string): number {
    return this.subscribers.get(topic)?.size ?? 0;
  }
}

// Demo
const bus = new PubSubBus<{ orderId: string; status: string }>();

const paymentHandler = (msg: any) => console.log(`Payment svc: order ${msg.orderId} -> ${msg.status}`);
const inventoryHandler = (msg: any) => console.log(`Inventory svc: update for ${msg.orderId}`);

bus.subscribe("order.placed", paymentHandler);
bus.subscribe("order.placed", inventoryHandler);
bus.publish("order.placed", { orderId: "ORD_001", status: "PLACED" });
// Both handlers fire

bus.unsubscribe("order.placed", paymentHandler);
bus.publish("order.placed", { orderId: "ORD_002", status: "PLACED" });
// Only inventoryHandler fires
```

### Tradeoffs

| Feature                | This (in-memory) | Kafka         | Redis Pub/Sub |
| ---------------------- | ---------------- | ------------- | ------------- |
| Persistence            | No               | Yes           | No            |
| Message ordering       | Yes (sync)       | Per-partition | No            |
| At-least-once delivery | No               | Yes           | No            |
| Replay                 | No               | Yes           | No            |

---

## Q98. Connection Pool

### Problem Statement

Design a database connection pool: `acquire()` gets a connection (blocks/waits if none available), `release(conn)` returns it to the pool.

**Real scenario:** HikariCP (Java), pgbouncer (PostgreSQL) â€” every backend service uses a connection pool.

- **Pattern:** Object pool with idle queue | **Difficulty:** Medium | **Asked at:** Backend system design

---

### TypeScript Solution

```typescript
interface DBConnection { id: number; execute(query: string): Promise<string>; }

class ConnectionPool {
  private readonly idle: DBConnection[] = [];
  private readonly waiters: ((conn: DBConnection) => void)[] = [];
  private readonly allConnections: Set<number> = new Set();
  private connCounter: number = 0;
  private readonly maxSize: number;

  constructor(maxSize: number) {
    this.maxSize = maxSize;
    // Pre-create connections
    for (let i = 0; i < maxSize; i++) this.createConnection();
  }

  private createConnection(): DBConnection {
    const id = ++this.connCounter;
    const conn: DBConnection = {
      id,
      execute: async (query: string) => `[conn:${id}] result of "${query}"`
    };
    this.idle.push(conn);
    this.allConnections.add(id);
    return conn;
  }

  async acquire(timeoutMs: number = 5000): Promise<DBConnection> {
    if (this.idle.length > 0) return this.idle.pop()!;

    // Wait for a connection to be released
    return new Promise<DBConnection>((resolve, reject) => {
      const timer = setTimeout(() => reject(new Error("Connection pool timeout")), timeoutMs);
      this.waiters.push(conn => { clearTimeout(timer); resolve(conn); });
    });
  }

  release(conn: DBConnection): void {
    if (!this.allConnections.has(conn.id)) return; // foreign connection

    if (this.waiters.length > 0) {
      // Hand off directly to waiting acquirer
      const waiter = this.waiters.shift()!;
      waiter(conn);
    } else {
      this.idle.push(conn);
    }
  }

  stats() {
    return { total: this.allConnections.size, idle: this.idle.length, waiting: this.waiters.length };
  }
}

// Demo
async function demo() {
  const pool = new ConnectionPool(2);
  const c1 = await pool.acquire();
  const c2 = await pool.acquire();
  console.log(pool.stats()); // {total:2, idle:0, waiting:0}

  const c3Promise = pool.acquire(1000); // will wait
  console.log(pool.stats()); // {total:2, idle:0, waiting:1}

  pool.release(c1);
  const c3 = await c3Promise; // gets c1's connection
  console.log(`Got connection: ${c3.id}`);
  console.log(await c3.execute("SELECT 1")); // [conn:1] result of "SELECT 1"
}
demo();
```

### Tradeoffs

- **Pre-allocation (eager):** Connections created upfront â€” no latency on first request but wastes resources if pool is idle.
- **Lazy allocation:** Create connections on demand up to maxSize â€” better resource utilisation.
- **Health checking:** Production pools (HikariCP) ping idle connections to detect stale ones and replace them.

---

## Q99. Bloom Filter

### Problem Statement

Design a space-efficient probabilistic data structure that answers "is this element in the set?" with no false negatives but possible false positives.

**Real scenario:** Chrome Safe Browsing checks URLs against a Bloom filter before querying the server. Cassandra uses Bloom filters to avoid unnecessary disk reads.

- **Pattern:** Bit array + multiple hash functions | **Difficulty:** Medium | **Asked at:** Flipkart, Databricks

---

### TypeScript Solution

```typescript
class BloomFilter {
  private readonly bits: Uint8Array;
  private readonly size: number;
  private readonly numHashes: number;

  constructor(sizeBits: number = 1024, numHashes: number = 3) {
    this.size = sizeBits;
    this.numHashes = numHashes;
    this.bits = new Uint8Array(Math.ceil(sizeBits / 8));
  }

  private hash(item: string, seed: number): number {
    let h = seed;
    for (let i = 0; i < item.length; i++) {
      h = (Math.imul(h, 31) + item.charCodeAt(i)) >>> 0;
    }
    return h % this.size;
  }

  private setBit(pos: number): void {
    this.bits[pos >> 3] |= (1 << (pos & 7));
  }

  private getBit(pos: number): boolean {
    return (this.bits[pos >> 3] & (1 << (pos & 7))) !== 0;
  }

  add(item: string): void {
    for (let i = 0; i < this.numHashes; i++) {
      this.setBit(this.hash(item, i * 0xdeadbeef));
    }
  }

  mightContain(item: string): boolean {
    for (let i = 0; i < this.numHashes; i++) {
      if (!this.getBit(this.hash(item, i * 0xdeadbeef))) return false;
    }
    return true; // might be false positive
  }
}

// Demo
const bf = new BloomFilter(2048, 3);
["rahul@example.com","priya@example.com","spam.com"].forEach(s => bf.add(s));
console.log(bf.mightContain("rahul@example.com")); // true (definitely in set)
console.log(bf.mightContain("priya@example.com")); // true
console.log(bf.mightContain("unknown@test.com"));  // false (likely, not guaranteed)
```

### Complexity | add/contains: O(k) where k = numHashes | Space: O(m bits)

### Tradeoffs

- **False positive rate:** ~(1 - e^(-kn/m))^k where n=elements, m=bits, k=hashes. Set m = -n*ln(FPR)/(ln2)^2.
- **No deletion:** Standard Bloom filter can't delete. **Counting Bloom Filter** replaces bits with counters.
- **Alternatives:** Cuckoo filter â€” same space, supports deletion, faster in practice.

---

## Q100. Circuit Breaker (State Machine)

### Problem Statement

Design a Circuit Breaker that wraps service calls. After N consecutive failures, it OPENS (blocks calls for a cooldown period). After cooldown, it goes HALF-OPEN (tries one call). If successful, CLOSES again; if not, OPENS again.

**Real scenario:** Hystrix (Netflix), Resilience4j â€” every microservice architecture uses circuit breakers to prevent cascade failures.

- **Pattern:** State machine (CLOSED -> OPEN -> HALF_OPEN -> CLOSED) | **Difficulty:** Hard | **Asked at:** Uber, Amazon, PayPal

---

### TypeScript Solution

```typescript
type CBState = "CLOSED" | "OPEN" | "HALF_OPEN";

interface CircuitBreakerConfig {
  failureThreshold: number;  // how many failures before opening
  successThreshold: number;  // how many successes in HALF_OPEN to close
  cooldownMs: number;        // how long to stay OPEN before trying HALF_OPEN
}

class CircuitBreaker {
  private state: CBState = "CLOSED";
  private failureCount: number = 0;
  private successCount: number = 0;
  private openedAt: number = 0;
  private readonly config: CircuitBreakerConfig;

  constructor(config: CircuitBreakerConfig) {
    this.config = config;
  }

  async call<T>(fn: () => Promise<T>): Promise<T> {
    this.transition();

    if (this.state === "OPEN") {
      throw new Error(`CircuitBreaker OPEN â€” service unavailable. Retry after ${
        this.config.cooldownMs - (Date.now() - this.openedAt)
      }ms`);
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (err) {
      this.onFailure();
      throw err;
    }
  }

  private transition(): void {
    if (this.state === "OPEN" && Date.now() - this.openedAt >= this.config.cooldownMs) {
      this.state = "HALF_OPEN";
      this.successCount = 0;
      console.log("CircuitBreaker -> HALF_OPEN: probing...");
    }
  }

  private onSuccess(): void {
    this.failureCount = 0;
    if (this.state === "HALF_OPEN") {
      this.successCount++;
      if (this.successCount >= this.config.successThreshold) {
        this.state = "CLOSED";
        console.log("CircuitBreaker -> CLOSED: service recovered");
      }
    }
  }

  private onFailure(): void {
    this.failureCount++;
    if (this.state === "HALF_OPEN") {
      this.state = "OPEN";
      this.openedAt = Date.now();
      console.log("CircuitBreaker -> OPEN: probe failed, backing off");
      return;
    }
    if (this.state === "CLOSED" && this.failureCount >= this.config.failureThreshold) {
      this.state = "OPEN";
      this.openedAt = Date.now();
      console.log(`CircuitBreaker -> OPEN: ${this.failureCount} consecutive failures`);
    }
  }

  getState(): CBState { return this.state; }
}

// Demo
async function run() {
  const cb = new CircuitBreaker({ failureThreshold: 3, successThreshold: 2, cooldownMs: 2000 });
  let failUntil = 5;
  let call = 0;

  const unreliableService = async () => {
    call++;
    if (call <= failUntil) throw new Error(`Service down (call ${call})`);
    return `OK (call ${call})`;
  };

  for (let i = 0; i < 8; i++) {
    try {
      const result = await cb.call(unreliableService);
      console.log(`Call ${i+1}: ${result} | State: ${cb.getState()}`);
    } catch (e: any) {
      console.log(`Call ${i+1}: FAILED â€” ${e.message.split(".")[0]} | State: ${cb.getState()}`);
    }
    await new Promise(r => setTimeout(r, i === 4 ? 2100 : 100)); // wait past cooldown after 5th call
  }
}
run();
```

### State Transition Diagram

```
             N failures
CLOSED  ------------------> OPEN
  ^                          |
  |    success threshold      |  cooldown expires
  |    reached               v
  +-------------------- HALF_OPEN
         (if probe fails) --> OPEN
```

### Tradeoffs

| Property                            | Value                                                                          |
| ----------------------------------- | ------------------------------------------------------------------------------ |
| **Prevents cascade failures** | OPEN state fast-fails immediately                                              |
| **Auto-recovery**             | HALF_OPEN probes the service                                                   |
| **Threshold tuning**          | Too sensitive: frequent trips. Too loose: delays recovery.                     |
| **Bulkhead pattern**          | Limit concurrent calls per service (isolate) â€” pairs with circuit breaker |

---

## Pattern 12 â€” Summary

| Q                    | Pattern               | Key Insight                                  |
| -------------------- | --------------------- | -------------------------------------------- |
| Q96 Event Simulator  | Min-Heap by fireAt    | tick() fires all events <= currentTime       |
| Q97 Pub-Sub          | HashMap of topic->Set | Set enables O(1) unsubscribe                 |
| Q98 Connection Pool  | Queue + waiters       | Hand off directly to waiting acquirer        |
| Q99 Bloom Filter     | Bit array + k hashes  | No false negatives; false positives possible |
| Q100 Circuit Breaker | State machine         | CLOSED->OPEN->HALF_OPEN->CLOSED              |

---

# MASTER CHEATSHEET

## By Data Structure

| Structure     | Questions                 | Key Operations               |
| ------------- | ------------------------- | ---------------------------- |
| Trie          | Q16-Q22                   | insert O(L), search O(L+N)   |
| Max-Heap      | Q23, Q27, Q30, Q34        | push/pop O(log n), peek O(1) |
| Min-Heap      | Q24-Q26, Q28-Q29, Q31-Q34 | push/pop O(log n)            |
| HashMap       | Q35-Q46, Q97              | get/put/delete O(1) avg      |
| Stack         | Q47-Q52                   | push/pop O(1)                |
| Queue / Deque | Q53-Q58                   | enqueue/dequeue O(1)         |
| Union-Find    | Q64-Q65                   | union/find O(alpha(n))       |
| BIT/Fenwick   | Q84                       | update/prefix O(log n)       |
| Bloom Filter  | Q99                       | add/contains O(k)            |

## By Algorithm

| Algorithm           | Questions         |
| ------------------- | ----------------- |
| Topo Sort (Kahn's)  | Q59-Q62           |
| Dijkstra            | Q39, Q66          |
| BFS                 | Q63, Q68-Q72      |
| DFS + Backtracking  | Q62, Q71, Q85-Q87 |
| Sweep Line          | Q73-Q79           |
| Binary Search       | Q80-Q84           |
| Two Heaps           | Q28-Q29           |
| Sliding Window      | Q1-Q10            |
| Dynamic Programming | Q91-Q95           |

## By Pattern

| Pattern          | Qs       | Real-World System                    |
| ---------------- | -------- | ------------------------------------ |
| Rate Limiting    | Q1-Q5    | API Gateway, AWS WAF                 |
| Cache Design     | Q11-Q15  | Redis, Memcached                     |
| Trie / Prefix    | Q16-Q22  | Search autocomplete, DNS             |
| Heap / Priority  | Q23-Q34  | Task queues, leaderboards            |
| HashMap O(1)     | Q35-Q46  | Key-value stores, ID generation      |
| Stack / Queue    | Q47-Q58  | Editors, simulators, game engines    |
| Graph / Topo     | Q59-Q72  | Build systems, navigation, social    |
| Interval / Sweep | Q73-Q79  | Calendars, booking systems           |
| Binary Search    | Q80-Q84  | Search, hashing, range queries       |
| Backtracking     | Q85-Q90  | Constraint solving, image processing |
| DP / Greedy      | Q91-Q95  | Optimization, text formatting        |
| Design Patterns  | Q96-Q100 | Microservices, event systems         |
