# 同步操作

本章主要内容

- 带有future的等待
- 在限定时间内等待
- 使用同步操作简化代码

## 4.1 等待事件或条件

C++标准库对条件变量有两套实现：`std::condition_variable`和`std::condition_variable_any`，两者都需要与互斥量一起才能工作，前者只能和`std::mutex`一起工作，后者可以和多种`mutex`工作,但会带来更多的性能开销。

### 4.1.1等待条件达成

使用wait函数等待条件达成，注意防止虚假唤醒，以下给出一个cppreference使用条件变量的示例代码：

```cpp
#include <iostream>
#include <string>
#include <thread>
#include <mutex>
#include <condition_variable>
 
std::mutex m;
std::condition_variable cv;
std::string data;
bool ready = false;
bool processed = false;
 
void worker_thread()
{
    // Wait until main() sends data
    std::unique_lock lk(m);
    cv.wait(lk, []{return ready;});
 
    // after the wait, we own the lock.
    std::cout << "Worker thread is processing data\n";
    data += " after processing";
 
    // Send data back to main()
    processed = true;
    std::cout << "Worker thread signals data processing completed\n";
 
    // Manual unlocking is done before notifying, to avoid waking up
    // the waiting thread only to block again (see notify_one for details)
    lk.unlock();
    cv.notify_one();
}
 
int main()
{
    std::thread worker(worker_thread);
 
    data = "Example data";
    // send data to the worker thread
    {
        std::lock_guard lk(m);
        ready = true;
        std::cout << "main() signals data ready for processing\n";
    }
    cv.notify_one();
 
    // wait for the worker
    {
        std::unique_lock lk(m);
        cv.wait(lk, []{return processed;});
    }
    std::cout << "Back in main(), data = " << data << '\n';
 
    worker.join();
}
```

### 4.1.2构建线程安全队列

一个线程安全的队列代码：

```c++
#include <queue>
#include <memory>
#include <mutex>
#include <condition_variable>

template<typename T>
class threadsafe_queue
{
private:
  mutable std::mutex mut;  // 1 互斥量必须是可变的 
  std::queue<T> data_queue;
  std::condition_variable data_cond;
public:
  threadsafe_queue()
  {}
  threadsafe_queue(threadsafe_queue const& other)
  {
    std::lock_guard<std::mutex> lk(other.mut);
    data_queue=other.data_queue;
  }

  void push(T new_value)
  {
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(new_value);
    data_cond.notify_one();
  }

  void wait_and_pop(T& value)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    value=data_queue.front();
    data_queue.pop();
  }

  std::shared_ptr<T> wait_and_pop()
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk,[this]{return !data_queue.empty();});
    std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
    data_queue.pop();
    return res;
  }

  bool try_pop(T& value)
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return false;
    value=data_queue.front();
    data_queue.pop();
    return true;
  }

  std::shared_ptr<T> try_pop()
  {
    std::lock_guard<std::mutex> lk(mut);
    if(data_queue.empty())
      return std::shared_ptr<T>();
    std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
    data_queue.pop();
    return res;
  }

  bool empty() const
  {
    std::lock_guard<std::mutex> lk(mut);
    return data_queue.empty();
  }
};
```

条件变量在多个线程等待同一个事件时也很有用。当线程用来分解工作负载，并且只有一个线程可以对通知做出反应时，与代码4.1中结构完全相同。当数据准备完成时，调用notify_one()将会唤醒一个正在wait()的线程，检查条件和wait()函数的返回状态(因为仅是向data_queue添加了一个数据项)。 这里不保证线程一定会被通知到，即使只有一个等待线程收到通知，其他处理线程也有可能因为在处理数据，而忽略了这个通知。

另一种可能是，很多线程等待同一事件。对于通知，都需要做出回应。这会发生在共享数据初始化的时候，当处理线程使用同一数据时，就要等待数据被初始化，或等待共享数据的更新，比如：*周期性初始化*(periodic reinitialization)。这些情况下，线程准备好数据时，就会通过条件变量调用notify_all()，而非调用notify_one()。顾名思义，这就是全部线程在都去执行wait()(检查他们等待的条件是否满足)的原因。

