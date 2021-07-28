+++
title = "Go语言的小细节"
description = ""
draft = true

[taxonomies]
tags = ["Golang"]
categories = ["Blog"]
+++

Go语言相对比较简单, 但其中的细节也不少. 本文持续更新, 盘点一下Go语言中的那些小细节, 希望看到这篇文章的你能少踩些坑. ^_^

<!-- more -->

## 标准库

### math/rand并发安全吗?

A: rand包下的函数是并发安全的, 但Rand结构的方法不是并发安全的.

解释: rand包下面的Rand结构是随机数生成器, 构造时需要传入一个随机源Source. 以`Rand.Int63()`方法为例:

```go
func (r *Rand) Int63() int64 { return r.src.Int63() }
```

其中`r.src`的实现为`rngSource`结构, 该结构的`Int63()`方法实现为:

```go
// Int63 returns a non-negative pseudo-random 63-bit integer as an int64.
func (rng *rngSource) Int63() int64 {
    return int64(rng.Uint64() & rngMask)
}

func (rng *rngSource) Uint64() uint64 {
    rng.tap--
    if rng.tap < 0 {
        rng.tap += rngLen
    }

    rng.feed--
    if rng.feed < 0 {
        rng.feed += rngLen
    }

    x := rng.vec[rng.feed] + rng.vec[rng.tap]
    rng.vec[rng.feed] = x
    return uint64(x)
}
```

很明显, `Uint64()`不是并发安全的, 涉及到内部数据的多次读写操作. 当多个goroutine并发调用Rand的获取随机数相关方法时, 会产生数据竞争, 导致返回的数据出现问题. 例如`Int63()`正常应该返回一个非负的63位随机整数, 但在并发获取时有可能出现返回负数.

标准库中的相关函数, 是在访问全局rand对象时加锁了. **个人觉得这样设计API比较坑, 缺乏一致性** (提供的函数是并发安全的, 而其结构本身又不是并发安全的).
