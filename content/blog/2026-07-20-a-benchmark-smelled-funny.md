+++
title = "A Benchmark Smelled Funny"
date = 2026-07-19
description = "Or how I learned that a 1.77x slower benchmark was misleading... and about store-to-load forwarding stall hiding something more."
[taxonomies]
tags = ["go", "performance", "cpu", "benchmarking"]
[extra]
draft = true
+++

I've been building [go-spmc-ring](https://github.com/pintomau/go-spmc-ring), a single-writer, multiple-reader ring
buffer in Go. The whole point of the project is mechanical sympathy: every design decision is supposed to have a
benchmark behind it. I came across a benchmark that I didn't quite understand, and went digging deeper.

## The accusation

The ring buffer has two ways to publish an event. You can hand it a value:

```go
rb.Publish(event)
```

Or you can hand it a callback that fills the destination slot in place:

```go
rb.PublishFunc(func(slot *object) {
    slot.x[0] = '0'
})
```

My payload is `object{ x [64]byte }`, which is exactly one cache line. And my benchmark table said this:

| Benchmark        | ns/op | Notes                                  |
|------------------|------:|----------------------------------------|
| `Publish`        |  5.19 | callback API, keep-up reader           |
| `Publish_Direct` |  9.21 | constructs and publishes a value/event |

There it is. Publishing by value is **1.77x slower**. The obvious lesson writes itself: *don't pass structs by value in
a hot loop, use the callback.*

Something nagged, though. 1.77x is a *lot* for one struct copy. 64 bytes is four 16-byte SSE moves, or one AVX-512 store
on a good day. It should be nearly free. I decided to take it apart.

## Four benchmarks, one variable at a time

