# C++ STL源码剖析 13-hashtable

[TOC]

## Overview

---

## 1.hashtable

在 C++ 中，`std::map` 和 `std::multimap` 是基于红黑树实现的有序关联容器，而 `std::unordered_map` 和 `std::unordered_multimap` 是基于哈希表实现的无序关联容器。哈希表提供了平均时间复杂度为 O(1) 的插入、查找和删除操作，但在最坏情况下（例如，所有元素都映射到同一个哈希桶时）这些操作的时间复杂度可能退化为 O(n)。

**常见的内存泄漏风险**：

1. **动态分配内存**：如果在 `std::map` 或 `std::multimap` 的值中存储了指向动态分配内存的指针，而这些指针没有在析构函数中被释放，就会造成内存泄漏。

2. **异常安全**：如果在插入元素的过程中发生异常，而没有适当的异常处理机制，已经分配的资源可能不会被释放。

3. **容器内部对象的复制**：如果容器中的元素是通过动态分配内存得到的，需要确保这些元素的复制构造函数和赋值操作符正确实现深拷贝。

**避免内存泄漏的措施**：

1. **使用智能指针**：使用 `std::shared_ptr` 或 `std::unique_ptr` 来自动管理动态分配的内存。

2. **确保异常安全**：使用异常处理机制来确保即使发生异常，也能正确释放已分配的资源。

3. **避免在容器中存储裸指针**：尽量不要在 `std::map` 或 `std::multimap` 中存储指向动态分配内存的裸指针。

4. **定期内存检测**：使用内存检测工具，如 Valgrind、AddressSanitizer 等，定期检查程序的内存使用情况。

5. **合理使用内存分配器**：可以通过自定义分配器来优化内存分配和释放的效率。

6. **使用 `std::swap` 清空容器**：当需要清空 `std::map` 或 `std::multimap` 时，使用 `std::swap` 将容器与一个临时空容器交换，这样可以释放容器占用的内存。

通过这些措施，可以有效地减少 `std::map` 和 `std::multimap` 使用中的内存泄漏风险。

## 2.在 C++ 中，除了 std::map 和 std::multimap，还有哪些容器是基于哈希表实现的？

在 C++ 标准模板库（STL）中，除了 `std::unordered_map` 和 `std::unordered_multimap` 这两个直接基于哈希表实现的关联容器外，还有一些其他容器和库也使用哈希表作为其底层数据结构。以下是一些例子：

1. **`std::unordered_set` 和 `std::unordered_multiset`**：
   - 这两个容器分别提供不包含重复元素和允许包含重复元素的哈希集合功能。

2. **`std::hash`**：
   - `std::hash` 是一个模板类，为多种标准类型提供了默认的哈希函数。它通常与哈希容器一起使用来计算元素的哈希值。

3. **自定义哈希容器**：
   - 你可以基于 `std::unordered_map` 和 `std::unordered_set` 的实现方式自定义哈希容器。

4. **第三方库**：
   - 许多第三方库提供了自己的哈希表实现，例如 Boost库中的 `Boost.Unordered`。

5. **`std::shared_ptr` 和 `std::weak_ptr` 的内部实现**：
   - 实现控制共享和弱引用的内部机制时，有时会使用哈希表。

6. **标准库中的其他组件**：
   - 一些标准库组件，如 `std::thread` 的管理，可能在内部使用哈希表来处理元素。

7. **缓存和协程库**：
   - 某些实现缓存或协程的库可能会使用哈希表来快速查找缓存条目或协程状态。

8. **编译器内部**：
   - 编译器在实现模板实例化、符号表等时，可能会使用哈希表来提高效率。

9. **标准库扩展**：
   - 一些标准库的实现可能会提供扩展，包括基于哈希表的容器。

10. **`std::pmr::unordered_map` 和 `std::pmr::unordered_multimap`**：
    - 在 C++17 中引入了内存资源库（Polymorphic Allocators），其中包含了使用这种资源的无序关联容器。

哈希表是一种非常通用的数据结构，因此在 C++ 的许多领域和库中都可能找到它的应用。

