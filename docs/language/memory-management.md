Great, I’ll research memory management approaches—both theoretical and in use—for designing a new low-level language
focused on manual memory management with strong safety guarantees, flexibility, and ease of use. This will include
borrow-checking, regions, arenas, linear types, hybrid models, and lessons from existing systems like Rust and Vale.
I’ll factor in the resources you provided as well.

I’ll get back to you with a detailed comparison and recommendations tailored to your language’s goals.

# Memory Management Strategies for a Safe, Expressive Systems Language

Designing a low-level language for a secure and expressive systems kernel requires carefully balancing **memory safety
**, performance, and **ergonomics**. We need to **avoid garbage collection** while preventing errors like
use-after-free, double free, or leaks, and also make the language **approachable for developers** with flexible manual
control when needed. Below we explore various memory management models – from manual allocation to sophisticated type
systems – and compare their use in real languages (Rust, Vale, Cyclone, Pony, Zig, etc.), highlighting trade-offs in
safety, usability, compiler complexity, and user control. Finally, we suggest a hybrid model that could meet the design
goals of a “Big Iron” kernel language.

## Requirements and Challenges

**Memory Safety without GC:** Tracing garbage collection (GC) is a common route to memory safety, but it introduces
runtime overheads (extra memory use, nondeterministic pauses). We seek **memory safety** (no dangling pointers or
unchecked memory errors) *without* relying on a traditional GC. This means exploring static compile-time checks and
alternative runtime strategies.

**Prevent Use-After-Free & Leaks:** The language must *prevent* common memory bugs – **use-after-free**, double freeing,
buffer overruns, **memory leaks**, etc. In safe Rust, for example, use-after-free and double free are eliminated by
design (ownership prevents freeing memory still in use). Memory leaks are also undesirable; while Rust considers leaks
safe, a systems kernel likely needs to minimize leaks for long-running stability.

**Ease of Use vs Manual Control:** Low-level systems developers often demand **fine-grained control** over memory (for
performance and special cases), but fully manual memory management (like C’s `malloc`/`free`) is error-prone. Our
language should provide **high-level safety by default** with ergonomic features (so the average user doesn’t constantly
fight the compiler), *yet* allow opting into manual control or specialized allocators when needed. Achieving this often
means offering multiple memory management paradigms within one language or a highly flexible one.

**Deterministic Performance:** In a kernel context, **predictable performance** and low latency are crucial. Thus, **no
stop-the-world pauses** or high runtime overhead. Techniques like reference counting or region-based deallocation can be
more deterministic than tracing GC. We will weigh each model’s impact on throughput and latency.

**Compiler and Implementation Feasibility:** Approaches like Rust’s borrow checker provide strong guarantees but come
with complex compiler logic (lifetime inference, extensive error checking) and a steep learning curve for programmers.
We must consider how complex a memory model is to implement and use – e.g. is it worth designing a new static analysis
or type system from scratch, or leveraging simpler runtime checks?

With these goals in mind, let’s examine the memory management models that could be candidates or components for our new
language.

## Memory Management Models and Their Trade-offs

### Manual Memory Management (Malloc/Free)

**Description:** Manual memory management means the programmer explicitly allocates and frees memory. Languages like C
and older C++ rely on this model. The advantage is **complete control** and zero overhead beyond the allocations
themselves – no automatic tracking or collector running. This model can be *very efficient* when used carefully, and
it’s flexible enough to implement any allocation scheme.

**Trade-offs:** Pure manual management is notoriously error-prone. The burden is on the programmer to free every
allocation exactly once at the right time. Mistakes lead to **memory leaks (if never freed)** or **use-after-free** and
double-free bugs (if freed too early or twice), causing undefined behavior and security vulnerabilities. There are *no
built-in safety nets* here.

Real-world systems code often compensates by establishing conventions or patterns. For example, C programmers document
which function “owns” a heap object and must eventually free it. C++ introduced RAII (Resource Acquisition Is
Initialization) and smart pointers (like `unique_ptr` and `shared_ptr`) to automate calling `delete` in destructors,
which helps prevent leaks and double frees by tying object lifetime to scope.

**Pros:** maximal flexibility and performance (no runtime overhead, memory usage fully under program control).

**Cons:** extremely unsafe by default – **error-prone** and no automatic prevention of memory misuse. Requires
discipline or additional tooling for safety.

**Notable Implementations:** **Zig** is a modern systems language that leans toward manual memory management. Zig has no
built-in garbage collector; instead, it encourages passing allocators explicitly and using defer statements to free
memory, making memory lifetime explicit but still developer-managed. Zig does perform some safety checks (e.g. bounds
checking in safe mode), but freeing memory is up to the programmer – meaning it has similar risks to C if not carefully
handled. **C** and **C++** (without using smart pointers) are the classic examples – unsafe by default. **Odin**,
another systems language, also favors manual and arena allocators, giving control to the programmer.

**Relevance to our design:** Pure manual management alone is not acceptable for our goals, since we need strong memory
safety guarantees. However, **patterns from manual management (ownership conventions, RAII)** can be incorporated into
safer systems. Indeed, the concept of **ownership** – tracking which code “owns” an allocation – is the foundation of
more advanced compile-time checks.

### RAII and Deterministic Destruction

Before moving on, it’s worth noting **RAII (deterministic destruction)** as a pattern that augments manual management.
In RAII, objects free their resources in their destructor automatically when they go out of scope. This is used in C++
and Rust (where it’s called the Drop trait). RAII by itself doesn’t enforce correct *aliasing* rules, but it *does*
automate timely freeing of memory. For example, a `unique_ptr<T>` in C++ will free its pointed object when it goes out
of scope – if you never copy that unique pointer (maintaining single ownership), you get safety similar to Rust’s
ownership model, but **without a compiler-enforced guarantee**. RAII makes freeing **easier and less error-prone**,
contributing to ease-of-use, but without a strict compiler check, a programmer could still violate uniqueness by
aliasing a raw pointer. RAII will free once, and an outstanding raw pointer becomes a dangling reference (undefined
behavior).

In summary, RAII helps manage *when* memory is freed (preventing leaks) but not *what* is pointing to it at that time.
To truly prevent use-after-free, we need static or runtime checks on pointer usage in addition to RAII’s timely freeing.

### Ownership and Borrowing (Compile-Time Affine Types)

