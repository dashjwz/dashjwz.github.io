---
title: golang下map的性能分析
author: dashjay
date: '2020-01-12'
slug: golang-map
categories: []
tags: []
---

## golang 中 map 性能优化[低阶]

### 简单介绍

golang 中的 build-in 的 map 这个 map 是非线程安全的，但是也是最常用的一个家伙。
为了测试多个 map 的性能我写了个接口 Map

```
type Map interface {
	Set(key interface{}, val interface{})
	Get(key interface{}) (interface{}, bool)
	Del(key interface{})
}
```

然后这是封装的普通的 map

```
type OriginMap struct {
	m map[interface{}]interface{}
}

func NewOriginMap() *OriginMap {
	return &OriginMap{m: make(map[interface{}]interface{})}
}

func (o *OriginMap) Get(key interface{}) (interface{}, bool) {
	v, ok := o.m[key]
	return v, ok
}
func (o *OriginMap) Set(key interface{}, value interface{}) {
	o.m[key] = value
}

func (o *OriginMap) Del(key interface{}) {
	delete(o.m, key)
}
```

别看一堆代码，其实就是 get 和 set 操作。在这里我们要使用 golang 自带的 test 工具

```
func TestOriginMaps(t *testing.T) {
	hm := NewOriginMap()
	wg := sync.WaitGroup{}
	for i := 0; i < Writer; i++ {
		wg.Add(1)
		go func(wg *sync.WaitGroup) {
			for k := 0; k < 100; k++ {
				hm.Set(strconv.Itoa(k), k*k)
				val, _ := hm.Get(strconv.Itoa(k))
				t.Logf("Get %d = %d", k, val)
			}
			wg.Done()
		}(&wg)
	}
	wg.Wait()
}
```

这其中有个变量 Writer 就是写者的数量，如果只有 1 的时候程序能安全运行退出

```
1264 ± : go test map_test/map_performance_test.go -v           ⏎ [3h1m] ✹ ✚ ✭
=== RUN   TestOriginMaps
--- PASS: TestOriginMaps (0.00s)
    map_performance_test.go:71: Get 0 = 0
    map_performance_test.go:71: Get 1 = 1
......
    map_performance_test.go:71: Get 99 = 9801
PASS
ok      command-line-arguments  0.339s
```

但是一旦我们把 Writer 数量改为 2

```
1264 ± : go test map_test/map_performance_test.go -v             [3h2m] ✹ ✚ ✭
=== RUN   TestOriginMaps
fatal error: concurrent map writes

goroutine 21 [running]:
```

立马就爆炸了。那么？？？？golang 自己官方心理没数么？