## 3.使用Valgrind进行内存泄漏检测时，有哪些常见的问题和解决方案？

使用 Valgrind 进行内存泄漏检测时，可能会遇到一些常见的问题和挑战，以下是一些解决方案：

1. **忽略第三方库的泄漏**：有时候，第三方库可能会有内存泄漏，而这些泄漏不应该算作你的程序的问题。你可以使用 Valgrind 的 `--trace-children=no` 选项来忽略子进程的内存泄漏，或者使用 `--gen-suppressions=yes` 来生成抑制文件，然后在后续的检测中使用 `--suppressions=suppressions.supp` 来忽略特定的泄漏。

2. **内存泄漏定位困难**：Valgrind 会提供内存泄漏的堆栈跟踪，但有时候这个信息可能不够详细。你可以增加编译时的调试信息（使用 `-g` 选项），这样 Valgrind 就能提供更详细的源代码位置信息。

3. **大量内存泄漏**：如果程序中有大量的内存泄漏，Valgrind 的输出可能会非常长，难以分析。你可以使用 `--show-leak-kinds=all` 选项来显示所有类型的内存泄漏，或者使用 `--track-origins=yes` 来跟踪内存泄漏的来源。

4. **性能问题**：Valgrind 会在运行时插入额外的代码来检测内存操作，这可能会导致程序运行缓慢。为了减少性能开销，你可以在开发过程中使用 Valgrind 进行测试，而在生产环境中关闭它。

5. **误报**：Valgrind 可能会产生误报，特别是当程序使用复杂的内存管理技术时。你可以通过仔细分析 Valgrind 的输出来确定是否真的存在问题。

6. **内存分配不匹配**：有时候，内存分配和释放不匹配可能会导致 Valgrind 报告错误。确保每次 `malloc` 都有对应的 `free`，每次 `new` 都有对应的 `delete`。

7. **使用 Valgrind 检测 C++ 程序**：Valgrind 通常用于检测 C 程序，但也可以用于 C++ 程序。确保你的程序是使用 C++ 编译器编译的，并且链接了正确的库。

8. **集成到自动化测试中**：你可以将 Valgrind 集成到你的自动化测试流程中，以便在每次构建后自动运行内存泄漏检测。

9. **处理大量动态内存分配**：如果你的程序中有大量的动态内存分配和释放，Valgrind 的输出可能会变得非常复杂。你可以尝试减少动态内存分配的数量，或者使用智能指针等现代 C++ 特性来帮助管理内存。

通过这些方法，你可以更有效地使用 Valgrind 来检测和解决内存泄漏问题。

## 4.std:unordered_map 和 std:unordered_set 在性能上有什么不同?

`std::unordered_map` 和 `std::unordered_set` 都是基于哈希表实现的无序关联容器。它们在性能上的主要差异通常体现在以下几个方面：

1. **存储内容**：
   - `std::unordered_map` 存储键值对（key-value pairs），允许通过键快速访问、插入或删除对应的值。
   - `std::unordered_set` 仅存储唯一的键（keys），不存储值。

2. **内存使用**：
   - `std::unordered_map` 可能比 `std::unordered_set` 使用更多的内存，因为它需要存储额外的值数据。

3. **操作性能**：
   - 两者在插入、查找和删除操作上都具有平均时间复杂度为 O(1) 的性能，但最坏情况下可能会退化到 O(n)，这取决于哈希函数的质量和冲突解决策略。

4. **遍历性能**：
   - 尽管 `std::unordered_map` 和 `std::unordered_set` 都是无序的，但 `std::unordered_map` 在遍历时可以同时访问键和值，而 `std::unordered_set` 只能访问键。

5. **应用场景**：
   - 如果需要存储额外的数据或者需要通过键快速查找值，`std::unordered_map` 是更好的选择。
   - 如果只需要存储唯一的元素并且关注插入和查找的速度，`std::unordered_set` 可能更适合。

6. **内存碎片**：
   - 由于哈希表的动态扩容特性，两者都可能面临内存碎片的问题，但通常 `std::unordered_map` 因为存储了更多的数据，可能会有更多内存碎片。