**Description:** The **ownership/borrowing model**, epitomized by Rust, enforces at compile time that each piece of
memory has a unique owner, and that *aliases* (borrows) are tightly controlled. In Rust, when an object is created, one
variable “owns” it, and when that variable goes out of scope, the object is freed. You can temporarily **borrow**
references to the object, but the rules (the *borrow checker*) ensure you cannot use a reference after the original is
freed, nor have conflicting mutable accesses. This provides *memory safety through compile-time verification*, avoiding
use-after-free and data races without a garbage collector.

Rust’s system is essentially an **affine type system** (similar to linear types but allowing values to be dropped
unused). It ensures at compile time that every allocated object is freed exactly once (or “moved” to an equivalent
owner) and no live references outlast the object’s lifetime. In practice, this means many memory errors are caught as
compile errors.

**Trade-offs:** This model yields performance comparable to manual memory management (no runtime GC overhead) and strong
safety guarantees (almost on par with a high-level safe language). However, it comes at the cost of **language
complexity and restrictions** that programmers must adapt to. The Rust borrow checker introduces strict rules: for
example, you cannot form certain cyclic data structures without extra indirection (like using `Rc<RefCell>`), and
patterns like interior pointers, callbacks that outlive their owner, or intrusive linked structures can be hard to
express. Developers sometimes struggle with compiler errors until they internalize the ownership model.

Yorick Peterse, reflecting on designing the Inko language, noted that if we demand sound, powerful memory safety *with
no runtime checks*, a compile-time borrow checker is likely **inevitable**. Rust demonstrates this – it eliminates
entire classes of bugs. But he also warns that implementing a sound borrow checker *requires* explicit lifetime/region
annotations or sophisticated inference, which can lead to complexity “leaking” into the language’s surface (lots of
lifetime parameters and rules). Indeed, Rust’s lifetimes sometimes show up in types and can complicate API designs. The
compiler is also quite complex internally to handle all edge cases soundly.

**Pros:** Excellent **performance** (no GC pauses, minimal overhead), strong **memory safety** (no use-after-free,
etc.), and even data-race freedom in concurrent code as an added benefit (Rust’s model ensures only one thread can
mutate a value or else only reads occur, preventing data races). Memory can often be allocated on the stack or inlined
in structures (no need for everything on the heap), which improves cache usage.

**Cons:** **Ergonomics and flexibility** are the main issues. The strict rules forbid certain patterns or require
workarounds (like reference-counted wrappers or unsafe code). For example, **graphs with cycles** or tree nodes with
parent pointers are not trivial – you must use reference-counting or unsafe pointers for those. Observers (callbacks)
that hold references into data are tricky since Rust doesn’t allow a long-lived reference if the observed data might
mutate or drop. These constraints sometimes force design changes. The learning curve for newcomers is significant, and
even experienced users run into borrow-checker errors that require non-intuitive fixes.

**Notable Implementations:** **Rust** is the flagship example. **Cyclone**, a research language (early 2000s), was a
safe C dialect that introduced ownership and region ideas; Cyclone’s approach was a precursor in some ways – it
prevented dangling pointers by tracking lifetimes of allocated regions and disallowing pointers that outlive their
referent. Cyclone’s system was less flexible than Rust’s (it often required manual region annotations), but it proved
the viability of static checking in C-like languages. **Austral** is a newer language that uses a Rust-like borrow
checker *plus* linear types, which enforce that certain values must be used (not just dropped) to ensure actions
happen (Austral can guarantee, for instance, that a cleanup function is called by making the resource a linear type).
This concept of **linear/affine types for liveness** (termed “higher RAII” by Vale’s designers) ensures not only memory
is freed, but any programmer-intended finalization occurs exactly once.

Given the power of Rust’s model, we likely want to incorporate *some form* of **ownership and borrowing** in our new
language, but perhaps in a more ergonomic or blended way to mitigate the downsides. As Yorick Peterse concluded, a
borrow checker might be *“inevitable”* for strong safety – but we should explore if it can be simplified or combined
with other techniques.

### Linear and Affine Type Systems

**Description:** **Linear types** (and their weaker cousin, **affine types**) come from type theory and enforce a form
of single ownership at the type level. A *linear* type means a value **must be used exactly once** – it cannot be copied
or discarded. An *affine* type is similar but allows a value to be discarded (used at most once). These types ensure
that resources are neither duplicated nor forgotten: if something is linear, the compiler won’t let you drop it without
performing the designated “use” (like calling a destructor), preventing leaks and double frees by design.

In practice, Rust’s ownership can be viewed as an **affine type system** (you can drop a value by not using it again,
which is allowed; if it were truly linear, you’d have to explicitly call some function to consume it). Some experimental
languages and research use *strict linear types*: for instance, **Clean** (a functional language) used *uniqueness
types*, which are a form of linearity ensuring only one reference to a value exists, allowing destructive in-place
updates in a functional setting. **Austral** (mentioned above) uses linear types to enforce certain actions (like
freeing or other side effects) are done. Even Haskell now has an extension for linear types to guarantee functions use
their arguments exactly once (useful for resource management in pure code).

**Trade-offs:** Linear/affine types can guarantee **memory and resource cleanup** at compile time. If every allocation
is a linear resource, the program cannot compile unless you free (consume) it exactly once along every code path – thus
leaks are eliminated (the compiler forces you to handle the resource). They also prevent aliasing by default: if
something is linear, you can’t have two pointers to it at the same time (unless temporarily “borrowed”, which
essentially converts the linear resource into a nonlinear one within a scope). This model can therefore prevent not just
memory errors but logical errors (e.g., ensure a file handle is closed).

The downside is **inflexibility and verbosity**. Not every piece of data in a program fits the model of “exactly one
owner, used once.” Forcing linear types everywhere would make programming very cumbersome – you’d be inserting a lot of
`.clone()` or explicit copy operations when you really want to share read-only data, for example. Vale’s developer notes
that one *can* write entire programs in a move-only (affine) style, but it requires “acrobatics” like removing an item
from a data structure to inspect it, then putting it back, since you can’t have two aliases to it. This is why most
practical systems (Rust included) blend affine types with some borrowing or other escape hatches.

**Notable Implementations:** As mentioned, **Rust** and **C++** effectively use affine types for their core (move
semantics, unique ownership). **Vale** introduces what they call a *“linear aliasing”* model, giving the programmer the
option to treat certain structures as linear or affine as needed. **Clean** and **Linear Haskell** explored these in
functional languages. **Austral** builds its entire safety on linear + borrow types, leaning toward a very pure linear
type discipline for memory safety.

