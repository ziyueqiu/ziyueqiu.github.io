---
layout:     post
title:      Introduction to HHVM LRU Cache
date:       2022-02-03
tags:
    - Cache
categories: caching
comments: true
---

## What you will see

I am impressed by this 400 LOC pure implementation of LRU Cache [source code](https://github.com/facebook/hhvm/blob/master/hphp/util/concurrent-lru-cache.h), and decide to introduce its design through scanning the code. I believe learning HHVM LRU Cache implementation can help when needing to understand all kinds of cache implementations.

- Data structures including doubly-linked list and hash table
- APIs including `insert()`, `find()`
- Thread-safe
- non-blocking (try_lock) for hit path
- Thinking questions

## Terms

- **Key** Keys are hashed to metadata and searched in hash table here. Related to `m_key`, `key` in source code.

- **Object** values for a Key-Value Cache or blocks for a Block Cache.

- **Eviction** Eviction refers an item exiting from cache because the cache storage is full. Evictions are triggerred internally.

- **LRU List** LRU List maintains the LRU order. Related to `m_head` and `m_tail` in source code.
  - **Node** Node refers node in LRU List, with type of `ListNode`.
  - **Lock** Locks are used for protection under concurrent accesses, such as `m_listMutex`
- **Hash Table** Related to `HashMap` and `m_map` in source code

## Data structures

Architecture TBD :)

### ListNode

Each node's prev pointer is always inited into `OutOfListMarker`, which is a global const.

The main usage of `OutOfListMarker` is to judge whether in list (`isInList()`). To elaborate more, `isInList()` is used in `find()` to avoid conflict where hash table returned node is being evicted. The conflict can happen because `find()` search hash table to get node without lock, and adding a condition can ensure thread-safety.

```c
  struct ListNode {
    ListNode()
      : m_prev(OutOfListMarker), m_next(nullptr)
    {}

    explicit ListNode(const TKey& key)
      : m_key(key), m_prev(OutOfListMarker), m_next(nullptr)
    {}

    TKey m_key;
    ListNode* m_prev;
    ListNode* m_next;

    bool isInList() const {
      return m_prev != OutOfListMarker;
    }
  };

  static ListNode* const OutOfListMarker;
```

### HashMap

Key is `key`, value include object (`m_value`) and node (`m_listNode`).

Use Intel TBB `concurrent_hash_map` as Hash Table.

Note that `const_accessor` and `accessor` are different. (generally read lock v.s write lock here).

```c
  struct HashMapValue {
    HashMapValue()
      : m_listNode(nullptr)
    {}

    HashMapValue(const TValue& value, ListNode* node)
      : m_value(value), m_listNode(node)
    {}

    TValue m_value;
    ListNode* m_listNode;
  };

  typedef tbb::concurrent_hash_map<TKey, HashMapValue, THash> HashMap;
  typedef typename HashMap::const_accessor HashMapConstAccessor;
  typedef typename HashMap::accessor HashMapAccessor;
  typedef typename HashMap::value_type HashMapValuePair;
  typedef std::pair<const TKey, TValue> SnapshotValue;
```

Question: Try to use other hash table implementations (like CLHT), and test.

### Global Variables

Note that `m_size` is **approximately** equal to the number of elements in the container, and is **atomic**. The gap comes from concurrency, and does not become larger with time.

For the LRU List, the "head" is the most-recently used node, and the "tail" is the least-recently used node. The list mutex must be held during both read and write.

```c
  /**
   * The maximum number of elements in the container.
   */
  size_t m_maxSize;

  /**
   * This atomic variable is used to signal to all threads whether or not
   * eviction should be done on insert. It is approximately equal to the
   * number of elements in the container.
   */
  std::atomic<size_t> m_size;

  /**
   * The underlying TBB hash map.
   */
  HashMap m_map;

  /**
   * The linked list. The "head" is the most-recently used node, and the
   * "tail" is the least-recently used node. The list mutex must be held
   * during both read and write.
   */
  ListNode m_head;
  ListNode m_tail;
  typedef std::mutex ListMutex;
  ListMutex m_listMutex;
```

Question: spinlock can perform different. Replace `std::mutex` and test.

