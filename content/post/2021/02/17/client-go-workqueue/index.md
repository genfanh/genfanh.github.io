---
title: "client-go WorkQueue工作队列简介"
description: "对client-go WorkQueue学习总结"
date: 2021-02-17T13:38:57+08:00
draft: false
tags: 
  - k8s
  - client-go
categories: 
  - k8s
slug: "client-go-WorkQueue"
---

## 1. 概述
WrokQueue是k8s的工作队列，与普通FIFO队列相比，实现起来更加复杂，主要特性如下：
- **有序**：按照添加顺序处理元素（item）。
- **去重**：相同元素在同一时间不会被重复处理，例如一个元素在处理之前被添加了多次，它只会被处理一次。
- **并发性**：多生产者和多消费者。
- **标记机制**：支持标记功能，标记一个元素是否被处理，也允许元素在处理时重新排队。
- **通知机制**：ShutDown方法通过信号量通知队列不再接收新的元素，并通知metric goroutine退出。
- **延迟**：支持延迟队列，延迟一段时间后再将元素存入队列。
- **限速**：支持限速队列，元素存入队列时进行速率限制。限制一个元素被重新排队（Reenqueued）的次数。
- **Metric**：支持metric监控指标，可用于Prometheus监控。
client-go中WorkQueue支持3种队列同时提供了3种接口，每种队列对应特定的应用场景，具体如下：
- Interface：FIFO队列接口，先进先出队列，并支持去重机制。
- DelayingInterface：延迟队列接口，基于Interface接口封装，延迟一段时间后再将元素存入队列。
- RateLimitingInterface：限速队列接口，基于DelayingInterface接口封装，支持元素存入队列时进行速率限制。
## 2. 队列介绍
### 2.1. FIFO队列
FIFO队列支持最基本的队列方法，例如插入元素、获取元素、获取队列长度等。另外，WorkQueue中的限速及延迟队列都基于Interface接口实现，其提供如下方法：

**代码路径：vendor/k8s.io/client-go/util/workqueue/queue.go**
```go
type Interface interface {
	Add(item interface{})
	Len() int
	Get() (item interface{}, shutdown bool)
	Done(item interface{})
	ShutDown()
	ShuttingDown() bool
}
```
FIFO队列接口方法说明：
- **Add**：给队列添加元素（item），可以是任意类型元素。
- **Len**：返回当前队列的长度。
- **Get**：获取队列头部的一个元素。
- **Done**：标记队列中该元素已被处理。
- **ShutDown**：关闭队列。
- **ShuttingDown**：查询队列是否正在关闭。
FIFO队列数据结构如下：
```go
type Type struct {
	queue []t
	dirty set
	processing set
	cond *sync.Cond
	shuttingDown bool
	metrics queueMetrics
	unfinishedWorkUpdatePeriod time.Duration
	clock                      clock.Clock
}
```
FIFO队列数据结构中最主要的字段有queue、dirty和processing。其中：
- **queue字段**是实际存储元素的地方，它是slice结构的，用于保证元素有序；
- **dirty字段**非常关键，除了能保证去重，还能保证在处理一个元素之前哪怕其被添加了多次（并发情况下），但也只会被处理一次；
- **processing字段**用于标记机制，标记一个元素是否正在被处理。
### 2.2. 延迟队列
迟队列，基于FIFO队列接口封装，在原有功能上增加了AddAfter方法，其原理是延迟一段时间后再将元素插入FIFO队列。延迟队列数据结构如下：

**代码路径：vendor/k8s.io/client-go/util/workqueue/delaying_queue.go**
```go
type DelayingInterface interface {
	Interface
	AddAfter(item interface{}, duration time.Duration)
}

type delayingType struct {
	Interface
	clock clock.Clock
	stopCh chan struct{}
	stopOnce sync.Once
	heartbeat clock.Ticker
	waitingForAddCh chan *waitFor
	metrics retryMetrics
}
```
AddAfter方法会插入一个item（元素）参数，并附带一个duration（延迟时间）参数，该duration参数用于指定元素延迟插入FIFO队列的时间。如果duration小于或等于0，会直接将元素插入FIFO队列中。
delayingType结构中最主要的字段是waitingForAddCh，其默认初始大小为1000，通过AddAfter方法插入元素时，是非阻塞状态的，只有当插入的元素大于或等于1000时，延迟队列才会处于阻塞状态。waitingForAddCh字段中的数据通过goroutine运行的waitingLoop函数持久运行。
### 2.3. 限速队列
限速队列，基于延迟队列和FIFO队列接口封装，限速队列接口（RateLimitingInterface）在原有功能上增加了AddRateLimited、Forget、NumRequeues方法。限速队列的重点不在于RateLimitingInterface接口，而在于它提供的4种限速算法接口（RateLimiter）。其原理是，限速队列利用延迟队列的特性，延迟某个元素的插入时间，达到限速目的。RateLimiter数据结构如下：