In our language, linear/affine types could be used as a *foundation* (ensuring unique ownership by default). We might
say that by default, all heap allocations are affine – you can transfer ownership but not copy freely. This gives us a
baseline of no double frees (because only one thing can free). To allow more flexible patterns, we’d layer other
strategies on top, which we’ll discuss below.

### Region-Based Memory Management

**Description:** **Region-based management** allocates objects in a *context* (region or arena) and frees all objects in
that region at once. A classic example is stack allocation: local variables exist in the “region” of a function call (
the stack frame) and are freed automatically when the function returns. We can generalize this idea – for instance,
allocate a bunch of related objects in a region and destroy the entire region when it’s no longer needed. This approach
was studied in the 90s for ML and others (Memory *Region Inference* by Tofte and Talpin), and implemented in languages
like **Cyclone** and **MLKit** (an ML variant without GC, using inferred regions).

**Trade-offs:** Region allocation can be extremely efficient: allocation is often just pointer bumping (if the region is
a large contiguous chunk, you just bump an offset for each new object), and freeing is one bulk operation (reset the
pointer or free the whole chunk). It also provides **deterministic lifetime**: objects live at most as long as their
region. If we can ensure no references outlive their region, we prevent dangling pointers by construction.

The challenge is **inferring or managing regions** in real programs. Some lifetimes don’t nest neatly. For example, two
objects may have overlapping but not nested lifetimes – one common region could last too long (causing memory usage to
spike) or too short (freeing while some references still needed it). If the programmer must manually annotate regions,
it can become complex to maintain. Cyclone required programmers to sometimes annotate functions with region parameters (
like templates for lifetimes) so that the compiler knows which region an object belongs to. This is analogous to Rust’s
lifetimes, but tied to manual deallocation of region memory.

In practice, region systems can suffer either **leaks** (if regions are coarse-grained and hold objects long after
they’re needed) or **safety issues** if not checked strictly. Cyclone and others solved safety by having the compiler
check that no pointer to region A is stored in region B that outlives A, etc., effectively a simpler form of lifetime
check. Ada’s SPARK subset has a similar rule: a pointer cannot point to an object with a shorter lifetime (no inward
pointing to deeper stack frames). This prevents dangling stack references statically.

**Pros:** Very fast allocation/free, batch deallocation can simplify reasoning in some cases, and no background GC
thread or counters. If integrated with the type system, **use-after-free can be prevented** by compile-time checks that
a reference doesn’t outlive its region. Good fit for certain workloads (e.g. phase-oriented computations, request
handling where you allocate per request and free everything at end).

**Cons:** Less flexible lifetimes; can be **wasteful** if memory isn’t freed until much later than necessary (objects
die, but region lives on until a bigger scope ends). Manual region annotation or inference algorithms add complexity. If
not carefully designed, can restrict programming style (for instance, long-lived global regions vs short-lived local
regions – moving data between them safely can be tricky).

**Notable Implementations:** **Cyclone** allowed explicit region annotations and had a static analysis to ensure safety.
**MLKit** (an ML implementation) successfully inferred regions for many allocations, eliminating GC in some cases, but
it would fall back to a global heap if it couldn’t statically prove region safety for some data. **Verona**, a Microsoft
research language, combines region ideas with others – it has *“cgroups”* or compartments that each can use different
strategies (some might be collected, some freed en masse). **Ada/SPARK** (a safety-critical Ada subset) enforces that
heap allocations are tied to accessibility levels to avoid dangling references. **Rust** doesn’t have regions in the
same sense for memory, but its lifetimes accomplish a similar goal for safety (ensuring no reference outlives what it
points to). Also, **Zig** and **Odin** encourage the use of arena allocators for efficiency; Odin even lets you inject
an arena allocator into procedures automatically.

For our language, we can incorporate **arena/region support**: for example, allow the programmer to create an `Arena`
context and allocate objects into it, then statically prevent those objects’ references from escaping the arena’s scope.
This gives manual control (the user decides to use an arena for a bunch of objects) but with safety checks. It’s a
proven concept (Cyclone did it, and Verona is exploring it). It satisfies “manual control when desired” – the user can
opt into an arena for performance or deterministic deallocation – without sacrificing safety.

### Arena Allocation (Stack/Arena Pools)

*Arenas* are a special case of region management often used explicitly by systems programmers. We mention them
separately because they are very common in practice and can be a *subset* of features in a language. An **arena** is a
chunk of memory from which you allocate many objects and then free them all at once. This is essentially manual memory
management but at a coarse granularity (freeing in bulk). It trades memory consumption for speed by avoiding per-object
deallocations and possibly improving locality.

Languages like **Zig, D, Odin** support arenas or have standard libraries for them. It’s “more of a memory management
approach than a memory safety approach” in itself – by default, using arenas doesn’t magically prevent you from taking
an object out of the arena and using it after the arena is freed. However, with language support, **arena allocation can
be made memory-safe**. Cyclone’s region system and Ada’s rules, as noted, *track which pointers point into which arena*
and ensure you don’t use them after the arena is destroyed. This could be done via compile-time checks (similar to
lifetimes) or even runtime tagging of pointers to arenas (with checks).

Arenas are great for **ease-of-use in certain domains** (you allocate a lot of objects related to each other, then one
call frees all, relieving the programmer of individually freeing each). They also satisfy the “no GC, manual control”
criterion – the programmer decides when to create and destroy arenas. We will likely include arenas as an *option* in
the new language, with static analysis to prevent dangling references outside the arena scope (this is similar to how
some safe languages handle stack references or pools).

### Reference Counting (with or without Cycle Collection)

**Description:** **Reference counting (RC)** is a form of automatic memory management without a traditional GC pause.
Each object maintains a count of references to it; when the count hits zero, the object is destroyed immediately. This
is used in many environments: **Swift and Objective-C’s ARC**, **Python** (which combines ref counting with a
cycle-detecting GC), **Nim’s default ARC/ORC**, and **C++** `shared_ptr`. Reference counting provides a deterministic
destruction (so it plays nicely with RAII patterns – objects free as soon as last reference is gone) and avoids long
pauses by doing little work spread out (increment/decrement operations).

**Trade-offs:** Simplicity is a big plus – it’s easy to understand for programmers. Memory is reclaimed promptly and
predictably, which is great for real-time concerns (no surprise pauses). Also, **immutable or acyclic data** works
extremely well with RC.

