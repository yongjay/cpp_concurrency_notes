# 线程管理

## 2.1线程的基本操作

使用C++线程库启动线程，就是构造`std::thread`对象。

启动线程需要注意，当把函数对象传入到线程构造函数中时，需要避免语法解析的错误：

```c++
std::thread my_thread(background_task());
```

这相当于声明了一个名为my\_thread的函数，这个函数带有一个参数\(函数指针指向没有参数并返回background\_task对象的函数\)，返回一个`std::thread`对象的函数。

解决办法是使用lambda或者增加多个括号或者使用大括号

```c++
std::thread my_thread((background_task()));  // 1
std::thread my_thread{background_task()};    // 2
```

```c++
std::thread my_thread([]{
  do_something();
  do_something_else();
});
```

线程启动之后需要在线程对象析构之前决定线程是否需要汇入，join还是detach，如果不决定，程序会调用`std::terminate()`来终止程序，如果选择了detach，则需要注意在线程是否可能会访问会被销毁的变量。

线程只能join一次，可以通过`joinable()`来检查

## 2.2传递参数

向可调用对象或函数传递参数很简单，只需要将这些参数作为 `std::thread`构造函数的附加参数即可。需要注意的是，这些参数会拷贝至新线程的内存空间中(同临时变量一样)。即使函数中的参数是引用的形式，拷贝操作也会执行。

由于无法保证隐式转换的操作和`std::thread`构造函数的拷贝操作的顺序，所以需要避免在传递参数给线程函数时发生变量的隐式转换，需要显示转换为线程函数所需要的类型。

```c++
void f(int i,std::string const& s);
void oops(int some_param)
{
  char buffer[1024]; // 1
  sprintf(buffer, "%i",some_param);
  std::thread t(f,3,buffer); // 在传递到线程函数时char*到string的类型转换可能发生在oops函数结束之后，可能会导致未定义的行为
  // 解决方案是在参数传入线程函数之前将buffer转换为string
  // stf::thread t(f,3,std::string(buffer));
  t.detach();
}
```

如果线程函数的参数为一个非常量引用，需要添加`std::ref()`来包裹实参，否则会引起编译错误，这是因为thread的构造函数默认拷贝提供的变量，内部代码会将拷贝的参数以右值的方式进行传递，这是为了那些只支持移动的类型，而后会尝试以右值为实参调用update_data_for_widget。但因为函数期望的是一个非常量引用作为参数(而非右值)，所以会在编译时出错。

对于**不支持拷贝仅支持移动**的类型，需要使用`std::move()`将参数以移动的方式传递到线程函数中。

```c++
void process_big_object(std::unique_ptr<big_object>);

std::unique_ptr<big_object> p(new big_object);
p->prepare_data(42);
std::thread t(process_big_object,std::move(p));
```

## 2.3转移所有权

`std::thread`支持移动但不支持拷贝，但是线程对象支持作为函数的参数和函数的返回值，由于线程不支持拷贝，此时会隐式的调用`std::move()`。

```c++
std::thread f()
{
  void some_function();
  return std::thread(some_function);
}

std::thread g()
{
  void some_other_function(int);
  std::thread t(some_other_function,42);
  return t;
}
```

## 2.4确定线程数量

在c++11之后的版本中可以使用`std::thread::hardware_concurrency()`来确定机器最大支持的并发数。

完整的并行版std::accumulate。

```c++
#include <iostream>
#include <thread>
#include <vector>
#include <algorithm>
#include <numeric>

// 单个块的计算
template <typename Iterator, typename T>
struct accumulate_block
{
    void operator()(Iterator first, Iterator last, T &result)
    {
        result = std::accumulate(first, last, result);
    }
};

template <typename Iterator, typename T>
T parallel_accumulate(Iterator first, Iterator last, T init)
{
    // 计算数据的长度
    unsigned long const length = std::distance(first, last);

    if (!length) // 1
        return init;

    // 设定每个线程计算的数据个数
    unsigned long const min_per_thread = 25;
    // 计算最大的线程个数
    unsigned long const max_threads =
        (length + min_per_thread - 1) / min_per_thread; // 2

    // 机器支持的最大并发个数,核心个数
    unsigned long const hardware_threads =
        std::thread::hardware_concurrency();

    unsigned long const num_threads = // 3
        std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);

    unsigned long const block_size = length / num_threads; // 4

    std::vector<T> results(num_threads);
    // 此处减一的原因在于在启动之前主线程已经在运行了
    std::vector<std::thread> threads(num_threads - 1); // 5

    Iterator block_start = first;
    for (unsigned long i = 0; i < (num_threads - 1); ++i)
    {
        Iterator block_end = block_start;
        std::advance(block_end, block_size); // 6
        threads[i] = std::thread(            // 7
            accumulate_block<Iterator, T>(),
            block_start, block_end, std::ref(results[i]));
        block_start = block_end; // 8
    }

    // 此处处理剩余的未被其他线程处理的结果
    accumulate_block<Iterator, T>()(
        block_start, last, results[num_threads - 1]); // 9

    // 汇入所有的线程
    for (auto &entry : threads)
        entry.join(); // 10

    // 将各个块的结果累加返回
    return std::accumulate(results.begin(), results.end(), init); // 11
}

int main()
{
    std::vector<int> my_vec{3, 4, 5, 5, 6, 6, 4, 3, 3, 2, 2, 3, 4, 4, 4};
    auto result = std::accumulate(my_vec.begin(), my_vec.end(), 0);
    auto concurrency_result = parallel_accumulate(my_vec.begin(), my_vec.end(), 0);

    // print
    std::cout << "result:" << result << "concurrency_result:" << concurrency_result << std::endl;
}
```



## 2.5识别线程

线程ID可以作为线程的唯一标识符，获取线程的ID有：

```c++
// 通过thread对象来获取
thread t;
t.get_id();
// 在线程中
std::this_thread::get_id();
```

