# Problems with Shared Pointers

- **Shared pointers (`std::shared_ptr`)** manage dynamic memory with reference counting.
- While useful, they come with several potential issues.

---

# 1. Cyclic References

- **Problem**: Shared pointers can create memory leaks if objects reference each other in a cycle.

- **Solution**: Use std::weak_ptr to break the cycle

- **Over simplified example**

  ```cpp
  struct Foo { std::shared_ptr<Bar> bar; };
  struct Bar { std::shared_ptr<Baz> baz; };
  struct Baz { std::shared_ptr<Bar> bar; };

  auto foo = std::make_shared<Foo>();
  auto bar = std::make_shared<Bar>();
  auto baz = std::make_shared<Baz>();
  
  foo->bar = bar; // When foo gets deleted, bar does not
  bar->baz = baz;
  baz->bar = bar;
  ```

- **Solution**

  ```cpp
  struct Foo { std::shared_ptr<Bar> bar; };
  struct Bar { std::shared_ptr<Baz> baz; };
  struct Baz { std::weak_ptr<Bar> bar; }; // Use weak_ptr

  auto foo = std::make_unique<Foo>();
  auto bar = std::make_shared<Bar>();
  auto baz = std::make_shared<Baz>();
  
  foo->bar = bar;
  bar->baz = baz;
  baz->bar = bar;
  ```

---

# 2. Overhead

- **Problem**: Shared pointers introduce performance overhead:
  - Reference counting (incrementing/decrementing counters).
  - Allocating memory for the control block.
- **Impact**: Can slow down high-performance or low-latency systems.

---

# 3. Thread Safety

- **Problem**: Reference counting is thread-safe, but the managed object is **not**.
- **Impact**: Concurrent access to the object can lead to race conditions.
- **Solution**: Use synchronization mechanisms (e.g., `std::mutex`).

---

# 4. Unintended Ownership

- **Problem**: Assigning a raw pointer to multiple shared pointers can cause double deletion.
- **Example**:

  ```cpp
  int* rawPtr = new int(42);
  std::shared_ptr<int> sp1(rawPtr);
  std::shared_ptr<int> sp2(rawPtr); // Undefined behavior
  ```

- **Solution**: Use `std::make_shared` or ensure only one `std::shared_ptr` owns the pointer.

---

# 5. Custom Deleters

- **Problem**: Custom deleters can complicate code and lead to resource leaks if not handled properly.
- **Example**: Forgetting to release resources in the custom deleter.

---

# 6. Ownership Semantics

- **Problem**: Shared pointers can obscure ownership semantics.
- **Impact**: Makes it unclear which part of the code manages the object's lifetime.

---

# 7. Performance vs. `std::unique_ptr`

- **Problem**: Shared pointers are slower than `std::unique_ptr` due to:
  - Reference counting.
  - Thread-safety mechanisms.
- **Solution**: Use `std::unique_ptr` when shared ownership is not required.

---

# 8. Misuse in Containers

- **Problem**: Using shared pointers in containers (e.g., `std::vector`) can lead to unintended copies.
- **Impact**: Increases reference count unnecessarily.

  ```cpp
  std::shared_ptr<int> sp = std::make_shared<int>(42);
  std::vector<std::shared_ptr<int>> vec;

  // Adding shared pointers to the vector
  vec.push_back(sp); // Reference count increases to 2
  vec.push_back(sp); // Reference count increases to 3
  ```
  
- **Solution**

  ```cpp
  std::shared_ptr<int> sp = std::make_shared<int>(42);
  std::vector<std::shared_ptr<int>> vec;

  // Adding shared pointers to the vector
  vec.push_back(std::move(sp)); // Transfer ownership
  ```
  
---

# 9. Over simplified code smells

```cpp
auto foo = std::make_shared<Foo>();
workQueue.push_back(std::move(foo));
```

_n_ threads pick `foo` out of the queue and do stuff, `foo` might even be stored. Ownership and lifetime are
left to chance. This is a lazy way of managing the lifetime of `foo`. This is OK for temporaries, but long
lived objects should be explicitly managed.

The below code **might** be OK, if leftThread and rightThread merely reads a value out of `foo` and then finishes.
This pattern of constructing a shared pointer and then immediately passing it to a thread smells, though, and should
be avoided.

```cpp
auto foo = std::make_shared<Foo>();
leftThread(foo);
rightThread(foo);
```