However, naive reference counting has several downsides:

* **Overhead:** Every pointer assignment or copy requires adjusting counters. In multithreaded contexts, these
  adjustments are typically atomic operations, which can be costly (though techniques like deferred counting or
  thread-local buffers exist to mitigate this).
* **Cache performance:** The counter updates can cause a lot of traffic on shared memory (ping-pong cache lines between
  cores, etc.). It can be slower than tracing GC in throughput for large heaps because of these continual updates.
* **Cycles:** Reference counting alone cannot reclaim cyclic structures (two objects referring to each other will never
  drop to count zero). Solutions include *cycle collectors* (like Python’s GC for cycles or JavaScript’s approach when
  using reference counting historically), or requiring the programmer to break cycles manually via weak references. This
  adds complexity or potential for leaks if not handled.

Despite these issues, reference counting is often considered *“simpler and uses less memory”* than tracing GC, and is a
reasonable compromise for many applications. Languages like **Swift** prove that RC can be made relatively seamless to
the user, though under the hood it introduces ARC optimization in the compiler.

**Pros:** **Deterministic cleanup**, fairly straightforward model, works well with hybrid strategies (you can mix RC
with other methods). No big pause events; memory usage is generally more predictable.

**Cons:** **Performance overhead** (especially atomic ops in multi-threaded scenarios), potential for memory leaks with
cycles unless extra mechanisms are introduced. Also, RC doesn’t prevent *use-after-free* in unsafe languages – it will
free at count 0, but if a raw pointer was still lingering, that pointer becomes dangling (RC by itself doesn’t track raw
uncounted references). In safe implementations, you typically don’t allow raw unchecked pointers; everything must bump
the count, or you use weak references that are set to null when target is freed.

**Notable Implementations:** **Swift** (ARC for class instances), **Nim** (default ARC and an alternate ORC which adds
cycle collection), **Python** (ref counts + cycle GC), **Rust** (provides `Rc`/`Arc` smart pointers as library types –
not baked into the language, but widely used when shared ownership is needed). Rust notably allows mixing: you use the
borrow checker for most cases, and if you need shared ownership, you opt in by wrapping in `Rc`. This is a form of *
*hybrid model**: static checks where possible, runtime counting where necessary. We’ll discuss hybrid approaches more
soon.

One interesting development is **optimizing reference counting with compile-time analysis**. The language **Koka**
introduced *Perceus*, a system that uses compile-time knowledge to insert and elide reference count operations,
achieving performance close to GCs but with deterministic destruction. Essentially, the compiler figures out where
increments/decrements are redundant (like a value created and used entirely in a function can be allocated with count =
1 and freed without intermediate updates). Vale’s author also points out this direction, noting that *“there’s a whole
spectrum between \[ref counting and tracing GC]”* and that Koka’s approach eliminates most runtime overhead of RC. This
increases compiler complexity but improves runtime speed significantly.

For our new language, reference counting could be offered as an **opt-in mechanism** for certain scenarios – e.g.,
perhaps the language by default uses ownership (no overhead) but if the user wants multiple owners of something, they
can mark it as `refcounted` and the compiler either automatically inserts RC or provides a library type for it. This is
exactly how Rust’s `Arc<T>` works, or C++ `shared_ptr`. The key is to integrate it in a way that’s *visible and
controllable* by the programmer (for transparency), yet potentially optimized by the compiler (to reduce overhead where
possible). We might also integrate **cycle handling** or encourage patterns that avoid cycles (like using weak
references or observer lists that don’t strongly hold objects).

### Hybrid and Novel Memory Models

No single technique is perfect; modern safe systems languages often **combine approaches** to get the best of each.
Let’s examine some hybrid strategies and novel ideas that have emerged, which could inspire our language’s design:

* **Ownership + Reference Counting:** We already mentioned how Rust blends them: the primary model is
  ownership/borrowing (zero cost, static enforcement), and when you *need* shared mutable state or aliases that outlive
  their owners, you deliberately switch to using `Rc`/`Arc` with `RefCell` (interior mutability). This incurs runtime
  overhead and the possibility of runtime borrow errors (if you misuse RefCell) or leaks (if you create reference
  cycles), but it’s opt-in. This hybrid lets you choose safety/performance trade-offs on a case-by-case basis. Many
  languages can similarly let the user choose between unique or shared pointers.

* **Borrow Checking + GC/Arena**: Some projects attempt to use static borrowing for most of the program and a GC for the
  few parts that don’t fit (for example, a garbage-collected “escape hatch” for complex cyclic structures or long-lived
  graphs). **Verona** is interesting here: it introduces the notion of regions that can each have their own memory
  management strategy. One region might be collected with a concurrent collector, another might be freed manually (bump
  allocator, freed all at once), and the key is the language ensures references don’t cross regions unsafely. Verona
  essentially allows mixing approach per region: the user has *fine-grained control* over *when and where* GC happens.
  In a systems kernel context, this could mean mostly manual/RAII style memory, but perhaps certain subsystems could use
  a GC region if that simplifies something non-performance-critical (though the requirement says avoid GC, so perhaps
  not).

* **Generational References (Use-After-Free Guards):** A novel approach used by **Vale** is **generational references**.
  This is like a software-implemented memory tagging system: every allocation carries a generation tag, and any
  non-owning pointer must hold the correct generation to be valid. When an object is freed, its generation is
  incremented, so any stale pointer’s tag won’t match and the runtime can catch the use (turning it into a safe
  error/abort instead of memory corruption). Essentially, dereferencing a non-owning pointer involves an *assertion*
  that the object is still alive. Vale’s design makes owning pointers normal (no overhead) and only *borrowed* pointers
  carry a tag and incur a check. This is somewhat like hardware memory tagging (e.g., ARM’s ARMv8.5-A memory tagging or
  the HWASan model) but done in software with 64-bit tags for reliability.

  The **advantage** of generational refs is flexibility: you can freely create aliases (raw pointers) and you won’t get
  compile errors – if you misuse them (use-after-free), the bug is caught at runtime deterministically. It’s a bit like
  having a very fast memory sanitizer always on. Vale’s approach even *optimizes away* most checks by using **regions**:
  if you mark a region of memory as immutable for a duration, pointers into it don’t need to be checked during that
  time. This can remove virtually all the overhead in many cases. The Vale team claims this blend gives *C++’s
  architectural flexibility with Rust’s safety, while being simpler than both*. The simplicity comes from not having an
  elaborate borrow checker or lifetime syntax – the rules are more runtime-oriented and thus easier to use at the cost
  of those occasional checks.

  **Drawbacks:** generational references make pointer sizes larger (non-owning pointer = pointer + generation, often 128
  bits total) and consume space for the generation counter in each object. There’s a runtime cost to each checked
  access (though if optimized well and with regions, this can be minimal). It’s also a relatively new idea, so compilers
  and optimizers haven’t matured around it yet. But it does directly address **use-after-free** in a way that doesn’t
  require GC or a strict borrow checker, trading some performance for a simpler model.

