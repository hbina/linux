# Kernel Foundations Study Plan

Each item is a specific, completable task: read a named function, answer a
named question, or find a named thing in the source.  Tick it off only when
you can answer the question without looking it up again.

Phases must be done in order — each one is a prerequisite for the next.

Track high-level completion in `docs/PROGRESS.md`.

---

## Phase 1 — Core Data Structures

### 1.1 Intrusive Linked Lists

- [x] Read `include/linux/list.h`: `struct list_head` — it is just `{next, prev}` pointers with no data payload. Understand what "intrusive" means: you embed it inside your own struct.
- [x] Read `include/linux/container_of.h`: expand `container_of(ptr, type, member)` by hand. Answer: what is `offsetof(type, member)` doing, and why does the cast go through `char *`?
- [x] Read `include/linux/list.h`: `INIT_LIST_HEAD()` — an empty list is a self-loop (`next == prev == self`), not NULL. Why is this safer than NULL?
- [x] Read `list_add()` and `list_add_tail()` — which end does each insert at? Which gives LIFO behaviour, which gives FIFO?
- [x] Read `list_del()` — it sets `prev`/`next` to `LIST_POISON1`/`LIST_POISON2` instead of NULL. Find those constants in `include/linux/poison.h`. Answer: why poison values instead of NULL?
- [x] Read `list_del_init()` — how does it differ from `list_del()`? When would you use each?
- [x] Read `list_for_each_entry(pos, head, member)` — trace through the macro: it calls `container_of` on each `list_head` in the ring to recover the enclosing struct.
- [x] Read `list_for_each_entry_safe(pos, n, head, member)` — `n` is a saved-next cursor. Answer: why can you safely call `list_del(pos)` inside this loop but not inside `list_for_each_entry`?
- [x] Grep for `list_for_each_entry_safe` in `kernel/workqueue.c`. Find one call site and explain why deletion during iteration is needed there.
- [x] Read `include/linux/hlist.h`: `struct hlist_head` has one pointer; `struct hlist_node` has `next` and `**pprev` (pointer-to-pointer). Answer: why `**pprev` instead of `*prev`, and what memory does this save in a hash table with millions of buckets?
- [x] Find `hlist` used for PID lookup: open `kernel/pid.c`, find `struct upid` — it embeds `struct hlist_node pid_chain`. Find the `pid_hash[]` array. Answer: why is `hlist` better than `list_head` here?
- [x] Read `include/linux/llist.h`: `llist_add()` uses `cmpxchg` — it is lock-free. Find one caller in `kernel/` and explain why lock-freedom matters in that context.

### 1.2 Red-Black Trees

- [ ] Read `include/linux/rbtree.h`: `struct rb_node` embeds colour in the low bit of the parent pointer (`__rb_parent_color`). Answer: why use a pointer bit rather than a separate `bool color` field?
- [ ] Read `rb_entry(ptr, type, member)` — it is `container_of`. Same pattern as `list_entry`.
- [ ] Read `lib/rbtree.c`: `rb_insert_color()` — skim the rotation cases. You do not need to memorise them, but identify: what are the three cases handled after inserting a red node?
- [ ] Open `kernel/time/hrtimer.c`. Find `struct timerqueue_node` — it wraps `struct rb_node`. Find `timerqueue_add()` — it inserts into the RB-tree and tracks the leftmost node separately. Answer: why cache the leftmost node?
- [ ] Open `mm/mmap.c`. Find where `struct vm_area_struct` is inserted into the RB-tree (`vma_rb_insert` or similar). Find `find_vma()` — it walks the tree. Answer: what is the key used for ordering VMAs?

### 1.3 XArrays

- [ ] Read `Documentation/core-api/xarray.rst` — the full overview. Answer: what problem did XArray replace (hint: radix tree), and what did the API simplify?
- [ ] Read `include/linux/xarray.h`: `xa_load(xa, index)` and `xa_store(xa, index, entry, gfp)` — these are the fundamental read and write operations.
- [ ] Read `xa_for_each(xa, index, entry)` — it iterates over all present entries. Note that gaps (absent entries) are skipped automatically.
- [ ] Open `include/linux/fs.h`. Find `struct address_space` — it contains `struct xarray i_pages`. This is the page cache: `i_pages` maps file offset (in pages) to `struct folio *`.
- [ ] Open `mm/filemap.c`. Find `filemap_get_folio()` — it calls `xa_load(&mapping->i_pages, index)`. Answer: what does it do when `xa_load` returns NULL (page not in cache)?
- [ ] Answer: what is an "exceptional entry" in XArray? Grep for `xa_is_value()` in `mm/` to find where swap entries are stored as exceptional XArray entries.

