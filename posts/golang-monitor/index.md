# Golang监控


看了[一篇文章](https://blog.csdn.net/u010278923/article/details/78774914)，里面涉及到了一些golang程序监控的问题，回过头总结了一下实现方式，简单介绍一下

## expvar

go自带的runtime包拥有各种功能，包括goroutine数量，设置逻辑线程数量，当前go版本，当前系统类型等等。前两天发现了go标准库还有一个更好用的可以监控服务运行各项指标和状态的包—-expvar。  expvar包为监控变量提供了一个标准化的接口，它以 JSON 格式通过 /debug/vars 接口以 HTTP 的方式公开这些监控变量以及我自定义的变量。通过它，再加上metricBeat，ES和Kibana，可以很轻松的对服务进行监控。我这里是用gin把接口暴露出来，其实用别的web框架也都可以。下面我们来看一下如何使用它（示例代码使用[GIN HTTP web framework](https://github.com/gin-gonic/gin)）：

```go
package main

import (
	"expvar"
	"github.com/gin-gonic/gin"
	"net/http"
	"net/http/pprof"
	"time"
)

func main() {
	router := gin.Default()
	router.GET("/debug/vars", monitor.GetCurrentRunningStats)
	s := &http.Server{
		Addr:           ":9090",
		Handler:        router,
		ReadTimeout:    5 * time.Second,
		WriteTimeout:   5 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}

	s.ListenAndServe()
}
```

对应的handler

```go
package monitor

import (
	"encoding/json"
	"expvar"
	"fmt"
	"github.com/gin-gonic/gin"
	"net/http"
	"runtime"
	"time"
)

// 开始时间
var start = time.Now()

// calculateUptime 计算运行时间
func calculateUptime() interface{} {
	return time.Since(start).String()
}

// currentGoVersion 当前 Golang 版本
func currentGoVersion() interface{} {
	return runtime.Version()
}

// getNumCPUs 获取 CPU 核心数量
func getNumCPUs() interface{} {
	return runtime.NumCPU()
}

// getGoOS 当前系统类型
func getGoOS() interface{} {
	return runtime.GOOS
}

// getNumGoroutins 当前 goroutine 数量
func getNumGoroutins() interface{} {
	return runtime.NumGoroutine()
}

// getNumCgoCall CGo 调用次数
func getNumCgoCall() interface{} {
	return runtime.NumCgoCall()
}

var lastPause uint32

// getLastGCPauseTime 获取上次 GC 的暂停时间
func getLastGCPauseTime() interface{} {
	var gcPause uint64
	ms := new(runtime.MemStats)

	statString := expvar.Get("memstats").String()
	if statString != "" {
		json.Unmarshal([]byte(statString), ms)

		if lastPause == 0 || lastPause != ms.NumGC {
			gcPause = ms.PauseNs[(ms.NumGC+255)%256]
			lastPause = ms.NumGC
		}
	}

	return gcPause
}

// GetCurrentRunningStats 返回当前运行信息
func GetCurrentRunningStats(c *gin.Context) {
	c.Writer.Header().Set("Content-Type", "application/json; charset=utf-8")

	first := true
	report := func(key string, value interface{}) {
		if !first {
			fmt.Fprintf(c.Writer, ",\n")
		}
		first = false
		if str, ok := value.(string); ok {
			fmt.Fprintf(c.Writer, "%q: %q", key, str)
		} else {
			fmt.Fprintf(c.Writer, "%q: %v", key, value)
		}
	}

	fmt.Fprintf(c.Writer, "{\n")
	expvar.Do(func(kv expvar.KeyValue) {
		report(kv.Key, kv.Value)
	})
	fmt.Fprintf(c.Writer, "\n}\n")

	c.String(http.StatusOK, "")
}

func init() { //这些都是我自定义的变量，发布到expvar中，每次请求接口，expvar会自动去获取这些变量，并返回给我
	expvar.Publish("运行时间", expvar.Func(calculateUptime))
	expvar.Publish("version", expvar.Func(currentGoVersion))
	expvar.Publish("cores", expvar.Func(getNumCPUs))
	expvar.Publish("os", expvar.Func(getGoOS))
	expvar.Publish("cgo", expvar.Func(getNumCgoCall))
	expvar.Publish("goroutine", expvar.Func(getNumGoroutins))
	expvar.Publish("gcpause", expvar.Func(getLastGCPauseTime))
}
```

运行程序，访问http://localhost:9090/debug/vars，如下

![1530461378444](expvar.png)

可以看到，expvar返回给了我我之前自定义的数据，以及它本身要默认返回的数据，比如memstats。这个memstats是干嘛的呢，其实看到这些字段名就可以知道，是各种内存堆栈以及GC的一些信息，具体可以看源码注释： 

```go
type MemStats struct {
   // General statistics.

   // Alloc is bytes of allocated heap objects.
   //
   // This is the same as HeapAlloc (see below).
   Alloc uint64

   // TotalAlloc is cumulative bytes allocated for heap objects.
   //
   // TotalAlloc increases as heap objects are allocated, but
   // unlike Alloc and HeapAlloc, it does not decrease when
   // objects are freed.
   TotalAlloc uint64

   // Sys is the total bytes of memory obtained from the OS.
   //
   // Sys is the sum of the XSys fields below. Sys measures the
   // virtual address space reserved by the Go runtime for the
   // heap, stacks, and other internal data structures. It's
   // likely that not all of the virtual address space is backed
   // by physical memory at any given moment, though in general
   // it all was at some point.
   Sys uint64

   // Lookups is the number of pointer lookups performed by the
   // runtime.
   //
   // This is primarily useful for debugging runtime internals.
   Lookups uint64

   // Mallocs is the cumulative count of heap objects allocated.
   // The number of live objects is Mallocs - Frees.
   Mallocs uint64

   // Frees is the cumulative count of heap objects freed.
   Frees uint64

   // Heap memory statistics.
   //
   // Interpreting the heap statistics requires some knowledge of
   // how Go organizes memory. Go divides the virtual address
   // space of the heap into "spans", which are contiguous
   // regions of memory 8K or larger. A span may be in one of
   // three states:
   //
   // An "idle" span contains no objects or other data. The
   // physical memory backing an idle span can be released back
   // to the OS (but the virtual address space never is), or it
   // can be converted into an "in use" or "stack" span.
   //
   // An "in use" span contains at least one heap object and may
   // have free space available to allocate more heap objects.
   //
   // A "stack" span is used for goroutine stacks. Stack spans
   // are not considered part of the heap. A span can change
   // between heap and stack memory; it is never used for both
   // simultaneously.

   // HeapAlloc is bytes of allocated heap objects.
   //
   // "Allocated" heap objects include all reachable objects, as
   // well as unreachable objects that the garbage collector has
   // not yet freed. Specifically, HeapAlloc increases as heap
   // objects are allocated and decreases as the heap is swept
   // and unreachable objects are freed. Sweeping occurs
   // incrementally between GC cycles, so these two processes
   // occur simultaneously, and as a result HeapAlloc tends to
   // change smoothly (in contrast with the sawtooth that is
   // typical of stop-the-world garbage collectors).
   HeapAlloc uint64

   // HeapSys is bytes of heap memory obtained from the OS.
   //
   // HeapSys measures the amount of virtual address space
   // reserved for the heap. This includes virtual address space
   // that has been reserved but not yet used, which consumes no
   // physical memory, but tends to be small, as well as virtual
   // address space for which the physical memory has been
   // returned to the OS after it became unused (see HeapReleased
   // for a measure of the latter).
   //
   // HeapSys estimates the largest size the heap has had.
   HeapSys uint64

   // HeapIdle is bytes in idle (unused) spans.
   //
   // Idle spans have no objects in them. These spans could be
   // (and may already have been) returned to the OS, or they can
   // be reused for heap allocations, or they can be reused as
   // stack memory.
   //
   // HeapIdle minus HeapReleased estimates the amount of memory
   // that could be returned to the OS, but is being retained by
   // the runtime so it can grow the heap without requesting more
   // memory from the OS. If this difference is significantly
   // larger than the heap size, it indicates there was a recent
   // transient spike in live heap size.
   HeapIdle uint64

   // HeapInuse is bytes in in-use spans.
   //
   // In-use spans have at least one object in them. These spans
   // can only be used for other objects of roughly the same
   // size.
   //
   // HeapInuse minus HeapAlloc esimates the amount of memory
   // that has been dedicated to particular size classes, but is
   // not currently being used. This is an upper bound on
   // fragmentation, but in general this memory can be reused
   // efficiently.
   HeapInuse uint64

   // HeapReleased is bytes of physical memory returned to the OS.
   //
   // This counts heap memory from idle spans that was returned
   // to the OS and has not yet been reacquired for the heap.
   HeapReleased uint64

   // HeapObjects is the number of allocated heap objects.
   //
   // Like HeapAlloc, this increases as objects are allocated and
   // decreases as the heap is swept and unreachable objects are
   // freed.
   HeapObjects uint64

   // Stack memory statistics.
   //
   // Stacks are not considered part of the heap, but the runtime
   // can reuse a span of heap memory for stack memory, and
   // vice-versa.

   // StackInuse is bytes in stack spans.
   //
   // In-use stack spans have at least one stack in them. These
   // spans can only be used for other stacks of the same size.
   //
   // There is no StackIdle because unused stack spans are
   // returned to the heap (and hence counted toward HeapIdle).
   StackInuse uint64

   // StackSys is bytes of stack memory obtained from the OS.
   //
   // StackSys is StackInuse, plus any memory obtained directly
   // from the OS for OS thread stacks (which should be minimal).
   StackSys uint64

   // Off-heap memory statistics.
   //
   // The following statistics measure runtime-internal
   // structures that are not allocated from heap memory (usually
   // because they are part of implementing the heap). Unlike
   // heap or stack memory, any memory allocated to these
   // structures is dedicated to these structures.
   //
   // These are primarily useful for debugging runtime memory
   // overheads.

   // MSpanInuse is bytes of allocated mspan structures.
   MSpanInuse uint64

   // MSpanSys is bytes of memory obtained from the OS for mspan
   // structures.
   MSpanSys uint64

   // MCacheInuse is bytes of allocated mcache structures.
   MCacheInuse uint64

   // MCacheSys is bytes of memory obtained from the OS for
   // mcache structures.
   MCacheSys uint64

   // BuckHashSys is bytes of memory in profiling bucket hash tables.
   BuckHashSys uint64

   // GCSys is bytes of memory in garbage collection metadata.
   GCSys uint64

   // OtherSys is bytes of memory in miscellaneous off-heap
   // runtime allocations.
   OtherSys uint64

   // Garbage collector statistics.

   // NextGC is the target heap size of the next GC cycle.
   //
   // The garbage collector's goal is to keep HeapAlloc ≤ NextGC.
   // At the end of each GC cycle, the target for the next cycle
   // is computed based on the amount of reachable data and the
   // value of GOGC.
   NextGC uint64

   // LastGC is the time the last garbage collection finished, as
   // nanoseconds since 1970 (the UNIX epoch).
   LastGC uint64

   // PauseTotalNs is the cumulative nanoseconds in GC
   // stop-the-world pauses since the program started.
   //
   // During a stop-the-world pause, all goroutines are paused
   // and only the garbage collector can run.
   PauseTotalNs uint64

   // PauseNs is a circular buffer of recent GC stop-the-world
   // pause times in nanoseconds.
   //
   // The most recent pause is at PauseNs[(NumGC+255)%256]. In
   // general, PauseNs[N%256] records the time paused in the most
   // recent N%256th GC cycle. There may be multiple pauses per
   // GC cycle; this is the sum of all pauses during a cycle.
   PauseNs [256]uint64

   // PauseEnd is a circular buffer of recent GC pause end times,
   // as nanoseconds since 1970 (the UNIX epoch).
   //
   // This buffer is filled the same way as PauseNs. There may be
   // multiple pauses per GC cycle; this records the end of the
   // last pause in a cycle.
   PauseEnd [256]uint64

   // NumGC is the number of completed GC cycles.
   NumGC uint32

   // NumForcedGC is the number of GC cycles that were forced by
   // the application calling the GC function.
   NumForcedGC uint32

   // GCCPUFraction is the fraction of this program's available
   // CPU time used by the GC since the program started.
   //
   // GCCPUFraction is expressed as a number between 0 and 1,
   // where 0 means GC has consumed none of this program's CPU. A
   // program's available CPU time is defined as the integral of
   // GOMAXPROCS since the program started. That is, if
   // GOMAXPROCS is 2 and a program has been running for 10
   // seconds, its "available CPU" is 20 seconds. GCCPUFraction
   // does not include CPU time used for write barrier activity.
   //
   // This is the same as the fraction of CPU reported by
   // GODEBUG=gctrace=1.
   GCCPUFraction float64

   // EnableGC indicates that GC is enabled. It is always true,
   // even if GOGC=off.
   EnableGC bool

   // DebugGC is currently unused.
   DebugGC bool

   // BySize reports per-size class allocation statistics.
   //
   // BySize[N] gives statistics for allocations of size S where
   // BySize[N-1].Size < S ≤ BySize[N].Size.
   //
   // This does not report allocations larger than BySize[60].Size.
   BySize [61]struct {
      // Size is the maximum byte size of an object in this
      // size class.
      Size uint32

      // Mallocs is the cumulative count of heap objects
      // allocated in this size class. The cumulative bytes
      // of allocation is Size*Mallocs. The number of live
      // objects in this size class is Mallocs - Frees.
      Mallocs uint64

      // Frees is the cumulative count of heap objects freed
      // in this size class.
      Frees uint64
   }
}
```

如果需要查看其对应的翻译，请看[这里](http://lib.csdn.net/article/go/68270?knId=1441)

## pprof

有关pprof的基本介绍和使用，可以参考[这里](https://www.cnblogs.com/yjf512/archive/2012/12/27/2835331.html)，最简单的使用方式如下

```go
package main

import (
	"net/http"
	_ "net/http/pprof"
)

func main() {
	http.ListenAndServe(":9090", nil)
}
```

运行程序，访问http://localhost:9090/debug/pprof可以看到如下结果

![1530462279152](pprof.png)

配合go tool pprof一起使用，例如查看cpu详情，可以通过如下命令实现

```shell
root@kaku-Inspiron-7537:~/blog# go tool pprof localhost:9090/debug/pprof/profile
Fetching profile over HTTP from http://localhost:9090/debug/pprof/profile
Saved profile in /root/pprof/pprof.___go_build_main_go__1_.samples.cpu.002.pb.gz
File: ___go_build_main_go__1_
Type: cpu
Time: Jul 2, 2018 at 12:44am (CST)
Duration: 30s, Total samples = 0 
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) web
```

执行完之后，如果本地安装了graphviz和chrome，则自动打开浏览器，可以在其中看到如下图片

![1530463634094](pprof_cpu.png)

因为示例程序没有任何其他代码，之暴露了http服务，所以看到的结果比较单调

## Prometheus

Prometheus client内置了golang metrics暴露的handler，只需要简单调用即可实现，如下

```go
package main

import (
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

func main() {
    http.Handle("/metrics", promhttp.Handler())
    panic(http.ListenAndServe(":9090", nil))
}
```

运行程序，访问http://localhost:9090/metrics 即可。

同时可以通过Prometheus来采集此Endpoint暴露出来的数据，也可以进行自定义数据的采集，参考[这里](https://pierrevincent.github.io/2017/12/prometheus-blog-series-part-4-instrumenting-code-in-go-and-java/)