* **Constraint References (Unique + Checked Aliases):** Another hybrid pattern (used in some game development
  communities and an old language called **Gel**) is what Evan (Vale’s author) calls *constraint references*. In this
  model, each object has at most one *owning* pointer, but you can have any number of *borrowed* pointers. However, when
  the owner goes to free the object, the runtime will **check if any borrowed pointers are still extant**. If yes, that
  indicates a logic bug – in debug mode it might crash or log an error instead of actually freeing and causing a
  use-after-free. Essentially it’s a runtime assertion that no outstanding references exist when freeing. This is
  simpler than a full Rust borrow checker (no compile-time enforcement of borrow lifetimes, just a runtime fail-safe).
  It supports patterns like graphs, intrusive data structures, etc., which static borrow rules might reject, but one
  must be careful to ensure those references are gone to avoid runtime errors. It’s a bit like a delayed reference
  count: you don’t count every increment/decrement, you only verify at destruction time that count is zero. The downside
  is if such a check fails in production, you either leak (skip freeing) or crash – not great for a kernel. So this is
  more of a debug-time safety net than a fully safe model, unless combined with other restrictions.

* **Hardware-Assisted Memory Safety:** Technologies like **CHERI** (Capability Hardware Enhanced RISC Instructions)
  provide fat-pointer capabilities in hardware. Pointers carry bounds and permission metadata, and the hardware traps on
  out-of-bounds or unauthorized access, enforcing spatial safety. Temporal safety can be added by versioning
  allocations (CHERI has an approach called **Cornucopia** that avoids reuse of memory until it’s safe). If we were
  designing for a platform with CHERI, we could offload a lot of safety to hardware: use plain C-like manual management
  but count on the CPU to catch any misuse. This is attractive for performance (capability checks are optimized in
  hardware) and strong security (if hardware says you can’t use a pointer after free, you can’t). However, CHERI is not
  yet mainstream; it’s still experimental (RISC-V, ARM prototypes). Relying on it would limit the language to those
  platforms or require a software emulator for others.

* **Never-free / region-reuse models:** A tongue-in-cheek “strategy” is **never freeing memory** (just let the OS
  reclaim on program exit). Obviously not viable for long-running systems like kernels! But it underlies some real
  patterns: for instance, **MMM++ (Mostly Manual Memory++**) as Vale’s blog calls it, where you allocate from pools or
  global arrays and never reuse memory for a different purpose. Many embedded and safety-critical systems do this: they
  have a fixed pool for objects of type X, and they never free them for reuse as a different type, avoiding temporal
  bugs at the cost of potential fragmentation or memory use. This is safe (no dangling pointers because memory never
  gets reallocated to a new object with a different meaning). Some high-performance servers and games follow similar
  patterns (reuse object slots to avoid allocation churn). We could incorporate this as an optional mode (e.g., user
  could mark that an allocation is from a permanent pool, so the language knows it won’t be freed until program end or
  pool reset, thus references to it are always valid).

* **Interaction Nets (for immutable data):** A very specialized technique, interaction nets (used in the HVM project for
  Haskell) can manage **purely immutable** data extremely efficiently without GC or RC. This is more relevant to
  functional languages and might not directly apply to a systems kernel (which will have lots of mutable state). The
  idea here is more about graph rewriting optimization and reuse of nodes. It’s probably beyond our scope, except if the
  kernel language had a pure functional sublanguage.

In summary, hybrids try to mix the **safety of static checks** with the **flexibility of runtime techniques**. The
language we envision will likely combine several of these: for example, *ownership/affine types by default* (like Rust)
for most data, plus *optional reference counting or constraint references* for shared scenarios, plus *region-based
allocation* for bulk memory management when performance dictates, plus possibly *generational tags or hardware traps* as
a backstop against mistakes. Each addition comes with complexity: e.g. if we add a generational system, that’s extra
metadata and runtime code; if we add a borrow checker, that’s compiler complexity. A guiding principle should be *
*pay-for-what-you-use**: if a user sticks to simple patterns (like unique ownership with RAII), they should get zero
runtime overhead and not have to deal with heavy syntax. Only when they opt into more complex patterns (shared
mutability, long-lived aliasing) should additional machinery kick in (be it runtime checks, reference counts, or
explicit lifetime annotations).

## Comparison of Approaches

Let’s tabulate the key trade-offs of major approaches, to see how they meet the goals:

* **Manual (Malloc/Free):** **Flexibility:** total. **Safety:** none by default (high risk of UAF, leaks). **Ergonomics:
  ** poor (burden on programmer). **Performance:** excellent (no overhead beyond allocation, but risk of fragmentation).
  **Compiler complexity:** minimal (no special analysis needed). **Use in Kernel:** Used in C kernels, but unsafe; needs
  supplementary discipline or tools.

* **Ownership/Borrowing (Affine types, Rust model):** **Flexibility:** high, but with patterns restrictions (no cycles
  without opting out). **Safety:** very high (formal memory safety proven in many cases). **Ergonomics:** moderate to
  low for newbies (requires understanding the model; some patterns need verbose workarounds). **Performance:**
  excellent (zero-cost at runtime for safety). **Compiler:** very complex (lifetime analysis, error messages, etc.). *
  *Use in Kernel:** Rust is increasingly used in OS components due to safety, proving this model’s viability, but
  requires training and careful API design.

* **Linear Types:** **Flexibility:** low if strictly applied (everything must be used once), but can be combined with
  borrowing to increase it. **Safety:** extremely high (statically prevents leaks and multiple frees, can even enforce
  that cleanup code runs). **Ergonomics:** low if pervasive (lots of `clone()`ing or explicit consumes; however,
  localized use for specific resource types can be fine). **Performance:** excellent (no runtime overhead, just
  restrictions). **Compiler:** complex (similar to ownership, plus requiring each linear value’s flow to be tracked
  exactly). **Use in Kernel:** Not directly used yet widely, but research (like Austral, or Rust’s potential linear type
  future) could bring this in. Great for particular resources (e.g., making file handles linear to ensure close is
  called).