### 1.4 Hash Tables

- [ ] Read `include/linux/hashtable.h`: `DECLARE_HASHTABLE(name, bits)` declares an array of `2^bits` `hlist_head` buckets.
- [ ] Read `hash_add(htable, node, key)` — it calls `hash_min(key, HASH_BITS(htable))` to select the bucket. Read `include/linux/hash.h`: `hash_32()` uses Fibonacci (golden-ratio) hashing. Answer: why is Fibonacci hashing better than `key % table_size`?
- [ ] Read `hash_for_each_possible(htable, obj, member, key)` — iterates only the bucket matching `key`. Contrast with `hash_for_each` which iterates all buckets.
- [ ] Open `kernel/pid.c`. Find `find_pid_ns()` — it calls `hash_for_each_possible` on `pid_hash`. Trace the full lookup: hash key → bucket → `hlist` walk → `struct upid` → `struct pid`.

---

## Phase 2 — Concurrency Primitives

### 2.1 Spinlocks

- [ ] Read `include/linux/spinlock.h`: understand the three lock variants and when each is required:
  - `spin_lock(l)` — process context only, no IRQ interaction
  - `spin_lock_bh(l)` — disables softirqs; use when lock is shared with a softirq handler
  - `spin_lock_irqsave(l, flags)` — disables hard IRQs; use when lock is shared with a hard IRQ handler
- [ ] Answer: if a spinlock is taken in a hard IRQ handler and also in process context on the same CPU, which variant must the process-context caller use, and why? (Hint: what happens if the IRQ fires while process context holds the lock with just `spin_lock`?)
- [ ] Read `include/linux/spinlock_types.h`: find `spinlock_t` vs `raw_spinlock_t`. Answer: under `PREEMPT_RT`, `spinlock_t` becomes a sleeping mutex — so what is the only correct type to use in code that truly cannot sleep (e.g., inside an NMI handler)?
- [ ] Find a real `spin_lock_irqsave` in `kernel/irq/manage.c` — identify which shared data structure it protects and why IRQs must be disabled.

### 2.2 Mutexes

- [ ] Read `include/linux/mutex.h`: `mutex_lock()` may sleep — it is only valid in process context. `mutex_trylock()` returns 0 on failure without sleeping.
- [ ] Read `kernel/locking/mutex.c`: find the fast path — `atomic_long_try_cmpxchg_acquire()`. Answer: what value does the `owner` field hold when the mutex is unlocked?
- [ ] Read the slow path: `__mutex_lock_slowpath()` — the caller is put on a wait list and goes to sleep via `schedule()`. Contrast with a spinlock, which busy-waits.
- [ ] Answer: you are writing an interrupt handler (hard IRQ context). Can you call `mutex_lock()`? Why or why not?

### 2.3 RW Semaphores