7. **内存泄漏风险**：
   - 如果 `std::unordered_map` 的值部分包含指向动态分配内存的指针，那么需要确保这些指针在析构时被正确释放，否则会造成内存泄漏。

总的来说，选择 `std::unordered_map` 还是 `std::unordered_set` 取决于具体的应用需求和性能要求。在大多数情况下，`std::unordered_map` 提供了更灵活的数据存储和访问方式，而 `std::unordered_set` 在只需要存储键的情况下更为轻量级。

## 5.std:hash函数是如何影响哈希表性能的？

`std::hash` 函数在 C++ 中是哈希表性能的关键因素之一，因为它负责为容器（如 `std::unordered_map`、`std::unordered_set` 等）中的每个元素计算一个哈希值，这个哈希值决定了元素在哈希表中的位置。以下是 `std::hash` 函数如何影响哈希表性能的几个方面：

1. **哈希值的分布**：
   - 一个好的哈希函数能够生成均匀分布的哈希值，减少哈希碰撞，从而提高哈希表的性能。如果哈希值分布不均匀，很多元素可能会被分配到同一个桶（bucket），导致性能下降。

2. **冲突的频率**：
   - 冲突是指不同的元素具有相同的哈希值。一个高效的哈希函数可以最小化冲突的发生，因为冲突需要通过链表或其他方法解决，这会增加操作的时间复杂度。

3. **计算的复杂度**：
   - 哈希函数的计算复杂度影响哈希表操作的速度。如果哈希函数过于复杂，会增加计算哈希值的时间，从而影响整体性能。

4. **确定性**：
   - 哈希函数应该是确定性的，即对于同一个输入总是产生相同的哈希值。这有助于确保哈希表的行为是可预测的。

5. **快速失败**：
   - 如果哈希函数能够快速识别出不可能存在的键，那么它可以帮助哈希表快速失败，而不是在解决冲突时浪费时间。

6. **哈希表的负载因子**：
   - 负载因子是哈希表中元素数量与桶数量的比例。一个好的哈希函数可以帮助维持一个较低的负载因子，从而减少重新哈希（rehashing）的需要。

7. **内存使用**：
   - 哈希函数的设计也会影响哈希表的内存使用。一个好的哈希函数可以在保持性能的同时，减少哈希表的内存占用。

8. **特定类型的优化**：
   - 对于某些特定的数据类型，可能有特定的哈希函数能够提供更好的性能。例如，对于字符串，使用 djb2、MurmurHash 或其他专门为字符串设计的哈希函数可能会比标准的 `std::hash` 表现更好。

总的来说，`std::hash` 函数的性能直接影响哈希表的性能。一个好的哈希函数应该能够提供均匀的哈希值分布、减少冲突、快速计算，并且对于特定的数据类型有优化。在某些情况下，可能需要为自定义类型提供自定义的哈希函数，以确保哈希表的性能。

## 6.在C++中，如何自定义一个基于哈希表的容器？

在 C++ 中，自定义一个基于哈希表的容器需要对哈希表的工作原理有深入的理解。以下是创建一个简单的自定义哈希表容器的步骤：

### 6.1. 定义哈希函数

首先，你需要定义一个哈希函数，它将键（key）映射到哈希表的一个桶（bucket）上。

```cpp
struct MyHash {
    std::size_t operator()(const MyKeyType& key) const {
        // 实现键到哈希值的映射
    }
};
```

### 6.2. 定义键值比较函数

定义一个函数来比较两个键是否相等。

```cpp
struct MyKeyEqual {
    bool operator()(const MyKeyType& lhs, const MyKeyType& rhs) const {
        // 实现键的比较逻辑
    }
};
```

### 6.3. 定义桶结构

定义一个存储元素的桶结构，通常是一个链表来解决冲突。

```cpp
struct Bucket {
    std::list<std::pair<MyKeyType, MyValueType>> list;
};
```

### 6.4. 定义哈希表容器

定义你的哈希表容器类，包括桶数组、哈希函数和键值比较函数。

