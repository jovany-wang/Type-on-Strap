---
layout: post
title:  "使用asio实现RepeatedTimer定时器"
date:   2020-09-02
---

<p class="intro">使用过boost::asio的同学都知道,asio中的steady_timer是一个较为简陋的组件，其可以提供一个异步等待超时的机制，并且其异步等待是一次性的。这就意味着你想要一个和闹钟一样的定时器，每隔固定时间就滴答一次是需要做不少额外的工作。这篇文章带大家使用boost::asio中的steady_timer实现一个RepeatedTimer。</p>

## 1. boost::asio中的steady_timer
如果我们想做一次超时的定时器，使用steady_timer写如下代码即可：
```c++
asio::io_service io_service;
asio::steady_timer {io_service};
steady_timer.expires_from_now(std::chrono::milliseconds(5 * 1000));
steady_timer.async_wait([](const asio::error_code &e) {
    if (e.value() == 0) {
        std::cout << "Time is up!" << std::endl;
    }
});
io_service.run();
```
上述代码将输出:
```shell
qingw > Time is up! 
```
可以看出其在5s之后将输出一行文字，随后程序结束。
上述代码问题比较明显：  
(1) 无法持续不断的进行超时回调，就是每个5s都调一次回调。  
(2) 无法进行`续租约式`的对timer进行续租。就是说在5s超时时间还没到的时候按下`reset()`按钮，timer就重新开始计时。  

为了解决这个问题，提供一个便利高效易用的timer给用户，我们可以对其进行如下封装，实现一个`RepeatedTimer`。

## 2. 实现一个现代化的RepeatedTimer
首先说一下我们对这个类期望的使用方式应该是这样的。
```c++
asio::io_service io_service;
asio::io_service::work work {io_service}; // work是为了io_service在没有pending task的时候不退出
std::thread th { io_service.run(); };

RepeatedTimer timer(io_service, [](const asio::error_code &e) { std::cout << "Time is up!"; });
timer.Start(/*ms=*/1000);
timer.Stop();
timer.Reset(/*ms=*/2000);
```


### 2.1 一把梭实现
根据前面总结的几点需求，我们可以直观地进行如下实现。
```c++
class RepeatedTimer final {
public:
    RepeatedTimer(asio::io_service &io_service,
                  std::function<void(const asio::error_code &e)> timeout_handler)
        : io_service_(io_service), timer_(io_service_),
          timeout_handler_(std::move(timeout_handler)) {}

    void Start(const uint64_t timeout_ms) { Reset(timeout_ms); }

    void Stop() { is_running_ = false; }

    void Reset(const uint64_t timeout_ms) {
        is_running_ = true;
        DoSetExpired(timeout_ms);

    }

private:
    void DoSetExpired(const uint64_t timeout_ms) {
        if (!is_running_) { return; }

        timer_.expires_from_now(std::chrono::milliseconds(timeout_ms));
        timer_.async_wait([this, timeout_ms](const asio::error_code &e) {
            if (e.value() == asio::error::operation_aborted || !is_running_) { return; }            
            timeout_handler_(e);
            this->DoSetExpired(timeout_ms);
        });
    }

private:
    // The io service that runs this timer.
    asio::io_service &io_service_;
    // The actual boost timer.
    asio::steady_timer timer_;
    
    bool is_running_ = false;
    // The handler that will be triggered once the time's up.
    std::function<void(const asio::error_code &e)> timeout_handler_;
};
```
测试代码：
```c++
// 准备工作
asio::io_service io_service;
asio::io_service::work work {io_service};
io_service.run();
std::thread th{ io_service.run(); };

// 使用RepeatedTimer
RepeatedTimer timer(io_service, []() { std::cout << "Time is up!" << std::endl; });
timer.Start(/*ms=*/1000);
std::this_thread::sleep_for(std::chrono::millisecond(3 * 1000));
timer.Stop();
timer.Reset(/*ms=*/2000);
```
上述测试代码将会先每隔1s输出一个`Time is up!`，一共输出3个(也可能是2个， 看具体时间消耗)，以后每隔2s输出一句。
```shell
/// 每隔1s输出一句
qingw >  Time is up!
qingw >  Time is up!
qingw >  Time is up!
/// 后面的都是每隔2s输出一行
qingw >  Time is up!
qingw >  Time is up!
...
```

