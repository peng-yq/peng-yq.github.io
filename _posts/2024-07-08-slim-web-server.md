---
layout: post
title: Slim Web Server
subtitle: Introduction to Slim Web Server
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - C++
---

本文是对个人项目 [Slim Web Server的详细介绍](https://github.com/peng-yq/slim-web-server)。

## Overview

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/202407081243298.png"/>

主要特性：

- 利用标准库容器封装char，实现动态增长的字节流缓冲区；
- 利用锁、条件变量和队列实现单生产者消费者的阻塞队列；
- 利用单例模式与阻塞队列实现支持异步和同步的日志系统，记录服务器运行状态；
- 基于小根堆实现的定时器，关闭超时的非活动连接，节约系统资源；
- 利用RAII机制实现了数据库连接池，减少频繁打开和关闭数据库连接的开销，提高数据库操作的效率；
- 利用正则与状态机解析HTTP请求报文，实现处理静态资源的请求、用户注册和登录；
- 利用IO复用技术Epoll与线程池实现多线程的Reactor高并发模型，使用webbench-1.5进行压力测试可以实现上万的QPS；

## Server

`Server`类是本项目的核心所在，通过初始化并启动实现对客户端请求的监听和事件处理。并通过`IO`复用技术`Epoll`与线程池实现多线程的`Reactor`高并发模型。

### 初始化

1. 初始化`web server`配置：
   - 端口号 (`port_`)
   - 触发模式 (`trigMode`)
   - 连接超时时间 (`timeoutMs_`)
   - 是否启用监听套接字的LINGER选项 (`openLinger_`)
   - 数据库相关设置
   - 定时器（`timer_`）
   - `Epoll`
   - 线程池大小 (`threadNum`)
   - 日志系统设置
   - 资源目录（`srcDir_`）
2. 初始化日志系统，设置日志级别、日志文件存放路径和队列大小。
3. 初始化数据库连接池，配置包括数据库服务器地址、端口、用户名、密码、数据库名以及连接池大小。
4. 根据传入的触发模式 (`trigMode`) 设置epoll的事件监听模式，决定监听事件和连接事件是使用边缘触发还是水平触发。
5. 调用`InitListenSocket_() `方法初始化监听套接字。如果初始化失败，设置`isClose_ `标志为`true`，表示服务器初始化失败。初始化监听套接字会根据`openLinger_`决定是否开启LINGER选项，初始化成功后会将监听套接字fd加入epoll进行监听。
6. 如果启用了日志，根据`isClose_`的状态记录不同的日志信息。如果服务器初始化成功，记录服务器的配置信息，如端口、是否启用`SO_LINGER`、监听模式、日志级别、资源目录、数据库连接池容量和线程池容量等。

### 启动

1. 设置`epoll_wait`超时：
   - `timeMs`初始化为`-1`，意味着如果没有任何事件，`epoll_wait`将会阻塞，直到有事件发生。
2. 检查服务器是否关闭：
   - 如果`isClose_`标志为true，则不进入主循环，服务器不会启动。
   - 如果服务器正常启动，记录启动日志。
3. 主事件循环：
   - 循环继续执行，直到`isClose_`被设置为 true。
   - 如果设置了超时时间 (`timeoutMs_ `> 0)，通过`timer_`对象调用`GetNextTick()`方法（此函数会先将小根堆中已经超时的连接调用回调函数进行取消连接）计算下一个最早过期事件的时间。这个时间用于`epoll_wait`的超时参数，以便在没有网络事件时处理超时连接。`GetNextTick()`方法会清理超时的连接。
4. 调用`epoll_wait`监听事件：
   - 使用`epoller_->Wait(timeMs)`监听文件描述符上的事件，`timeMs`可能是超时时间或`-1`（永不超时）。`-1`对应定时器中没有添加的`http`连接，即`epoll`监听监听套接字是否有可读事件，只要无事件发生（无连接）就一直阻塞。返回的是发生事件数量`eventCnt`。
5. 处理每个事件：
   - 遍历每个事件，使用`epoller_->GetEventFd(i)`和`epoller_->GetEvents(i)`获取事件相关的文件描述符fd和事件类型`events`。
6. 根据不同的情况处理事件：
   - 监听套接字事件：如果`fd`是监听文件描述符`listenFd`_，调用`DealListen_()`方法接受新的连接。如果使用的是边缘触发（`EPOLLET`）模式，则需要在一个循环中不断调用 `accept`，直到没有更多的连接请求。这是因为在边缘触发模式下，一个事件可能意味着有多个连接请求需要处理。`DealListen_()`方法会将连接的套接字描述符加入定时器和`epoll`进行监听（读事件，也就是监听`httprequest`）。
   - 错误或挂起的连接：如果事件类型包含`EPOLLRDHUP`、`EPOLLHUP`或`EPOLLERR`，表示连接出现错误或挂起，调用`CloseConn_()`方法关闭连接，并从`epoll`队列中删除。
   - 可读事件：如果事件类型包含`EPOLLIN`，表示数据可读，调用`DealRead_()`方法处理读取数据。`DealRead_()`方法首先会去读套接字中的内容至读缓冲区，并调用`OnProcess_`方法处理http事件，该方法首先会调用`HttpConn->Process()`方法去处理`http`请求，并构造对应的`http`响应至写缓冲区。此外，`Process`方法会根据返回修改该连接套接字的监听事件类：返回`true`则监听可写事件，用于将`http`响应进行发送；返回`false`表示当前没有`http`请求，需要监听可读事件。
   - 可写事件：如果事件类型包含`EPOLLOUT`，表示连接可写，调`用DealWrite_()`方法处理数据发送。`DealWrite_()`方法首先会让HttpConn去往套接字写入写缓冲区的内容，也就是构造的`http`响应，如果全部写完则根据`http`请求是否需要保持连接（`keep-alive`）决定是否需要继续处理http连接，否则直接关闭。如果没有完全写完，则继续监听可写事件。
   - 异常处理：如果事件不是以上任何一种已知类型，记录错误日志。

> 处理可读可写事件均是将任务推送至线程池任务队列，由工作线程来取出完成。此外，在处理可读和可写事件时，均会调整该`fd`的超时时间，避免因超时而未处理

### 单reactor多线程

1. 单个`Reactor`（主线程）：单个`Reactor`运行在主线程（`WebServer.Start()`）中，负责监听所有的`I/O`事件，包括新的客户端连接请求以及现有连接上的读写事件。`Reactor`使用非阻塞`I/O`和事件通知机制（`epoll`），能够高效地处理成千上万的并发连接。
2. 事件分发：当事件发生时，`Reactor`会从`epoll`事件队列中获取事件，并根据事件类型（新连接、数据读取、数据写入等）将事件分派给相应的处理程序（ `DealListen_`, `DealRead_`或`DealWrite_`方法。）。
3. 多线程处理：对于读写事件，`Reactor`不直接处理具体的数据读取或写入逻辑。相反，它将这些任务委托给后端的线程池。这样可以快速地返回到事件循环中，继续监听其他`I/O`事件，而具体的`I/O`处理则由工作线程并行处理。
4. 异步执行：工作线程处理完任务后，需要再次通知`Reactor`线程（主线程）以进行进一步的操作，如发送响应到客户端。这里是通过修改监听对应套接字描述符的事件状态或再次注册事件来实现。

### epoll

```c++
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;    /* Epoll events */
    epoll_data_t data;      /* User data variable */
};
```

`events`：这个字段是一个位掩码，指定了需要监听的事件类型。常见的事件类型包括：

- `EPOLLIN`：表示对应的文件描述符可以读取数据（如有新的连接请求或者有数据可读）。
- `EPOLLOUT`：表示对应的文件描述符可以写入数据（如写缓冲区有空间可写）。
- `EPOLLET`：将 epoll 行为设置为边缘触发（Edge Triggered）模式，这是与水平触发（Level Triggered）模式相对的一种高效模式。
- `EPOLLERR`：表示对应的文件描述符发生了错误。
- `EPOLLHUP`：表示对应的文件描述符被挂断。
- `EPOLLRDHUP`：表示套接字的远端关闭或半关闭连接。
- `EPOLLONESHOT`：表示一次性监听，事件发生一次后，如果需要再次监听该文件描述符的话，需要再次把它加入到 EPOLL 队列中。

```c++
listenEvent_ = EPOLLRDHUP;
connEvent_ = EPOLLRDHUP | EPOLLONESHOT;
```

`EPOLLONESHOT`标志是一种特殊的`epoll`行为模式，用于确保一个`socket`连接上的事件在任何时刻只被一个线程处理。设置了`EPOLLONESHOT`后，一旦`epoll`报告了某个事件，该事件会被自动从`epoll`监听集合中移除，直到应用程序再次显式地重新将这个事件添加到`epoll`集合中。这样做主要是为了防止多个线程同时处理同一个`socket`的相同事件，从而避免竞态条件和不一致的情况。

为什么只对`connEvent_`设置`EPOLLONESHOT`：

- 连接处理复杂性：对于服务器来说，处理客户端的连接通常涉及读取数据、处理请求和发送响应等多个步骤。这些步骤可能需要更多的状态管理和更复杂的处理逻辑，尤其是在多线程环境中。使用`EPOLLONESHOT`可以确保连接处理的独占性，避免多个线程同时操作同一个连接造成数据错乱。
- 监听套接字与连接套接字的区别：`listenEvent`_通常用于监听套接字，它主要负责接受新的连接请求。监听套接字的事件（新的连接请求）相对简单，不涉及复杂的状态或数据处理，直接在主线程中处理，因此通常不需要`EPOLLONESHOT`。一旦接受了新的连接，相应的连接套接字（由 `connEvent_`管理）将负责后续的数据交换和更复杂的交互。
- 性能考虑：如果监听套接字也使用`EPOLLONESHOT`，那么每处理完一个新的连接请求后，服务器必须显式地重新将监听事件添加到`epoll`集合中。这会增加额外的系统调用开销，并可能降低服务器接受新连接的能力。

### Linger

```c++
optLiner.l_onoff = 1;
optLiner.l_linger = 1;
```

当设置了这样的`linger`选项后，套接字的关闭行为会按照以下方式改变：

- 如果有未发送完的数据在套接字的发送缓冲区中：关闭套接字的操作（例如调用` close()`）不会立即返回。相反，操作系统会延迟套接字的实际释放，最多延迟`l_linger`指定的秒数。在这个例子中，系统会最多延迟`1`秒。
- 在这`1`秒内：操作系统会尝试继续发送缓冲区中的数据，并等待网络对端确认。
- 如果在`1`秒内数据成功发送并得到确认：套接字正常关闭，`close()`调用返回。
- 如果`1`秒后仍有数据未被发送或未得到确认：`close()`调用将返回，未发送完的数据可能会丢失，套接字被强制关闭。

## Buffer

动态字节缓冲区，用于处理数据的临时存储，优化数据的读取和发送过程，主要通过`vector`对`char`进行封装，并使用读写指针。

> 一开始在`buffer`的实现中使用了互斥锁来实现线程安全，后面发现写来死锁了。但其实在`slim web server`的实现中，`buffer`这里可以不要互斥锁，锁的获取和释放也是需要代价的，并且场景中基本没有多个线程同时操作一个`buffer`。另外`buffer`都是封装在其它类中使用，其它类均使用互斥同步机制确保了线程安全。只需要保证对`readPos_`和`writePos_`的操作是原子操作即可

**主要特性**

- 动态扩展：根据需要动态调整缓冲区大小，以适应不同大小的数据负载。
- 高效的内存管理：优化内存使用，减少数据拷贝，提高性能。
- 支持多种数据操作：提供多种方法来读取、写入、追加和清除缓冲区中的数据。

**缓冲区操作**

- 读写指针管理：通过管理读写指针来优化数据的处理，避免不必要的数据复制。
- 自动扩容：当可写空间不足时，缓冲区能自动扩容以存储更多数据。
- 数据追加：支持多种数据类型的追加，包括字符串、原始数据和其他缓冲区的内容。

## Block Deque

线程安全的阻塞双端队列（`BlockDeque`）使用`deque`，并通过模板类实现，支持在多线程环境中进行同步访问。采用生产者-消费者模式，确保数据的安全生产和消费，适合用于任务调度、消息队列等场景。`Slim Web Server`项目中主要用于服务日志的异步写，日志服务作为生产者往队列中推送需要记录的日志条目，异步写线程作为消费者取出写入日志文件。

**主要特性**

- 线程安全：通过互斥锁（`mutex`）和条件变量（`condition_variable`）确保在多线程环境下的互斥和同步。
- 阻塞操作：支持阻塞的数据插入和移除操作，当队列满或空时，生产者和消费者线程将会等待。
- 条件通知：使用条件变量来实现线程的正确唤醒和阻塞，优化了线程间的协调。
- 动态控制：允许在运行时清空队列、关闭队列，并通知所有阻塞的线程。
- 容量限制：队列大小有上限，防止无限制增长导致的资源耗尽。

**互斥锁和条件变量**

- 互斥锁：保护队列的内部状态，确保在多线程操作时数据的一致性和完整性。
- 条件变量：用于在队列为空时阻塞消费者线程，在队列满时阻塞生产者线程。当条件得到满足时，相应的线程会被唤醒。

**生产者-消费者模式**

> 项目里的应用是单生产者单消费者模式

生产者：负责向队列中添加新的元素，并通知阻塞的消费者。如果队列已满，则生产者线程会阻塞，直到队列中有空间可用。
消费者：从队列中取出元素进行处理，并通知阻塞的生产者。如果队列为空，消费者线程会阻塞，直到队列中有新的元素可用。

## Log

高效、线程安全的日志系统，**支持同步和异步日志记录**。每日的日志将会被保存在一个文件中，并且单个文件有最大行数限制，如果超过则生成单日的第二个日志文件。 

**主要特性**

- 线程安全：确保多线程环境下日志记录的正确性。
- 支持同步与异步日志记录：可配置为同步或异步模式，以满足不同的性能需求。
- 自动日志文件管理：自动按日期管理和分割日志文件。
- 多级别日志：支持不同级别的日志记录（`debug`、`info`、`warn`、`error`）。

**单例模式**

使用单例模式设计，确保全局只有一个日志系统实例。这样可以集中管理日志记录资源，并减少资源使用冲突。单例通过一个静态方法`Instance()`实现，保证了实例的唯一性和线程安全的初始化。

**std::unique_ptr**

使用`unique_ptr`管理资源，如异步写线程和阻塞队列，确保资源的正确释放。`unique_ptr`是一个智能指针，它拥有其指向的对象，在`unique_ptr`被销毁时会自动释放所管理的对象。

**std::mutex**

使用互斥锁`std::mutex`来保护共享数据，确保日志操作的线程安全。在写入文件或修改共享状态时，会先锁定互斥锁，操作完成后再释放，从而避免数据竞态。

**同步与异步写**

- 同步写：直接将缓冲区的日志信息写入文件，适用于对日志实时性要求较高的场景。
- 异步写：将缓冲区的日志消息首先放入一个阻塞队列中，由后台线程负责将队列中的日志信息批量写入文件。这种方式可以显著减少日志写入对程序性能的影响，适用于高并发环境。

**宏定义**

提供了一系列宏定义，如`LOG_DEBUG`、`LOG_INFO`、`LOG_WARN`、`LOG_ERROR`，包装了日志级别、格式化输出等操作。

## Timer

使用小根堆实现的定时器，关闭超时的非活动连接。通过维护一个小根堆，能够高效地管理和执行定时器。最小堆的插入和删除操作都在对数时间内完成，因此能够确保定时器管理的高效性。本项目使用`vector`模拟小根堆，并使用`unordered_map<int, size_t>`标记每个节点在`vector`中的位置。

每个`timer`节点的属性包括：

1. `int id`：连接套接字的文件描述符号
2. `Timestamp expires`：到期时间
3. `TimeoutCallBack cb`：回调函数，`TimeoutCallBack`是`std::function<void()>`类型

`SiftUp_` 方法用于在插入新节点或调整节点到期时间时，将节点向上移动以维持最小堆的性质。

```cpp
void Timer::SiftUp_(size_t i) {
    assert(i >= 0 && i < heap_.size());
    while (i > 0) {
        size_t j = (i - 1) / 2; // 计算父节点的索引
        if (heap_[j] < heap_[i]) {
            break;
        }
        SwapNode_(i, j); // 交换节点
        i = j;
    }
}
```

`SiftDown_` 方法用于在删除堆顶元素或调整节点到期时间时，将节点向下移动以维持最小堆的性质。

```cpp
void Timer::SiftDown_(size_t i, size_t n) {
    assert(i >= 0 && i < heap_.size());
    assert(n >= 0 && n <= heap_.size());
    size_t j = i * 2 + 1; // 计算左子节点的索引
    while (j < n) {
        if (j + 1 < n && heap_[j + 1] < heap_[j]) {
            j++;
        }
        if (heap_[i] < heap_[j]) {
            break;
        }
        SwapNode_(i, j); // 交换节点
        i = j;
        j = i * 2 + 1;
    }
}
```

`Add` 方法用于添加新的定时器节点，将元素添加到堆的末尾，然后通过“向上调整”（sift-up）操作将其移动到正确的位置，以维持堆的性质。如果定时器已经存在，则更新其到期时间和回调函数，并根据新的到期时间调整堆。

```cpp
void Timer::Add(int id, int timeOut, const TimeoutCallBack& cb) {
    assert(id > 0);
    size_t i;
    if (ref_.count(id) == 0) {
        i = heap_.size();
        ref_[id] = i;
        heap_.push_back({id, HighResolutionClock::now() + Milliseconds(timeOut), cb});
        SiftUp_(i);
    } else {
        i = ref_[id];
        auto oldExpires = heap_[i].expires;
        auto newExpires = HighResolutionClock::now() + Milliseconds(timeOut);
        heap_[i].expires = newExpires;
        heap_[i].cb = cb;
        if (newExpires < oldExpires) {
            SiftUp_(i);
        } else if (newExpires > oldExpires) {
            SiftDown_(i, heap_.size());
        }
    }
}
```

`Delete_` 方法用于删除指定的定时器节点，逻辑为该节点与堆的末尾进行交换，并根据需要调整堆。

```cpp
void Timer::Delete_(size_t i) {
    assert(!heap_.empty() && i >= 0 && i < heap_.size());
    size_t n = heap_.size() - 1;
    if (i < n) {
        TimerNode node = heap_[i];
        SwapNode_(i, n);
        if (heap_[i] < node) {
            SiftUp_(i);
        } else {
            SiftDown_(i, n);
        }
    }
    ref_.erase(heap_.back().id);
    heap_.pop_back();
}
```

## Thread Pool

线程池提供一个高效的方式来并发执行多个任务，特别适用于需要处理大量独立任务的网络服务器。线程池能够有效地管理线程资源，避免了频繁创建和销毁线程的开销。线程池中的工作线程是通过`std::vector<std::thread>`来管理的。这不仅使得管理多个线程变得简单，而且在关闭线程池时，可以遍历这个向量来逐个加入（`join`）各个线程，确保所有线程都正确完成任务后再结束程序。

**主要特性**

- 并发执行：允许并行处理多个任务，提高程序的执行效率。
- 资源重用：通过重用已存在的线程，减少了线程创建和销毁的开销。
- 任务队列：使用队列管理待执行的任务，确保按顺序执行。
- 优雅关闭：支持安全地关闭线程池，确保所有任务都能完成后再退出。

**std::shared_ptr**

在这个线程池实现中，使用`std::shared_ptr`来管理`Pool`结构的生命周期。`Pool`结构包含了**任务队列、互斥锁、条件变量和关闭标志**。通过使用`std::shared_ptr`，可以确保只要有线程还在运行，`Pool`的资源就不会被提前释放。

**完美转发**

在添加任务到线程池的`AddTask`方法中，使用了模板和`std::forward`来实现完美转发。这允许我们将各种不同类型的函数对象和参数以最高效的方式传递到线程池中，减少不必要的拷贝，提高性能。

**std::mutex、std::condition_variable**

互斥锁用于保护共享资源，主要是任务队列 (`tasks`)。条件变量用于线程间的同步。在`ThreadPool`中，工作线程可能会在没有任务可执行时进入等待状态。这时，它们会通过条件变量进入休眠，直到有新的任务被添加到队列中并通过条件变量被唤醒。这种机制有效地减少了`CPU`的无效使用，因为它允许线程在没有工作时不占用`CPU`资源。

在具体实现中，工作线程在尝试从任务队列中取出任务时，会使用`std::unique_lock<std::mutex>`加锁；如果发现队列为空，调用`std::condition_variable::wait`方法进入等待状态。这个等待可以被以下两种情况之一打断：

- 新任务被添加到队列中：当`AddTask`方法向队列中添加一个新任务后，它会调用`std::condition_variable::notify_one`方法，这将唤醒一个正在等待的线程（如果有的话），让它取出任务并执行。

- 线程池关闭：如果调用`Close`方法关闭线程池，会设置关闭标志(`isClosed`)并调用`std::condition_variable::notify_all`。这会唤醒所有等待的线程，这些线程会检查关闭标志，然后退出循环并结束执行。

## Sql Connect

数据库连接池用于高效地管理`MySQL`数据库连接。通过使用连接池，我们可以减少频繁打开和关闭数据库连接的开销，提高数据库操作的效率。

**主要特性**

- 连接复用：复用已经建立的数据库连接，减少连接开销。
- 线程安全：确保多线程环境下数据库连接的安全使用。
- 资源控制：限制最大连接数，避免过多的连接耗尽服务器资源。
- 自动管理：通过`RAII`包装器自动获取和释放数据库连接。

**std::mutex**

互斥锁用于保护`mysql`连接队列，确保在多线程环境下对队列的访问是安全的。在获取或释放连接时，必须先获取互斥锁，这样可以防止多个线程同时修改连接队列，从而避免数据竞争和潜在的错误。

**sem_t**

信号量用于控制最大`mysql`可用连接的数量，是一种有效的线程同步机制。**初始化时，信号量的值设置为最大连接数。每当一个线程获取一个连接时，信号量减一；每当连接被释放回池中时，信号量加一。如果所有连接都在使用中，信号量值为零，此时任何请求连接的线程都会阻塞，直到有连接被释放回池中**。

**单例模式**

使用了单例模式来确保整个程序中只存在一个数据库连接池的实例。利用`C++11`特性，通过一个静态方法`Instance()`保证了全局只有一个`SqlConnPool`实例。这个方法内部使用了一个局部静态变量来存储实例，确保线程安全并且延迟初始化（即在第一次使用时才创建实例）。

**RAII**

通过一个`RAII`包装类`SqlConnRAII`来管理数据库连接的获取和释放。这样可以确保即使在发生异常或`return early`的情况下，数据库连接也总是被正确释放。

```c++
// RAII wrapper class for automatic MySQL connection management.
class SqlConnRAII {
public:
    // Constructor acquires a connection from the pool and stores it.
    SqlConnRAII(MYSQL** sqlCoon, SqlConnPool* connPool) {
        assert(connPool);
        *sqlCoon = connPool->GetConn();
        sqlCoon_ = *sqlCoon;
        connPool_ = connPool;
    }

    // Destructor releases the connection back to the pool.
    ~SqlConnRAII() {
        if (sqlCoon_ && connPool_) {
            connPool_->FreeConn(sqlCoon_);
        }
    }
private:
    MYSQL* sqlCoon_;            // Pointer to the MySQL connection.
    SqlConnPool* connPool_;     // Pointer to the connection pool.
};

#endif //SLIM_WEB_SERVER_SQL_CONNECT_RAII_H
```

注意`RAII`中传入的参数为`MYSQL**`，因为获取的`sql`连接类型为`MYSQL*`，如果传入`MYSQL*`，接收的是一个指向`MYSQL`结构的指针的拷贝。在构造函数内部对这个拷贝进行的任何修改（例如改变它指向的地址）都不会影响到原始的指针。这意味着，即使你在`SqlConnRAII`构造函数中获取了一个新的数据库连接并将其赋给这个拷贝，原始的`MYSQL*`变量仍然是未初始化的或指向错误的地址。因此需要传入`MYSQL**`，并使用`*`解引用来更改`MYSQL*`的`sql`连接。

## Http

`HTTP`连接处理模块，负责处理`HTTP`请求和响应。支持多线程环境下的高效`HTTP`请求解析和响应生成，适用于处理静态资源请求、用户登录验证等功能。

**主要特性**

- 高效的请求解析：使用正则表达式和状态机解析`HTTP GET`和`POST`请求，包括请求行、头部字段和消息体。
- 动态响应生成：根据请求动态生成`HTTP`响应，包括状态行、响应头和响应体。
- 文件映射支持：使用内存映射技术优化文件访问速度，适用于静态文件服务。
- 连接管理：支持长连接，根据`HTTP/1.1`的`Connection: keep-alive`管理`TCP`连接。
- 并发用户统计：通过原子操作和静态变量统计并发连接数，确保数据的准确性。

**HttpConn类**

封装了`HttpRequest`类和`HttpResponse`类，负责单个`HTTP`连接的管理，包括初始化连接、读写数据、处理请求和生成响应。

**HttpRequest类**

解析客户端发来的`HTTP`请求，包括请求行、请求头和消息体。支持解析`URL`编码的`POST`数据。

**HttpResponse类**

根据`HttpRequest`的解析结果生成`HTTP`响应。支持错误处理，能够根据不同的错误码返回不同的错误页面。

### HTTP GET请求示例

一个完整的 `HTTP`请求示例包括**请求行、请求头部以及可选的请求体**。下面是一个使用 `GET`方法的 `HTTP 1.1` 请求示例，该请求可能用于从服务器获取一个 `HTML`页面，同时指定连接应保持活跃（`Keep-Alive`）。

**请求行**:

```http
GET /index.html HTTP/1.1
```

- `GET`：这是 HTTP 请求方法，用于请求访问服务器上的资源。
- `/index.html`：这是请求的资源的路径。
- `HTTP/1.1`：这指明了使用的 HTTP 版本。

**请求头部**:

```http
Host: www.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
```

- `Host: www.example.com`：这指定了请求将发送到的服务器。
- `User-Agent: Mozilla/5.0 ... Safari/537.36`：这提供了关于发出请求的客户端软件的信息，通常用于统计和兼容性处理。
- `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8`：这告诉服务器客户端能够接收哪些媒体类型。
- `Accept-Language: en-US,en;q=0.5`：这指明了客户端优先接收的语言。
- `Accept-Encoding: gzip, deflate`：这告诉服务器客户端支持哪些压缩格式。
- `Connection: keep-alive`：这指示服务器保持连接打开，以便客户端可以通过同一连接发送进一步的请求。

**请求体**:

```http
(此处没有请求体，因为GET请求通常不包括请求体)
```

这种类型的HTTP请求非常常见，特别是在浏览网页时。浏览器会发送类似的请求来获取网页内容，并通过 `Connection: keep-alive` 头部指示服务器保持连接，这样浏览器就可以快速连续请求网页上的其他资源，如图片、`CSS`文件和`JavaScript`文件，无需每次都重新建立连接。

### HTTP POST请求示例

下面是一个使用`POST`方法的`HTTP 1.1`请求示例。`POST`请求通常用于向服务器提交数据，如表单数据、文件上传等。在这个示例中，我们将模拟一个用户通过表单提交用户名和密码的情况。

**请求行**:
```http
POST /login HTTP/1.1
```

- `POST`：这是HTTP请求方法，用于向服务器提交数据。
- `/login`：这是数据提交到的服务器上的资源路径。
- `HTTP/1.1`：这指明了使用的HTTP版本。

**请求头部**:
```http
Host: www.example.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 27
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/58.0.3029.110 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Connection: keep-alive
```

- `Host: www.example.com`：这指定了请求将发送到的服务器。
- `Content-Type: application/x-www-form-urlencoded`：这指明了发送的数据类型，表示表单数据被编码为键值对，如同查询字符串。
- `Content-Length: 27`：这指明了请求体的长度，必须正确指定以便服务器正确接收全部数据。
- `User-Agent: Mozilla/5.0 ... Safari/537.36`：这提供了关于发出请求的客户端软件的信息。
- `Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8`：这告诉服务器客户端能够接收哪些媒体类型。
- `Accept-Language: en-US,en;q=0.5`：这指明了客户端优先接收的语言。
- `Accept-Encoding: gzip, deflate`：这告诉服务器客户端支持哪些压缩格式。
- `Connection: keep-alive`：这指示服务器保持连接打开，以便客户端可以通过同一连接发送进一步的请求。

**请求体**:
```http
username=johndoe&password=12345
```

- `username=johndoe&password=12345`：这是实际的数据部分，包含了表单中填写的用户名和密码。数据以键值对形式发送，每对键值用 `&` 符号分隔。

### HTTP响应示例

`HTTP`响应是服务器在接收到客户端的HTTP请求后返回的数据。一个`HTTP`响应包括状态行、响应头部和响应体。下面是一个典型的`HTTP`响应示例：假设客户端请求一个网页，服务器的响应可能如下：

```http
HTTP/1.1 200 OK
Date: Thu, 11 May 2024 12:00:00 GMT
Server: Apache/2.4.1 (Unix)
Last-Modified: Wed, 10 May 2024 23:11:55 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 1234
Connection: close

<html>
<head>
    <title>An Example Page</title>
</head>
<body>
    <h1>Hello, World!</h1>
    <p>This is an example of a simple HTML page with one paragraph.</p>
</body>
</html>
```

1. **状态行**：
   - `HTTP/1.1 200 OK`：这一行表明使用的`HTTP`版本为`1.1`，状态码为`200`，表示请求成功处理，"`OK`" 是状态消息。

2. **响应头部**：
   - `Date: Thu, 11 May 2024 12:00:00 GMT`：表示响应生成的日期和时间。
   - `Server: Apache/2.4.1 (Unix)`：描述了服务器的软件信息。
   - `Last-Modified: Wed, 10 May 2024 23:11:55 GMT`：页面的最后修改时间。
   - `Content-Type: text/html; charset=UTF-8`：响应内容的类型和字符编码。
   - `Content-Length: 1234`：响应体的长度，单位是字节。
   - `Connection: close`：指示完成本次响应后关闭连接。

3. **响应体**：
   - 包含具体的内容，这里是一个简单的`HTML`页面。它展示了一个标题和一个段落。

## Pressure Test

性能测试主要使用[Webbench 1.5](https://github.com/EZLippi/WebBench)，`Webbench`是一个在`linux`下使用的非常简单的网站压测工具。它使用`fork()`模拟多个客户端同时访问我们设定的`URL`，测试网站在压力下工作的性能，最多可以模拟`3`万个并发连接去测试网站的负载能力。

**测试环境**

<img src="https://cdn.jsdelivr.net/gh/peng-yq/Gallery/202406051255703.png"/>

**测试结果**

<img src="https://camo.githubusercontent.com/2e3d5a9e8a7c962360b52d2543f214961d8645fbb42e1862dd01d874f13e627e/68747470733a2f2f63646e2e6a7364656c6976722e6e65742f67682f70656e672d79712f47616c6c6572792f3230323430363037313033383236382e706e67">

由于`Webbench`测试方法比较简单，因此测出来的数据只能说见仁见智，我后面又用`apache-jmeter 5.6.3`进行了测试，测出来的数据肯定是没有这么好看的，但也算能在高压下稳定，“高性能”的运行。当然还是那句话，脱离业务、机器和请求量谈高性能都是耍流氓。