* **Region/Arena:** **Flexibility:** medium. Works well for hierarchical lifetimes, less so for arbitrary graphs. The
  programmer might have to adapt design to fit region scopes. **Safety:** high if checked (no dangling pointers from
  freed regions). **Ergonomics:** fairly good for certain patterns (e.g. allocate many objects in a loop then free all).
  But if the lifetime relationships are non-trivial, managing region annotations can be hard. **Performance:**
  excellent (fast alloc/free, good locality). **Compiler:** medium complexity (must track region lifetimes like an
  extended borrow checker). **Use in Kernel:** Many kernels use arenas for specific subsystems (Linux slabs, etc.),
  though enforced by discipline not language. A language that natively supports arenas with checks (like Cyclone did)
  can make this safer.

* **Reference Counting (ARC):** **Flexibility:** high – you can create arbitrary graphs of objects, and memory is
  reclaimed when no one references them. Cycles are the big caveat (need breaking). **Safety:** high if used purely (no
  raw pointers) – it ensures no use-after-free because an object lives as long as someone has a reference. But if the
  language allows non-counted references, then not inherently safe. **Ergonomics:** very good; it’s automatic.
  Programmers mostly don’t need to think about memory ownership for basic cases (just keep a reference to what you
  need). Must be careful to avoid cycles or use weak references. **Performance:** moderate – continuous cost of
  counting, slower in multithreading due to atomics, memory overhead for counts. No pause though. **Compiler:** low to
  medium – basic RC is simple, but optimizing it (like Swift’s ARC optimizer or Koka’s Perceus) is complex. **Use in
  Kernel:** Not typically used in kernel-space historically (due to overhead and lack of real-time guarantees in naive
  form), but for a new language, a deterministic ARC might be considered if overhead can be kept low. Some embedded
  systems avoid RC due to atomic overhead on weak CPUs.

* **Hybrid (Rust+RC, etc.):** **Flexibility:** very high – you choose the right tool for each scenario. **Safety:** high
  if the safe subsets are used appropriately (Rust ensures safety either way; if you use `Rc` it’s still memory safe,
  just might leak on cycles). **Ergonomics:** potentially the best of both: you use simple patterns where possible, and
  only engage with complexity (like Arc or unsafe) when needed. However, context-switching can confuse some (one must
  learn multiple models: the base ownership model and the RC/RefCell model for exceptions). **Performance:** can be
  optimized case by case. Most code runs with zero overhead (ownership), only specific parts pay for RC or checks. *
  *Compiler:** highest complexity – it includes the union of features.

* **Runtime Checks (Generational or Constraint Refs):** **Flexibility:** high – similar to manual or C++ in what you can
  express (not much is prohibited at compile time). **Safety:** high at runtime (dangling uses are caught), but not as
  absolute as compile-time (you push errors to runtime exceptions). **Ergonomics:** good – easier to write code when you
  don’t get compile errors for aliasing; issues only surface if you actually misuse memory, likely as a crash or trap (
  which is still better than silent corruption). **Performance:** slight cost on pointer use (like an array bounds check
  cost). Possibly acceptable in systems code if not overused, especially if it can be mostly eliminated via static
  regions as Vale suggests. **Compiler:** simpler than a full borrow checker (mostly instrumentation). **Use in Kernel:
  ** Not used yet, but could be analogous to enabling something like memory tagging/sanitizer in a production system –
  might be viable if overhead is small. For a kernel, failing a check would ideally panic that subsystem – which might
  be acceptable (better than a security hole). Still, relying on runtime checks means bugs aren’t prevented, only
  caught; for a secure kernel, we’d prefer to prevent them entirely or prove them absent.

Given these, an ideal model might try to maximize safety and performance, while offering an easier learning curve than
Rust if possible. A purely static approach (Rust) is extremely safe and fast, but not “ease of use” as some would like.
A purely dynamic checked approach (runtime tagging or RC) is easier to use, but either slower or catches errors late. A
**blend** can aim for the **best of both worlds**: static enforcement for most cases, with escape hatches that are
runtime-checked for those that the static system can’t handle.

## A Hybrid Model for the Big Iron Language

Considering all the above, we propose a **hybrid memory management model** that could suit a “Big Iron” secure kernel
language:

1. **Default to Unique Ownership with RAII:** Every heap allocation returns a unique (affine) object reference that will
   be automatically freed when it goes out of scope. This covers the majority of use cases in systems programming in a
   safe, zero-cost manner – like Rust’s ownership but aim for simpler usage if possible (perhaps infer lifetimes to
   avoid explicit annotations in most cases). Use linear/affine types so that the compiler ensures no double frees and
   no leaks of owned data (any owned data not moved will be freed at scope end). This provides memory safety *and*
   avoids GC. For example:

   ```rust
   let packet = Packet::new(); // unique owner
   kernel.process(packet);
   // freed here if not moved
   ```

   At compile time, ensure `packet` was either moved into `process` or not accessible here.

2. **Borrowing & Regions for Temporaries:** Allow functions to take borrowed references to data without transferring
   ownership, for efficiency (avoiding copies). Use a lifetime or region system (could be implicit or explicit) to
   ensure these borrows don’t outlive the owned data. This can be done Rust-style (with explicit lifetimes for complex
   cases) or in a simpler way like Peterse’s experiment where borrows are tied to an explicit lexical scope. The latter
   means you can do:

   ```rust
   borrow mut file {
       let writer = BufferedWriter::new(file);
       writer.write(...);
   } // file is automatically returned here
   ```

   Ensuring `writer` (and any references inside it) doesn’t escape the `borrow` block. The exact syntax isn’t crucial;
   what matters is borrowed pointers are allowed but checked. This enables safe **aliasing for reads** and temporary
   mutable loans, just like Rust’s &/\&mut. Compiler complexity here is significant but manageable (similar to Rust’s
   borrow checker or Cyclone’s region checker).