**代码路径：vendor/k8s.io/client-go/util/workqueue/default_rate_limiters.go**
```go
type RateLimiter interface {
	When(item interface{}) time.Duration
	Forget(item interface{})
	NumRequeues(item interface{}) int
}
```
限速算法每个方法说明如下：
- **When**：获取指定元素应该等待的时间。
- **Forget**：释放指定元素，清空该元素的排队数。
- **NumRequeues**：获取指定元素的排队数。

**注意**：这里有一个非常重要的概念——**限速周期**，一个限速周期是指从执行AddRateLimited方法到执行完Forget方法之间的时间。如果该元素被Forget方法处理完，则清空排队数。下面会分别详解WorkQueue提供的4种限速算法，应对不同的场景，这4种限速算法分别如下。
- 令牌桶算法（BucketRateLimiter）。
- 排队指数算法（ItemExponentialFailureRateLimiter）。
- 计数器算法（ItemFastSlowRateLimiter）。
- 混合模式（MaxOfRateLimiter），将多种限速算法混合使用。
#### 2.3.1. 令牌桶算法
令牌桶算法是通过Go语言的第三方库golang.org/x/time/rate实现的。令牌桶算法内部实现了一个存放token（令牌）的“桶”，初始时“桶”是空的，token会以固定速率往“桶”里填充，直到将其填满为止，多余的token会被丢弃。每个元素都会从令牌桶得到一个token，只有得到token的元素才允许通过（accept），而没有得到token的元素处于等待状态。令牌桶算法通过控制发放token来达到限速目的。
WorkQueue在默认的情况下会实例化令牌桶，代码示例如下：
```go
rate.NewLimiter(rate.Limit(10), 100)
```
在实例化rate.NewLimiter后，传入r和b两个参数，其中r参数表示每秒往“桶”里填充的token数量，b参数表示令牌桶的大小（即令牌桶最多存放的token数量）。我们假定r为10，b为100。假设在一个限速周期内插入了1000个元素，通过`r.Limiter.Reserve().Delay`函数返回指定元素应该等待的时间，那么前b（即100）个元素会被立刻处理，而后面元素的延迟时间分别为item100/100ms、item101/200ms、item102/300ms、item103/400ms，以此类推。
#### 2.3.2. 排队指数算法
排队指数算法将相同元素的排队数作为指数，排队数增大，速率限制呈指数级增长，但其最大值不会超过maxDelay。元素的排队数统计是有限速周期的，一个限速周期是指从执行AddRateLimited方法到执行完Forget方法之间的时间。如果该元素被Forget方法处理完，则清空排队数。排队指数算法的核心实现代码示例如下：

**代码路径：vendor/k8s.io/client-go/util/workqueue/default_rate_limiters.go**
```go
exp := r.failures[item]
r.failures[item] = r.failures[item] + 1
backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
if backoff > math.MaxInt64 {
  return r.maxDelay
}
```
该算法提供了3个主要字段：failures、baseDelay、maxDelay。其中，failures字段用于统计元素排队数，每当AddRateLimited方法插入新元素时，会为该字段加1；另外，baseDelay字段是最初的限速单位（默认为5ms），maxDelay字段是最大限速单位（默认为1000s）。限速队列利用延迟队列的特性，延迟多个相同元素的插入时间，达到限速目的。
#### 2.3.3. 计数器算法
计数器算法是限速算法中最简单的一种，其原理是：限制一段时间内允许通过的元素数量，例如在1分钟内只允许通过100个元素，每插入一个元素，计数器自增1，当计数器数到100的阈值且还在限速周期内时，则不允许元素再通过。但WorkQueue在此基础上扩展了fast和slow速率。计数器算法提供了4个主要字段：failures、fastDelay、slowDelay及maxFastAttempts。其中，failures字段用于统计元素排队数，每当AddRateLimited方法插入新元素时，会为该字段加1；而fastDelay和slowDelay字段是用于定义fast、slow速率的；另外，maxFastAttempts字段用于控制从fast速率转换到slow速率。计数器算法核心实现的代码示例如下：
```go
r.failures[item] = r.failures[item] + 1
if r.failures[item] <= r.maxFastAttempts {
  return r.fastDelay
}
return r.slowDelay
```
#### 2.3.4. 混合模式
混合模式是将多种限速算法混合使用，即多种限速算法同时生效。例如，同时使用排队指数算法和令牌桶算法。