- [ ] Read `include/linux/rwsem.h`: `down_read()` / `up_read()` for shared access; `down_write()` / `up_write()` for exclusive access. Multiple readers can hold it simultaneously; a writer is exclusive.
- [ ] Find `mmap_lock` in `include/linux/mm_types.h` — it is `struct rw_semaphore`. Answer: why is an rwsem better than a mutex for protecting the VMA list? (Hint: how often do concurrent readers vs writers occur for a process's memory map?)
- [ ] Read `include/linux/percpu-rwsem.h`: `percpu_down_write()` is used for data that is almost never written. Find one use in `kernel/` — identify what it protects.

### 2.4 Atomics

- [ ] Read `include/linux/atomic.h`: `atomic_read(v)`, `atomic_set(v, i)`, `atomic_inc(v)`, `atomic_dec_and_test(v)`. Note these are 32-bit; `atomic64_t` exists for 64-bit.
- [ ] Read `atomic_cmpxchg(v, old, new)` — returns the previous value. Answer: what does the caller check to know if the CAS succeeded?
- [ ] Read `atomic_inc_return(v)` and `atomic_fetch_add(i, v)` — understand the difference in return value.
- [ ] Answer: does `atomic_inc` imply a memory barrier? What do you need to add to ensure stores before `atomic_inc` are visible to another CPU that observes the incremented value? (Hint: look at `smp_mb__before_atomic()`.)

### 2.5 Memory Barriers

- [ ] Read `Documentation/memory-barriers.txt`, sections 1–4 (Abstract Memory Access Model through Explicit Kernel Barriers). This is the most important document in this phase.
- [ ] Answer from the doc: what is the difference between a "compiler barrier" (`barrier()`) and a CPU memory barrier (`smp_mb()`)?
- [ ] Answer: what does `smp_wmb()` guarantee that a plain store does not?
- [ ] Read `include/asm-generic/barrier.h`: find `smp_store_release(p, v)` and `smp_load_acquire(p)`. These are the preferred modern primitives. Answer: `smp_store_release` is equivalent to what combination of a store and a barrier?
- [ ] Find a producer/consumer pattern using `smp_store_release` / `smp_load_acquire` in `kernel/events/ring_buffer.c` (perf ring buffer). Identify the produced and consumed pointers.

### 2.6 RCU

RCU is the most used synchronisation mechanism in the kernel after spinlocks. Spend the most time here.

- [ ] Read `Documentation/RCU/whatisRCU.rst` section 1: "What is RCU, Fundamentally?" — the three guarantees: read-side is lock-free, updaters wait for a grace period, then free.
- [ ] Read section 2: "What is RCU's Usage?" — the three primitives: `rcu_read_lock()`, `rcu_dereference()`, `rcu_assign_pointer()`, `synchronize_rcu()`, `call_rcu()`.
- [ ] Answer: what is a "grace period"? What must be true of every CPU before a grace period can end?
- [ ] Answer: why must you use `rcu_dereference()` rather than a plain pointer load inside `rcu_read_lock()`? (Hint: compiler optimisations and `READ_ONCE`.)
- [ ] Answer: what is the difference between `synchronize_rcu()` and `call_rcu()`? Give a scenario where you cannot use `synchronize_rcu()`.
- [ ] Read `include/linux/rculist.h`: `list_add_rcu()`, `list_del_rcu()`, `list_for_each_entry_rcu()`. Answer: why does `list_for_each_entry_rcu()` use `rcu_dereference()` internally on every step?
- [ ] Open `net/core/dev.c`. Grep for `rcu_read_lock()`. Find one call site, identify the RCU-protected pointer being dereferenced, and find the corresponding `call_rcu()` or `synchronize_rcu()` on the write side.
- [ ] Read `kernel/rcu/tree.c`: find `rcu_read_lock_held()` and the quiescent-state reporting (`rcu_qs()`). Answer: what is a "quiescent state" and what kernel events constitute one? (Context switch, idle, user-mode execution.)

---

## Phase 3 — Memory Management

### 3.1 Page Allocator

- [ ] Read `include/linux/gfp.h`: understand the three GFP flag categories:
  - Zone flags: `__GFP_DMA`, `__GFP_HIGHMEM`, `__GFP_MOVABLE`
  - Action flags: `__GFP_IO`, `__GFP_FS`, `__GFP_RECLAIM`
  - Behaviour flags: `__GFP_ZERO`, `__GFP_NOWARN`, `__GFP_NOFAIL`
- [ ] Answer: you are in a hard IRQ handler and need one page. Which GFP flag combination do you use, and why can't you use `GFP_KERNEL`?
- [ ] Read `include/linux/mmzone.h`: find `struct zone` — it contains `struct free_area free_area[MAX_ORDER]`. Each `free_area` is a list of free page blocks of size `2^order`. This is the buddy allocator.
- [ ] Read `mm/page_alloc.c`: find `__alloc_pages()` — this is the top of the allocator. Find `get_page_from_freelist()` — this is the fast path. Answer: what happens when `get_page_from_freelist()` fails?
- [ ] Read `__alloc_pages_slowpath()` — find the three escalating steps it tries: kswapd wakeup, direct reclaim, OOM kill. Answer: which step is skipped when `GFP_ATOMIC` is set?
- [ ] Find `struct per_cpu_pages` in `include/linux/mmzone.h`. Answer: what is the purpose of the per-CPU page cache, and at what count does it refill from the zone free list (`high` watermark)?

### 3.2 SLUB

- [ ] Read `include/linux/slab.h`: `kmalloc(size, gfp)` is the main interface. Find the `kmalloc_caches` array — each entry is a `struct kmem_cache` for a power-of-2 size class.
- [ ] Read `include/linux/slub_def.h`: `struct kmem_cache_cpu` — it has `freelist` (next free object) and `slab` (the current slab page). This per-CPU structure makes allocation lock-free in the common case.
- [ ] Read `mm/slub.c`: `kmem_cache_alloc()` → `slab_alloc()` → `slab_alloc_node()`. Find the fast path: pop the head of `freelist`. Answer: how many atomic operations does the fast path require?
- [ ] Read `__slab_alloc()` — the slow path, called when `freelist` is empty. It tries the per-CPU partial list, then the node partial list, then allocates a new slab page from the page allocator.
- [ ] Answer: what is the difference between `kmalloc` and `kmem_cache_alloc`? When would you create a dedicated `kmem_cache` rather than using `kmalloc`?
- [ ] Find where `kfree()` calls back into SLUB. Trace: `kfree()` → `slab_free()`. Answer: what check does `kfree_hook()` perform if `CONFIG_KASAN` is enabled?

### 3.3 VMAs and Page Tables

- [ ] Read `include/linux/mm_types.h`: `struct vm_area_struct` — note `vm_start`, `vm_end` (byte addresses), `vm_flags`, `vm_file` (NULL for anonymous), and `vm_ops`.
- [ ] Read `include/linux/mm.h`: find the `VM_*` flag constants — `VM_READ`, `VM_WRITE`, `VM_EXEC`, `VM_SHARED`, `VM_GROWSDOWN` (stack). Answer: what do `VM_READ | VM_WRITE` on a VMA mean for page table permissions?
- [ ] Read `mm/mmap.c`: `do_mmap()` — the kernel implementation of `mmap(2)`. Find where it calls `vma_merge()` — merging adjacent VMAs with identical flags. Answer: why merge VMAs rather than always creating new ones?
- [ ] Read `mm/memory.c`: `handle_mm_fault()` — called from the architecture page-fault handler. It dispatches to `do_anonymous_page()` (first access to anon memory), `do_cow_fault()` (write to a shared page), or `do_read_fault()` / `do_shared_fault()`.
- [ ] Read `do_anonymous_page()` — it allocates a zeroed page with `alloc_zeroed_user_highpage_movable()` and installs it in the page table. Answer: why is the page zeroed?
- [ ] Read `do_cow_fault()` — it allocates a new page and copies the old page into it. Answer: at what point does the page table entry get updated, and what prevents another CPU from seeing a half-copied page?
- [ ] Open `arch/x86/mm/fault.c`: find `do_user_addr_fault()` — this is the x86 page fault handler. Trace the call to `handle_mm_fault()`. Answer: what does the handler do if `handle_mm_fault()` returns `VM_FAULT_OOM`?

### 3.4 Page Cache and Folios

- [ ] Read `include/linux/fs.h`: `struct address_space` — find `i_pages` (the XArray of folios), `a_ops` (the `address_space_operations` vtable), and `host` (back-pointer to the inode).
- [ ] Read `include/linux/pagemap.h`: `filemap_get_folio(mapping, index)` — looks up a folio by page index. If not found, returns an error pointer.
- [ ] Read `mm/filemap.c`: `filemap_read()` — the implementation of `read(2)` for regular files. It loops: look up folio in page cache, if missing submit readahead, wait for folio to be uptodate, copy to userspace.
- [ ] Read `mm/readahead.c`: `page_cache_async_readahead()` — find where it decides how many pages to read ahead. Answer: what heuristic does it use to grow the window?
- [ ] Answer: what is a `struct folio`? How does it differ from `struct page`, and why was the change made in 5.16?
- [ ] Find `address_space_operations` in `fs/ext4/inode.c` — the `ext4_aops` table. Find the `.readpage` / `.readahead` callback. Answer: what does it do at the bottom of the call chain? (Hint: it eventually calls `submit_bio()`.)

---

## Phase 4 — Process Model and Scheduling

### 4.1 task_struct

- [ ] Read `include/linux/sched.h`: find `struct task_struct`. It is large (~700 fields). Focus on:
  - `__state` — `TASK_RUNNING`, `TASK_INTERRUPTIBLE`, `TASK_UNINTERRUPTIBLE`, `TASK_DEAD`
  - `on_rq` — 1 if on a run queue (eligible to run), 0 if blocked
  - `prio`, `normal_prio`, `static_prio` — effective, computed, and configured priority
  - `mm` — pointer to `mm_struct` (NULL for kernel threads)
  - `files` — open file table
  - `pid`, `tgid` — process ID and thread group ID
- [ ] Answer: what is the difference between `TASK_INTERRUPTIBLE` and `TASK_UNINTERRUPTIBLE`? Give one real example of each from the kernel.
- [ ] Read `kernel/fork.c`: `copy_process()` — find where it copies the memory map (`copy_mm()`), file table (`copy_files()`), and signal handlers (`copy_sighand()`). Answer: for `fork()` vs `clone(CLONE_VM)`, which of these are shared vs copied?
- [ ] Read `kernel/exit.c`: `do_exit()` — find the sequence: release file descriptors → release mm → send SIGCHLD to parent → set state to `TASK_DEAD`. Answer: what is a zombie process, and at what point does it become one?
- [ ] Read `kernel/pid.c`: `find_task_by_vpid(pid)` — it calls `find_pid_ns()` (the hlist hash lookup you traced in Phase 1.4), then `pid_task()` to get the `task_struct`. Confirm the path connects Phase 1.4 knowledge.

### 4.2 CFS Scheduler

- [ ] Read `Documentation/scheduler/sched-design-CFS.rst` — the full document. Answer: what is `vruntime`, and what property does CFS maintain about `vruntime` across runnable tasks?
- [ ] Read `kernel/sched/sched.h`: `struct sched_entity` — find `vruntime`. Find `struct cfs_rq` — find `min_vruntime` and the `tasks_timeline` RB-tree (keyed by `vruntime`).
- [ ] Read `kernel/sched/fair.c`: `enqueue_task_fair()` — when a task becomes runnable, it is inserted into the RB-tree. Find the call to `__enqueue_entity()` and trace to `rb_add_cached()`.
- [ ] Read `pick_next_task_fair()` — it picks the leftmost node (smallest `vruntime`). Answer: why leftmost = smallest `vruntime` = the most deserving task?
- [ ] Read `task_tick_fair()` — called every scheduler tick. Find `update_curr()` which advances `vruntime`. Find `check_preempt_tick()` — when does it set `TIF_NEED_RESCHED`?
- [ ] Read `kernel/sched/core.c`: `__schedule()` — find the call to `pick_next_task()`, then `context_switch()`. Answer: what is saved/restored in `context_switch()` for a switch between two user tasks?

### 4.3 RT Scheduling Classes

- [ ] Read `kernel/sched/rt.c`: `pick_next_task_rt()` — it picks the highest-priority runnable SCHED_FIFO/SCHED_RR task. Note it uses a priority bitmap (`struct rt_prio_array`) for O(1) selection.
- [ ] Answer: what is the difference between `SCHED_FIFO` and `SCHED_RR`? In which case does the kernel preempt a running RT task in favour of another RT task at the same priority?
- [ ] Read `kernel/sched/deadline.c`: skim the first 100 lines for the CBS (Constant Bandwidth Server) model — each task has `runtime` and `period`. Answer: what happens when a `SCHED_DEADLINE` task overruns its runtime budget?

---

## Phase 5 — Filesystems

### 5.1 VFS Layer

- [ ] Read `Documentation/filesystems/vfs.rst` — the full document. Answer: what are the four main VFS objects (super_block, inode, dentry, file) and how do they relate to each other?
- [ ] Read `include/linux/fs.h`: memorise the four vtable types:
  - `struct super_operations` — mount/unmount/sync
  - `struct inode_operations` — create/lookup/link/unlink/mkdir
  - `struct file_operations` — open/read/write/ioctl/mmap
  - `struct address_space_operations` — readpage/writepage/direct_IO
- [ ] Read `include/linux/dcache.h`: `struct dentry` — find `d_inode` (NULL = negative dentry), `d_parent`, `d_name`, `d_subdirs` (list of children). Answer: what is a negative dentry and why does the kernel keep them?
- [ ] Read `fs/dcache.c`: `d_lookup(parent, name)` — it computes a hash of `(parent, name)` and walks a `hlist` in `dentry_hashtable`. Answer: what lock protects dentry hash bucket lookup?
- [ ] Read `fs/namei.c`: `link_path_walk()` — the core path resolution loop. It processes one component at a time, calling `lookup_fast()` (dcache) then `lookup_slow()` (filesystem) for each. Trace one iteration.
- [ ] Answer: `open("/tmp/foo", O_CREAT)` — at what point does VFS call the filesystem's `inode_operations->create()`? Trace from `do_sys_openat2()` to the `create` call.
- [ ] Read `fs/read_write.c`: `vfs_read()` — it calls `file->f_op->read_iter()`. Answer: what is `struct kiocb` and what does it carry through the read path?

### 5.2 ramfs

**Read every line of both files.** ramfs is ~300 lines total and is the simplest complete filesystem.

- [ ] Read `fs/ramfs/inode.c`: `ramfs_get_inode()` — find where it sets `i_op` and `i_fop` differently for directories vs regular files vs symlinks.
- [ ] Read `ramfs_fill_super()` — find where it creates the root inode and root dentry. Answer: what does `d_make_root()` do?
- [ ] Read the `ramfs_inode_operations` table — it uses `simple_lookup()` from `fs/libfs.c`. Open `fs/libfs.c` and read `simple_lookup()`. Answer: why does ramfs not need to implement its own `lookup()`?
- [ ] Read `fs/ramfs/file-mmu.c`: find the `ramfs_file_operations` table. Note it uses `generic_file_read_iter` and `generic_file_write_iter`. Answer: where does the file data actually live in ramfs? (Hint: there is no block device — look at the `address_space_operations`.)
- [ ] Answer: what happens to ramfs data when the system is rebooted? What prevents ramfs from filling all RAM (and what does tmpfs add to solve this)?
- [ ] Read `mm/shmem.c`: `shmem_fill_super()` — compare with `ramfs_fill_super()`. Find where tmpfs adds the size/inode limits. Answer: what struct field stores the size limit?

### 5.3 ext4

- [ ] Read `fs/ext4/super.c`: `ext4_fill_super()` — find where it reads the superblock from disk (`__read_mmp_block` / `sb_bread`), validates the magic number (`EXT4_SUPER_MAGIC`), and sets up the journal (`ext4_load_journal()`).
- [ ] Read `fs/ext4/inode.c`: `ext4_get_block()` — maps a logical block number to a physical block. Find where it branches between extent-based and block-map-based lookup.
- [ ] Read `fs/ext4/extents.c`: `ext4_ext_map_blocks()` — extent tree lookup. Find `ext4_find_extent()` — it walks the B-tree of extents in the inode. Answer: where does the root of the extent tree live? (Hint: it is in the inode itself, not a separate block.)
- [ ] Read `fs/ext4/namei.c`: `ext4_lookup()` — find where it uses the HTree (hashed B-tree directory index) via `ext4_dx_find_entry()`. Answer: what is the HTree an optimisation over?
- [ ] Answer: what is the journal (JBD2) for? Find `ext4_journal_start()` in `fs/ext4/ext4_jbd2.h` — what does a "transaction" represent? What happens to uncommitted transactions if the system crashes?

---

## Phase 6 — IRQ Subsystem Deep Dive

*You have existing notes from investigation work. Use this phase to fill gaps
and connect the pieces into a coherent mental model.*

- [ ] Read `kernel/irq/irqdesc.c`: `alloc_desc()` — find the fields of `struct irq_desc` that you had not seen before in your investigation notes. In particular: `irq_data`, `handle_irq`, `action` list.
- [ ] Read `kernel/irq/handle.c`: `handle_level_irq()` vs `handle_edge_irq()` — answer: what is the difference in when the IRQ chip's `mask`/`unmask` is called between level and edge triggered interrupts?
- [ ] Read `kernel/irq/manage.c`: `request_threaded_irq()` — find where the threaded handler kthread is created. Answer: what scheduling class does the IRQ thread run in by default?
- [ ] Read `kernel/irq/irqdomain.c`: `irq_domain_add_linear()` — answer: what problem does an IRQ domain solve? (Hardware IRQ numbers are not globally unique — e.g., two PCI devices can both have "IRQ 0".)
- [ ] Read `kernel/irq/msi.c`: `msi_domain_alloc_irqs()` — find the path from MSI vector allocation to the call that writes the MSI address/data into the device's PCI config space.
- [ ] Read `kernel/softirq.c`: `do_softirq()` — find the bitmask loop over `softirq_vec[]`. Answer: what prevents softirq starvation of process context? (Hint: find `MAX_SOFTIRQ_TIME` and the `ksoftirqd` handoff.)

---

## Phase 7 — Networking Stack

*You have detailed notes on the SolarFlare EF100 RX path. Use this phase to
understand the generic kernel networking layer that path feeds into.*

- [ ] Read `include/linux/skbuff.h`: `struct sk_buff` — find and understand:
  - `head`, `data`, `tail`, `end` — raw buffer boundaries (byte pointers / offsets)
  - `len` vs `data_len` — linear data length vs paged (fragmented) data length
  - `transport_header`, `network_header`, `mac_header` — stored as offsets from `head`
  - `cb[48]` — the per-layer control block; each protocol layer writes its own metadata here
- [ ] Answer: what does `skb_pull(skb, len)` do? What does `skb_push(skb, len)` do? When is each used (TX vs RX path)?
- [ ] Read `net/core/dev.c`: `netif_receive_skb_internal()` — find the protocol demux: it walks `ptype_all` (sniffers like tcpdump) and `ptype_base` (protocol handlers keyed by `ETH_P_*`). Connect this to your SolarFlare notes at step 9.
- [ ] Read `net/ipv4/ip_input.c`: `ip_rcv()` — find the Netfilter hook (`NF_INET_PRE_ROUTING`). Answer: what would happen here if `iptables -A INPUT -j DROP` were active?
- [ ] Read `net/ipv4/tcp_input.c`: `tcp_v4_rcv()` — find the lookup of the socket (`__inet_lookup_skb()`), then the dispatch to `tcp_rcv_established()` or `tcp_rcv_state_process()`. Answer: what state does a socket need to be in to reach `tcp_rcv_established()`?
- [ ] Read `net/xdp/xsk.c`: `xsk_rcv()` — find where an incoming frame is placed into the UMEM ring. Answer: what happens if the userspace consume ring is full and the kernel tries to deliver a packet?
- [ ] Read `Documentation/networking/af_xdp.rst` — the full document. Answer: what is the difference between "copy mode" and "zero-copy mode" in AF_XDP?

---

## Phase 8 — io_uring

- [ ] Read `Documentation/filesystems/io_uring.rst` — the full overview. Answer: what are the Submission Queue (SQ) and Completion Queue (CQ)? Who writes to each, kernel or userspace?
- [ ] Read `io_uring/io_uring.c`: `io_uring_setup()` — find where the SQ and CQ rings are allocated and memory-mapped so both kernel and userspace can access them without copying. Answer: what `mmap` offset constants are used?
- [ ] Read `io_uring_enter()` — find the two modes: submitting SQEs (bit `IORING_ENTER_SQ_WAKEUP`) vs waiting for CQEs (`IORING_ENTER_GETEVENTS`).
- [ ] Read `io_uring/sqpoll.c`: `io_sq_thread()` — this is the SQPOLL kernel thread. Answer: what does it do in a tight loop, and what is the timeout after which it goes to sleep if there are no new SQEs?
- [ ] Read `io_uring/net.c`: find `IORING_OP_RECV` handling. Answer: how does io_uring reuse the existing `sock_recvmsg()` path rather than reimplementing socket receive?
- [ ] Read `io_uring/napi.c`: `io_napi_busy_loop()` — answer: what is "busy-poll" in this context, and when does it help reduce latency compared to waiting for a socket wake-up?
- [ ] Read `io_uring/zcrx.c`: `io_zcrx_recv()` — find where NIC buffer pages are mapped directly into the userspace ring without copying. Answer: what prevents the NIC from overwriting the buffer while userspace is reading it?
