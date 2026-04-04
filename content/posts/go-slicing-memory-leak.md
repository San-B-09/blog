---
title: "The Slice That Wouldn't Let Go"
date: 2026-04-03
slug: "go-slicing-memory-leak"
description: "How Go slices pin memory you've already forgotten about. Three patterns, three proofs, one backing array."
summary: "How Go slices pin memory you've already forgotten about. Three patterns, three proofs, one backing array."
categories: ["golang"]
draft: false
---

There's a category of bugs that don't announce themselves. No panic, no error log, no wrong answer. Just a memory utilization graph doing this:
 
```
  memory
    ▲
    │       /|     /|     /|     /
    │      / |    / |    / |    /
    │     /  |   /  |   /  |   /
    │    /   |  /   |  /   |  /
    │   /    | /    | /    | /
    └────────────────────────────► time
                                  ↑ OOM
```

Then container does what container does: `OOMKilled`. Restart. Climb. Kill. Restart.

I learned this the hard way. A game scoreboard service that held a ranked leaderboard in memory and served the top-N over WebSocket. No goroutine leak, no unclosed connections, no forgotten tickers. The heap profiler pointed at one thing: **slices**.

Slices in Go are three-word headers pointing to a shared backing array. Most Go developers know this. Far fewer internalize what it means for memory lifetime. This post walks through three distinct patterns where idiomatic slice operations silently pin memory that should have been freed. All demonstrated through the same scoreboard domain, all with a short program you can run.
 
Every pattern below includes just the key function. The complete runnable program is at the end. Run it with `go run main.go <pattern>` to reproduce any of them.

---

## The Data Model

Say you're building a near real-time game scoreboard. Thousands of players, scores updating every second, ranked leaderboard held in memory.

Everything that follows operates on this:
 
```go
type Player struct {
    ID       string
    Score    int64
}
 
type Scoreboard struct {
    players []Player
}
```
 
A `Player` is 24 bytes, a string header (16 bytes: pointer + length) and an int64 (8 bytes). Small. Innocent. That's what makes these patterns deceptive. The struct is tiny, but the backing arrays they live in are not.
 
---

## Pattern 1: The Sub-Slice Anchor
You need to serve the top 10 players. You have 50,000 in a sorted slice. The natural instinct:
 
```go
func GetTopTen(board *Scoreboard) []Player {
    return board.players[:10] // the anchor
}
```
 
The returned slice has `len=10`. But it shares the backing array with the full 50,000 element slice. If the caller stores this result in a cache, a response struct, a channel buffer, it pins the **entire backing array** in memory.
 
```
  Backing Array (50,000 players)
  ┌────┬────┬────┬─────┬────────────────────────────────────┐
  │ P0 │ P1 │ .. │ P9  │  P10  P11  ...  ...  ...   P49999  │
  └────┴────┴────┴─────┴────────────────────────────────────┘
   ▲ what you use (10)    ▲ what you're paying for (49,990)
```

We simulate 100 WebSocket connections, each caching a `board[:10]` sub-slice from a fresh 50,000 player scoreboard snapshot:
 
```go
// BUG: sub-slice anchors entire 50k backing array
topTen := board[:10]
cache = append(cache, topTen)
```

**Output (`go run main.go subslice-leak`):**
```shell
[  0s] cache_len=1    heap_alloc=3.5 MB     heap_objects=131
[ 10s] cache_len=11   heap_alloc=37.9 MB    heap_objects=147
[ 20s] cache_len=21   heap_alloc=72.3 MB    heap_objects=162
[ 30s] cache_len=31   heap_alloc=106.7 MB   heap_objects=171
[ 60s] cache_len=61   heap_alloc=209.8 MB   heap_objects=210
[100s] cache_len=100  heap_alloc=347.3 MB   heap_objects=241
```

