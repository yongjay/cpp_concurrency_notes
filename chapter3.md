# 共享数据

本章讨论以下几个问题：

- 共享数据的问题
- 使用互斥保护数据
- 保护数据的替代方案

## 3.1共享数据的问题

由于不确定线程的访问顺序，A线程在修改共享数据时，可能发生线程B来读取共享数据，此时线程B读到的值不能确定，会存在竞态条件(race condition)。

### 3.1.1条件竞争

**良性条件竞争可以接受**并发中的竞争条件，取决于一个以上线程的执行顺序，每个线程都抢着完成自己的任务。大多数情况下，即使改变执行顺序，也是良性竞争，结果是可以接受的。例如，两个线程同时向一个处理队列中添加任务，因为不变量保持不变，所以谁先谁后都不会有什么影响。

当不变量遭到破坏时，才会产生条件竞争，比如：双向链表的例子。并发中对数据的条件竞争通常表示为**恶性竞争**(我们对不产生问题的良性条件竞争不感兴趣)。C++标准中也定义了数据竞争这个术语，一种特殊的条件竞争：并发的去修改一个独立对象，数据竞争是未定义行为的起因。

### 3.1.2避免恶性条件竞争

保护共享数据结构的最基本的方式，使用C++标准库提供的互斥量。

## 3.2使用互斥量

### 3.2.1互斥量

使用`std::mutex`创建互斥量，一般使用标准库提供的RAII模板类`std::lock_guard`，构造时加锁，析构时解锁：

使用互斥量保护列表：

```c++
#include <list>
#include <mutex>
#include <algorithm>

std::list<int> some_list;    // 1
std::mutex some_mutex;    // 2

void add_to_list(int new_value)
{
  // 此处使用了c++17引入的模版类型推导，简化了代码
  std::lock_guard guard(some_mutex); 
  some_list.push_back(new_value);
}

bool list_contains(int value_to_find)
{
  // 此处使用了c++17引入的模版类型推导，简化了代码
  std::lock_guard guard(some_mutex);   // 4
  return std::find(some_list.begin(),some_list.end(),value_to_find) != some_list.end();
}
```

互斥量和需要保护的数据，在类中都定义为private成员，这会让代码更清晰，并且方便了解什么时候对互斥量上锁。所有成员函数都会在调用时对数据上锁，结束时对数据解锁，这就保证了访问时数据不变量的状态稳定。

**具有访问能力的指针或引用可以访问(并可能修改)保护数据，而不会被互斥锁限制。**

### 3.2.2保护共享数据

保护共享数据不是单纯的增加`lock_guard`那么简单，例如如下的例子，无意中把受保护数据的引用传递出去了：

```c++
class some_data
{
  int a;
  std::string b;
public:
  void do_something();
};

class data_wrapper
{
private:
  some_data data;
  std::mutex m;
public:
  template<typename Function>
  void process_data(Function func)
  {
    std::lock_guard<std::mutex> l(m);
    func(data);    // 1 传递“保护”数据给用户函数
  }
};

some_data* unprotected;

void malicious_function(some_data& protected_data)
{
  unprotected=&protected_data;
}

data_wrapper x;
void foo()
{
  x.process_data(malicious_function);    // 2 传递一个恶意函数
  unprotected->do_something();    // 3 在无保护的情况下访问保护数据
}
```

### 3.2.3接口间的条件竞争

使用了互斥量或其他机制保护了共享数据，就不必再为条件竞争所担忧吗？并不是，依旧需要确定数据是否受到了保护。回想之前双链表的例子，为了能让线程安全地删除一个节点，需要确保防止对这三个节点(待删除的节点及其前后相邻的节点)的并发访问。如果只对指向每个节点的指针进行访问保护，那就和没有使用互斥量一样，条件竞争仍会发生。

以栈为例，接口之间存在条件竞争：

```c++
template<typename T,typename Container=std::deque<T> >
class stack
{
public:
  explicit stack(const Container&);
  explicit stack(Container&& = Container());
  template <class Alloc> explicit stack(const Alloc&);
  template <class Alloc> stack(const Container&, const Alloc&);
  template <class Alloc> stack(Container&&, const Alloc&);
  template <class Alloc> stack(stack&&, const Alloc&);
  
  bool empty() const;
  size_t size() const;
  T& top();
  T const& top() const;
  void push(T const&);
  void push(T&&);
  void pop();
  void swap(stack&&);
  template <class... Args> void emplace(Args&&... args); // C++14的新特性
};
```

削减接口可以获得最大程度的安全,甚至限制对栈的一些操作。

线程安全的堆栈：

