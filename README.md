# Teaching Heap Allocation with the Simulator

**Tool:** <https://arifpucit.github.io/heap-simulator/>
**Target:** glibc 2.39, x86-64
**Companion slides:** 49–53 (Heap Chunks → free())

Every sequence below was executed against the simulator's own engine and the
outputs are transcribed, not predicted. If a demo does not match, the deployed
build is out of date.

---

## How the tool behaves

Three things shape every demo:

1. **`free() last` frees the highest-address in-use chunk.** You cannot pick which
   chunk to free. Every sequence here is designed around that.
2. **A chunk released next to the top chunk is absorbed into it** and vanishes from
   the arena. To keep a freed chunk visible in a bin, you must *pin* it — leave a
   small allocation above it. This is the single most important trick in the guide.
3. **The walkthrough length tells you which path was taken.** 6 steps = tcache hit.
   7 steps = everything else. 8 steps = `brk()` was needed. Say the step count out
   loud; students learn to predict it.

Reset between demos. Each demo below assumes a fresh heap.

---

## Demo 1 — A chunk is not the size you asked for

**Goal:** kill the belief that `malloc(n)` reserves `n` bytes.

Use the boundary pills. For each, run the walkthrough and read **Step 2**.

| type | chunk | usable | what to say |
|---|---|---|---|
| `4` | 32 | 24 | You asked for 4 and can legally touch 24. |
| `24` | 32 | 24 | **Same chunk as `malloc(4)`.** Nothing changed. |
| `25` | 48 | 40 | One byte more, one whole size class up. |
| `1000` | 1008 | 1000 | Overhead has vanished — why? |

**The question to ask before revealing `24` → 32:** *"We add a 16-byte header, so
`malloc(24)` needs 40, rounds to 48. Yes or no?"* Most of the class says yes. It is
32.

**The answer.** `request2size` adds only `SIZE_SZ = 8`:

```c
(req + 8 + 15) & ~15,  floored at MINSIZE = 32
```

While a chunk is in use, the *next* chunk's `prev_size` field is dead space, so the
allocator lends it to you. Effective overhead is 8 bytes, not 16. That is why
`malloc(1000)` gives usable exactly 1000.

> Ties to slide 49, bullet 2. Have the arena panel open — it prints
> `1000B req / 1000B usable` next to the chunk.

---

## Demo 2 — The header, and the flag that lies

**Goal:** connect the slide-49 diagram to bytes on screen.

```
malloc(4) → malloc(100) → malloc(4)
```

Point at the arena rows. Each shows `prev_size: 0x0` and `size: 0x20|P`.

Ask: **"Why is `prev_size` zero on all three?"**
Because the previous chunk is in use, so that field is not size metadata at all —
it is the tail of the previous chunk's payload. The grey colour in the tool means
"not written."

Now `free() last`. The third chunk turns green (in tcache) — **and the top chunk
above it still reads `|P`.**

**This is the payoff.** `tcache_put()` writes `next` and `key` *inside* the freed
chunk and touches nothing else. The neighbour cannot tell the chunk was freed.
That is not an oversight; it is precisely why tcache and fastbin chunks are never
coalesced. Contrast it in Demo 6, where `P=0` appears.

---

## Demo 3 — tcache: the fast path

**Goal:** show that most `malloc` calls never reach the allocator proper.

```
malloc(100)      → 7 steps, carved from top
malloc(100)      → 7 steps
free() last      → tcache shows 1×112B
malloc(100)      → 6 steps, TCACHE HIT
```

Two things to have students notice:

- **The step count drops from 7 to 6.** Step 5 reads *"returns here without ever
  calling `_int_malloc()`"* — no lock, no bin scan.
- **The returned address is identical to the one just freed.** (Verified: the tool
  hands back the same pointer.) tcache is LIFO, so the most recently freed chunk
  is the first reused — good for cache locality, and the reason use-after-free bugs
  are so reproducible.

**Ask:** *"Is that a coincidence?"* Then free and allocate twice more to show it is
not.

---

## Demo 4 — Seven, then somewhere else

**Goal:** the tcache is capped per size class, and the overflow route depends on size.

```
malloc(100) × 9        (nine allocations)
free() last × 9
```

Observed destinations:

```
free #1..#7  → tcache      (bin now 7×112B)
free #8, #9  → fastbins
```

The tcache bin fills to exactly **7** (`TCACHE_FILL_COUNT`) and stops. Chunk 112 ≤
128, so the surplus goes to a **fastbin** — still no coalescing, still no
`PREV_INUSE` change.

**Now cross the fastbin boundary.** Reset, and repeat with a pin so nothing is
absorbed by top:

```
malloc(120) × 8, then malloc(4)     → free the eight 120s
   frees 1–7 → tcache,  free 8 → fastbins        (chunk 128)

malloc(121) × 8, then malloc(4)     → free the eight 121s
   frees 1–7 → tcache,  free 8 → unsorted bin    (chunk 144)
```

One byte of request moves the eighth free from a fastbin to the unsorted bin,
because 128 is the default `global_max_fast` and 144 exceeds it.

> This is the slide-51 correction made visible: the fastbin cutoff is **128**, not
> 160. 160 is `MAX_FAST_SIZE`, a request size that never becomes the cutoff.

---

## Demo 5 — The tcache ceiling

```
malloc(1032) × 8, then malloc(4)   → frees 1–7 tcache, free 8 → unsorted bin
malloc(1033) × 8, then malloc(4)   → EVERY free → unsorted bin
```