The naive math says 100 × 50,000 × 24 bytes = ~115 MB. But each `Player.ID` was created with `fmt.Sprintf`, that's a separate string allocation on the heap. The 24-byte `Player` struct in the backing array holds a string header that *points to* that allocation. Pin the backing array, and you pin every string header inside it, which pins every string allocation they reference. The real cost is **~3.5 MB per snapshot × 100 = ~347 MB**. For 10 players per cache entry, you'd expect under **4 KB**.

**Now with the fix, `copy` into a fresh slice:**
 
```go
// FIX: copy decouples from backing array
topTen := make([]Player, 10)
copy(topTen, board[:10])
cache = append(cache, topTen)
```

**Output (`go run main.go subslice-fix`):**
```
[  0s] cache_len=1    heap_alloc=3.5 MB     heap_objects=133
[ 10s] cache_len=11   heap_alloc=7.0 MB     heap_objects=149
[ 20s] cache_len=21   heap_alloc=3.6 MB     heap_objects=158
[ 30s] cache_len=31   heap_alloc=3.6 MB     heap_objects=168
[ 60s] cache_len=61   heap_alloc=17.3 MB    heap_objects=202
[100s] cache_len=100  heap_alloc=3.6 MB     heap_objects=241
```
 
Same logic. ~3.6 MB (with GC fluctuations up to ~17 MB between cycles) vs a steady **347 MB**. The difference is one `copy` call. The spikes you see are just the transient 50,000 player board allocations being built and collected each tick. The *retained* memory after GC is what matters, and it stays flat.
 
> *Any sub-slice returned from a function, sent to a channel, or stored in a struct with a longer lifetime than the parent slice.*

---

## Pattern 2: The Append Ambiguity
After `append`, does the new slice share memory with the old one? 

**It depends on the capacity.** If there's room, `append` reuses the backing array. If not, it allocates fresh. This dual behavior creates bugs that don't reproduce at small scale.
 
We build a scoreboard with excess capacity and derive two top-5 sub-slices, one with inherited cap and one clamped with the three-index expression:
```go
board := make([]Player, 50, 10_000)
 
topA := board[:5]         // cap = 10,000 (inherited!)
topB := board[:5:5]       // cap = 5     (clamped with 3-index)
 
newPlayer := Player{ID: "intruder", Score: 999}
topA = append(topA, newPlayer)
topB = append(topB, newPlayer)
```
 
**Output (`go run main.go append-leak`):** 
```
[round    0] board[5].ID=intruder   (expect p-5)  topA: len=6   cap=10000   topB: len=6   cap=10      heap=345.3 KB
[round   50] board[5].ID=intruder   (expect p-5)  topA: len=6   cap=10000   topB: len=6   cap=10      heap=1.5 MB
[round  100] board[5].ID=intruder   (expect p-5)  topA: len=6   cap=10000   topB: len=6   cap=10      heap=2.7 MB
[round  500] board[5].ID=intruder   (expect p-5)  topA: len=6   cap=10000   topB: len=6   cap=10      heap=2.0 MB
```

`topA` had room in the inherited capacity, so `append` wrote directly into the parent's backing array. `board[5]` is now `"intruder"`. That's silent corruption. `topA` still carries `cap=10000`, holding the full allocation alive.

`topB` tells a different story. The three-index expression `board[:5:5]` gave it `cap=5`, so `append` had no room and allocated a fresh backing array. Go's growth strategy doubled the capacity from 5 to 10 and that's why you see `cap=10` in the output, not 5. The parent board is untouched.


---

## Pattern 3: The Batch Residue
 
The scoreboard ingests player data from a network stream in large batches. Many protocol decoders like protobuf, JSON unmarshalers, custom binary parsers work by allocating one large `[]byte` buffer for the entire message and then producing string fields by sub-slicing into that buffer. The strings they return are lightweight views into the original bytes.
 
Here's what that looks like in a scoreboard context:
 