```c++
#include <exception>
#include <memory>
#include <mutex>
#include <stack>

struct empty_stack: std::exception
{
  const char* what() const throw() {
	return "empty stack!";
  };
};

template<typename T>
class threadsafe_stack
{
private:
  std::stack<T> data;
  mutable std::mutex m;
  
public:
  threadsafe_stack()
	: data(std::stack<T>()){}
  
  threadsafe_stack(const threadsafe_stack& other)
  {
    std::lock_guard<std::mutex> lock(other.m);
    data = other.data; // 1 在构造函数体中的执行拷贝
  }

  threadsafe_stack& operator=(const threadsafe_stack&) = delete;

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lock(m);
    data.push(new_value);
  }
  
  std::shared_ptr<T> pop()
  {
    std::lock_guard<std::mutex> lock(m);
    if(data.empty()) throw empty_stack(); // 在调用pop前，检查栈是否为空
	
    std::shared_ptr<T> const res(std::make_shared<T>(data.top())); // 在修改堆栈前，分配出返回值
    data.pop();
    return res;
  }
  
  void pop(T& value)
  {
    std::lock_guard<std::mutex> lock(m);
    if(data.empty()) throw empty_stack();
	
    value=data.top();
    data.pop();
  }
  
  bool empty() const
  {
    std::lock_guard<std::mutex> lock(m);
    return data.empty();
  }
};
```

### 3.2.4死锁

一个给定操作需要两个或两个以上的互斥量时，另一个潜在的问题将出现：死锁。与条件竞争完全相反——不同的两个线程会互相等待，从而什么都没做。

可以使用`std::lock()`来锁住多个锁，c++17提供了`std::scoped_lock`可以替换`std::lock()`，从而减少错误的发生：

```
// 这里的std::lock()需要包含<mutex>头文件
class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
  some_big_object some_detail;
  std::mutex m;
public:
  X(some_big_object const& sd):some_detail(sd){}

  friend void swap(X& lhs, X& rhs)
  {
    if(&lhs==&rhs)
      return;
    std::scoped_lock guard(lhs.m,rhs.m);
    swap(lhs.some_detail,rhs.some_detail);
  }
};
```

### 3.2.5避免死锁的进阶指导

- **避免使用嵌套锁**：线程获得一个锁时，就别再去获取第二个。每个线程只持有一个锁，就不会产生死锁。当需要获取多个锁，使用`std::lock`来做这件事(对获取锁的操作上锁)，避免产生死锁。
- **避免在持有锁时调用外部代码**
- **使用固定顺序获取锁**
- **使用层次锁结构**

### 3.2.6 std::unique_lock

**`std::lock_guard`和`std::unique_lock`的区别**

`std::lock_guard`和`std::unique_lock`都是C++标准库中的互斥量封装类，用于在多线程编程中实现资源的互斥访问。它们的主要区别在于灵活性和功能上。

1. **作用范围**:
   - `std::lock_guard`: 用于自动管理互斥量的锁定和解锁，适用于较简单的情况。一旦`std::lock_guard`对象被创建，它会立即锁定给定的互斥量，并在其作用域结束时自动解锁互斥量。
   - `std::unique_lock`: 提供了更大的灵活性。你可以选择在何时锁定和解锁互斥量，甚至可以将锁定的时间范围延长到多个作用域。这使得`std::unique_lock`适用于更复杂的情况，比如需要在锁定期间进行条件变量的等待。

2. **锁定策略**:
   - `std::lock_guard`: 采用的是"锁定即构造，解锁即析构"的策略。当创建`std::lock_guard`对象时，它会尝试锁定互斥量，并在作用域结束时自动解锁。这确保了在离开作用域时互斥量总是被解锁。
   - `std::unique_lock`: 提供了更多的灵活性，可以手动锁定和解锁互斥量。它可以在不同的作用域中锁定和解锁，也可以在不同的时间点进行锁定和解锁。这使得你可以更精细地控制锁定的时机，甚至在锁定期间进行等待。

3. **等待能力**:
   - `std::lock_guard`: 由于其"锁定即构造，解锁即析构"策略，`std::lock_guard`在锁定期间不能手动解锁，也不能在锁定期间等待某些条件。
   - `std::unique_lock`: 具有等待能力，可以在锁定期间手动解锁互斥量，并在解锁期间等待条件变量满足。这在需要在锁定期间等待一些条件时非常有用。

综上所述，如果只需要简单地在一段代码中保护临界区，可以使用`std::lock_guard`。而如果需要更多的灵活性，比如手动锁定/解锁，或者在锁定期间等待条件变量，那么`std::unique_lock`会更适合。

在构造std::unique_lock时可以提供不同的参数来指定如何锁定互斥量，参数的区别分别如下：

