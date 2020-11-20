---
layout: post
title: "Circuit Breaker Pattern"
author: "Gaius"
categories: Design pattern
tags: [documentation]
image: circuit-breaker-pattern.jpg
---

## 介绍
熔断模式，类比现实当中电路熔断机制。当线路电压过高时会保险丝会熔断, 维修成功后可进行恢复供电。分布式场景下也会面临服务异常以及网络超时等问题，需要一定时间进行恢复。如果一直进行重试请求，在未恢复的这段时间内都会返回失败，并且占用资源。所以需要熔断模式的设计核心为 "防止无限重试一个已知的失败操作" 。

## 原理
熔断器相当于 Proxy，检测请求成功率当达到一定阈值时通过切换状态，给出正确的处理方式。

熔断器模式使用状态机进行实现，主要有以下三种状态:
- Closed: 默认状态，允许操作执行。会监听操作失败次数，当达到阈值时熔断器转换为 Open 状态。
- Open: 执行操作失败会抛出异常，且会根据提前设置好恢复需要时间进行计时。当超过恢复时间时切换为 Half-Open 状态。
- Half-Open: 允许执行一定次数操作，如果全部成功即认为故障恢复，切换为 Closed 状态。如果有一次失败操作，则认为故障未恢复，转换为 Open 状态。