### 2.2 线程安全
上述实现的RepeatedTimer并不是一个线程安全的，因为我们自己定义了一些状态来做判断，因此，要想实现一个线程安全的，我们还需要做一些工作。
在这里我选择使用读写锁来作为多线程状态的保护。
```c++
class RepeatedTimer final {
public:
    RepeatedTimer(asio::io_service &io_service,
                  std::function<void(const asio::error_code &e)> timeout_handler)
        : io_service_(io_service), timer_(io_service_), 
        timeout_handler_(std::move(timeout_handler)) {}

    void Start(const uint64_t timeout_ms) { Reset(timeout_ms); }

    void Stop() {
        std::unique_lock<std::shared_mutex> guard {shared_mutex_};
        is_running_ = false;
    }

    void Reset(const uint64_t timeout_ms) {
        {
            std::unique_lock<std::shared_mutex> guard {shared_mutex_};
            is_running_ = true;
        }
        DoSetExpired(timeout_ms);
    }

private:
    void DoSetExpired(const uint64_t timeout_ms) {
        {
            std::shared_lock<std::shared_mutex> guard {shared_mutex_};
            if (!is_running_) { return; }
        }
        timer_.expires_from_now(std::chrono::milliseconds(timeout_ms));
        timer_.async_wait([this, timeout_ms](const asio::error_code &e) {
            if (e.value() == asio::error::operation_aborted) {
                return;
            }
            {
                std::shared_lock<std::shared_mutex> guard {shared_mutex_};
                if (!is_running_) {
                    return;
                }            
                timeout_handler_(e);
            }
            // 这一步放到锁范围之外，避免递归上锁
            this->DoSetExpired(timeout_ms);
        });
    }

private:
    // The io service that runs this timer.
    asio::io_service &io_service_;
    // The actual boost timer.
    asio::steady_timer timer_;
    std::shared_mutex shared_mutex_;
    bool is_running_ = false;
    // The handler that will be triggered once the time's up.
    std::function<void(const asio::error_code &e)> timeout_handler_;
};
```
上述改进的实现版本能够确保在多线程下使用RepeatedTimer是安全的，因为我们将状态使用读写锁保护了起来。测试代码就不继续写了，也很简单，大家可以自己写一下。 

### 2.3 死锁分析
上述实现虽然是线程安全的，而且我们一眼看过去，是没有死锁的可能的，因为使用的是一个读写锁，并且没有递归上锁。事实上这样的想法是正确的，只是我们在实现一个通用组件的时候，不能仅仅考虑组件本身，还应该考虑用户使用过程中的心智负担以及使用安全性问题。怎么理解这句话呢？其实很简单，说白了就是用户不管怎么用你这个组件，都不应该出现问题！  

考虑用户在多线程环境中使用RepeatedTimer，他很容易地就会写出如下代码：
```c++
class MyClass {
public:
    explicit MyClass(asio::io_service &io_service)
            : timer_(io_service, [this](){ TimeoutHandler(); }) {
        timer_.Start();
    }

    void TimeoutHandler() {
        std::lock_guard<std::mutex> guard {mutex_};
        state_.ChangeSth();
    }

    void F() {
        std::lock_guard<std::mutex> guard {mutex_};
        state_.ChangeSth();
        timer_.Reset(1000);
    }

private:
    RepeatedTimer timer_;
    // The mutex that protects state_.
    std::mutex mutex_;
    MyState state_;
};
```
我们简单的测试这段代码之后，会发现这段代码会有概率发生hang住的问题，使用pstack或者其他debug工具可以发现其发生死锁。
![死锁流程]({{"assets/img/for_posts/20190104/dead_lock_anal.jpeg" | relative_url }})
线程A, B分别按照时间顺序执行到第3, 第6步之后，紧接着A，B线程分别acquire shared_mutex_和mutex_，但这两者都被对方线程锁住，因此造成死锁。  

解决方法也很简单，就是不要在`DoSetExpired()`中shared_mutex_的锁范围内调用`timeout_handler_()`，shared_mutex_只作用于更改timer的状态即可。改进后的代码如下，只需要简单更改`DoSetExpired()`方法中的锁范围即可。
```c++
class RepeatedTimer final {
public:
    /// 其他方法没有改变

private:
    void DoSetExpired(const uint64_t timeout_ms) {
        {
            std::shared_lock<std::shared_mutex> guard {shared_mutex_};
            if (!is_running_) { return; }
        }
        timer_.expires_from_now(std::chrono::milliseconds(timeout_ms));
        timer_.async_wait([this, timeout_ms](const asio::error_code &e) {
            if (e.value() == asio::error::operation_aborted) { return; }
            {
                // 注意，这里的锁范围缩小，只作用于更改自身的状态，timeout_handler_()在锁范围之外
                std::shared_lock<std::shared_mutex> guard {shared_mutex_};
                if (!is_running_) { return; }
            }
            timeout_handler_(e);
            this->DoSetExpired(timeout_ms);
        });
    }
};
```