`malloc(1032)` → chunk 1040, the largest the tcache holds. `malloc(1033)` → chunk
1056, past the ceiling, so the tcache is skipped entirely — Step 4 says so
explicitly.

**Ask before running the second one:** *"How many of these eight frees land in the
tcache?"* The expected answer is 7. The correct answer is 0.

---

## Demo 6 — The unsorted bin and coalescing

**Goal:** the best single demo in the tool. Use it as the centrepiece.

```
malloc(1033) × 4
malloc(4)              ← the pin. Do not skip this.
free() last × 4        (the pin stays allocated)
```

Watch the unsorted bin after each free:

```
free #1 → unsorted[1056]
free #2 → unsorted[2112]
free #3 → unsorted[3168]
free #4 → unsorted[4224]
```

Four frees, **one** chunk. Each release merges with the free chunk above it. The
arena collapses from five rows to two.

Three things to point out, in order:

1. **`P=0` appears** on the chunk below the merged one, and its `prev_size` turns
   green with a real value. Compare directly with Demo 2, where it stayed `|P`.
   The footer is what makes backward coalescing possible.
2. **Nothing was erased.** The bytes are still there; only the reservation is gone.
3. **Explain the pin.** Remove `malloc(4)` and re-run: every chunk is absorbed into
   the top chunk instead and the unsorted bin stays empty. That is also correct
   glibc behaviour, and it is the same effect as the "why doesn't my program's
   memory go back to the OS" problem — one live allocation above a free region
   blocks the whole thing.

---

## Demo 7 — Sorting: unsorted → smallbins / largebins

**Goal:** the unsorted bin is a queue, not a destination.

```
malloc(4000) → malloc(4000) → malloc(100)
free() last × 2                       → unsorted[4016]
malloc(3000)                          → unsorted[1008]      (split; see Demo 8)
malloc(5000)                          → smallbins[1008]     ← sorted at last
```

And the large version:

```
malloc(9000) → malloc(9000) → malloc(100)
free() last × 2                       → unsorted[9008]
malloc(3000)                          → unsorted[6000]
malloc(20000)                         → largebins[6000]
```

**The mechanism.** Chunks are filed into small/large bins only when a later
`malloc` scans the unsorted bin and does not take them. 1008 < 1024 so it is a
smallbin; 6000 ≥ 1024 so it is a largebin.

**Why the last `malloc` must be big.** If you ask for something that *fits*, the
allocator takes the chunk straight out of the unsorted bin and it never gets
sorted. Try `malloc(50)` instead of `malloc(5000)` and show that the smallbin stays
empty — a useful accident to stage deliberately.

> Slide-51 boundary check: 1024, not 512. 512 is the 32-bit figure.

---

## Demo 8 — Splitting

Inside Demo 7, step 3 deserves its own moment.

```
before:  unsorted[4016]
malloc(3000)  → chunk 3008 returned, remainder 1008 → unsorted
after:   arena shows  4016(in use) | 3008(in use) | 1008(unsorted) | 112(tcache)
```

One 4016-byte free chunk became a 3008-byte allocation plus a 1008-byte free
remainder. Ask students to compute `4016 − 3008` before you reveal it.

Note `request2size(3000) = 3008`, not 3024 — a second chance to rehearse Demo 1.

---

## Demo 9 — The top chunk and `brk()`

**Goal:** show that most allocations cost no system call.

```
malloc(5000)  repeatedly
```

Watch the **brk() calls** counter and the top-chunk reading. The counter stays at
1 for a long time, then:

```
brk() fires at allocation #27
```

Twenty-six allocations served with no system call; the twenty-seventh has an
**8-step** walkthrough instead of 7, with the extra step badged `brk() CALLED`.

**Say this:** the heap was extended once at the start and then handed out in user
space. The kernel is involved roughly 1 time in 27 here. In a real process it is
rarer still, because glibc over-allocates by `top_pad` = 128 KB on each extension.

Step 6b also notes what the tool does not model: requests whose chunk reaches
`mmap_threshold` (128 KB) bypass the heap entirely, and thread arenas never use
`brk` at all.

---

## Demo 10 — Search order

**Goal:** correct the ordering most textbooks get wrong.

No new clicks needed — read the bins panel top to bottom:

```
tcache → fastbins → smallbins → unsorted bin → largebins → top chunk
```

**Smallbins are searched before the unsorted bin.** An exact-size smallbin hit is
O(1) and needs no unsorted scan; only a *large* request must drain the unsorted bin
first, because the chunk it needs may still be sitting there unsorted.

Step 5 of any tcache-miss walkthrough states the order. Have a student read it
aloud and compare it with whatever ordering your textbook gives.


---

## Verifying on a real machine

Anything in Demo 1 can be checked live, which is worth doing once so students trust
the tool:

```c
#include <stdio.h>
#include <stdlib.h>
#include <malloc.h>

int main(void) {
    for (size_t n = 1; n <= 41; n += 8) {
        void *p = malloc(n);
        size_t hdr = *((size_t *)p - 1);
        printf("malloc(%2zu) -> chunk 0x%02zx  flags=%zu  usable=%zu\n",
               n, hdr & ~7UL, hdr & 7UL, malloc_usable_size(p));
        free(p);
    }
    return 0;
}
```

`*((size_t *)p - 1)` is the size field: eight bytes immediately before the pointer
`malloc` gave you. The low three bits are the flags; `& ~7` strips them.