在 `std::unique_lock` 的构造函数中，有三个不同的参数：`defer_lock`、`try_to_lock` 和 `adopt_lock`，它们用于控制锁的初始化行为。这些参数用于指定构造函数在创建 `std::unique_lock` 对象时是否应该锁定互斥量，以及如何处理互斥量的锁定状态。以下是它们的区别和在何时使用的情况：

1. **`defer_lock`**:
   - 在构造 `std::unique_lock` 对象时，不会自动锁定互斥量。互斥量会保持解锁状态。
   - 适用于当你希望在稍后的代码中根据需要手动锁定互斥量时，可以使用 `lock()` 成员函数来锁定它。

```cpp
std::unique_lock<std::mutex> myLock(myMutex, std::defer_lock);
// ... some other code ...
myLock.lock(); // 手动锁定互斥量
```

2. **`try_to_lock`**:
   - 在构造 `std::unique_lock` 对象时会尝试锁定互斥量，但不会阻塞。如果锁定成功，则互斥量会被锁定；如果锁定失败（即互斥量已经被其他线程锁定），则 `std::unique_lock` 对象会保持解锁状态。
   - 适用于当你想要尝试锁定互斥量，但又不希望阻塞等待的情况。

```cpp
std::unique_lock<std::mutex> myLock(myMutex, std::try_to_lock);
if (myLock.owns_lock()) {
    // 成功锁定互斥量
} else {
    // 互斥量已被其他线程锁定
}
```

3. **`adopt_lock`**:
   - 假设在构造 `std::unique_lock` 对象之前，你已经手动锁定了互斥量。使用 `adopt_lock` 会将 `std::unique_lock` 对象与已锁定的互斥量关联，以便在 `std::unique_lock` 对象析构时自动解锁互斥量。
   - 适用于当你已经手动锁定互斥量，并希望将锁定的状态传递给 `std::unique_lock` 对象进行管理。

```cpp
std::mutex myMutex;
myMutex.lock();
// ... some other code ...
std::unique_lock<std::mutex> myLock(myMutex, std::adopt_lock);
// 此时 myMutex 已经被锁定，myLock 将管理它的锁定状态
```

综上所述，可以根据具体的情况选择不同的构造函数参数，以便更好地控制互斥量的锁定状态。

### 3.2.7 不同域中互斥量的传递

`std::unique_lock`实例没有与自身相关的互斥量，互斥量的所有权可以通过移动操作，在不同的实例中进行传递。

### 3.2.8锁的粒度

锁的粒度通常用来描述通过一个锁保护着的数据量的大小。细粒度保护较小的数据，粗粒度保护较多的数据。在操作过程中，自己要注意锁的粒度问题。

## 3.3 保护共享数据的方式

### 3.3.1 保护共享数据的初始化过程

对于构建代价很昂贵的资源，在单线程代码中的做法是延迟初始化，在多线程代码中，为了安全的访问，需要使用互斥锁进行同步，以下是一个线程安全的做法：

```cpp
std::shared_ptr<some_resource> resource_ptr;
std::mutex resource_mutex;

void foo()
{
  std::unique_lock<std::mutex> lk(resource_mutex);  // 所有线程在此序列化 
  if(!resource_ptr)
  {
    resource_ptr.reset(new some_resource);  // 只有初始化过程需要保护 
  }
  lk.unlock();
  resource_ptr->do_something();
}
```

这段虽然是线程安全的，但是锁的粒度比较大，降低了性能，有一些方法来优化这段代码，例如双检锁：

```cpp
void undefined_behaviour_with_double_checked_locking()
{
  if(!resource_ptr)  // 1
  {
    std::lock_guard<std::mutex> lk(resource_mutex);
    if(!resource_ptr)  // 2
    {
      resource_ptr.reset(new some_resource);  // 3
    }
  }
  resource_ptr->do_something();  // 4
}
```

这段代码会带来隐藏的条件竞争，其中未被锁保护的读取操作①没有与其他线程里被锁保护的写入操作③进行同步，因此就会产生条件竞争。C++标准库提供了`std::once_flag`和`std::call_once`来处理这种情况。比起锁住互斥量并显式的检查指针，每个线程只需要使用`std::call_once`就可以，在`std::call_once`的结束时，就能安全的知晓指针已经被其他的线程初始化了。

使用`std::call_once`作为类成员的延迟初始化(线程安全)

