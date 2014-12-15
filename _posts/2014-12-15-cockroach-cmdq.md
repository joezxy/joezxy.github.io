---
layout: post
category: "code"
title:  "Cockroach的CommandQueue机制"
tags: [Cockroach]
---

Cockroach的每个Range都有一个CommandQueue，内部包含一个IntervalCache，用于将该Range的读写操作进行排队（阻塞等待）。

```
type CommandQueue struct {
	cache *util.IntervalCache
}

type cmd struct {
	readOnly bool
	pending  []*sync.WaitGroup // Pending commands gated on cmd
}

func (cq *CommandQueue) Add(start, end proto.Key, readOnly bool) interface{} {
	if len(end) == 0 {
		end = start.Next()
	}
	key := cq.cache.NewKey(start, end)
	cq.cache.Add(key, &cmd{readOnly: readOnly})
	return key
}
```

在Range的addReadOnlyCmd以及addReadWriteCmd方法中，会先调用beginCmd进行请求的排队操作；

```
func (r *Range) beginCmd(start, end proto.Key, readOnly bool) interface{} {
	r.Lock()
	var wg sync.WaitGroup
	r.cmdQ.GetWait(start, end, readOnly, &wg)
	cmdKey := r.cmdQ.Add(start, end, readOnly)
	r.Unlock()
	wg.Wait()
	return cmdKey
}

func (cq *CommandQueue) GetWait(start, end proto.Key, readOnly bool, wg *sync.WaitGroup) {
	// This gives us a memory-efficient end key if end is empty.
	if len(end) == 0 {
		end = start.Next()
		start = end[:len(start)]
	}
	for _, c := range cq.cache.GetOverlaps(start, end) {
		c := c.Value.(*cmd)
		// Only add to the wait group if one of the commands isn't read-only.
		if !readOnly || !c.readOnly {
			c.pending = append(c.pending, wg)
			wg.Add(1)
		}
	}
}
```

在begineCmd中，先声明一个WaitGroup；
再调用CommandQueue的GetWait方法，获取到所有Key区间重叠的请求。如果两个请求中有一个是写操作，则将本请求附加到该条目的pending列表中。
然后将本请求添加到CommandQueue中，并开始进行阻塞等待
在完成实际的数据读写后，将使用r.cmdQ.Remove(cmdKey)将此操作移出CommandQueue，从而触发CommandQueue的回调函数onEvicted，以解除其他读写请求的阻塞

```
func (cq *CommandQueue) onEvicted(key, value interface{}) {
	c := value.(*cmd)
	for _, wg := range c.pending {
		wg.Done()
	}
}
```