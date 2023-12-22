---
layout: post
title: Concurrency and Parallelism
subtitle: No Subtitle ：>
author: "PYQ"
header-mask: 0.3
mathjax: true
catalog: true
tags:
  - OS
---

## Concurrency and Parallelism

- [Talk：Rob Pike：Concurrency and Parallelism ](https://www.bilibili.com/video/BV1EN411o7FY/)
- [Slides：Rob Pike：Concurrency and Parallelism](https://go.dev/talks/2012/waza.slide)

### Concurrency vs. parallelism

- Concurrency is about **dealing with lots of things at once**.
- Parallelism is about **doing lots of things at once**.
- Not the same, but **related**.
- Concurrency is about **structure**, parallelism is about **execution**.
- **Concurrency** provides a way to structure a solution to **solve a problem that may (but not necessarily) be parallelizable**.

<img src="https://1484576603-files.gitbook.io/~/files/v0/b/gitbook-legacy-files/o/assets%2F-MV9vJFv4kmvRLgEog6g%2F-Mb_6O3J2tKkCGRwXx-i%2F-Mb_8rPEKSwKYp6bHv-6%2Fimage_%E5%89%AF%E6%9C%AC.png?alt=media&token=72531224-2848-440d-bc49-901f13d95b0e">

### Concurrency plus communication

- Concurrency is a way to **structure a program by breaking it into pieces that can be executed independently**.
- Communication is the means to **coordinate the independent executions**.

### Conclusion

- Concurrency is powerful.
- Concurrency is **not parallelism**.
- Concurrency **enables parallelism**.
- Concurrency **makes parallelism (and scaling and everything else) easy**.