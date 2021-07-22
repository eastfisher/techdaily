+++
title = "Dingo入坑指南"
description = ""

[taxonomies]
tags = ["Golang", "Dingo", "Dependency Injection"]
categories = ["Blog"]
+++

最近转组去了业务开发团队, 公司主语言采用Go, 且服务治理体系, 各种SDK等基础设施基本上只提供了Go语言的版本. 个人认为, 实现大型复杂项目的开发, 依赖注入 (Dependency Injection, DI) 框架还是非常有必要的. 对于Java语言来说, 使用Spring全家桶进行业务开发, 自然就拥有了Spring框架提供的IoC能力. 然而Go在这方面的差距还是比较明显的, 使用Go语言进行业务开发的案例本身就比较少, DI的最佳实践就更少了. 而且除了Uber的 [Dig][1], Google的 [Wire][2], 鲜有其他成熟的开源DI框架可用. 不过就在最近, 我发现了一个宝藏项目, 它就是[Dingo][3].

<!-- more -->

## 简介

Dingo框架的关注度不高, 只有100+ star. 在github搜索dingo, 需要翻到第2页的末尾才能找到这个项目. Dingo框架归属于一个叫作`i-love-flamingo`的组织, 该组织提供了一个开源的Web开发框架, 名叫Flamingo (关注度同样不高), 并基于该框架实现了一个简版电商平台demo. 而Dingo就是Flamingo的DI容器实现, 有点类似于Spring MVC和Spring的关系.

Dingo是基于反射机制实现的DI框架, 可通过 struct tag 和 Inject()回调方法 两种方式声明依赖注入.

Dingo代码量很少, 逻辑也和Google的 [Guice][4] (发音同juice) 非常相像, 有Java DI框架使用经验的话应该会很容易上手.

## 基本用法

了解3个概念即可使用Dingo: Dependency, Module, Injector. Dependency即为依赖, Module用来声明依赖的绑定关系, Injector为Dingo提供的DI容器. 给个例子:

```go
package main

import (
    "context"
    "log"

    "flamingo.me/dingo"
)

type User struct{
    Id int64
}

type IUserRepo interface {
    FindById(context.Context, int64) (*User, error)
}

// IUserRepo的内存型实现
type InMemoryUserRepo struct {
}

func (ur *InMemoryUserRepo) FindById(ctx context.Context, userId int64) (*User, error) {
    return &User{Id: userId}, nil
}

// 用户服务, 需要注入IUserRepo接口
type UserService struct {
    UserRepo IUserRepo `inject:""`
}

func (us *UserService) CheckUserExist(ctx context.Context, userId int64) (bool, error) {
    user, err := us.UserRepo.FindById(ctx, userId)
    if err != nil {return false, err}
    return user != nil, nil
}

type Module struct {
}

// 声明依赖的绑定关系
func (m *Module) Configure(injector *dingo.Injector) {
    injector.Bind(new(IUserRepo)).To(InMemoryUserRepo{})
}

func main() {
    injector, _ := dingo.NewInjector(
        new(Module),
    )

    userServiceObj, _ := injector.GetInstance((*UserService)(nil))
    log.Println(userServiceObj.(*UserService).CheckUserExist(context.Background(), 1))
}
```

上面的代码创建了一个Module, 里面声明了一个IUserRepo到InMemoryUserRepo的绑定关系. 在main中, 我们创建了Dingo的Injector容器, 并获取了一个UserService对象, 由于通过struct tag声明了UserRepo的注入, 因此容器会自动帮我们注入一个IUserRepo实例, 也就是在Module中声明的InMemoryUserRepo对象. 该对象是由Injector自动创建的. 请求注入时, 注入的对象由 [这个规则][5] 进行控制.

此外, Injector API还提供了诸如具名绑定, Eager绑定, Provider, 父子容器等功能, 看一遍文档就差不多能明白了.

Dingo还提供了拦截器 (Interception) 的功能, 但是与AOP完全无关, 就是个基于interface的装饰器, 除了某些特定场景以外 (比如web框架的HTTP middleware), 基本没啥用, 和Spring AOP差了十万八千里.

