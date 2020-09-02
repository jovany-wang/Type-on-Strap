---
layout: post
title:  "使用asio实现一个RepeatedTimer定时器"
date:   2020-09-02
---

<p class="intro">使用过boost::asio的同学都知道,boost中的steady_timer是一个较为简陋的组件，其可以提供一个异步等待超时的机制，并且其异步等待是一次性的。这就意味着你想要一个和闹钟一样的定时器，每隔固定时间就滴答一次是需要做不少额外工作。这篇文章带大家使用boost::asio中的定时器实现一个RepeatedTimer。</p>

## 1. boost::asio中的steady_timer
如果我们想做一次超时的定时器，使用steady_timer写如下代码即可：
```c++
asio::io_service io_service;
asio::steady_timer {io_service};
steady_timer.expires_from_now(std::chrono::milliseconds(5 * 1000));
steady_timer.async_wait([](const asio::error_code &e) {
    if (e.value() == 0) {
        std::cout << "Time's up!" << std::endl;
    }
});
io_service.run();
```
上述代码将输出:
```shell
qingw > Time's up! 
qingw >
```
可以看出其在5s之后将输出一行文字，随后程序结束。
上述代码问题比较明显：  
(1) 无法持续不断的进行超时回调，就是每个5s都调一次回调。
(2) 无法进行`续租约式`的对timer进行续租。就是说在5s超时时间还没到的时候按下`reset()`按钮，timer从新开始计时。

为了解决这个问题，提供一个便利高效易用的timer，我们可以对其进行如下封装，实现一个`RepeatedTimer`。

## 2. 实现一个现代化的RepeatedTimer
首先说一下我们对这个类期望的使用方式应该是这样的。
```c++
asio::io_service io_service;
asio::io_service::work work {io_service}; // work是为了io_service在没有pending task的时候不退出
std::thread th{ io_service.run(); };
RepeatedTimer timer(io_service, []() { std::cout << "Time's up!"; });
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
        : io_service_(io_service),
          timer_(io_service_),
          timeout_handler_(std::move(timeout_handler)) {}

    void Start(const uint64_t timeout_ms) {
        Reset(timeout_ms); 
    }

    void Stop() {
        is_running_ = false;
    }

    void Reset(const uint64_t timeout_ms) {
        is_running_ = true;
        DoSetExpired(timeout_ms);

    }

private:
    void DoSetExpired(const uint64_t timeout_ms) {
        if (!is_running_) {
            return;
        }

        timer_.expires_from_now(std::chrono::milliseconds(timeout_ms));
        timer_.async_wait([this, timeout_ms](const asio::error_code &e) {
            if (e.value() == asio::error::operation_aborted || !is_running_) {
                // The timer was canceled or the timer is stopping.
                return;
            }            
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
RepeatedTimer timer(io_service, []() { std::cout << "Time's up!" << std::endl; });
timer.Start(/*ms=*/1000);
std::this_thread::sleep_for(std::chrono::millisecond(3 * 1000));
timer.Stop();
timer.Reset(/*ms=*/2000);
```
上述测试代码将会先每个1s输出一个`Time's up!`，一共输出3个(也可能是2个， 看具体时间消耗)，以后每隔2s输出一句。
```shell
qingw >  Time's up!
qingw >  Time's up!
qingw >  Time's up!
/// 后面的都是每隔2s输出一行
qingw >  Time's up!
qingw >  Time's up!
...
```

### 2.2 线程安全
上述实现的RepeatedTimer并不是一个线程安全的，因为我们自己定义了一些状态来做判断，因此，要想实现一个线程安全的，我们还需要做一些工作。
在这里我选择采用读写锁来作为多线程状态的保护。
```c++
class RepeatedTimer final {
public:
    RepeatedTimer(asio::io_service &io_service,
                  std::function<void(const asio::error_code &e)> timeout_handler)
        : io_service_(io_service),
          timer_(io_service_),
          timeout_handler_(std::move(timeout_handler)) {}

    void Start(const uint64_t timeout_ms) {
        Reset(timeout_ms); 
    }

    void Stop() {
        std::lock_guard<std::shared_mutex> guard{shared_mutex_};
        is_running_ = false;
    }

    void Reset(const uint64_t timeout_ms) {
        {
            std::lock_guard<std::shared_mutex> guard{shared_mutex_};
            is_running_ = true;
        }
        DoSetExpired(timeout_ms);
    }

private:
    void DoSetExpired(const uint64_t timeout_ms) {
        std::shared_lock<std::shared_mutex> guard{shared_mutex_};
        if (!is_running_) {
            return;
        }

        timer_.expires_from_now(std::chrono::milliseconds(timeout_ms));
        timer_.async_wait([this, timeout_ms](const asio::error_code &e) {
            if (e.value() == asio::error::operation_aborted) {
                // The timer was canceled.
                return;
            }
            {
                std::shared_lock<std::shared_mutex> guard{shared_mutex_};
                if (!is_running_) {
                    // The timer is stopping, should we should trigger any handler.
                    return;
                }
            }
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
OK, 上述的实现就可以在多线程下安全的使用RepeatedTimer了，测试的代码就不继续写了，也很简单，大家可以自己写一下。其中用到的读写锁其实必要性不大，因为读写锁还是针对于多读的场景，然而在RepeatedTimer中，只有`DoSetExpired`中才会是读状态，而`DoSetExpired`里读状态的地方，只有在极少数的情况下才会多读，比如线程A调用`Reset()`刚好执行到`DoSetExpired`最开始读状态的地方，线程B刚好是io_service进行超时回调刚到进入`DoSetExpired`中，这个时候才恰好是多读。

## 3. 总结
1. `RepeatedTimer`这样的类是一个非常有用且用途非常广泛的类。例如在常见的一些RAFT实现里，需要用到RepeatedTimer作为其ElectionTimer, VoteTimer及HeartbeatTimer等。这些tiemr都需要`Reset()`的续租能力。
2. 这个类是一定需要考虑线程安全的问题。原因在于其基于asio的staedy_timer，这里的回调都是在io_service中完成，其大概率是在io_service的work pool中进行回调，然后用户的线程是一定会大量对其进行操作，因而没有什么单线程的使用场景。
3. 在这个实现中，`DoSetExpired()`方法还是进行了一次状态`is_running_`的判断，这一步是不能省略的。因为`DoSetExpired()`不仅是用户`Start()`和`Reset()`中直接调用，还是反复滴答的一个实现，因此在下轮滴答的时候，有可能用户已经`Stop()`过了，因此没必要进行后续操作。