Instead of trusting one row that changes several things at once, I wrote four benchmarks that each change exactly one
thing. (They live in a [tiny standalone repo](https://github.com/pintomau/store-forwarding-demo) so you can run them
yourself. No ring buffer required.) Every one writes a *different* value each iteration, so none of them can be
waved away as a constant the compiler folded flat:

```go
// build a fresh value every iteration, then publish by value
func BenchmarkConstructed(b *testing.B) {
    var seq byte
    for b.Loop() {
        o := object{}
        o.x[0] = seq
        publishByValue(o)
        seq++
    }
}

// publish by value from buffers built AHEAD of the loop
func BenchmarkPrepared(b *testing.B) {
    var bufs [4]object
    for i := range bufs {
        bufs[i].x[0] = byte(i)
    }
    var i int
    for b.Loop() {
        publishByValue(bufs[i&3]) // value changes, but it was built long ago
        i++
    }
}

// fill all 64 bytes of the slot in place, via callback
func BenchmarkFullWrite(b *testing.B) { /* ... */ }

// write a single changing byte into the slot, via callback
func BenchmarkOneByteWrite(b *testing.B) { /* ... */ }
```

Stripped of the ring buffer's own overhead, the gap is stark. On my Ryzen 5 9600X:

```
BenchmarkConstructed-12     5.51 ns/op
BenchmarkPrepared-12        1.69 ns/op
BenchmarkFullWrite-12       1.61 ns/op
BenchmarkOneByteWrite-12    1.53 ns/op
```

Read that table and the story flips completely.

**Payload size is free.** `FullWrite` fills all 64 bytes; `OneByteWrite` writes one. They're basically the same
(1.61 vs 1.53 ns). Writing the whole cache line costs almost nothing over writing a single byte, because what
you're really paying for is the cache line changing hands, not the bytes going into it. So the gap was never
"we copy 64 bytes."

**The by-value API is cheap too.** `Prepared` passes a 64-byte struct by value, exactly like `Constructed` does
(and it runs at 1.69 ns), right down near the floor. Passing a struct by value, when the value was built
ahead of time, is not the villain either.

So what is `Constructed` doing that the others aren't? Only one thing: it builds the value *in place, right
before the call*. That construction (`o := object{}; o.x[0] = seq`), is the entire difference between 1.69 ns
and 5.51 ns.

The original "1.77x API penalty" was a lie of composition. The honest by-value cost is a rounding error. Everything
else was the caller's construction pattern, and the benchmark had quietly folded the two together.

## So why does building it *in place* cost 3x?

This is where you have to stop reading Go and start reading assembly. `go test -gcflags=-S` on `Constructed` shows
exactly what the CPU is being asked to do:

```asm
; o := object{}; o.x[0] = seq
MOVUPS X15, (CX)        ; 16-byte zeroing store
MOVUPS X15, 16(CX)      ; 16-byte zeroing store
MOVUPS X15, 32(CX)      ; 16-byte zeroing store
MOVUPS X15, 48(CX)      ; 16-byte zeroing store
MOVB   DL, o+80(SP)     ; 1-byte store        <- note the size

; publishByValue(o): copy the argument out
MOVUPS (CX), X14        ; 16-byte LOAD over the bytes just written
MOVUPS X14, (DX)
```

Zeroing the struct is four 16-byte stores. Setting the first byte is one 1-byte store. Then, to pass `o` by value,
the argument copy immediately *reads those bytes back*, starting with a 16-byte load that overlaps the first
16-byte zeroing store **and** the 1-byte store.

And this is the crux.

1. **Stores don't go to the cache. Not at first**: When the CPU executes a store, it does not write to L1. It appends
   an entry to a small hardware queue called the store buffer (~64 entries on Zen): "address X, N bytes, this data."
   The entry sits there until the instruction retires, and only then drains to L1, quietly, in the background. Why?
   Speculation. The CPU executes ahead of branches it has guessed. If the guess was wrong, it must throw work away,
   and you can't "unwrite" the cache. The store buffer is the holding pen for writes that aren't yet official.
2. **That creates a problem for every load**: If pending writes live in the store buffer and not in the cache,
   then the cache is stale by definition. So every load must check: "is there a store buffer entry for my
   address, newer than me?" If yes, reading L1 would return old data.
3. **The fast path: forwarding**: If a single pending store entry fully contains the load's bytes, the hardware
   just copies the data straight out of that entry into the load. No cache involved, done in a few cycles. This
   is store-to-load forwarding, and it's why x = 5; y = x costs nothing. AMD's own
   [optimization guide](https://docs.amd.com/v/u/en-US/58455_1.00) states the covering rule plainly: the
   load-store unit forwards "when there is an older store that contains **all of the load's bytes**"
   (Software Optimization Guide for the AMD Zen5 Microarchitecture, #58455, ch. 2, Load-Store Unit).
4. **The slow path: our stall**: Now the crux. Each store buffer entry is one packet: (addr, size, data). The
   forwarding hardware is a matcher, not a merger: it can grab data from one entry, but it has no ALU to
   stitch bytes from entry #1 together with bytes from entry #5, in age order, in the load's shadow.\
\
   So when our 16-byte load arrives and finds byte 0's newest value in the MOVB entry and bytes 1–15's newest value
   in the older MOVUPS entry, the matcher shrugs: no single entry covers me. The only correct fallback is to make 
   the cache authoritative again: wait for those entries to retire and drain to L1, then read L1 normally. That
   wait, pipeline twiddling its thumbs while stores age out, is the 15–20 cycles
   ([Agner Fog](https://www.agner.org/optimize/microarchitecture.pdf) measures ~12 for the partial-overlap case
   on Zen 5, §25.17 of the microarchitecture manual; the end-to-end gap here works out to ~20 cycles per iteration).\
\
   That's the whole thing. Not a cache miss, not memory bandwidth. A pattern-matching limitation in a queue-lookup
   circuit, and the penalty is "wait until the queue drains." (For a perf-counter walkthrough of the same
   narrow-write-then-wide-read shape on Intel, see
   [Store forwarding by example](https://easyperf.net/blog/2018/03/09/Store-forwarding) on Easyperf.)

Here's the collision, drawn out. Building the value leaves several stores sitting in the store buffer, all still
in flight (newest at the bottom, in program order):

```text
  store buffer (pending writes to o on the stack)
  ┌────────────────────────────────────────────────┐
  │  MOVUPS X15  ->  bytes [ 0 .. 15]  (16 bytes)  │  ◄ zeroes byte 0
  │  MOVUPS X15  ->  bytes [16 .. 31]  (16 bytes)  │
  │  MOVUPS X15  ->  bytes [32 .. 47]  (16 bytes)  │
  │  MOVUPS X15  ->  bytes [48 .. 63]  (16 bytes)  │
  │  MOVB   seq  ->  byte [0]          (1 byte)    │  ◄ overwrites byte 0 (newest)
  └────────────────────────────────────────────────┘

Then the argument copy immediately issues:  LOAD bytes [0 .. 15].
Draw every pending store and the load as spans over the same byte axis:

           byte offset within o
           0               15 16          31 32   ...
           |----------------|----------------|--------

  pending  ################                     MOVUPS zero [ 0..15]
  stores                    ################     MOVUPS zero [16..31]
                                          ...    (two more, [32..63])
           #                                     MOVB seq -> [0] <- NEWEST

  the load ================                     LOAD [ 0..15]
           ^
           byte 0's newest data lives in the MOVB entry,
           bytes 1..15's newest data in the first MOVUPS:
           no single store spans the load window
           ->  cannot forward  ->  stall
```

The 1-byte store runs *last*, so at load time byte 0's freshest value lives in the `MOVB` while bytes 1..15 live in
the earlier 16-byte `MOVUPS`. The load wants all 16 as a unit, but its bytes have two different most-recent 
writers. No single pending store covers the read, so the core can't shortcut it. It stalls until those writes
land in L1, then reads them back the slow way.

Now compare `Prepared`. Its loop assembly is *almost identical*, it still does the double copy, the value into the 
argument area and then into the slot. The one difference is where the construction lives:

```asm
; buffers built ONCE, above the loop:
MOVUPS X15, (CX)
...                 ; the stores that fill bufs

loop:
MOVUPS (CX), X14    ; same 16-byte readback...
MOVUPS X14, (DX)    ; ...but the stores that wrote CX retired long ago
...
```

Same instructions, same straddling load. But the stores that built those buffers ran *before* the loop. They retired
and drained to L1 long before the first readback. Draw the same moment and the collision is simply gone:

```text
  store buffer (writes to bufs already retired)
  ┌──────────────────────────────┐
  │           (empty)            │   <- nothing pending
  └──────────────────────────────┘

  L1 cache (bufs live here, settled)
  ┌──────────────────────────────┐
  │ bytes [0 .. 63] of bufs[i&3] │
  └──────────────────────────────┘
           ▲
           │  LOAD bytes [0 .. 15]
           └──  no pending store to reconcile
               served straight from L1  ->  no stall
```

By the time the copy runs a million times, there is nothing fresh in the store buffer for it to trip over, so the load
is served from cache at full speed.

That's the whole difference between 5.5 ns and 1.7 ns: not the copy, not the API, just *when* the stores that feed the
copy were issued. Same load instruction, same bytes, the only thing that changed is whether a half-finished store 
was still sitting under it.

The benchmark, in other words, was never measuring the `Publish` API. It was measuring a *construct-then-copy
anti-pattern*, one that happens to be really easy to write, which is exactly why the number still matters
even though the blame was in the wrong place.

## Can't the library just fix this?

The stall doesn't happen in the ring buffer's memory. It happens in **the caller's stack frame, before the value ever
reaches the ring.** Here's `Publish`:

```go
func (r *RingBuffer[T]) Publish(payload T) {
    ...
    r.buffer[nextSequence&r.mask] = payload   // copy the value INTO the slot
    ...
}
```

By the time that line runs, it's already over. Trace the caller instead:

```go
o := object{}       // zeroing stores  -> caller's stack
o.x[0] = seq        // 1-byte store    -> caller's stack
rb.Publish(o)       // argument copy: read o back off the stack  <- the stall
```

The straddling load is the *argument copy*, the CPU reading `o` back out of your stack to pass it by value.
Both the stores and the load live in your frame, and they all finish before the ring buffer's own store
into the slot even executes. The slot is just the innocent final destination. You cannot fix a stall
in someone else's stack frame by rearranging your data structure.

But the library *does* have a structural answer. The sibling method hands you a pointer straight to the slot:

```go
func (r *RingBuffer[T]) PublishFunc(f func(*T)) {
    ...
    f(&r.buffer[nextSequence&r.mask])   // you fill the slot in place
    ...
}
```

There's no intermediate `o`, so there's nothing to copy and nothing to read back. You build the event where it will
live.

## What to actually do in production

Strip away the microbenchmark scaffolding, the rotating buffers were only ever a way to *simulate* "a value whose
stores already retired", and the guidance is a three-line decision:

- **Constructing the event at publish time?** Fill it in the slot. Use `PublishFunc` for one, `PublishBatchFunc` for
  many (which also amortizes the cursor store across the batch):

  ```go
  rb.PublishFunc(func(slot *object) {
      *slot = object{}   // or just overwrite every field
      slot.x[0] = seq
  })
  ```

  No argument copy, no stall, and it stays fast even when every field differs every time (`FullWrite`, 1.61 ns).

- **Already holding a finished value**: one that arrived from a channel, a decoded frame, a slice you're forwarding?
  Then `Publish(v)` is fine. Passing an existing value by value was never the stall; that's exactly what `Prepared`
  measured, and it ran near the floor. The trap is *construct-then-copy in the same breath*, not by-value itself.

  ```go
  for ev := range incoming { // ev built elsewhere, earlier
      rb.Publish(ev)         // no fresh stores under the copy
  }
  ```

- **Never** write `o := T{}; fill(o); rb.Publish(o)` back to back in a hot loop. That one shape pays the full stall,
  and no API change can save it.

## One more twist: your CPU may disagree

Running on an Apple M4 did not render the same results. On arm64 the gap mostly evaporates, the by-value construction
path is roughly on par with the rest, and in go-spmc-ring's full benchmark it's actually the *faster* option.
The store-forwarding stall that dominates on x86 simply doesn't reproduce there. (Apple silicon has its own,
entirely different stall lurking in that code, but that's a story for another post.)

---

*The full standalone reproduction (four benchmarks, the disassembly, and the arm64 note) is on [GitHub](https://github.com/pintomau/store-forwarding-demo). It runs in about 15 seconds and needs nothing but a Go toolchain. Clone it and watch your own CPU stall.*