3. **Parameterized Manual Control – Arenas and Pools:** Provide first-class support for **arena allocators** or memory
   pools. The programmer can create an `Arena<T>` which can allocate many `T` objects quickly. The language’s type
   system ensures that a pointer into an arena cannot outlive that arena (much like how references to a stack frame
   don’t live after function returns). This might mean functions that allocate from an arena are annotated as such (or
   the arena is passed in). When the arena is destroyed, all objects are freed. This gives manual control for
   performance-critical sections (e.g., allocate a bunch of objects during one iteration of a game loop or one network
   request, free them all at end). It’s memory safe because of the checks, and there’s no GC involvement. Verona’s and
   Cyclone’s work show the feasibility of this.

4. **Opt-In Reference Counting for Shared Ownership:** When multiple ownership *truly* is needed (e.g., a graph of nodes
   where multiple references exist, or a cache where many pieces hold pointers to a shared resource), the language can
   offer a `Shared<T>` smart pointer. Under the hood, this uses reference counting or a similar mechanism. The compiler
   can generate the retains/releases, or it could be a library type. This is akin to Rust’s `Arc` or C++ `shared_ptr`.
   To integrate with our system:

    * A `Shared<T>` when dereferenced yields a borrow of `T` (so within a scope you can access it as if \&T or \&mut T
      under borrow rules, possibly using interior mutability for \&mut).
    * The count ensures the `T` lives as long as any `Shared<T>` exists.
    * We should encourage avoiding cycles; perhaps provide `Weak<T>` to break cycles if needed.
    * If possible, apply optimizations: for instance, in single-threaded mode, use non-atomic counts; elide increments
      when passing into functions where analysis shows it’s not needed (like Rust’s new `Arc::new_cyclic` or some future
      linear handling of refcounts).

   By making this opt-in, casual users don’t pay a cost unless they need it. It covers scenarios that strict ownership
   doesn’t (like trees with parent and child pointers – make child hold a `Weak<Parent>` or parent a `Shared` that
   children use).

5. **Optional Runtime Checks (Generational or Assertions) for Unsafe Escapes:** For the ultimate flexibility, allow an
   `unsafe` escape hatch – but with assistance. For example, a developer might sometimes want to store a raw pointer or
   do something the borrow checker disallows for performance or needed semantics. In Rust, `unsafe` lets you do it but
   then you must uphold safety manually. In our language, we could augment `unsafe` pointers with **generational tagging
   ** behind the scenes. For instance, an `UnsafeRef<T>` type could behave like Vale’s generational references: you can
   convert an owning reference to an `UnsafeRef` (maybe to store in a data structure without lifetime generics), and the
   compiler won’t complain, but if you use that ref after the original is freed, it will trap rather than corrupt
   memory. This gives an extra layer of safety even in “unsafe” scenarios. It’s like having a debug mode in release – it
   turns a nasty bug into a caught error. This feature would rely on each allocation carrying a generation ID and each
   `UnsafeRef` carrying the expected ID. The overhead might be acceptable if such refs are used sparingly.
   Alternatively, simpler: adopt the “constraint reference” idea – in debug builds, check that unsafe refs aren’t
   dangling on free. In a kernel, you might choose to crash if such a bug is detected, which is better than silent
   corruption. Over time, as the kernel is hardened, these runtime checks should never trigger, effectively proving the
   design.

6. **Linear Types for Critical Resources:** We can designate certain resource types as **linear**, meaning the compiler
   will enforce they are used exactly once. For instance, a memory mapping or a lock guard could be linear – you must
   eventually hand it back or close it, you can’t just drop it. This ensures vital operations (like unlocking a mutex or
   deallocating a region) happen. The concept of *Higher RAII* from Vale/Austral is achievable here. We might not make
   *all* types linear (that’s too onerous), but for a secure kernel, being able to mark something like “this capability
   token must be consumed exactly once” could enforce security protocols.

7. **Controlled GC for Specific Cases (Optional):** While the requirement is to avoid GC, one could consider allowing *
   *tiny GC regions** as an opt-in for convenience in non-critical code. For example, maybe user-space facing components
   or scripting parts of the kernel could use a small tracing collector. But since the question explicitly says avoid
   GC, we likely won’t include this in the core plan. Instead, we rely on the above techniques to manage all memory
   without a traditional GC.

**Summing up the Hybrid Model:** The language’s default behavior is much like Rust (ownership and borrowing) but
possibly with less annotation if we can manage more inference or use simpler lexical scopes for borrows. When the strict
rules get in the way for a valid scenario, the programmer has alternatives: use a `Shared` (RC) pointer for shared
ownership, use an Arena for bulk allocation, or use an UnsafeRef (with runtime checking) for exotic patterns. Each of
these escapes has a defined cost and implication, which the programmer is aware of. This way, we uphold **memory safety
** (either statically or via runtime checks) and give developers the **flexibility** to manually control memory layout
and lifetime when needed. Importantly, we avoid hidden global garbage collectors – any automated memory management is
either deterministic (RAII, RC) or confined (per arena or per object via refcount). There are no stop-the-world pauses;
even the reference counting is incremental and localized.

## Theoretical Foundations vs Real Implementations

By drawing on theory (linear type systems, region calculus, capabilities) and practice (Rust’s success, Cyclone’s
experiments, Vale’s innovations, etc.), our design tries to take the best of each:

* From **Rust**, we take the idea that compile-time ownership can eliminate most memory bugs with zero runtime cost. We
  also take the lesson that this can be made workable in real systems and even OS kernels, as proven by ongoing Rust
  adoption in Linux and other projects.
* From **Cyclone**, we take the idea of region-based pointers to handle arenas safely.
* From **Pony**, we’re reminded that even GC can be made more deterministic by isolating heaps per actor (though we
  likely won’t use GC, Pony’s *ORCA* shows how to avoid stop-the-world by design).
* From **Vale**, we embrace the creative blending of techniques: generational references plus regions to minimize
  overhead, and the concept that we can achieve memory safety *without* an explicit borrow checker or GC by leveraging
  unique ownership + runtime checks. Vale’s approach influenced our idea of offering a runtime-checked unsafe reference
  to avoid full complexity for certain patterns.
* From **SPARK Ada**, we adopt a simple but strong rule for pointers (no pointing to shorter-lived memory) – this will
  naturally be enforced by our lifetime system.
* From **linear type research**, we ensure the language can guarantee cleanup of critical resources, improving not just
  memory safety but overall program correctness (no forgetting to release something).
