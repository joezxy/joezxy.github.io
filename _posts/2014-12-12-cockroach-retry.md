---
layout: post
category: "code"
title:  "Cockroach的Retry机制"
tags: [Cockroach]
---

Retry框架的核心方法RetryWithBackoff
```
func RetryWithBackoff(opts RetryOptions, fn func() (RetryStatus, error)) error 
```
RetryOptions设置缺省及最大重试间隔、最大重试次数
```
type RetryOptions struct {
	Tag         string        // Tag for helpful logging of backoffs
	Backoff     time.Duration // Default retry backoff interval
	MaxBackoff  time.Duration // Maximum retry backoff interval
	Constant    float64       // Default backoff constant
	MaxAttempts int           // Maximum number of attempts (0 for infinite)————Cockroach中，重试次数基本设置为0
	UseV1Info   bool          // Use verbose V(1) level for log messages
}
```
根据传入闭包函数的返回字段RetryStatus的值，RetryWithBackoff将结束重试（RetryBreak）、立即重试（RetryReset）、或者在一段时间后重试（RetryContinue）
```
const (
	// RetryBreak indicates the retry loop is finished and should return
	// the result of the retry worker function.
	RetryBreak RetryStatus = iota
	// RetryReset indicates that the retry loop should be reset with
	// no backoff for an immediate retry.
	RetryReset
	// RetryContinue indicates that the retry loop should continue with
	// another iteration of backoff / retry.
	RetryContinue
)
```
如何在返回RetryContinue时确定重试的间隔，代码如下：
```
wait = backoff + time.Duration(rand.Float64()*float64(backoff.Nanoseconds())*retryJitter)
// Increase backoff for next iteration.
backoff = time.Duration(float64(backoff) * opts.Constant)
if backoff > opts.MaxBackoff {
	backoff = opts.MaxBackoff
}
```
每次重试的等待时间wait=重试时间backoff*（1+0.X*jitter）————Store的backoff设置为50ms，每一次会加上少许
按Constant值增加重试时间backoff给下一次重试使用————Store的Constant值设置为2，这样就是以2的指数方式递增
设置递增后的backoff不能大于MaxBackoff————Store的MaxBackoff设置为5s