```cpp
template <class KeyType, class ValueType>
class MyHashtable {
private:
    std::vector<Bucket> buckets;
    MyHash hash_function;
    MyKeyEqual key_equal;

    std::size_t bucket_count() const { /* ... */ }
    std::size_t bucket_index(const MyKeyType& key) const { /* ... */ }

public:
    std::pair<bool, ValueType> insert(const MyKeyType& key, const MyValueType& value) { /* ... */ }
    ValueType get(const MyKeyType& key) { /* ... */ }
    void remove(const MyKeyType& key) { /* ... */ }
    // 其他成员函数...
};
```

### 6.5. 实现插入操作

实现元素的插入逻辑，包括计算键的哈希值和解决冲突。

```cpp
std::pair<bool, ValueType> MyHashtable::insert(const MyKeyType& key, const MyValueType& value) {
    std::size_t index = bucket_index(key);
    // 插入逻辑...
}
```

### 6.6. 实现查找操作

实现元素的查找逻辑。

```cpp
ValueType MyHashtable::get(const MyKeyType& key) {
    std::size_t index = bucket_index(key);
    // 查找逻辑...
}
```

### 6.7. 实现删除操作

实现元素的删除逻辑。

```cpp
void MyHashtable::remove(const MyKeyType& key) {
    std::size_t index = bucket_index(key);
    // 删除逻辑...
}
```

### 6.8. 处理动态扩容

当哈希表的负载因子过高时，实现动态扩容逻辑。

```cpp
void MyHashtable::rehash() {
    // 扩容逻辑...
}
```

### 6.9. 测试你的哈希表

创建测试用例来验证你的哈希表容器是否正确实现了所有功能。

```cpp
int main() {
    MyHashtable<int, std::string> hashtable;
    hashtable.insert(1, "Kimi");
    hashtable.insert(2, "Moonshot AI");
    // 测试插入、查找、删除等操作...
    return 0;
}
```

### 6.10. 注意事项

- 确保哈希函数尽可能均匀地分布键值，以减少冲突。
- 考虑负载因子，当桶中元素过多时进行扩容。
- 考虑线程安全，如果需要在多线程环境中使用你的哈希表，可能需要添加锁。
- 考虑异常安全性，确保在插入和删除操作中不会出现资源泄漏。

自定义哈希表是一个复杂的任务，需要仔细设计和测试。在实际应用中，推荐使用标准库中的 `std::unordered_map` 和 `std::unordered_set`，因为它们经过了广泛的测试和优化。如果需要，可以通过特化 `std::hash` 和 `std::equal_to` 来满足自定义类型的需求。

## 7.在C++中，如何优化哈希表的哈希函数以减少冲突？

要优化哈希表的性能，减少冲突是关键。以下是一些优化哈希函数以减少冲突的策略：

1. **均匀分布**：设计哈希函数时，确保它可以将键均匀分布在哈希表中。这可以通过使用质数、随机化技术或根据键的特征来实现。

2. **减少碰撞**：选择或设计哈希函数时，应考虑其在实际使用中的表现，以最小化不同键之间的碰撞。

3. **动态调整哈希表大小**：根据负载因子调整哈希表的大小，以保持较低的冲突率。负载因子是已存储元素数量与哈希表容量的比率。

4. **使用开放寻址法**：当发生冲突时，使用开放寻址法（如线性探测、二次探测或伪随机探测）来寻找下一个空闲位置。

5. **再哈希法**：使用多个哈希函数，当一个哈希函数产生冲突时，使用另一个哈希函数来计算新的位置。

6. **链地址法**：将所有具有相同哈希地址的元素存储在一个链表中。这种方法简单且易于实现，但可能会导致链表过长，从而影响性能。

7. **建立公共溢出区**：为所有冲突的元素创建一个单独的溢出区，可以是另一个哈希表或一个链表。

8. **优化哈希函数的计算**：使用快速且计算简单的哈希函数，如位运算和累加操作，以提高哈希函数的计算速度。

9. **使用现代哈希函数**：考虑使用已经广泛测试和验证的现代哈希函数，如MurmurHash、CityHash或SHA系列。

10. **考虑键的特征**：根据键的特性定制哈希函数，例如，对于字符串键，可以使用基于字符值和位置的哈希函数。

