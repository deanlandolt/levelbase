# levelbase

This is just some notes on a more consistent low-level interface for leveldb -- more primitive than leveldown but with the addition of a few critical missing primitives.

## Writing

There is one fundamental write api: an async iterable of operations representing a batch. By default all operations in the batch must be committed atomically. Individual operations can include an `atomic: false` property to override this behavior for that particular operation.

Array and chained batch style interfaces can be constructed on top of this.

Order of operations is relevant in a batch -- subsequent operations will override previous operations (unless the operations in question are commutative, of course).


## Reading

There is one fundamental read API: interval queries. Query options have coherent semantics, with min and max being the keys, and minExclusive and maxExclusive determining whether they represent open or closed intervals. The result is an async iterable "cursor" that can be iterated using for-of. It may also be consumed as a raw generator, with its `next` argument allowing an alternate key to be provided.

The typical `get` interface can be constructed on top of this.

There is no `reverse` in query options -- a `previous` method exists that behaves as the opposite of `next`. A `reverse` method exists on the cursor instance that will swap the `next` and `previous` methods.


## Observing

The ability to install pre and post commit hooks is also a primitive API. This could be accomplished locally by wrapping the batch operation, but cannot be done remotely without coordination from the host. There are other optimizations that are possible when treating this as a fundamental primitive, like building distribution trees based on interval containment properties.


## Subspacing

Promoting the notion of creating sublevels to a fundamental primitive allows guarantees about access to be preserved regardless of key encoding. A reference to a subspace can be passed around safely without leaking references to its parent or even any child subspaces. A subspace is a completely isolated keyspace nested within a larger keyspace, which may contain infinitely many isolated keyspaces nested within itself.

Batches can be written transactionally across subspaces, but only by providing a reference to the subspaces in question. These references could be leaked if batch is allowed to be overriden. Instead, batches containing subspaces references could be passed through a transaction method which partitions the batch stream by subspace and invokes the batch method for each subspace. This is also where atomic boundary switches should live, as non-atomic operations would each get a distinct batch. The transaction method is sealed to prevent overwriting, and is shared across all subspace instances (transaction method identity could also function as an instance check).

Write capabilities can be revoked on any given subspace reference, but never reenabled. Writes can be restricted to a range or set of ranges (even across multiple subspaces) but never widened. The allowable ranges for read and observe may also be narrowed but never widened. The authority of each subspace is mapped to a backing db by a WeakMap reference, which it uses to gain its authority. In this design, it's possible but not necessary to ensure all subspaces are isolated.


