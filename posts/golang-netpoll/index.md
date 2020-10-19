# Golang netpoll源码分析


## 简介

go针对不同的操作系统，其网络io模型不同，可以从go源码目录结构和对应内容清楚的看到各平台的io模型，如针对linux系统实现的`epoll`，针对windows操作系统实现的`iocp`等，这里主要看针对linux系统的实现，涉及到的文件大体如下：

- [runtime/netpoll.go](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/netpoll.go)
- [runtime/netpoll_epoll.go](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/netpoll_epoll.go)
- [runtime/proc.go](https://github.com/golang/go/blob/release-branch.go1.10/src/runtime/proc.go)
- [net/fd_unix.go](https://github.com/golang/go/blob/release-branch.go1.10/src/net/fd_unix.go)
- [internal/poll/fd_poll_runtime.go](https://github.com/golang/go/blob/release-branch.go1.10/src/internal/poll/fd_poll_runtime.go)
- [internal/poll/fd_unix.go](https://github.com/golang/go/blob/release-branch.go1.10/src/internal/poll/fd_unix.go)

在开始正式看源码之前需要具备一些基础知识，如`同步`、`异步`、`阻塞`、`非阻塞`、`io多路复用`、`go调度模型`等，不了解的话可以参考下面的链接：

- [同步异步阻塞非阻塞](https://blog.csdn.net/historyasamirror/article/details/5778378)
- [io多路复用select poll epoll](https://blog.csdn.net/tennysonsky/article/details/45745887)
- [goroutine实现原理](http://www.cnblogs.com/zkweb/p/7815600.html)

也可以网上自行搜索，文章很多

## 源码分析（v1.10.2）

golang通过对`epoll`的封装来取得使用同步编程达异步执行的效果。总结来说，所有的网络操作都以网络描述符`netFD`为中心实现，netFD通过将`Sysfd`与`pollDesc`结构绑定，当在一个netFD上读写遇到`EAGAIN`错误时，就将**当前goroutine**存储到这个netFD对应的pollDesc中，同时将goroutine给park住，直到这个netFD上再次发生读写事件，才将此goroutine设置为ready放入待运行队列等待重新运行，在底层通知goroutine再次发生读写等事件的方式就是靠的epoll事件驱动机制。

### netFD

服务端通过Listen方法返回的`Listener`接口的实现和通过listener的`Accept`方法返回的Conn接口的实现都包含一个网络文件描述符netFD，netFD中包含一个poll.FD数据结构，而poll.FD中包含两个重要的数据结构Sysfd和pollDesc，前者是真正的系统文件描述符，后者对是底层事件驱动的封装，所有的读写超时等操作都是通过调用后者的对应方法实现的。

1. 服务端的netFD在`listen`时会创建epoll的实例，并将listenFD加入epoll的事件队列
2. netFD在`accept`时将返回的connFD也加入epoll的事件队列
3. netFD在读写时出现`syscall.EAGAIN`错误，通过pollDesc将当前的goroutine park住，直到ready，从pollDesc的`waitRead`中返回

涉及到的一些结构：

```go
type TCPListener struct {
	fd *netFD
}

type conn struct {
	fd *netFD
}

// Network file descriptor.
type netFD struct {
	pfd poll.FD

	// immutable until Close
	family      int
	sotype      int
	isConnected bool
	net         string
	laddr       Addr
	raddr       Addr
}

// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	fdmu fdMutex

	// System file descriptor. Immutable until Close.
    // 系统文件描述符
	Sysfd int

	// I/O poller.
	pd pollDesc

	// Writev cache.
	iovecs *[]syscall.Iovec

	// Semaphore signaled when file is closed.
	csema uint32

	// Whether this is a streaming descriptor, as opposed to a
	// packet-based descriptor like a UDP socket. Immutable.
	IsStream bool

	// Whether a zero byte read indicates EOF. This is false for a
	// message based socket connection.
	ZeroReadIsEOF bool

	// Whether this is a file rather than a network socket.
	isFile bool

	// Whether this file has been set to blocking mode.
	isBlocking bool
}
```

### pollDesc

上面提到的`net.conn`的读写等操作实际上就是调用的`poll.FD`对应的方法，`poll.FD`中包含一个重要结构`poll.pollDesc`，其定义如下：

```go
type pollDesc struct {
	runtimeCtx uintptr
}
```

可以看到其中只包含一个指针，这个指针具体代表的其实是另一个同名不同包的结构`runtime.pollDesc`，定义如下：

```go
// Network poller descriptor.
//
// No heap pointers.
//
//go:notinheap
type pollDesc struct {
	link *pollDesc // in pollcache, protected by pollcache.lock

	// The lock protects pollOpen, pollSetDeadline, pollUnblock and deadlineimpl operations.
	// This fully covers seq, rt and wt variables. fd is constant throughout the PollDesc lifetime.
	// pollReset, pollWait, pollWaitCanceled and runtime·netpollready (IO readiness notification)
	// proceed w/o taking the lock. So closing, rg, rd, wg and wd are manipulated
	// in a lock-free way by all operations.
	// NOTE(dvyukov): the following code uses uintptr to store *g (rg/wg),
	// that will blow up when GC starts moving objects.
	lock    mutex // protects the following fields
	fd      uintptr
	closing bool
	seq     uintptr // protects from stale timers and ready notifications
	rg      uintptr // pdReady, pdWait, G waiting for read or nil
	rt      timer   // read deadline timer (set if rt.f != nil)
	rd      int64   // read deadline
	wg      uintptr // pdReady, pdWait, G waiting for write or nil
	wt      timer   // write deadline timer
	wd      int64   // write deadline
	user    uint32  // user settable cookie
}
```

`runtime.pollDesc`包含自身类型的一个指针，用来保存下一个`runtime.pollDesc`的地址，go中有很多类似的实现，用来实现链表，可以减少数据结构的大小，所有的`runtime.pollDesc`保存在`runtime.pollCache`结构中，定义如下：

```go
type pollCache struct {
	lock  mutex
	first *pollDesc
	// PollDesc objects must be type-stable,
	// because we can get ready notification from epoll/kqueue
	// after the descriptor is closed/reused.
	// Stale notifications are detected using seq variable,
	// seq is incremented when deadlines are changed or descriptor is reused.
}
```

以tcp连接为例，分析一下Listen和Accept调用过程：

### Listen

```go
func Listen(network, address string) (Listener, error) {
	addrs, err := DefaultResolver.resolveAddrList(context.Background(), "listen", network, address, nil)
	if err != nil {
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: nil, Err: err}
	}
	var l Listener
	switch la := addrs.first(isIPv4).(type) {
	case *TCPAddr:
		l, err = ListenTCP(network, la)
	case *UnixAddr:
		l, err = ListenUnix(network, la)
	default:
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: la, Err: &AddrError{Err: "unexpected address type", Addr: address}}
	}
	if err != nil {
		return nil, err // l is non-nil interface containing nil pointer
	}
	return l, nil
}

// ListenTCP acts like Listen for TCP networks.
//
// The network must be a TCP network name; see func Dial for details.
//
// If the IP field of laddr is nil or an unspecified IP address,
// ListenTCP listens on all available unicast and anycast IP addresses
// of the local system.
// If the Port field of laddr is 0, a port number is automatically
// chosen.
func ListenTCP(network string, laddr *TCPAddr) (*TCPListener, error) {
	switch network {
	case "tcp", "tcp4", "tcp6":
	default:
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: laddr.opAddr(), Err: UnknownNetworkError(network)}
	}
	if laddr == nil {
		laddr = &TCPAddr{}
	}
	ln, err := listenTCP(context.Background(), network, laddr)
	if err != nil {
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: laddr.opAddr(), Err: err}
	}
	return ln, nil
}
```

当我们调用net.Listen("tcp",addr)时，根据address类型会命中ListenTCP函数去执行，ListenTCP函数很简单，基本处理逻辑都在listenTCP函数中，往下看

```go
func listenTCP(ctx context.Context, network string, laddr *TCPAddr) (*TCPListener, error) {
	fd, err := internetSocket(ctx, network, laddr, nil, syscall.SOCK_STREAM, 0, "listen")
	if err != nil {
		return nil, err
	}
	return &TCPListener{fd}, nil
}

func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string) (fd *netFD, err error) {
	if (runtime.GOOS == "windows" || runtime.GOOS == "openbsd" || runtime.GOOS == "nacl") && mode == "dial" && raddr.isWildcard() {
		raddr = raddr.toLocal(net)
	}
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr)
}
```

listenTCP函数，在此函数中终于见到了期望看到的fd字眼，跳到internetSocket函数，可以看到最终的fd是由socket函数产生的，继续

```go
// socket returns a network file descriptor that is ready for
// asynchronous I/O using the network poller.
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
	if fd, err = newFD(s, family, sotype, net); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}

	// This function makes a network file descriptor for the
	// following applications:
	//
	// - An endpoint holder that opens a passive stream
	//   connection, known as a stream listener
	//
	// - An endpoint holder that opens a destination-unspecific
	//   datagram connection, known as a datagram listener
	//
	// - An endpoint holder that opens an active stream or a
	//   destination-specific datagram connection, known as a
	//   dialer
	//
	// - An endpoint holder that opens the other connection, such
	//   as talking to the protocol stack inside the kernel
	//
	// For stream and datagram listeners, they will only require
	// named sockets, so we can assume that it's just a request
	// from stream or datagram listeners when laddr is not nil but
	// raddr is nil. Otherwise we assume it's just for dialers or
	// the other connection holders.

	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	if err := fd.dial(ctx, laddr, raddr); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

首先调用sysSocket函数生成一个非阻塞的socket，并且设置[close-on-exec](https://blog.csdn.net/justmeloo/article/details/40184039)标志，生成过程如下首先调用socketFunc，也就是Socket函数，再调用socket函数，继续调用RawSyscall，RawSyscall是由[汇编](http://blog.rootk.com/post/golang-asm.html)实现的，上述过程最终返回生成的socket对应的系统文件描述符（并不是netFD）。涉及到的代码如下

```go
// Wrapper around the socket system call that marks the returned file
// descriptor as nonblocking and close-on-exec.
func sysSocket(family, sotype, proto int) (int, error) {
	s, err := socketFunc(family, sotype|syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC, proto)
	// On Linux the SOCK_NONBLOCK and SOCK_CLOEXEC flags were
	// introduced in 2.6.27 kernel and on FreeBSD both flags were
	// introduced in 10 kernel. If we get an EINVAL error on Linux
	// or EPROTONOSUPPORT error on FreeBSD, fall back to using
	// socket without them.
	switch err {
	case nil:
		return s, nil
	default:
		return -1, os.NewSyscallError("socket", err)
	case syscall.EPROTONOSUPPORT, syscall.EINVAL:
	}

	// See ../syscall/exec_unix.go for description of ForkLock.
	syscall.ForkLock.RLock()
	s, err = socketFunc(family, sotype, proto)
	if err == nil {
		syscall.CloseOnExec(s)
	}
	syscall.ForkLock.RUnlock()
	if err != nil {
		return -1, os.NewSyscallError("socket", err)
	}
	if err = syscall.SetNonblock(s, true); err != nil {
		poll.CloseFunc(s)
		return -1, os.NewSyscallError("setnonblock", err)
	}
	return s, nil
}

socketFunc func(int, int, int) (int, error)  = syscall.Socket

func Socket(domain, typ, proto int) (fd int, err error) {
	if domain == AF_INET6 && SocketDisableIPv6 {
		return -1, EAFNOSUPPORT
	}
	fd, err = socket(domain, typ, proto)
	return
}

func socket(domain int, typ int, proto int) (fd int, err error) {
	r0, _, e1 := RawSyscall(SYS_SOCKET, uintptr(domain), uintptr(typ), uintptr(proto))
	fd = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}

func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2 uintptr, err Errno)

// func RawSyscall(trap, a1, a2, a3 uintptr) (r1, r2, err uintptr)
TEXT ·RawSyscall(SB),NOSPLIT,$0-56
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	$0, R10
	MOVQ	$0, R8
	MOVQ	$0, R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok1
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	RET
ok1:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	RET
```

有了真正的系统文件描述符之后紧接着通过newFD函数来初始化一个netFD

```go
func newFD(sysfd, family, sotype int, net string) (*netFD, error) {
	ret := &netFD{
		pfd: poll.FD{
			Sysfd:         sysfd,
			IsStream:      sotype == syscall.SOCK_STREAM,
			ZeroReadIsEOF: sotype != syscall.SOCK_DGRAM && sotype != syscall.SOCK_RAW,
		},
		family: family,
		sotype: sotype,
		net:    net,
	}
	return ret, nil
}
```

netFD创建好之后调用其listenStream方法，此方法中实现了socket的bind和listen，bind和listen最终也是走汇编代码，经过系统调用实现的。涉及到的代码如下

```go
func (fd *netFD) listenStream(laddr sockaddr, backlog int) error {
	if err := setDefaultListenerSockopts(fd.pfd.Sysfd); err != nil {
		return err
	}
	if lsa, err := laddr.sockaddr(fd.family); err != nil {
		return err
	} else if lsa != nil {
		if err := syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
			return os.NewSyscallError("bind", err)
		}
	}
	if err := listenFunc(fd.pfd.Sysfd, backlog); err != nil {
		return os.NewSyscallError("listen", err)
	}
	if err := fd.init(); err != nil {
		return err
	}
	lsa, _ := syscall.Getsockname(fd.pfd.Sysfd)
	fd.setAddr(fd.addrFunc()(lsa), nil)
	return nil
}

func (fd *netFD) listenStream(laddr sockaddr, backlog int) error {
	if err := setDefaultListenerSockopts(fd.pfd.Sysfd); err != nil {
		return err
	}
	if lsa, err := laddr.sockaddr(fd.family); err != nil {
		return err
	} else if lsa != nil {
		if err := syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
			return os.NewSyscallError("bind", err)
		}
	}
	if err := listenFunc(fd.pfd.Sysfd, backlog); err != nil {
		return os.NewSyscallError("listen", err)
	}
	if err := fd.init(); err != nil {
		return err
	}
	lsa, _ := syscall.Getsockname(fd.pfd.Sysfd)
	fd.setAddr(fd.addrFunc()(lsa), nil)
	return nil
}

//bind
func Bind(fd int, sa Sockaddr) (err error) {
	ptr, n, err := sa.sockaddr()
	if err != nil {
		return err
	}
	return bind(fd, ptr, n)
}

func bind(s int, addr unsafe.Pointer, addrlen _Socklen) (err error) {
	_, _, e1 := Syscall(SYS_BIND, uintptr(s), uintptr(addr), uintptr(addrlen))
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}

// func Syscall(trap int64, a1, a2, a3 int64) (r1, r2, err int64);
// Trap # in AX, args in DI SI DX R10 R8 R9, return in AX DX
// Note that this differs from "standard" ABI convention, which
// would pass 4th arg in CX, not R10.
TEXT	·Syscall(SB),NOSPLIT,$0-56
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	$0, R10
	MOVQ	$0, R8
	MOVQ	$0, R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET

//listen
listenFunc        func(int, int) error              = syscall.Listen

func Listen(s int, n int) (err error) {
	_, _, e1 := Syscall(SYS_LISTEN, uintptr(s), uintptr(n), 0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}

// func Syscall(trap int64, a1, a2, a3 int64) (r1, r2, err int64);
// Trap # in AX, args in DI SI DX R10 R8 R9, return in AX DX
// Note that this differs from "standard" ABI convention, which
// would pass 4th arg in CX, not R10.
TEXT	·Syscall(SB),NOSPLIT,$0-56
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	$0, R10
	MOVQ	$0, R8
	MOVQ	$0, R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok
	MOVQ	$-1, r1+32(FP)
	MOVQ	$0, r2+40(FP)
	NEGQ	AX
	MOVQ	AX, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok:
	MOVQ	AX, r1+32(FP)
	MOVQ	DX, r2+40(FP)
	MOVQ	$0, err+48(FP)
	CALL	runtime·exitsyscall(SB)
	RET
```

bind和listen执行完之后，netFD的init函数执行，初始化底层的epoll实例，并将fd添加到了epoll的事件队列中，最后设置析构函数供gc阶段调用，至此完成了整个Listen过程。代码如下

```go
func (fd *netFD) init() error {
	return fd.pfd.Init(fd.net, true)
}

// Init initializes the FD. The Sysfd field should already be set.
// This can be called multiple times on a single FD.
// The net argument is a network name from the net package (e.g., "tcp"),
// or "file".
// Set pollable to true if fd should be managed by runtime netpoll.
func (fd *FD) Init(net string, pollable bool) error {
	// We don't actually care about the various network types.
	if net == "file" {
		fd.isFile = true
	}
	if !pollable {
		fd.isBlocking = true
		return nil
	}
	return fd.pd.init(fd)
}

var serverInit sync.Once

func (pd *pollDesc) init(fd *FD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return syscall.Errno(errno)
	}
	pd.runtimeCtx = ctx
	return nil
}

//文件位置：internal/poll/fd_poll_runtime.go
//此函数在此文件处只有函数声明，一般这种类型的要么对应一个汇编，要么对应go:linkname
//此处对应后者
func runtime_pollOpen(fd uintptr) (uintptr, int)

//文件位置：runtime/netpoll.go
//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
	pd := pollcache.alloc()
	lock(&pd.lock)
	if pd.wg != 0 && pd.wg != pdReady {
		throw("runtime: blocked write on free polldesc")
	}
	if pd.rg != 0 && pd.rg != pdReady {
		throw("runtime: blocked read on free polldesc")
	}
	pd.fd = fd
	pd.closing = false
	pd.seq++
	pd.rg = 0
	pd.rd = 0
	pd.wg = 0
	pd.wd = 0
	unlock(&pd.lock)

	var errno int32
	errno = netpollopen(fd, pd)
	return pd, int(errno)
}

//文件位置：runtime/netpoll.go
func (c *pollCache) alloc() *pollDesc {
	lock(&c.lock)
	if c.first == nil {
		const pdSize = unsafe.Sizeof(pollDesc{})
		n := pollBlockSize / pdSize
		if n == 0 {
			n = 1
		}
		// Must be in non-GC memory because can be referenced
		// only from epoll/kqueue internals.
		mem := persistentalloc(n*pdSize, 0, &memstats.other_sys)
		for i := uintptr(0); i < n; i++ {
			pd := (*pollDesc)(add(mem, i*pdSize))
			pd.link = c.first
			c.first = pd
		}
	}
	pd := c.first
	c.first = pd.link
	unlock(&c.lock)
	return pd
}

//文件位置：runtime/netpoll_epoll.go
//epoll事件注册
func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

### Accept

TCPListner有两个暴露出来的Accept相关的函数，分别为Accept和AcceptTCP，这里主要从AcceptTCP分析，因为后者被tcpKeepAliveListener的Accept函数调用，而tcpKeepAliveListener的Accept方法就是常用的建立web项目时，http.ListenAndServe中会用到的Accept方法，如下：

```go
func (l *TCPListener) AcceptTCP() (*TCPConn, error) {
	if !l.ok() {
		return nil, syscall.EINVAL
	}
	c, err := l.accept()
	if err != nil {
		return nil, &OpError{Op: "accept", Net: l.fd.net, Source: nil, Addr: l.fd.laddr, Err: err}
	}
	return c, nil
}

```

可以看到实现逻辑基本都在accept方法内

```go
func (ln *TCPListener) accept() (*TCPConn, error) {
	fd, err := ln.fd.accept()
	if err != nil {
		return nil, err
	}
	return newTCPConn(fd), nil
}

func (fd *netFD) accept() (netfd *netFD, err error) {
	d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}

	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
	if err = netfd.init(); err != nil {
		fd.Close()
		return nil, err
	}
	lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
	netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
	return netfd, nil
}

// Accept wraps the accept network call.
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	if err := fd.readLock(); err != nil {
		return -1, nil, "", err
	}
	defer fd.readUnlock()

	if err := fd.pd.prepareRead(fd.isFile); err != nil {
		return -1, nil, "", err
	}
	for {
		s, rsa, errcall, err := accept(fd.Sysfd)
		if err == nil {
			return s, rsa, "", err
		}
		switch err {
		case syscall.EAGAIN:
			if fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		case syscall.ECONNABORTED:
			// This means that a socket on the listen
			// queue was closed before we Accept()ed it;
			// it's a silly error, so try again.
			continue
		}
		return -1, nil, errcall, err
	}
}
```

这里有两个函数比较重要，一个是accept，一个是fd.pd.waitRead，首先看accept的实现，最终还是会通过汇编进行系统调用。

```go
// Wrapper around the accept system call that marks the returned file
// descriptor as nonblocking and close-on-exec.
func accept(s int) (int, syscall.Sockaddr, string, error) {
	ns, sa, err := Accept4Func(s, syscall.SOCK_NONBLOCK|syscall.SOCK_CLOEXEC)
	// On Linux the accept4 system call was introduced in 2.6.28
	// kernel and on FreeBSD it was introduced in 10 kernel. If we
	// get an ENOSYS error on both Linux and FreeBSD, or EINVAL
	// error on Linux, fall back to using accept.
	switch err {
	case nil:
		return ns, sa, "", nil
	default: // errors other than the ones listed
		return -1, sa, "accept4", err
	case syscall.ENOSYS: // syscall missing
	case syscall.EINVAL: // some Linux use this instead of ENOSYS
	case syscall.EACCES: // some Linux use this instead of ENOSYS
	case syscall.EFAULT: // some Linux use this instead of ENOSYS
	}

	// See ../syscall/exec_unix.go for description of ForkLock.
	// It is probably okay to hold the lock across syscall.Accept
	// because we have put fd.sysfd into non-blocking mode.
	// However, a call to the File method will put it back into
	// blocking mode. We can't take that risk, so no use of ForkLock here.
	ns, sa, err = AcceptFunc(s)
	if err == nil {
		syscall.CloseOnExec(ns)
	}
	if err != nil {
		return -1, nil, "accept", err
	}
	if err = syscall.SetNonblock(ns, true); err != nil {
		CloseFunc(ns)
		return -1, nil, "setnonblock", err
	}
	return ns, sa, "", nil
}

// Accept4Func is used to hook the accept4 call.
var Accept4Func func(int, int) (int, syscall.Sockaddr, error) = syscall.Accept4

func Accept4(fd int, flags int) (nfd int, sa Sockaddr, err error) {
	var rsa RawSockaddrAny
	var len _Socklen = SizeofSockaddrAny
	nfd, err = accept4(fd, &rsa, &len, flags)
	if err != nil {
		return
	}
	if len > SizeofSockaddrAny {
		panic("RawSockaddrAny too small")
	}
	sa, err = anyToSockaddr(&rsa)
	if err != nil {
		Close(nfd)
		nfd = 0
	}
	return
}

// THIS FILE IS GENERATED BY THE COMMAND AT THE TOP; DO NOT EDIT

func accept4(s int, rsa *RawSockaddrAny, addrlen *_Socklen, flags int) (fd int, err error) {
	r0, _, e1 := Syscall6(SYS_ACCEPT4, uintptr(s), uintptr(unsafe.Pointer(rsa)), uintptr(unsafe.Pointer(addrlen)), uintptr(flags), 0, 0)
	fd = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}

func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2 uintptr, err Errno)

// func Syscall6(trap, a1, a2, a3, a4, a5, a6 uintptr) (r1, r2, err uintptr)
TEXT ·Syscall6(SB),NOSPLIT,$0-80
	CALL	runtime·entersyscall(SB)
	MOVQ	a1+8(FP), DI
	MOVQ	a2+16(FP), SI
	MOVQ	a3+24(FP), DX
	MOVQ	a4+32(FP), R10
	MOVQ	a5+40(FP), R8
	MOVQ	a6+48(FP), R9
	MOVQ	trap+0(FP), AX	// syscall entry
	SYSCALL
	CMPQ	AX, $0xfffffffffffff001
	JLS	ok6
	MOVQ	$-1, r1+56(FP)
	MOVQ	$0, r2+64(FP)
	NEGQ	AX
	MOVQ	AX, err+72(FP)
	CALL	runtime·exitsyscall(SB)
	RET
ok6:
	MOVQ	AX, r1+56(FP)
	MOVQ	DX, r2+64(FP)
	MOVQ	$0, err+72(FP)
	CALL	runtime·exitsyscall(SB)
	RET
```

再看waitRead方法

```go
func (pd *pollDesc) waitRead(isFile bool) error {
	return pd.wait('r', isFile)
}

func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}

func runtime_pollWait(ctx uintptr, mode int) int
```

又出现一个只有函数声明的函数，和上面出现过的一样，可以通过go:linkname找到具体实现，如下

```go
//go:linkname poll_runtime_pollWait internal/poll.runtime_pollWait
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
	err := netpollcheckerr(pd, int32(mode))
	if err != 0 {
		return err
	}
	// As for now only Solaris uses level-triggered IO.
	if GOOS == "solaris" {
		netpollarm(pd, mode)
	}
	for !netpollblock(pd, int32(mode), false) {
		err = netpollcheckerr(pd, int32(mode))
		if err != 0 {
			return err
		}
		// Can happen if timeout has fired and unblocked us,
		// but before we had a chance to run, timeout has been reset.
		// Pretend it has not happened and retry.
	}
	return 0
}

// returns true if IO is ready, or false if timedout or closed
// waitio - wait only for completed IO, ignore errors
func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	// set the gpp semaphore to WAIT
	for {
		old := *gpp
		if old == pdReady {
			*gpp = 0
			return true
		}
		if old != 0 {
			throw("runtime: double wait")
		}
		if atomic.Casuintptr(gpp, 0, pdWait) {
			break
		}
	}

	// need to recheck error states after setting gpp to WAIT
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, membarrier, load of rg/wg
	if waitio || netpollcheckerr(pd, mode) == 0 {
		gopark(netpollblockcommit, unsafe.Pointer(gpp), "IO wait", traceEvGoBlockNet, 5)
	}
	// be careful to not lose concurrent READY notification
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}
```

这里用了两个for循环，分别在调用netpollblock函数和netpollblock函数中，第一个for循环好理解，就是要一直等到io ready，第二个for循环用来等待io ready或者io wait，gpp为0，则会跳出循环，因为waitio在这里为true，所以会执行gopark函数，停靠当前G，设为当前G为waiting状态，并将G与M解绑，等到ioready后G讲重新变为Runnable状态并放入待运行队列等待被调度执行。

## 总结

通过上面的简单分析，我们大致可以理解整个流程，里面还涉及到很多具体细节以及go调度器相关的内容无法一一介绍，有兴趣的话可以直接查看源码。剩下的read，write大致相同，这里不再分析，最终都是通过netpoll的相关函数实现的，可以说整个核心实现都在netpoll.go这个文件中，外面只是进行了一些封装和状态的处理，至于G状态的变化的相关代码，可以自行搜索go调度器，已经有相当多博客进行过讲解了。

其实我们可以看到，知识就是个圈，缺少哪一块都串不起来，让我们一起努力，填补我们所缺失的部分吧，加油！
