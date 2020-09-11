---
layout: post
title:  "Modern C++之constexpr"
date:   2020-01-02
---

<p class="intro">使用过boost::asio的同学都知道,asio中的steady_timer是一个较为简陋的组件，其可以提供一个异步等待超时的机制，并且其异步等待是一次性的。这就意味着你想要一个和闹钟一样的定时器，每隔固定时间就滴答一次是需要做不少额外的工作。这篇文章带大家使用boost::asio中的steady_timer实现一个RepeatedTimer。</p>

## 1. boost::asio中的steady_timer