OK, 上述的实现就可以在多线程下安全地使用RepeatedTimer了，而且不论用户代码怎么写都不会和timer发生死锁。测试的代码就不继续写了，也很简单，大家可以自己写一下。其中用到的读写锁其实必要性不是非常大，因为读写锁还是针对于多读的场景，然而在RepeatedTimer中，只有`DoSetExpired`中才会是读状态，而`DoSetExpired`里读状态的地方，只有在极少数的情况下才会多读，比如线程A调用`Reset()`刚好执行到`DoSetExpired`最开始读状态的地方，线程B刚好是io_service进行超时回调刚到进入`DoSetExpired`中，这个时候才恰好是多读。所以这里到底是否需要使用读写锁，在没有做一些很深刻的性能测试之前，其实还只能是根据经验判断。但不管怎么样，都没什么大问题。

### 2.4 原子变量实现
用锁实现虽然能有效的解决多线程安全问题，但是在上述思考及实现过程中，还是有很多地方需要大家非常小心，才能避免掉进坑里。上述实现还有一个特点，就是我们其实只需要在多线程的情况下小心的保护`is_running_`这一个变量，而不是很多复杂的状态，因此，原子变量就是为这种场景而生的。

使用原子变量，`RepeatedTimer`的实现变得异常简单。
```c++
class RepeatedTimer final {
public:
    RepeatedTimer(asio::io_service &io_service,
                  std::function<void(const asio::error_code &e)> timeout_handler)
        : io_service_(io_service), timer_(io_service_),
          timeout_handler_(std::move(timeout_handler)) {}

    void Start(const uint64_t timeout_ms) { Reset(timeout_ms); }

    void Stop() { is_running_.store(false); }

    void Reset(const uint64_t timeout_ms) {
        is_running_.store(true);
        DoSetExpired(timeout_ms);

    }

private:
    void DoSetExpired(const uint64_t timeout_ms) {
        if (!is_running_.load()) { return; }

        timer_.expires_from_now(std::chrono::milliseconds(timeout_ms));
        timer_.async_wait([this, timeout_ms](const asio::error_code &e) {
            if (e.value() == asio::error::operation_aborted || !is_running_.load()) { return; }            
            timeout_handler_(e);
            this->DoSetExpired(timeout_ms);
        });
    }

private:
    // The io service that runs this timer.
    asio::io_service &io_service_;
    // The actual boost timer.
    asio::steady_timer timer_;
    
    std::atomic<bool> is_running_ = { false };
    // The handler that will be triggered once the time's up.
    std::function<void(const asio::error_code &e)> timeout_handler_;
};
```
搞定，上述的原子实现也非常简单，和最开始我们实现的版本并无逻辑区别，它保证了多线程场景下使用的安全性，并且性能还比加锁版本要好很多。

## 3. 总结与思考
(1) `RepeatedTimer`这样的类是一个非常有用且用途非常广泛的类。例如在常见的一些RAFT实现里，需要用到RepeatedTimer作为其ElectionTimer, VoteTimer及HeartbeatTimer等。这些tiemr都需要`Reset()`的续租能力，还有很多分布式系统中的一些心跳时间，续租时间等都可以使用这样的类。  

(2) 这个类是一定需要考虑线程安全的问题。原因在于其基于asio的staedy_timer，这里的回调都是在io_service中完成，其大概率是在io_service的work pool中进行回调，然后用户的线程是一定会大量对其进行操作，因而没有什么单线程的使用场景。  

(3) 使用锁的时候需要考虑的不仅仅是一个组件本身的死锁可能性，还需要尽可能避免同业务代码结合的时候的死锁问题。

(4) 原子变量使用的场景一般是对单状态的保护。 而锁的使用场景则是对很多复杂的状态的保护。因为这个时候我们即使对每个状态都使用原子变量，我们还需要考虑这些状态之间的协调性，这意味着很多状态之间是有状态互斥或者状态协同的。