```cpp
class X
{
private:
  connection_info connection_details;
  connection_handle connection;
  std::once_flag connection_init_flag;

  void open_connection()
  {
    connection=connection_manager.open(connection_details);
  }
public:
  X(connection_info const& connection_details_):
      connection_details(connection_details_)
  {}
  void send_data(data_packet const& data)  // 1
  {
    std::call_once(connection_init_flag,&X::open_connection,this);  // 2
    connection.send_data(data);
  }
  data_packet receive_data()  // 3
  {
    std::call_once(connection_init_flag,&X::open_connection,this);  // 2
    return connection.receive_data();
  }
};
```

`std::call_once` 是 C++ 标准库中的一个函数，用于实现一种仅执行一次的操作。它通常用于在多线程环境中确保某个函数或代码块只会被执行一次，无论有多少个线程尝试调用它。常见的用例包括初始化共享资源、注册回调函数等。以下是一些适合使用 `std::call_once` 的情况：

1. **初始化共享资源**:
   如果多个线程需要共享一个资源并进行初始化，你可以使用 `std::call_once` 来确保初始化只会执行一次。这可以防止多个线程重复初始化资源，避免不一致或错误的状态。

```cpp
std::once_flag flag;
std::shared_ptr<Resource> sharedResource;

void initializeSharedResource() {
    sharedResource = std::make_shared<Resource>();
}

void threadFunction() {
    std::call_once(flag, initializeSharedResource);
    // 使用 sharedResource
}
```

2. **注册回调函数**:
   在某些场景中，你可能希望在程序运行过程中注册一些回调函数，但只需要执行一次。使用 `std::call_once` 可以确保这些回调只会被注册一次，无论多少个线程尝试注册。

```cpp
std::once_flag callbackFlag;

void registerCallback(std::function<void()> callback) {
    std::call_once(callbackFlag, [callback]() {
        // 注册回调函数
        // 这段代码只会被执行一次
    });
}
```

3. **延迟初始化**:
   在需要在多线程环境下延迟初始化某些对象时，可以使用 `std::call_once` 来确保初始化操作只执行一次，从而避免竞态条件和不必要的开销。

```cpp
std::once_flag initializationFlag;
MyClass* globalInstance = nullptr;

void initializeGlobalInstance() {
    globalInstance = new MyClass();
}

MyClass* getGlobalInstance() {
    std::call_once(initializationFlag, initializeGlobalInstance);
    return globalInstance;
}
```

总之，`std::call_once` 在需要确保某个操作仅执行一次的情况下非常有用，能够提供线程安全的执行保证。

在c++11之后的标准中，线程安全的单例：

```cpp
class myclass;
static myclass& getInstance()
{
	static myclass instance_;
	return _instance;
}
```

### 3.3.2 保护不常更新的数据结构

本章实际上描述的是c++中的读写锁，对于一些读比写更加频繁的情况可以使用读写锁，当需要更新共享变量时，短暂的锁住共享变量。读操作则不需要使用锁进行同步。

C++17标准库提供了两种非常好的互斥量——`std::shared_mutex`和`std::shared_timed_mutex`。C++14只提供了`std::shared_timed_mutex`，并且在C++11中并未提供任何互斥量类型。如果还在用支持C++14标准之前的编译器，可以使用Boost库中的互斥量。`std::shared_mutex`和`std::shared_timed_mutex`的不同点在于，`std::shared_timed_mutex`支持更多的操作方式(参考4.3节)，`std::shared_mutex`有更高的性能优势，但支持的操作较少。

使用读写锁开同步DNS的代码示例：

```cpp
#include <map>
#include <string>
#include <mutex>
#include <shared_mutex>

class dns_entry;

class dns_cache
{
  std::map<std::string,dns_entry> entries;
  mutable std::shared_mutex entry_mutex;
public:
  dns_entry find_entry(std::string const& domain) const
  {
    std::shared_lock<std::shared_mutex> lk(entry_mutex);  // 1
    std::map<std::string,dns_entry>::const_iterator const it=
       entries.find(domain);
    return (it==entries.end())?dns_entry():it->second;
  }
  void update_or_add_entry(std::string const& domain,
                           dns_entry const& dns_details)
  {
    std::lock_guard<std::shared_mutex> lk(entry_mutex);  // 2
    entries[domain]=dns_details;
  }
};
```

### 3.3.3嵌套锁

在某些情况下，一个线程会尝试在释放一个互斥量前多次获取。因此，C++标准库提供了`std::recursive_mutex`类。除了可以在同一线程的单个实例上多次上锁，其他功能与`std::mutex`相同。其他线程对互斥量上锁前，当前线程必须释放拥有的所有锁，所以如果你调用lock()三次，也必须调用unlock()三次。正确使用`std::lock_guard<std::recursive_mutex>`和`std::unique_lock<std::recursive_mutex>`可以帮你处理这些问题。
