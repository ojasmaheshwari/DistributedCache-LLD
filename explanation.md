# Distributed Cache - Design Explanation

## How data is distributed across nodes

A `NodeSelectionStrategy` determines which node owns a given key. The current implementation (`SimpleModuloSelectionStrategy`) computes the ASCII sum of the key and takes `mod N` (where N = number of nodes) to pick a node index. This ensures the same key always maps to the same node.

The `NodeRegistry` (a singleton) maintains the mapping from node IDs to their URLs. On startup, all node URLs are registered and assigned sequential integer IDs.

**Flow:** `DistributedCache.put("name", "Ojas")` -> `NodeSelectionStrategy.chooseNode("name")` returns a node URL -> `RequestSenderStrategy.put(...)` sends the request to that node's `NodeService` -> the node's local `Cache` stores it.

## How cache miss is handled

It isn't, explicitly. When a `get` is called for a key that doesn't exist (or was evicted), the node's `Cache` returns `null` from its internal `HashMap`. There is no fallback to a backing store or other nodes - the caller simply receives `null`.

## How eviction works

Each node has its own local `Cache` paired with a `CacheEvictionPolicy`. With LRU eviction:

1. The policy maintains a `DoublyLinkedList` tracking access order (head = most recent, tail = least recent).
2. **On get:** the key is moved to the head of the list.
3. **On put (new key):** if at capacity, the tail node (LRU) is evicted - removed from both the linked list and the cache's `HashMap`. The new key is then inserted at the head.
4. **On put (existing key):** the key is moved to the head. No eviction occurs.
5. **On delete:** the key is removed from the linked list.

The eviction policy holds a direct reference to the cache's `HashMap` (set via `setStore`), allowing it to remove evicted entries from the map itself.

## How the design supports future extensibility

The design uses the **Strategy pattern** in three places, each allowing new behavior by adding a subclass without modifying existing code:

| Abstraction | Current Impl | Could be swapped for |
|---|---|---|
| `NodeSelectionStrategy` | `SimpleModuloSelectionStrategy` | Consistent hashing, weighted routing |
| `RequestSenderStrategy` | `SimpleRequestSendingStrategy` | HTTP client, gRPC, async sender |
| `CacheEvictionPolicy` | `LRUCacheEvictionPolicy` | LFU, FIFO, TTL-based eviction |
| `RequestParsingStrategy` | `SimpleRequestParsingStrategy` | JSON parser, binary protocol |

The `DistributedCacheFactory` wires these strategies together, so changing behavior is a matter of passing different implementations at construction time.