通过实施这些策略，可以显著提高哈希表的性能，减少冲突，并确保即使在高负载情况下也能保持高效的操作。

## 8.如何实现一个高效的哈希表，以提高C++程序的性能？

实现一个高效的哈希表对于提高 C++ 程序的性能至关重要。以下是一些关键步骤和技巧：

### 8.1. 选择合适的哈希函数

- **均匀分布**：确保哈希函数能够将键均匀分布在哈希表中，以减少冲突。
- **快速计算**：选择计算速度快的哈希函数，如使用位运算代替算术运算。
- **定制哈希函数**：针对特定类型的键设计哈希函数，例如，对于字符串，可以使用 DJB2、MurmurHash 或其他专门的哈希算法。

### 8.2. 动态调整哈希表大小

- **负载因子**：监控哈希表的负载因子（已存储元素数量与哈希表容量的比率），并在负载因子过高时扩容。
- **自动扩容**：实现自动扩容机制，通常在哈希表大小的增长达到一定阈值时进行。

### 8.3. 处理冲突

- **链表法**：使用链表解决冲突，但要注意链表的长度，以免影响性能。
- **二次探测**：使用二次探测或其他开放寻址法来减少因冲突导致的性能下降。
- **双散列**：使用双散列技术，通过改变探测步长来减少冲突。

### 8.4. 优化内存使用

- **内存分配**：使用内存池来管理内存分配，以减少动态内存分配的开销。
- **自定义分配器**：考虑使用自定义分配器，以优化内存分配和释放的性能。

### 8.5. 考虑多线程环境

- **线程安全**：如果哈希表将在多线程环境中使用，确保实现适当的同步机制，如使用锁。
- **无锁设计**：考虑使用无锁数据结构来提高并发性能。

### 8.6. 测试和调优

- **性能测试**：使用标准基准测试和分析工具来评估哈希表的性能。
- **分析瓶颈**：使用性能分析工具（如 Valgrind）来识别和解决性能瓶颈。

### 8.7. 使用标准库

- **std::unordered_map**：考虑使用 C++ 标准库中的 `std::unordered_map`，它已经经过了广泛的测试和优化。
- **std::hash**：利用 `std::hash` 来获取标准类型的哈希值，或为自定义类型提供特化的哈希函数。

### 8.8.示例代码

以下是一个简单的哈希表实现示例：

```cpp
#include <vector>
#include <list>
#include <functional>

template <typename K, typename V>
class HashTable {
public:
    explicit HashTable(size_t bucket_count) : buckets(bucket_count) {}

    void insert(const K& key, const V& value) {
        size_t index = hash_key(key);
        auto& bucket = buckets[index];
        for (auto& pair : bucket) {
            if (pair.first == key) {
                pair.second = value; // Update existing value
                return;
            }
        }
        bucket.push_back({key, value});
    }

    V get(const K& key) {
        size_t index = hash_key(key);
        auto& bucket = buckets[index];
        for (auto& pair : bucket) {
            if (pair.first == key) {
                return pair.second;
            }
        }
        throw std::runtime_error("Key not found");
    }

    void remove(const K& key) {
        size_t index = hash_key(key);
        auto& bucket = buckets[index];
        auto it = std::find_if(bucket.begin(), bucket.end(),
                               [&key](const std::pair<K, V>& pair) { return pair.first == key; });
        if (it != bucket.end()) {
            bucket.erase(it);
        }
    }

private:
    std::vector<std::list<std::pair<K, V>>> buckets;

    size_t hash_key(const K& key) {
        return std::hash<K>{}(key) % buckets.size();
    }
};
```

通过遵循这些步骤和技巧，你可以实现一个高效的哈希表，从而提高 C++ 程序的性能。

## 9.我想在哈希表中实现快速查找，有哪些高级技术可以应用？

要实现一个高效的哈希表以提高 C++ 程序的性能，可以考虑以下几个方面：

1. **选择或设计一个好的哈希函数**：哈希函数应该能够将键均匀分布在哈希表中，减少冲突。可以使用质数、随机化技术或根据键的特征来实现。一个好的哈希函数可以显著提高哈希表的性能。