当条件为true时，等待线程只等待一次，就不会再等待条件变量了，所以尤其是在等待一组可用的数据块时，一个条件变量并非同步操作最好的选择。

## 4.2使用future

### 4.2.1 后台任务的返回值

C++标准库中有两种future，`std::unique_ptr`和`std::shared_ptr`非常类似。`std::future`只能与指定事件相关联，而`std::shared_future`就能关联多个事件。后者的实现中，所有实例会在同时变为就绪状态，并且可以访问与事件相关的数据。

future对象本身并不提供同步访问。当多个线程需要访问一个独立future对象时，必须使用互斥量或类似同步机制进行保护。不过，当多个线程对一个`std::shared_future<>`副本进行访问，即使同一个异步结果，也不需要同步future。

`std::future`通过`std::async`来返回，future的等待取决于`std::async`是否启动一个线程，或是否有任务在进行同步。

可以通过参数指定`std::async`的执行方式，`std::launch::defered`，表明函数调用延迟到wait()或get()函数调用时才执行，`std::launch::async`表明函数必须在其所在的独立线程上执行。

### 4.2.2future与任务绑定

`std::packaged_task<>`会将future与函数或可调用对象进行绑定。当调用`std::packaged_task<>`对象时，就会调用相关函数或可调用对象，当future状态为就绪时，会存储返回值。`std::packaged_task`是个可调用对象，可以封装在`std::function`对象中，从而作为线程函数传递到`std::thread`对象中，或作为可调用对象传递到另一个函数中或直接调用。

以下是几个使用packaged_task的示例：

```c++
#include <iostream>
#include <cmath>
#include <thread>
#include <future>
#include <functional>
 
// unique function to avoid disambiguating the std::pow overload set
int f(int x, int y) { return std::pow(x,y); }
 
void task_lambda()
{
    std::packaged_task<int(int,int)> task([](int a, int b) {
        return std::pow(a, b); 
    });
    std::future<int> result = task.get_future();
 
    task(2, 9);
 
    std::cout << "task_lambda:\t" << result.get() << '\n';
}
 
void task_bind()
{
    std::packaged_task<int()> task(std::bind(f, 2, 11));
    std::future<int> result = task.get_future();
 
    task();
 
    std::cout << "task_bind:\t" << result.get() << '\n';
}
 
void task_thread()
{
    std::packaged_task<int(int,int)> task(f);
    std::future<int> result = task.get_future();
 		// 注意此处使用了std::move，因为packaged_task内部持有一个可调用对象，而这个可调用对象是不可拷贝的，因此需要使用移动操作
    std::thread task_td(std::move(task), 2, 10);
    task_td.join();
 
    std::cout << "task_thread:\t" << result.get() << '\n';
}
 
int main()
{
    task_lambda();
    task_bind();
    task_thread();
}
```

### 4.2.3 使用std::promises

`std::promise` 是 C++ 标准库提供的一个类模板，用于在多线程编程中，将一个值或异常与一个将来可能会设置该值或异常的异步操作关联起来。它通常与 `std::future` 一起使用，用于在一个线程中设置值或异常，然后在另一个线程中等待并获取这个值或异常。

promise提供了一种线程同步的手段。

示例代码：

```c++
#include <iostream>       // std::cout
#include <functional>     // std::ref
#include <thread>         // std::thread
#include <future>         // std::promise, std::future

void print_int(std::future<int>& fut) {
    int x = fut.get(); // 获取共享状态的值. 
    std::cout << "value: " << x << '\n'; // 打印 value: 10.
}

int main ()
{
    std::promise<int> prom; // 生成一个 std::promise<int> 对象.
    std::future<int> fut = prom.get_future(); // 和 future 关联.
    std::thread t(print_int, std::ref(fut)); // 将 future 交给另外一个线程t.
    prom.set_value(10); // 设置共享状态的值, 此处和线程t保持同步.
    t.join();
    return 0;
}
```

### 4.2.4将异常存与future中

可以通过调用`set_exception`设置异常值到`std::promise`中。

### 4.2.5多个线程的等待