* From **Nim/Swift** and others, we see that reference counting can be the “99% solution” that’s easier for many
  programmers than Rust’s model – but we also see its limitations (cycles, concurrency costs). By not using RC
  everywhere, we avoid global overhead, but by having it available, we give a softer learning curve: a developer who
  finds the ownership rules too restrictive in some case can switch that part to `Shared<T>` rather than rewrite their
  design completely. Nim, for instance, found that many algorithms were easier to write with ARC and decided to make ARC
  the default over pure GC – showing the appeal of deterministic destruction without borrow checking.

**Compiler Complexity vs User Benefit:** Our hybrid approach does mean the compiler has to implement multiple
mechanisms (ownership checking, lifetime analysis, perhaps an RC optimizer, and the generational scheme). This is
non-trivial, but each piece is grounded in existing work:

* Lifetimes/borrowing: Rust’s borrow checker is a guide; we might simplify by disallowing some complex cases or
  requiring manual annotation in rare tricky scenarios to keep inference simpler.
* Linear types: having an annotation for linear types of certain structs, like Austral, would require the compiler to
  enforce an additional rule (similar to ownership, but stricter).
* Reference counting: can be implemented as a library feature largely, with maybe some compiler hints (like inlining
  reference count ops or optimizing moves).
* Generational indices: require some runtime support, but conceptually simple (bump a counter on free, check on deref).
  The trickiest part is ensuring thread-safety if used in a multithreaded kernel.

In terms of **user experience**, the hope is to provide a safe default that feels natural (like using unique pointers in
C++ or variables in Rust), and to make advanced features opt-in with clear syntax. For example:

* `let x = new T` gives unique `x`.
* `let y = &x` gives a borrow (maybe implicitly limited to this scope or function).
* `let s = Shared::new(t)` gives a reference-counted pointer.
* `arena A { ... A.alloc(... ) ... }` creates an arena and uses it inside.
* `unsafe_ref z = x` produces an unchecked reference (the name `unsafe_ref` signaling to the user the risk, but we
  behind scenes make it a generational tagged pointer to mitigate danger).

Documentation would guide which method to use when. For maximum safety, stick to unique/borrow. Use Shared only when
necessary for multiple owners. Use unsafe\_ref extremely sparingly (and only if you need to e.g. store a pointer
long-term without the type system understanding it, such as in a self-referential structure, which could perhaps be
handled by safer means anyway).

**Trade-off Highlights:**

* **Safety vs Flexibility:** By combining static and dynamic methods, we ensure safety either at compile time or at
  runtime. Rust chose to reject code that doesn’t meet its rules; we choose to accept more code by sometimes deferring
  the check to runtime (with RC or tagged pointers). This means a buggy program might compile in our language but then
  crash at runtime if a memory safety rule is violated (e.g. if the programmer improperly used an unsafe ref). In Rust,
  that would have been a compile error or required `unsafe`. This is a conscious trade-off: slightly lower guarantees
  than Rust’s absolute provable safety, in exchange for *ergonomic gains and flexibility*. We mitigate it by making most
  checks deterministic and fast (so we catch the issue immediately where it occurs, not long after).
* **Ergonomics:** New users could initially use the language almost like a garbage-collected one: allocate objects and
  not worry about freeing because the unique ownership and RAII free things automatically, and if they need to share,
  use a Shared pointer which works much like a GC reference (just remember to break cycles). As they get comfortable,
  they can optimize by using arenas or restructuring to avoid refcounts. This gradual path is arguably more forgiving
  than hitting the borrow checker’s wall immediately without knowing why. We still expect some learning curve (the
  concept of ownership is unavoidable), but we can aim to simplify some aspects (perhaps default to move semantics
  without explicit `move` keywords, more implicit lifetime inference, etc.).
* **Performance:** In the critical parts of the kernel, developers can avoid any runtime overhead by sticking to
  unique/borrow/arena patterns (just like one would in Rust or C). In more complex subsystems where development speed
  matters more than micro-optimization, using `Shared` (RC) or others is fine and will still be deterministic (just
  somewhat slower due to atomic ops). If at some point a performance bottleneck is found, those parts can often be
  refactored to a more manual style (e.g., replace a graph of Shared nodes with an arena allocated graph and manual
  relationships, if needed).

In conclusion, the “Big Iron” language can achieve **memory safety without GC** by leveraging *ownership and linear
types at compile time*, supplemented by *region-based allocation and (optionally) runtime-checked references* for
patterns that static rules can’t easily cover. This approach is informed by Rust’s success, Vale’s experiments, and
decades of research. It balances safety and usability: preventing the vast majority of memory errors at compile time,
while giving the programmer pragmatic escape hatches (with controlled costs) to handle real-world complexity. The result
should be a language where building a secure kernel is feasible and memory-safe, without sacrificing the low-level
control and high performance that “big iron” systems demand.

**Sources:**

* Evan Ovadia, *“The Memory Safety Grimoire, Part 1”* – an overview of at least 14 memory management techniques beyond
  just GC/RC/borrow checking. This series compares approaches like arenas, regions, generational references, etc., and
  inspired the hybrid strategy (combining regions with generational checks to get C++-like flexibility with Rust-like
  safety).
* *Verdagon (Vale) blog*, *“Single Ownership and Memory Safety without Borrow Checking, RC, or GC”* – discusses how
  Vale’s linear + generational model achieves memory safety by default using unique ownership and runtime checks.
* Yorick Peterse, *“The inevitability of the borrow checker”* (Feb 2025) – an experience report from the Inko language,
  concluding that to avoid runtime costs, a Rust/Austral-style borrow checker is likely the only sound solution, despite
  the complexity. This emphasizes why our design includes a borrow-checking component for soundness.
* Discussions on Reddit (r/ProgrammingLanguages) such as *“Does Rust have the ultimate memory management solution?”* –
  highlighting community perspectives that Rust’s model, while powerful, is not the end-all and has limitations in
  ergonomics. This justifies exploring alternatives and hybrids instead of blindly copying Rust.
* **Rust language reference & Nomicon:** for details on how ownership and borrowing guarantee memory safety without a
  GC, and what patterns are disallowed (e.g., why observers and intrusive structures need workarounds).
* **Cyclone project documentation:** showing how region annotations and pointers with region lifetimes were used to
  prevent dangling pointers in a safe dialect of C.
* **Pony language tutorials:** explaining the ORCA GC and actor-isolated heaps for concurrency without stop-the-world
  GC, influencing the idea of per-region memory management and deterministic collection.
* **CHERI project papers:** for the concept of hardware capabilities that enforce spatial (and with Cornucopia,
  temporal) safety at potentially lower cost.