2. **处理冲突**：合理解决冲突是提高哈希表性能的关键。可以使用链地址法（将具有相同哈希值的元素存储在同一个链表中），或者开放寻址法（通过一定的探测策略在哈希表中寻找下一个空闲位置来存储元素）。

3. **动态调整哈希表大小**：根据负载因子（已存储元素数量与哈希表容量的比率）调整哈希表的大小。负载因子过高会导致冲突增多，影响性能。

4. **优化内存使用**：使用内存池来管理内存分配，以减少动态内存分配的开销。

5. **考虑多线程环境**：如果哈希表将在多线程环境中使用，确保实现适当的同步机制。

6. **测试和调优**：使用性能测试工具和分析工具来评估哈希表的性能，并根据测试结果进行调优。

7. **使用标准库**：考虑使用 C++11 标准库中的 `std::unordered_map` 和 `std::unordered_set`，它们已经经过了广泛的测试和优化。

8. **自定义分配器**：可以通过实现自定义分配器来进一步优化内存分配和释放的性能。

9. **使用 `malloc_trim(0)`**：在某些实现中，调用 `malloc_trim(0)` 可以将空闲的堆内存归还给操作系统，有助于减少内存碎片。

通过这些方法，可以有效地提高哈希表的性能，从而提升整个 C++ 程序的性能。

## 10.如何为哈希表容器添加线程安全的特性，以支持多线程环境？

为了在哈希表容器中实现线程安全，可以采取以下几种高级技术：

1. **使用读写锁（`std::shared_mutex`）**：读写锁可以允许多个读线程同时访问容器，但写线程则需要独占访问。这样可以在保持一定并发性的同时，确保写操作的安全性。例如，可以为哈希表的每个桶或分段实现独立的读写锁 。

2. **分段锁（Segmentation Locks）**：类似于 Java 中的 `ConcurrentHashMap`，可以将哈希表分成多个段（Segment），每个段使用独立的锁。这样，不同线程可以同时操作不同的段，从而提高并发性能 。

3. **无锁设计（Lock-free Design）**：使用原子操作（如 CAS - Compare-and-Swap）来实现无锁的哈希表。这种方法可以避免锁的开销，提高并发性能，但实现复杂度较高 。

4. **细粒度锁（Fine-grained Locks）**：为哈希表的每个桶或元素使用独立的锁，这样可以减少锁的粒度，提高并发性，但可能会增加锁的开销 。

5. **使用第三方库**：例如，使用 Intel TBB（Threading Building Blocks）库中的并发哈希表，或者使用其他已经实现线程安全的哈希表库 。

6. **避免共享**：如果可能，避免在多个线程之间共享同一个哈希表实例。每个线程可以使用独立的哈希表，然后通过合并结果来实现线程安全 。

7. **使用 `std::atomic`**：对于某些简单的哈希表操作，可以使用 `std::atomic` 类型来保证操作的原子性，从而避免使用锁 。

8. **自定义同步机制**：根据哈希表的使用场景，可以设计自定义的同步机制，比如使用信号量、条件变量等来控制线程对哈希表的访问 。

9. **利用 `std::mutex`**：在哈希表的每个操作前后使用 `std::mutex` 来确保线程安全。这是一种简单但可能影响性能的方法。

10. **使用 C++17 的 `std::shared_mutex`**：如果你的编译器支持 C++17，可以使用 `std::shared_mutex` 来实现读写锁，提高读操作的并发性 。

选择哪种技术取决于你的具体需求，包括对并发性能的要求、实现的复杂度以及对读操作和写操作的相对频率。在实际应用中，可能需要权衡各种因素来选择最合适的线程安全策略。

---

## 关于作者

- 微信公众号:WeSiGJ
- GitHub:[https://github.com/wesigj/cplusplusboys](https://github.com/wesigj/cplusplusboys)
- CSDN:[https://blog.csdn.net/wesigj](https://blog.csdn.net/wesigj)
- 微博:
- 版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。

<img src=/./img/wechat.jpg width=60% />
