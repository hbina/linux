# `WRITE_ONCE` and Compiler Barriers in Linux List Operations

`WRITE_ONCE` (and its counterpart `READ_ONCE`) are the kernel's primary tool for
controlling compiler optimisations on shared memory accesses. They appear
throughout `include/linux/list.h` and are essential to understanding how the
list primitives are safe for concurrent use.

---

## 1. What `WRITE_ONCE` Is

Defined in `include/asm-generic/rwonce.h`:

```c
#define __WRITE_ONCE(x, val) \
    *(volatile typeof(x) *)&(x) = (val);

#define WRITE_ONCE(x, val) \
do {                        \
    compiletime_assert_rwonce_type(x); \
    __WRITE_ONCE(x, val);   \
} while (0)
```

The implementation is a cast to `volatile` pointer followed by a store. The
`volatile` keyword has one precise effect: **the compiler must emit exactly one
store to that address, at exactly the point in the source where it appears.** The
compiler cannot:

- Reorder the store relative to other `volatile` accesses
- Hoist the store out of a loop
- Merge two stores to the same address into one
- Eliminate the store because it thinks the value is never read

`WRITE_ONCE` is **not** a CPU memory barrier — it does not prevent the CPU's
out-of-order execution hardware from reordering stores relative to each other.
It is purely a constraint on the compiler's code generation.

The file header (`rwonce.h:1`) summarises this:

> Prevent the compiler from merging or refetching reads or writes. The compiler
> is also forbidden from reordering successive instances of READ_ONCE and
> WRITE_ONCE, but only when the compiler is aware of some particular ordering.

---

## 2. Why Lists Need It

The C abstract machine gives the compiler wide latitude to reorder stores to
unrelated addresses. In `__list_add` and `__list_del`, the order of stores is
semantically significant for concurrent readers — even without CPU reordering,
a compiler that reorders the stores into the wrong sequence can break a
concurrent traversal.

### Insertion: `__list_add`

```c
static inline void __list_add(struct list_head *new,
                               struct list_head *prev,
                               struct list_head *next)
{
    next->prev = new;          // (1) wire new node's back-link in the successor
    new->next = next;          // (2) wire new node's forward link
    new->prev = prev;          // (3) wire new node's back-link to predecessor
    WRITE_ONCE(prev->next, new); // (4) publish: make new node visible to readers
}
```

Steps 1–3 fully initialise `new` before step 4. The `WRITE_ONCE` on step 4
is the **publication point**: a concurrent forward traversal following
`prev->next` will either see the old next node (if it reads before step 4) or
the fully-initialised `new` node (if it reads after). It can never see a
half-wired node.

Without `WRITE_ONCE` the compiler could legally hoist step 4 above steps 1–3,
making a half-initialised node briefly visible to readers.

### Deletion: `__list_del`

```c
static inline void __list_del(struct list_head *prev, struct list_head *next)
{
    next->prev = prev;           // plain store
    WRITE_ONCE(prev->next, next); // WRITE_ONCE
}
```

`prev->next` is the pointer a concurrent forward traversal will follow. The
moment it is updated to skip the deleted node, that node is gone from the
reader's perspective. This store is the publication point and must not be
reordered earlier by the compiler.

`next->prev` is the backward link. Standard forward-only traversals never read
`->prev` during iteration, so a plain store is sufficient — there is no
concurrent reader racing on it.

---

## 3. The Asymmetry Explained

The asymmetry (`WRITE_ONCE` on `->next` but not `->prev`) is a deliberate
encoding of the kernel's concurrency model for lists:

> **Concurrent lockless traversal is forward-only.**

`->next` is the pointer that concurrent RCU readers follow. It is the shared
mutable state that requires careful handling. `->prev` is only accessed by code
that already holds the list lock, or by specialised bidirectional RCU variants
(see section 5).

---

## 4. What Goes Wrong Without It

Consider a pathological compiler optimisation on `__list_add` without
`WRITE_ONCE`:

```c
// Compiler decides to reorder because both are "just stores":
WRITE_ONCE(prev->next, new);  // reordered first — new is now visible
new->next = next;             // new->next still garbage
new->prev = prev;             // new->prev still garbage
next->prev = new;
```

A concurrent reader following `prev->next` now has a pointer to `new`, but
`new->next` has not been set yet. Following it leads to an arbitrary address.

Or consider a loop:

```c
while (condition) {
    list_add(&item->list, head);  // without WRITE_ONCE, compiler might hoist
    ...                           // the store outside the loop
}
```

The `volatile` cast in `WRITE_ONCE` prevents the compiler from treating the
store as loop-invariant.

---

## 5. Backward Traversal: Three Tiers

Whether `->prev` needs protection depends on how the list is used.

### Tier 1 — Standard lists (always under a lock)

`list_for_each_prev` and `list_for_each_entry_reverse` require the caller to
hold the list lock. `next->prev = prev` is a plain store — the lock guarantees
mutual exclusion.

### Tier 2 — `list_del_rcu` (forward-only RCU lists)

When a list is accessed by lockless RCU readers, deletion uses `list_del_rcu`:

```c
static inline void list_del_rcu(struct list_head *entry)
{
    __list_del_entry(entry);
    entry->prev = LIST_POISON2;  // ← deliberately poisoned immediately
}
```

The comment in `rculist.h` is explicit:

> *it means that we can not poison the forward pointers that may still be used
> for walking the list*

`->next` is preserved (an RCU reader might still be traversing it). `->prev` is
poisoned immediately because **RCU lists only support lockless forward
traversal**. Any attempt to traverse backward on such a list will fault at
`LIST_POISON2`, making the bug obvious.

### Tier 3 — `list_bidir_del_rcu` (bidirectional RCU lists)

Added for the rare case where lockless backward traversal is genuinely needed.
`list_bidir_del_rcu` does **not** poison `->prev`, preserving it through the
RCU grace period so that `list_bidir_prev_rcu()` can safely follow it.

This is a **whole-list contract**: a list must use either `list_del_rcu` (forward
only) or `list_bidir_del_rcu` (bidirectional) — never both. Mixing them will
corrupt the `->prev` pointers that backward traversal depends on.

When backward traversal is lockless, `->prev` stores also need `WRITE_ONCE`
protection — which the bidirectional RCU helpers apply.

---

## 6. Summary Table

| Operation | `->next` store | `->prev` store | Reason |
|-----------|---------------|----------------|--------|
| `__list_add` | `WRITE_ONCE` | plain | `->next` is the publication point for forward readers |
| `__list_del` | `WRITE_ONCE` | plain | `->next` makes deletion visible; `->prev` not read concurrently |
| `list_del_rcu` | `WRITE_ONCE` (via `__list_del_entry`) | poisoned | Forward-only RCU; backward traversal is unsupported |
| `list_bidir_del_rcu` | `WRITE_ONCE` | preserved, not poisoned | Both directions traversed locklessly; `->prev` must stay valid |
| Plain locked operations | plain | plain | Lock provides all ordering guarantees |

---

## References

- `include/asm-generic/rwonce.h`
- `include/linux/list.h`
- `include/linux/rculist.h`
- `Documentation/memory-barriers.txt` — full treatment of compiler vs CPU barriers
- `Documentation/RCU/whatisRCU.rst` — RCU fundamentals (grace periods, readers)