## gobreaker
Sony 开源项目 [gobreaker](https://github.com/sony/gobreaker) 使用状态机实现熔断模式。源码实现总共 350 行相对比较简单。

熔断器 Struct
```golang
type Settings struct {
	Name          string
	MaxRequests   uint32
	Interval      time.Duration
	Timeout       time.Duration
	ReadyToTrip   func(counts Counts) bool
	OnStateChange func(name string, from State, to State)
}
```

熔断器创建以及配置
```golang
type Settings struct {
	Name          string
	MaxRequests   uint32
	Interval      time.Duration
	Timeout       time.Duration
	ReadyToTrip   func(counts Counts) bool
	OnStateChange func(name string, from State, to State)
}

func NewCircuitBreaker(st Settings) *CircuitBreaker {
	cb := new(CircuitBreaker)

	cb.name = st.Name
	cb.onStateChange = st.OnStateChange

	if st.MaxRequests == 0 {
		cb.maxRequests = 1
	} else {
		cb.maxRequests = st.MaxRequests
	}

	if st.Interval <= 0 {
		cb.interval = defaultInterval
	} else {
		cb.interval = st.Interval
	}

	if st.Timeout <= 0 {
		cb.timeout = defaultTimeout
	} else {
		cb.timeout = st.Timeout
	}

	if st.ReadyToTrip == nil {
		cb.readyToTrip = defaultReadyToTrip
	} else {
		cb.readyToTrip = st.ReadyToTrip
	}

	cb.toNewGeneration(time.Now())

	return cb
}
```

执行函数主要分为三个阶段: beforeRequest、Execute 以及 afterRequest
```golang
func (cb *CircuitBreaker) Execute(req func() (interface{}, error)) (interface{}, error) {
	generation, err := cb.beforeRequest()
	if err != nil {
		return nil, err
	}

	defer func() {
		e := recover()
		if e != nil {
			cb.afterRequest(generation, false)
			panic(e)
		}
	}()

	result, err := req()
	cb.afterRequest(generation, err == nil)
	return result, err
}
```

生成新的 generation，expiry 为过期时间。按照状态区分为:

- Open: expiry 为当前时间加 Setting 中的 Timeout 恢复时间，
- Closed: expiry 为当前时间加 Setting 中的 Interval 一个监视周期时间。生成新的 generation 会清空 counts 内值。
- Half-Open: expiry 清零，通过 maxRequests 来判断。
```golang
func (cb *CircuitBreaker) toNewGeneration(now time.Time) {
	cb.generation++
	cb.counts.clear()

	var zero time.Time
	switch cb.state {
	case StateClosed:
		if cb.interval == 0 {
			cb.expiry = zero
		} else {
			cb.expiry = now.Add(cb.interval)
		}
	case StateOpen:
		cb.expiry = now.Add(cb.timeout)
	default: // StateHalfOpen
		cb.expiry = zero
	}
}

func (c *Counts) clear() {
	c.Requests = 0
	c.TotalSuccesses = 0
	c.TotalFailures = 0
	c.ConsecutiveSuccesses = 0
	c.ConsecutiveFailures = 0
}
```

currentState 获取当前状态, 按照状态区分为:

- Closed: expiry 过期时间为 0 即达到一个监视周期时间则生成新的 generation, 清空 counts 内值。
- Open: expiry 过期时间为 0 时即到达恢复时间, 设置为 Half-Open 状态。
```golang
func (cb *CircuitBreaker) currentState(now time.Time) (State, uint64) {
	switch cb.state {
	case StateClosed:
		if !cb.expiry.IsZero() && cb.expiry.Before(now) {
			cb.toNewGeneration(now)
		}
	case StateOpen:
		if cb.expiry.Before(now) {
			cb.setState(StateHalfOpen, now)
		}
	}
	return cb.state, cb.generation
}
```

beforeRequest 给 Requests 变量加互斥锁, 防止竞争。currentState 获取当前状态, 通过熔断器三种状态执行不同操作:

- Open: 直接抛错。
- Half-Open && counts 中累计 Request 大于 Half-Open 状态 maxRequest 阈值: 直接返回错误。
- Closed && Half-Open 且累计 Request 小于 Half-Open 状态 maxRequest 阈值: Request 数加一。
```golang
func (cb *CircuitBreaker) beforeRequest() (uint64, error) {
	cb.mutex.Lock()
	defer cb.mutex.Unlock()

	now := time.Now()
	state, generation := cb.currentState(now)

	if state == StateOpen {
		return generation, ErrOpenState
	} else if state == StateHalfOpen && cb.counts.Requests >= cb.maxRequests {
		return generation, ErrTooManyRequests
	}

	cb.counts.onRequest()
	return generation, nil
}

func (c *Counts) onRequest() {
	c.Requests++
}
```

afterRequest 给 counts 变量加互斥锁, 防止竞争。操作执行后分为两种状态:

- onSuccess: 当状态为 Closed 时则更改 count 计数, 当状态为 Half-Open 时则更改 count 计数且对比 ConsecutiveSuccesses 量即连续成功操作次数是否大于 maxRequest，如果大于则更改状态为 Closed。
- onFailure: 当状态为 Closed 时则更改 count 计数, readyToTrip 为 true 则状态变为 Open, 当状态为 Half-Open 状态变为 Open。
```golang
func (cb *CircuitBreaker) afterRequest(before uint64, success bool) {
    cb.mutex.Lock()
    defer cb.mutex.Unlock()

    now := time.Now()
    state, generation := cb.currentState(now)
    if generation != before {
        return
    }

    if success {
        cb.onSuccess(state, now)
    } else {
        cb.onFailure(state, now) 
    }
}

func (cb *CircuitBreaker) onSuccess(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onSuccess()
	case StateHalfOpen:
		cb.counts.onSuccess()
		if cb.counts.ConsecutiveSuccesses >= cb.maxRequests {
			cb.setState(StateClosed, now)
		}
	}
}

func (c *Counts) onSuccess() {
	c.TotalSuccesses++
	c.ConsecutiveSuccesses++
	c.ConsecutiveFailures = 0
}

func (cb *CircuitBreaker) onFailure(state State, now time.Time) {
	switch state {
	case StateClosed:
		cb.counts.onFailure()
		if cb.readyToTrip(cb.counts) {
			cb.setState(StateOpen, now)
		}
	case StateHalfOpen:
		cb.setState(StateOpen, now)
	}
}

func (c *Counts) onFailure() {
	c.TotalFailures++
	c.ConsecutiveFailures++
	c.ConsecutiveSuccesses = 0
}
```