## 常见的坑

一开始使用dingo时经常会遇到依赖绑定失败发生panic的问题. 在此给出一些常见的case.

### 类型声明问题

Dingo在做类型Binding, 以及GetInstance操作时, 都需要声明对象类型. 这就相当于Spring反射获取对象时传入class对象. 待绑定的对象是接口还是非接口, 有一些区别. 对于接口的绑定, 在声明时需要使用`new(接口)`从而得到接口类型指针, dingo才能在反射时获取接口类型. 如果声明成`(接口)(nil)`, 得到的是nil指针, dingo反射时无法获取接口类型, 会报panic.

```go
func (m *Module) Configure(injector *dingo.Injector) {
    injector.Bind(new(IUserRepo)).To(InMemoryUserRepo{})    // OK
    injector.Bind((IUserRepo)(nil)).To(InMemoryUserRepo{})  // Panic, 接口类型的nil指针, 其类型为nil
    injector.Bind((*InMemoryUserRepo)(nil)).ToInstance(&InMemoryUserRepo{}) // OK
}

func main() {
    ......
    var repoObj interface{}
    repoObj, _ = injector.GetInstance(new(IUserRepo))   // OK
    repoObj, _ = injector.GetInstance((IUserRepo)(nil)) // Panic, nil类型
    repoObj, _ = injector.GetInstance(new(InMemoryUserRepo))    // OK
    repoObj, _ = injector.GetInstance((*InMemoryUserRepo)(nil)) // OK
}
```

### 单例问题

有3种方式可以实现单例:

- 1 使用 ToInstance() 绑定到某个对象
- 2 使用 ToProvider() 绑定并提供单例 Provider
- 3 使用 In(dingo.Singleton) 将绑定声明为单例
  - 可使用 AsEagerSingleton() 对单例进行及时加载

其中, 第1种方式与及时加载的第3种方式非常类似. 第2种方式与延迟加载的第3种方式非常类似, 区别在于需要自己控制加锁.

第1, 2种方式可以自己控制单例对象的实例化, 而第3种方式只能使用new()进行对象初始化, 如果其中声明了依赖, 会自动注入. 对于那些需要控制对象创建过程的对象 (例如, 数据库连接池对象), 只能采用第1, 2种方式.

### Bean可见性

Dingo提供了父子容器的功能, 父容器中的Bean对子容器可见, 反之不成立. 这与Spring MVC和Spring的父子容器是一样的: Spring父容器中的Component, Service对MVC容器的Controller可见, 而MVC容器的Controller对Service不可见.

### 依赖注入正确性

对struct中的接口属性字段, 可使用无名inject进行注入. 原因: 必须声明interface到某个具体类型的binding后, dingo才能处理.

对struct中的非接口属性字段, 建议使用具名inject进行注入. 原因: 对非接口对象, dingo如果找不到任何提供该对象的工厂, 则会使用反射创建空值对象. 这有可能导致该对象并没有正确创建. 而对于具名inject注入, 如果不显式声明binding, 则会在依赖注入过程中由于找不到对应annotation的bean而报错, 从而对注入的正确性进行约束.

## 总结一下

当时在发现Dingo时我是非常惊喜的, 它恰到好处地解决了我在使用Go编写复杂业务代码时的依赖注入问题. Dingo提供的API非常符合Java转Go开发者的使用习惯. 而上面提到的坑, 与其说是踩坑, 不如说是对Go的反射机制理解不足 (以及Go的反射确实比较烂? 逃...). 我们已经在生产环境中使用Dingo进行微服务开发, 体验良好.

## 参考资料

- [Dingo代码仓库](https://github.com/i-love-flamingo/dingo)
- [我的Dingo源码阅读笔记](https://github.com/eastfisher/dingo/tree/reading)

[1]: https://github.com/uber-go/dig
[2]: https://github.com/google/wire
[3]: https://github.com/i-love-flamingo/dingo
[4]: https://github.com/google/guice
[5]: https://github.com/eastfisher/dingo/tree/reading#requesting-injection