Question: `m_head` and `m_tail` can be organized into one marker. Try? Hint: [RocksDB LRU Cache](https://github.com/facebook/rocksdb/blob/main/cache/lru_cache.cc)

## APIs

I will highlight `find()`, `insert()` and `evict()` in this section. At the end of this part, I will attach my own implementation of `delete()`. Welcome discussion!

### API Overview

Note that：

- `find()` return value by filling ConstAccessor
- `insert()` will copy key and value (from input), then insert into hash table
- `insert()` will not update/replace the same key, but return false
- `delink()` and `pushFront()` should be used with lock (`m_listMutex`) when called

```c
public:
	/**
   * Find a value by key, and return it by filling the ConstAccessor, which
   * can be default-constructed. Returns true if the element was found, false
   * otherwise. Updates the eviction list, making the element the
   * most-recently used.
   */
  bool find(ConstAccessor& ac, const TKey& key);

  /**
   * Insert a value into the container. Both the key and value will be copied.
   * The new element will put into the eviction list as the most-recently
   * used.
   *
   * If there was already an element in the container with the same key, it
   * will not be updated, and false will be returned. Otherwise, true will be
   * returned.
   */
  bool insert(const TKey& key, const TValue& value);

  /**
   * Clear the container. NOT THREAD SAFE -- do not use while other threads
   * are accessing the container.
   */
  void clear();

  /**
   * Get the approximate size of the container. May be slightly too low when
   * insertion is in progress.
   */
  size_t size() const {
    return m_size.load();
  }

private:
  /**
   * Unlink a node from the list. The caller must lock the list mutex while
   * this is called.
   */
  void delink(ListNode* node);

  /**
   * Add a new node to the list in the most-recently used position. The caller
   * must lock the list mutex while this is called.
   */
  void pushFront(ListNode* node);

  /**
   * Evict the least-recently used item from the container. This function does
   * its own locking.
   */
  void evict();
```

### API details

#### LRU List initialization

```cpp
template <class TKey, class TValue, class THash>
ConcurrentLRUCache<TKey, TValue, THash>::
ConcurrentLRUCache(size_t maxSize)
  : m_maxSize(maxSize), m_size(0),
  m_map(std::thread::hardware_concurrency() * 4) // it will automatically grow
{
  m_head.m_prev = nullptr;
  m_head.m_next = &m_tail;
  m_tail.m_prev = &m_head;
}
```

#### find()

Use read lock in hash table (`HashMapConstAccessor`) and `try_lock` for LRU List to weaken lock contention problem. If `lock` true, the hit item will be delinked from list then push back to the front of the list, called promotion.

```cpp
template <class TKey, class TValue, class THash>
bool ConcurrentLRUCache<TKey, TValue, THash>::
find(ConstAccessor& ac, const TKey& key) {
  HashMapConstAccessor& hashAccessor = ac.m_hashAccessor;
  if (!m_map.find(hashAccessor, key)) {
    return false;
  }

  // Acquire the lock, but don't block if it is already held
  std::unique_lock<ListMutex> lock(m_listMutex, std::try_to_lock);
  if (lock) {
    ListNode* node = hashAccessor->second.m_listNode;
    // The list node may be out of the list if it is in the process of being
    // inserted or evicted. Doing this check allows us to lock the list for
    // shorter periods of time.
    if (node->isInList()) {
      delink(node);
      pushFront(node);
    }
    lock.unlock();
  }
  return true;
}
```

I personally feel that it will be better to add some `assert()` for people who want to make code contributions to debug, like me :(

Note that `delink()` need global variable `OutOfListMarker`, and `pushFront()` need global variable `m_head`.

```cpp
template <class TKey, class TValue, class THash>
inline void ConcurrentLRUCache<TKey, TValue, THash>::
delink(ListNode* node) {
  ListNode* prev = node->m_prev;
  ListNode* next = node->m_next;
  prev->m_next = next;
  next->m_prev = prev;
  node->m_prev = OutOfListMarker;
}

template <class TKey, class TValue, class THash>
inline void ConcurrentLRUCache<TKey, TValue, THash>::
pushFront(ListNode* node) {
  ListNode* oldRealHead = m_head.m_next;
  node->m_prev = &m_head;
  node->m_next = oldRealHead;
  oldRealHead->m_prev = node;
  m_head.m_next = node;
}
```

Question: If there is no condition `isInList()` before `delink()`, what will happen?

#### insert() & evict()

`insert()` fails when key exists, or `m_map.insert()` return true. Note that at this time, accessor made a write lock on the bucket of the key.

 `evict()` is only used by `insert()`, thus should be looked inside when making use of `evict()`.

```cpp
template <class TKey, class TValue, class THash>
bool ConcurrentLRUCache<TKey, TValue, THash>::
insert(const TKey& key, const TValue& value) {
  // Insert into the CHM
  ListNode* node = new ListNode(key);
  HashMapAccessor hashAccessor;
  HashMapValuePair hashMapValue(key, HashMapValue(value, node));
  if (!m_map.insert(hashAccessor, hashMapValue)) {
    delete node;
    return false;
  }

  // Evict if necessary, now that we know the hashmap insertion was successful.
  size_t size = m_size.load();
  bool evictionDone = false;
  if (size >= m_maxSize) {
    // The container is at (or over) capacity, so eviction needs to be done.
    // Do not decrement m_size, since that would cause other threads to
    // inappropriately omit eviction during their own inserts.
    evict();
    evictionDone = true;
  }

  // Note that we have to update the LRU list before we increment m_size, so
  // that other threads don't attempt to evict list items before they even
  // exist.
  std::unique_lock<ListMutex> lock(m_listMutex);
  pushFront(node);
  lock.unlock();
  if (!evictionDone) {
    size = m_size++;
  }
  if (size > m_maxSize) {
    // It is possible for the size to temporarily exceed the maximum if there is
    // a heavy insert() load, once only as the cache fills. In this situation,
    // we have to be careful not to have every thread simultaneously attempt to
    // evict the extra entries, since we could end up underfilled. Instead we do
    // a compare-and-exchange to acquire an exclusive right to reduce the size
    // to a particular value.
    //
    // We could continue to evict in a loop, but if there are a lot of threads
    // here at the same time, that could lead to spinning. So we will just evict
    // one extra element per insert() until the overfill is rectified.
    if (m_size.compare_exchange_strong(size, size - 1)) {
      evict();
    }
  }
  return true;
}
```

Question: Will it be possible for `evict()` to fail internally?

Question: Why `m_size.compare_exchange_strong(size, size - 1)`? Is there any other way to implement?

Question: Why in this order: insert hash table -> evict object -> pushFront node? What could be wrong with pushFront ahead of insert hash table?

```cpp
template <class TKey, class TValue, class THash>
void ConcurrentLRUCache<TKey, TValue, THash>::
evict() {
  std::unique_lock<ListMutex> lock(m_listMutex);
  ListNode* moribund = m_tail.m_prev;
  if (moribund == &m_head) {
    // List is empty, can't evict
    return;
  }
  delink(moribund);
  lock.unlock();

  HashMapAccessor hashAccessor;
  if (!m_map.find(hashAccessor, moribund->m_key)) {
    // Presumably unreachable
    return;
  }
  m_map.erase(hashAccessor);
  delete moribund;
}
```

Question: When will `moribund == &m_head` become true?

Question: Will `!m_map.find(hashAccessor, moribund->m_key)` become true?

### Add support of delete_key()

It seems easy to implement `delete_key()` since it is similar to `insert()`, but there is one thing to take care.

`delete_key()` should do `m_size--` but `evict()` shouldn't do `m_size++`, since insert maintain the consistency of `m_size`. Therefore, if user move `evict()` into public and calls it, there should add code like `m_size--`.

```cpp
template <class TKey, class TValue, class THash>
void ConcurrentLRUCache<TKey, TValue, THash>::delete_key(const TKey& key) {
  HashMapAccessor hashAccessor;
  if (!m_map.find(hashAccessor, key)) {
    return;
  }

  std::unique_lock<ListMutex> lock(m_listMutex);
  ListNode* node = hashAccessor->second.m_listNode;
  if (!node->isInList()) {
    return;
  }
  delink(node);
  lock.unlock();

  m_map.erase(hashAccessor);
  delete node;
  m_size--;
}
```

Question: Can accessor be replaced into const accessor?