```go
// simulateDecoder mimics what protocol decoders do internally:
// allocate one big buffer, return strings that point into it
func simulateDecoder() []Player {
    // In real code, this buffer comes from net.Conn.Read or proto.Unmarshal
    raw := strings.Repeat("X", 1_048_576) // 1 MB wire message
 
    // Decoder "parses" fields by slicing into the buffer
    // Each player ID is a tiny substring backed by the full 1 MB
    players := make([]Player, 100)
    for i := range players {
        offset := i * 20
        players[i] = Player{
            ID:    raw[offset : offset+10], // 10 bytes, pinning 1 MB
            Score: int64(i),
        }
    }
    return players
}
```
 
Each `Player.ID` is only 10 bytes, but the string header points into the 1 MB buffer. As long as even one of these strings is alive, the **entire buffer stays on the heap**. Now your `IngestBatch` function extracts just the IDs for ranking:
 
```go
func IngestBatch(batch []Player) []string {
    ids := make([]string, len(batch))
    for i, p := range batch {
        ids[i] = p.ID // looks harmless, pins the whole buffer
    }
    return ids
}
```
 
You're holding 100 ten-byte strings. You're paying for the 1 MB buffer they came from. And if you ingest 100 batches, you're paying for 100 MB.
 
**Output (`go run main.go batch-leak`):**
```
[  0s] retained=1     total_string_bytes=10      heap=1.1 MB
[ 10s] retained=11    total_string_bytes=110     heap=11.1 MB
[ 20s] retained=21    total_string_bytes=210     heap=21.1 MB
[ 30s] retained=31    total_string_bytes=310     heap=31.1 MB
[ 60s] retained=61    total_string_bytes=610     heap=61.1 MB
[100s] retained=100   total_string_bytes=1000    heap=101.1 MB
```
 
100 retained strings. Total content: **1,000 bytes**. Heap: **~100 MB**. Each 10-byte substring is anchoring its 1 MB parent.
 
**The fix, `strings.Clone()` (Go 1.20+):**
```go
func IngestBatch(batch []Player) []string {
    ids := make([]string, len(batch))
    for i, p := range batch {
        ids[i] = strings.Clone(p.ID) // copies bytes, severs the link
    }
    return ids
}
```
 
**Output (`go run main.go batch-fix`):**
```
[  0s] retained=1     total_string_bytes=10      heap=110.3 KB
[ 10s] retained=11    total_string_bytes=110     heap=110.8 KB
[ 20s] retained=21    total_string_bytes=210     heap=111.3 KB
[ 30s] retained=31    total_string_bytes=310     heap=111.5 KB
[ 60s] retained=61    total_string_bytes=610     heap=112.8 KB
[100s] retained=100   total_string_bytes=1000    heap=114.7 KB
```
 
Same data. 0.11 MB instead of 100 MB. `strings.Clone` copies the bytes into a fresh allocation, so the original 1 MB buffer can be garbage collected.
 
When a decoder hands you strings, you don't know whether they're independent allocations or views into a shared buffer. If those strings will outlive the request, cached in memory, stored in a long-lived struct, `strings.Clone` is cheap insurance. The cost is one allocation per string. The alternative is pinning arbitrarily large buffers for the lifetime of your process.
 
---

## The Common Thread
 
Three patterns. Three different mechanisms. One underlying cause: **the slice header is not the memory.**
 
The `len` tells you what you're using. The `cap` tells you what you're paying for. And the backing array, with every pointer, string header, and interface value it contains is what the GC actually traces. Every pattern in this post is a variation on the same disconnect: the code operates on the slice header, but the memory cost lives in the backing array.
 
None of these bugs triggered a compile error. None showed up in unit tests with 10-element slices. They only manifest at scale, over time, with production-sized data.
 
The Go runtime gives you powerful zero-cost abstractions with slices. But zero-cost isn't zero-consequence. The backing array is the reality. The slice header is the illusion.
 
Respect the backing array. It doesn't let go until you make it.
 
---

**All three patterns are in a single file. Run any pattern with `go run main.go <pattern-name>`.**

{{< gist id="c4fa145263508be0f50b92d9f3790473" >}}