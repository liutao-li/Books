---
description: '[TOC]'
---

# thread

## condition\_variable

类，定义在头文件[`<condition_variable>`](https://en.cppreference.com/w/cpp/header/condition\_variable)中，用于阻塞线程或同时的多线程，直到其它线程修改共享变量（线程条件）且通知condition\_variable的同步基元。

试图修改条件变量的线程必须：

> 1. 获取锁`std::mutex`，通常为`std::lock_guard`
> 2. 执行修改
> 3. `std::condition_variable`执行`notify_one`或者`notify_all`&#x20;

即使共享变量是原子，为了向等待进程发布正确的修改，也必须在加锁后修改。

试图等待条件变量的线程需要：

> 1. 获取`std::unique_lock<std::mutex>`，与保护条件变量相同的互斥锁；
> 2. 第一种
>    1. 检查条件，以防它已经更新和通知；
>    2. 执行`wait`、 `wait_for`或 `wait_until`。等待操作自动释放互斥锁并暂停线程的执行；
>    3. 当条件变量被通知、超时过期或出现虚假的唤醒时，线程被唤醒，互斥锁被原子地重新获取。如果唤醒是假的，线程应该检查条件并继续等待。
> 3. 第二种
>    1. 使用`wait`、 `wait_for`或 `wait_until`的谓词重载处理上述三个步骤。

| 成员函数                                                                                    | 作用                          |
| --------------------------------------------------------------------------------------- | --------------------------- |
| [notify\_one](https://en.cppreference.com/w/cpp/thread/condition\_variable/notify\_one) | 通知一个等待的线程                   |
| [notify\_all](https://en.cppreference.com/w/cpp/thread/condition\_variable/notify\_all) | 通知所有等待的线程                   |
| [wait](https://en.cppreference.com/w/cpp/thread/condition\_variable/wait)               | 阻塞当前线程，直到条件变量被唤醒            |
| [wait\_for](https://en.cppreference.com/w/cpp/thread/condition\_variable/wait\_for)     | 阻塞当前线程，直到条件变量被唤醒或超过指定的超时时间  |
| [wait\_until](https://en.cppreference.com/w/cpp/thread/condition\_variable/wait\_until) | 阻塞当前线程，直到条件变量被唤醒或到达指定的超时时间点 |

### notify\_one



```cpp
#include <iostream>
#include <atomic>
#include <condition_variable>
#include <thread>
#include <chrono>
 
using namespace std::chrono_literals;
std::condition_variable cv;
std::mutex cv_m; // This mutex is used for three purposes:
                 // 1) to synchronize accesses to i
                 // 2) to synchronize accesses to std::cerr
                 // 3) for the condition variable cv
int i = 0;
 
void waits()
{
    std::unique_lock<std::mutex> lk(cv_m);
    std::cerr << "Waiting... \n";
    cv.wait(lk, []{return i == 1;});
    std::cerr << "Thread waits finished waiting. i == 1\n";
}
// predicate overload
void waits_pred() {
    std::unique_lock<std::mutex> lk(cv_m);
    std::cerr << "Waiting... \n";
    while (i != 1) {
        cv.wait(lk);
    }
    std::cerr << i <<  "Thread waits_pred finished waiting. i == 1\n";
}

void waits_until() {
    std::unique_lock<std::mutex> lk(cv_m);
    auto now = std::chrono::system_clock::now();
    cv.wait_until(lk, now + 100ms, [](){return i == 1;});
    std::cout << "Thread waits_until finished waiting, i == 1\n";
}

void waits_until_pred() {
    std::unique_lock<std::mutex> lk(cv_m);
    auto now = std::chrono::system_clock::now();
    while (i != 1) {
        cv.wait_until(lk, now + 300ms);
    }
    std::cout << "Thread waits_until_pred finished waiting, i == 1\n";
}

void waits_for() {
    std::unique_lock<std::mutex> lk(cv_m);
    cv.wait_for(lk, 100ms, []{return i == 1;});
    std::cout << "Thread waits_for finished waiting, i == 1\n";
}

void waits_for_pred() {
    std::unique_lock<std::mutex> lk(cv_m);
    while (i != 1) {
        cv.wait_for(lk, 200ms);
    }
    std::cout << "Thread waits_for_pred finished waiting, i == 1\n";
}
 
void signals()
{
    {
        std::lock_guard<std::mutex> lk(cv_m);
        std::cerr << "Notifying...\n";
    }
    cv.notify_all();
 
    std::this_thread::sleep_for(std::chrono::milliseconds(200));
 
    {
        std::lock_guard<std::mutex> lk(cv_m);
        i = 1;
        std::cerr << "Notifying again...\n";
    }
    cv.notify_all();
}
 
int main()
{
    std::thread t1(waits), t2(waits_pred);
    std::thread t3(waits_until), t4(waits_until_pred);
    std::thread t5(waits_for), t6(waits_for_pred);
    std::thread t7(signals);
    t1.join(); 
    t2.join(); 
    t3.join();
    t4.join();
    t5.join();
    t6.join();
    t7.join();
}
```



## future



\[future]\([https://en.cppreference.com/w/cpp/thread/future](https://en.cppreference.com/w/cpp/thread/future))类，定义在头文件`<future>`中，类模板`std::future`提供了一种访问异步操作结果的机制。

* 一个异步操作（通过`std::async`、`std::packaged_task`或`std::promise`创建）可以向异步操作的创建者提供一个`std::future`对象；
* 异步操作的创建者可以使用各种方法来查询、等待或从`std::future`中提取值。如果异步操作尚未提供值，这些方法可能会被阻塞；
* 当异步操作者准备将结果发送给创建者时，它可以通过修改链接到创建者的`std::future`的共享状态（例如`std::promise::set_value`）来实现。

注意：`std::future引`用的共享状态不与任何其它异步操作返回对象共享（与`std::shared_future`相反）。

### 成员函数

| 函数          | 功能                                |
| ----------- | --------------------------------- |
| operator=   | 移动future对象                        |
| share       | 将共享状态从\*this转移到shared\_future并返回它 |
| get         | 返回结果                              |
| valid       | 检查shared是否有共享状态                   |
| wait        | 等待结果变得可用                          |
| wait\_for   | 等待结果，如果在指定的超时时间内该结果不可用则返回         |
| wait\_until | 等待结果，如果到达指定的时间点结果不可用则返回           |



