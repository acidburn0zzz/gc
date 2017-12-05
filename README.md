# Garbage Collector for Crystal

The garbage collector is based on Immix, a garbage collector introduced by
Stephen M. Blackburn and Kathryn S. McKinley in the
[Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection,
and Mutator Performance](http://www.cs.utexas.edu/users/speedway/DaCapo/papers/immix-pldi-2008.pdf)
paper published in 2008.

This GC implements a simplified Immix without the compaction, which requires to
move objects, thus update all references, thus be capable to identify and update
all pointers perfectly. I.e. implement a fully precise collector.

It otherwise implements most of Immix (except for overflow allocations) for
small objects (less than 256 bytes) and medium objects (more than 256 bytes),
with a single GlobalAllocator and a per-thread LocalAllocator (thought only one
LocalAllocator is supported). Implements a linked list of allocations for large
objets (8192 bytes and more) that aren't covered by Immix.

This GC shall eventually be improved to allow a precise marking of identified
objects in the HEAP, while still relying on a conservative stack iterator. It
may eventually be improved to allow a precise iteration of stacks.


## WARNING

- It's a WORK IN PROGRESS.

- The test suite is merely scaffold tests for TDD data structures and their
  isolated behavior.

- This is UGLY Crystal and highly UNSAFE code —it can't rely on safe Crystal
  classes that would in turn require the GC. It's thus as UNSAFE as C code
  written by a newbie. It MUST go throught intensive Valgrind sessions of stress
  test suites.


## TODO

1. Overflow Allocations (Immix optimization)

  If we can't allocate a medium object (larger than 256 bytes) in a block, the
  object should be allocated in an overflow block (one per LocalAllocator).

2. Object Finalizers.

  Crystal classes may have a finalizer (e.g. to free external pointers, close
  some IO resources, ...) that must be called when an object is collected.

  This requires to identify dead objects during the recycling phase, to call the
  finalizers and mark the objects as collected (none of which has been
  implemented).

  It should be noted that a finalizer could potentially allocate memory... but
  maybe it could just be forbidden: trying to allocate would simply abort with
  an error message.

  Since the collector is unmoving, it shouldn't be required to identify a tree
  graph of objects before running finalizers. Since objects are never zeroed
  their data will always be available during the recycling phase (whatever if
  they're marked or not). Otherwise we'd have to run finalizers in a particular
  order, that would have to be identified (we can't collect/finalize B if A
  references it, because A's finalizer may require B).

3. Multithreaded Event Loop support.

  One LocalAllocator should be created for each thread running Fibers, to avoid
  too many synchronization until its required, for example when accessing the
  GlobalAllocator (i.e. getting the next block, allocating large objects).

  Since the Immix Collector is a stop-the-world kind, the collector must prevent
  other threads from executing while it marks & recycles the HEAP.

  The marking phase COULD be started immediately, by iterating the stacks of
  pending fibers, eventually waiting for all fibers to have been stopped, but
  since any fiber may modify any object at any time, there could be a race
  condition, where a just marked object could be changed by another fiber,
  resulting in an object reference to be lost. Maybe GC barriers could help?

  The recycling phase requires all fibers to be pending.

4. Precise Collector:
    1. precise marking of identified objects;
    2. precise iteration of stacks (using LLVM GC intrinsics & stack maps).

5. Defragmentation (IMMIX, requires precise collector).


## Credits

Some implementation details are based on ImmixGC from the scala-native
project; for example how the heap, blocks and lines metadata are handled.
