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
// Execute runs the given request if the CircuitBreaker accepts it.
// Execute returns an error instantly if the CircuitBreaker rejects the request.
// Otherwise, Execute returns the result of the request.
// If a panic occurs in the request, the CircuitBreaker handles it as an error
// and causes the same panic again.
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