当然有数 golang 开发者其中之一可是拿图灵奖的。你可以点击[stackoverflow 上的讨论](https://stackoverflow.com/questions/11063473/map-with-concurrent-access "stackoverflow这里")和[github 这里](https://github.com/golang/go/issues/21035 "github 上的讨论")去查看相关的 issue

### Sync.Map

这是某大佬提出的解决方案，我们试试

```
type SyncMap struct {
	m sync.Map
}

func NewSyncMap() *SyncMap {
	return &SyncMap{}
}

func (o *SyncMap) Get(key interface{}) (interface{}, bool) {
	v, ok := o.m.Load(key)
	return v, ok
}
func (o *SyncMap) Set(key interface{}, value interface{}) {
	o.m.Store(key, value)
}

func (o *SyncMap) Del(key interface{}) {
	o.m.Delete(key)
}
```

我简单封装了一下，测试个性能没啥问题。

现在把 Write 增加也没问题了，可是真的没问题么？

我们现在小改一下第一种 map 加了个 RW 锁，然后和这种 map 做一下比较看看？

```
type OriginWithRWLock struct {
	m map[interface{}]interface{}
	l sync.RWMutex
}

func NewOriginWithRWLock() *OriginWithRWLock {
	return &OriginWithRWLock{
		m: make(map[interface{}]interface{}),
		l: sync.RWMutex{},
	}
}

func (o *OriginWithRWLock) Get(key interface{}) (interface{}, bool) {
	o.l.RLock()
	v, ok := o.m[key]
	o.l.RUnlock()
	return v, ok
}
func (o *OriginWithRWLock) Set(key interface{}, value interface{}) {
	o.l.Lock()
	o.m[key] = value
	o.l.Unlock()
}

func (o *OriginWithRWLock) Del(key interface{}) {
	o.l.Lock()
	delete(o.m, key)
	o.l.Unlock()
}
```

然后我们这次用 Test 里的 Benchmark 试试看

```
func BenchmarkMaps(b *testing.B) {

	b.Logf("Writer: %d,Reader: %d", Writer, Readers)
	b.Run("map with SyncLock", func(b *testing.B) {
		hm := NewSyncMap()
		benchmarkMap(b, hm)
	})

	b.Run("map with RWLock", func(b *testing.B) {
		hm := NewOriginWithRWLock()
		benchmarkMap(b, hm)
	})
}

func benchmarkMap(b *testing.B, hm Map) {
	for i := 0; i < b.N; i++ {
		var wg sync.WaitGroup
		for j := 0; j < Writer; j++ {
			wg.Add(1)
			go func() {
				for k := 0; k < 100; k++ {
					hm.Set(strconv.Itoa(k), k*k)
					hm.Set(strconv.Itoa(k), k*k)
					hm.Del(strconv.Itoa(k))
				}
				wg.Done()
			}()
		}
		for j := 0; j < Readers; j++ {
			wg.Add(1)
			go func() {
				for k := 0; k < 100; k++ {
					hm.Get(strconv.Itoa(k))
				}
				wg.Done()
			}()
		}
	}
}
```

首先是 BenchMark 的函数当使用

`go test .... -bench=.`

的时候会被调用，然后来测试两种 Map 性能，上面那个是测试性能的函数，分别对两个函数的进行测试~~拭目以待

> 当两者都是 100 的时候

```
1266 ± : go test map_test/map_performance_test.go -bench=. -v                                                                                       [3h29m] ✹ ✚ ✭
goos: darwin
goarch: amd64
BenchmarkMaps/map_with_SyncLock-8                   1459            749699 ns/op
BenchmarkMaps/map_with_RWLock-8                     1688           1405360 ns/op
--- BENCH: BenchmarkMaps
    map_performance_test.go:92: Writer: 100,Reader: 100
PASS
ok      command-line-arguments  4.674s

```

然后我测试了，各种情况下的比较画了个表格 单位是 ns/op ,每次操作需要的秒数

| R/W | map_with_SyncLock | map_with_RWLock |
| --- | --- | --- |
| 100/100 | 749699|1405360 |
|10/100 | 2053942| 333406|
| 100/10| 432235|1076423 |
|100/200 |1474482 | 1920310|
|200/100 |1135713 |2372780 |

那么从这个表里可以看出，SyncMap 的整体性能是优于 mapWithRWLock 的我来分析一下为什么

从古至今，人们一直在时间和空间上做斗争，这次也不例外，两种锁的实现原理不一样。

![](https://imgkr.cn-bj.ufileos.com/8ce8ae6e-8cee-421c-8d54-0057d287e448.png)

当我们使用普通 Map 带 RWMutex 会将整块内存锁住，然后其他请求就要等待。
SyncMap 是如何实现的呢？

![](https://imgkr.cn-bj.ufileos.com/9fd62de8-9760-47d5-a525-eb043b409a0c.png)

它分为两块内存（存的都是指针），一块只读区域，一块 Dirty 区域支持读写。

两边的指针指向原数据，当需要 Get 的时候他会执行`Load`操作从 Read 中去获取指针指向的值，如果没有找到（`miss`）发生了，就转而会去 dirty 中获得数据并且存入 Read 中。

当`miss`超过一定数量的时候，他就会用原子操作把 dirty 的数据 Promote 到 ReadOnly 中。

因此 Sync 这种机制，往往只适用于 Key-Value 相对稳定的业务情况，读多写少的业务。

> 手痒想写个内存的看看到底多花多少内存
> go tool pprof 是一个工具可以查看代码测评产生的内存日志

```
go test map_test/map_performance_test.go -bench=. -memprofile=mem.prof
go tool pprof map.test mem.prof
(pprof)top
...
      flat  flat%   sum%        cum   cum%
    1.54GB 57.95% 57.95%     1.57GB 59.14%  command-line-arguments_test.benchmarkMap.func2
    0.43GB 16.22% 74.17%     0.69GB 25.94%  command-line-arguments_test.benchmarkMap.func1

```

不用说了这看起来三倍的内存消耗，果然越快内存越大。那么？本次测评到此结束？

！！！ 并没有！！！
还有一个大佬写了个 concurrent-map 甚叼，我们来观摩一波。[concurrent-map](https://github.com/orcaman/concurrent-map "concurrent-map")

立马封装一波

```
type ConCurrentMap struct {
	m cmap.ConcurrentMap
}

func NewConCurrentMap() *ConCurrentMap {
	conMap := cmap.New()
	return &ConCurrentMap{m: conMap}
}

type OriginMap struct {
	m map[interface{}]interface{}
}

func (c *ConCurrentMap) Get(key interface{}) (interface{}, bool) {
	v, ok := c.m.Get(key.(string))
	return v, ok
}
func (c *ConCurrentMap) Set(key interface{}, value interface{}) {
	c.m.Set(key.(string), value)
}

func (c *ConCurrentMap) Del(key interface{}) {
	c.m.Remove(key.(string))
}
```

迫不及待开始测试，当 Write=100，Reader=100 的时候

```
1271 ± : go test map_test/map_performance_test.go -bench=. -v                                                                                       [4h11m] ✹ ✚ ✭
goos: darwin
goarch: amd64
BenchmarkMaps/map_with_SyncLock-8                   1374            760728 ns/op
BenchmarkMaps/map_with_RWLock-8                     1671           1556679 ns/op
BenchmarkMaps/concurrentMap-8                       2667           1060736 ns/op
--- BENCH: BenchmarkMaps
    map_performance_test.go:114: Writer: 100,Reader: 100
```

那么我同样做个表格吧，把读写的几种情况都列出来

| R/W | map_with_SyncLock | map_with_RWLock | concurrentMap |
| --- | --- | --- | --- |
| 100/100 |760728 |1556679 | 1060736|
|10/100 |1690986 |338078 |457065 |
| 100/10|443063 | 1026160|544635 |


最后说一下这个并发读map是怎么搞的


![](https://imgkr.cn-bj.ufileos.com/84240d5a-21a9-4df7-8cf6-97fd06dc2b11.png)

左边是普通的map，当有读写的时候锁上了，其他线程就无法读写了。右边的事`concurrentMap`，他利用了一种`partition`的思想，把`Map`的内存`SHARD`（分割）成N份，然后用不同的🔐锁上锁，那么降低了需要资源被锁的概率。

我们在日常中编程的时候容易陷入一种误区，就是这锁，那锁，全锁上，面试也在问各种锁，但是在真实QPS比价高的业务中，锁是一种很可怕的东西，如果能在编程的时候好好想想写出`LockFree`的程序是最好的啊。

我是北京某211的混子，从19年10月开始写两行golang到现在不知不觉已经过去了2个月，上手就开始拉框架写代码的我已经进化到开始分析性能，然后优化代码啦，如果有小伙伴想一起讨论讨论，欢迎。
