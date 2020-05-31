# 源码阅读笔记 Golang  与 epoll 

```
golang标准网络库通信模型 epoll + 协程
文件描述符是非阻塞的 其实阻塞在调用者的G上
```

疑问

```
系统调用accpet 返回的文件描述符 如何和epoll相互联系
为什么一个新的连接 就有一个accept 返回 而老的连接并不建立accept
应该是内核通过判断tcp的包 可以判断出来连接队列里面是否有这个client端口的连接 如果有 就不需要建立新的连接 而是去直接发送给epoll中断 epoll中断就会恢复阻塞的conn连接协程
```



## Listen

建立文件描述符：

```
1. 建立一个socket描述符 设置为Noblock 并返回FD （创建套接字）
s, err := socketFunc(family, sotype, proto)
syscall.SetNonblock(s, true) 设置为非阻塞
对应的内核函数：int socket(int domain, int type, int protocal) 
2. system call bind socketfd （绑定套接字到一个IP地址和端口上）
_, _, e1 := syscall(funcPC(libc_bind_trampoline), uintptr(s), uintptr(addr), 
uintptr(addrlen))
对应的内核函数：int bind(int fd, sockaddr *addr, socklen_t len)
3. system call listen socketfd （将套接字设置为监听模式）  内核会帮助监听 
_, _, e1 := syscall(funcPC(libc_listen_trampoline), uintptr(s), uintptr(backlog), 0)
对应的内核函数：int listen(int socketfd, int backlog)

```

包装操作系统的文件描述符到netFD 其中的Sysfd 为上面建立的s：

```
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

netFD数据结构：

```
type netFD struct {
    pfd poll.FD
 
    // immutable until Close
    family      int
    sotype      int
    isConnected bool // handshake completed or use of association with peer
    net         string
    laddr       Addr
    raddr       Addr
}
```

```
// FD is a file descriptor. The net and os packages use this type as a
// field of a larger type representing a network connection or OS file.
type FD struct {
	// Lock sysfd and serialize access to Read and Write methods.
	fdmu fdMutex

	// System file descriptor. Immutable until Close.
	Sysfd int

	// I/O poller.
	pd pollDesc
}
```

netFD是描述符的包装  

netFD.pfd.Sysfd 是操作系统返回的文件描述符

netFD.pfd.pd 内部封装了协程信息 这个变量会和epoll线程通信  

```
type pollDesc struct {
	runtimeCtx uintptr
}
```

pollDesc Init 实际上初始化了runtime包中的pollDesc

```
func (pd *pollDesc) init(fd *FD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		if ctx != 0 {
			runtime_pollUnblock(ctx)
			runtime_pollClose(ctx)
		}
		return errnoErr(syscall.Errno(errno))
	}
	pd.runtimeCtx = ctx
	return nil
}
```

runtime包中的pollDesc

```
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
   // proceed w/o taking the lock. So closing, everr, rg, rd, wg and wd are manipulated
   // in a lock-free way by all operations.
   // NOTE(dvyukov): the following code uses uintptr to store *g (rg/wg),
   // that will blow up when GC starts moving objects.
   lock    mutex // protects the following fields
   fd      uintptr
   closing bool
   everr   bool    // marks event scanning error happened
   user    uint32  // user settable cookie
   rseq    uintptr // protects from stale read timers
   rg      uintptr // pdReady, pdWait, G waiting for read or nil
   rt      timer   // read deadline timer (set if rt.f != nil)
   rd      int64   // read deadline
   wseq    uintptr // protects from stale write timers
   wg      uintptr // pdReady, pdWait, G waiting for write or nil
   wt      timer   // write deadline timer
   wd      int64   // write deadline
}
```

其中rt和wt分别是读写定时器 来防止读写超时

fd为系统文件描述符指针

rg 保存了用户态操作pollDecs的读协程地址

wg保存了用户态操作pollDesc的写协程地址

rg,wg默认是0，rg为pdReady表示读就绪，可以将协程恢复，为pdWait表示读阻塞，协程将要被挂起。wg也是如此

我们在在用户态协程调用read阻塞时rg就被设置为该读协程，当内核态epoll_wait检测read就绪后就会通过rg找到这个协程让后恢复运行

1.serverInit.Do(runtime_pollServerInit) 对应了epoll的create和epollctl  把netpollBreakRd中断添加到epoll句柄中

epoll描述符会在整个程序的生命周期中使用

```
根据系统的不同 调用systemcall netpollinit
因为调用的时候使用了sync.Once 所以只会被初始化一次 不用担心全局变量的问题
var (
	epfd int32 = -1 // epoll descriptor
	netpollBreakRd, netpollBreakWr uintptr // for netpollBreak
)
netpollinit{
    epfd = epollcreate1(_EPOLL_CLOEXEC)
    //表示为监听读的事件
    	r, w, errno := nonblockingPipe()
  	if errno != 0 {
		println("runtime: pipe failed with", -errno)
		throw("runtime: pipe failed")
	  }
    ev := epollevent{
		events: _EPOLLIN,
  	}
  	//把epoll和r绑定起来 r代表了一个文件描述符
    errno = epollctl(epfd, _EPOLL_CTL_ADD, r, &ev)
    //epoll_create1(int flags); 返回值: 若成功返回一个大于0的值，表示epoll实例；若返回-1表示出错
    //epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 返回值: 若成功返回0；若返回-1表示出错
    //第一个参数 epfd 是刚刚调用 epoll_create 创建的 epoll 实例描述字，可以简单理解成是 epoll 句柄。
    // 第二个参数表示增加还是删除一个监控事件，它有三个选项可供选择：EPOLL_CTL_ADD： 向 epoll 实例注册文件描述符对应的事件；EPOLL_CTL_DEL：向 epoll 实例删除文件描述符对应的事件；EPOLL_CTL_MOD： 修改文件描述符对应的事件。
    //第三个参数是注册的事件的文件描述符，比如一个监听套接字。
    //第四个参数表示的是注册的事件类型，并且可以在这个结构体里设置用户需要的数据，其中最为常见的是使用联合结构里的 fd 字段，表示事件所对应的文件描述符
    //把r w 赋值给netpollBreakRd, netpollBreakWr
    	netpollBreakRd = uintptr(r)
    	netpollBreakWr = uintptr(w)
}

```

2. nonblockingPipe() 创建一个用于通信的管道 管道为我们提供了中断多路复用等待文件描述符中事件的方法

```
func netpollBreak() {
	for {
		var b byte
		n := write(netpollBreakWr, unsafe.Pointer(&b), 1)
		if n == 1 {
			break
		}
		if n == -_EINTR {
			continue
		}
		if n == -_EAGAIN {
			return
		}
	}
}
```

3.runtime_pollOpen(uintptr(fd.Sysfd))

```
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
	pd.everr = false
	pd.rseq++
	pd.rg = 0
	pd.rd = 0
	pd.wseq++
	pd.wg = 0
	pd.wd = 0
	unlock(&pd.lock)

	var errno int32
	errno = netpollopen(fd, pd)
	return pd, int(errno)
}
 
 func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}

```

内存分配

```
 const pollBlockSize = 4 * 1024
 func (c *pollCache) alloc() *pollDesc {
	lock(&c.lock)
	if c.first == nil {
		const pdSize = unsafe.Sizeof(pollDesc{})
		//golang内存分配策略 小内存一次向上级申请多个 并形成链表 这次申请的返回c.first
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

 Wrapper around sysAlloc that can allocate small chunks
 func persistentalloc(size, align uintptr, sysStat *uint64) unsafe.Pointer {
	var p *notInHeap
	systemstack(func() {
		p = persistentalloc1(size, align, sysStat)
	})
	return unsafe.Pointer(p)
}
```

每次调用该结构体都会返回链表头还没有被使用的  pollDesc 这种批量初始化的做法能够增加网络轮询器的吞吐量。Go 语言运行时会调用 free 方法释放已经用完的pollDesc 结构，它会直接将结构体插入链表的最前面

```
func (c *pollCache) free(pd *pollDesc) {
	lock(&c.lock)
	pd.link = c.first
	c.first = pd
	unlock(&c.lock)
}
```



## Accept

关键代码

```
func (fd *netFD) accept() (netfd *netFD, err error) {
  //阻塞 不是IO阻塞 是协程阻塞
  // 会调用 int accept(*int listensockfd,strcuct sockaddr * cliaddr socklen_t *addrlen)
  d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}
  //根据监听文件描述符建立连接文件描述符
	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
	if err = netfd.init(); err != nil {
		netfd.Close()
		return nil, err
	}
	lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
	netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
	return netfd, nil
}

```



监听fd.Sysfd 等待可读事件的到来 当accept出现错误时，需要判断err类型，如果是EAGAIN说明当前没有连接到来，就调用waitRead等待连接

```
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	for {
	  //这里不会阻塞 因为把socket设置成了非阻塞 accept(fd.Sysfd)是调用系统调用
		s, rsa, errcall, err := accept(fd.Sysfd)
		if err == nil {
			return s, rsa, "", err
		}
		switch err {
		//非阻塞没有数据 会直接返回EAGAIN错误
		case syscall.EAGAIN:
			if fd.pd.pollable() {
			  //这里才会真正的阻塞
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

fd.pd.waitRead的调用了runtime_pollWait

```
func (pd *pollDesc) wait(mode int, isFile bool) error {
	if pd.runtimeCtx == 0 {
		return errors.New("waiting for unsupported file type")
	}
	res := runtime_pollWait(pd.runtimeCtx, mode)
	return convertErr(res, isFile)
}

```

runtime_pollWait 实现当前协程的阻塞

运行在内核中 轮询调用netpollblock 当返回true的时候循环退出 当前协程就会继续运行

```
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
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
```

netpollblock

netpollblock内部根据读模式还是写模式，获取pollDesc成员变量的读协程或者写协程地址，然后判断其状态是否为pdReady，这里要详细说一下，golang阻塞一个用户态协程是要将其状态设置为0(正在运行)或者pdWait(阻塞)，这里为0，所以逻辑继续往下走，之后做了一个原子操作将gpp设置为pdWait状态，接着根据这个状态，执行gopark函数，阻塞住用户态协程。当内核想激活用户协程时gopark会返回，然后该函数判断gpp是否为pdReady，从而激活用户态协程

```
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
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
	// be careful to not lose concurrent READY notification
	old := atomic.Xchguintptr(gpp, 0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}
```

gopark将用户态协程放在等待队列中，然后调用mcall触发汇编代码。之后会检测调用unlockf函数，如果unlockf返回false则说明可以解锁用户态协程了

```
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}

```

atomic.Casuintptr(gpp, 0, pdWait) golang的cas操作

```
atomic.Casuintptr(gpp, 0, pdWait)
compare and swap
内存值 旧的预期值A 要修改的新值
CPU去更新一个值，但如果想改的值不再是原来的值，操作就失败，因为很明显，有其它操作先改变了这个值

就是指当两者进行比较时，如果相等，则证明共享数据没有被修改，替换成新值，然后继续往下运行；如果不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作

```

激活挂起的用户协程

netpoll函数 感觉是原理是内核中的文件描述符epoll有事件到来 会调用这个函数 会调用epoll_wait函数 然后读取其返回的用户数据（就是netpollopen塞进去的pollDesc指针）

该方法有单独一个常驻的goroutine在高效快速的轮询检测执行，当有网络包收到后，每次都会取最多128个事件，然后在ev.data里就是当初注册事件时的pollDesc，也就找到了要唤醒的G，唤醒之

sysmon方法就是我们说的监控任务，它没有和任何的P(逻辑处理器)进行绑定，而是通过自身改变睡眠时间和时间间隔来一直循环下去(代码位于runtime/proc.go)。 golang中所有文件描述符都被设置成非阻塞的，某个goroutine进行网络io操作，读或者写文件描述符，如果此刻网络io还没准备好，则这个goroutine会被放到系统的等待队列中，这个goroutine失去了运行权，但并不是真正的整个系统“阻塞”于系统调用，后台还有一个poller会不停地进行poll，所有的文件描述符都被添加到了这个poller中的，当某个时刻一个文件描述符准备好了，poller就会唤醒之前因它而阻塞的goroutine，于是goroutine重新运行起来



```
// netpoll checks for ready network connections.
// Returns list of goroutines that become runnable.
// delay < 0: blocks indefinitely 无限期等待
// delay == 0: does not block, just polls 非阻塞轮询
// delay > 0: block for up to that many nanoseconds 阻塞特定时间轮询网络
func netpoll(delay int64) gList {
	if epfd == -1 {
		return gList{}
	}
	var waitms int32
	if delay < 0 {
		waitms = -1
	} else if delay == 0 {
		waitms = 0
	} else if delay < 1e6 {
		waitms = 1
	} else if delay < 1e15 {
		waitms = int32(delay / 1e6)
	} else {
		// An arbitrary cap on how long to wait for a timer.
		// 1e9 ms == ~11.5 days.
		waitms = 1e9
	}
	var events [128]epollevent
retry:
	n := epollwait(epfd, &events[0], int32(len(events)), waitms)
	if n < 0 {
		if n != -_EINTR {
			println("runtime: epollwait on fd", epfd, "failed with", -n)
			throw("runtime: netpoll failed")
		}
		// If a timed sleep was interrupted, just return to
		// recalculate how long we should sleep now.
		if waitms > 0 {
			return gList{}
		}
		goto retry
	}
	var toRun gList
	for i := int32(0); i < n; i++ {
		ev := &events[i]
		if ev.events == 0 {
			continue
		}

		if *(**uintptr)(unsafe.Pointer(&ev.data)) == &netpollBreakRd {
			if ev.events != _EPOLLIN {
				println("runtime: netpoll: break fd ready for", ev.events)
				throw("runtime: netpoll: break fd ready for something unexpected")
			}
			if delay != 0 {
				// netpollBreak could be picked up by a
				// nonblocking poll. Only read the byte
				// if blocking.
				var tmp [16]byte
				read(int32(netpollBreakRd), noescape(unsafe.Pointer(&tmp[0])), int32(len(tmp)))
			}
			continue
		}

		var mode int32
		if ev.events&(_EPOLLIN|_EPOLLRDHUP|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'r'
		}
		if ev.events&(_EPOLLOUT|_EPOLLHUP|_EPOLLERR) != 0 {
			mode += 'w'
		}
		if mode != 0 {
			pd := *(**pollDesc)(unsafe.Pointer(&ev.data))
			pd.everr = false
			if ev.events == _EPOLLERR {
				pd.everr = true
			}
			netpollready(&toRun, pd, mode)
		}
	}
	return toRun
}

```

处理的事件总共包含两种，一种是调用netpollbreak 函数触发的事件，该函数的作用是中断网络轮询器；另一种是其他文件描述符的正常读写事件，对于这些事件，我们会交给netpollread处理

拿到pollDesc指针后 调用netpollready将曾经挂起的协程放入gList中，然后返回该列表



netpollunblock函数修改pd所在协程的状态为0，表示可运行状态

```
func netpollready(toRun *gList, pd *pollDesc, mode int32) {
	var rg, wg *g
	if mode == 'r' || mode == 'r'+'w' {
		rg = netpollunblock(pd, 'r', true)
	}
	if mode == 'w' || mode == 'r'+'w' {
		wg = netpollunblock(pd, 'w', true)
	}
	if rg != nil {
		toRun.push(rg)
	}
	if wg != nil {
		toRun.push(wg)
	}
}

func netpollunblock(pd *pollDesc, mode int32, ioready bool) *g {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	for {
		old := *gpp
		if old == pdReady {
			return nil
		}
		if old == 0 && !ioready {
			// Only set READY for ioready. runtime_pollWait
			// will check for timeout/cancel before waiting.
			return nil
		}
		var new uintptr
		if ioready {
			new = pdReady
		}
		if atomic.Casuintptr(gpp, old, new) {
			if old == pdReady || old == pdWait {
				old = 0
			}
			return (*g)(unsafe.Pointer(old))
		}
	}
}
```

在runtime/proc.go中findrunnable会判断是否初始化epoll，如果初始化了则调用netpoll，从而获取glist，然后traceGoUnpark激活挂起的协程

```
func findrunnable() (gp *g, inheritTime bool) {
    _g_ := getg()
    //...
    if netpollinited() && atomic.Load(&netpollWaiters) > 0 && atomic.Load64(&sched.lastpoll) != 0 {
        if list := netpoll(0); !list.empty() { // non-blocking
            gp := list.pop()
            injectglist(&list)
            casgstatus(gp, _Gwaiting, _Grunnable)
            if trace.enabled {
                traceGoUnpark(gp, 0)
            }
            return gp, false
        }
    }
    //...
}
```

总结

```
内核会调用netpoll,netpoll会调用epoll_wait,epoll_wait会返回一个epoll事件和用户数据
用户数据就是一个pollDesc指针 pollDesc数据的rd表示了阻塞的读协程
```

数据的处理

1. newFD(d, fd.family, fd.sotype, fd.net) 其中的d是系统调用返回的连接文件描述符
2. 调用 runtime_pollOpen(uintptr(fd.Sysfd)), 上文已经分析过 把fd加入到epoll中
3. 返回一个newTCPConn(fd)

读数据 syscall.Read 如果返回为EAGAIN 同样调用fd.pd.waitRead 将当前的协程挂起 . 当内核的epollin事件到达（意味着事件到来） 内核将该协程变为runable 继续运行到continue  重新读取 必然会有数据

```
func (fd *FD) Read(p []byte) (int, error) {
   if err := fd.readLock(); err != nil {
      return 0, err
   }
   defer fd.readUnlock()
   if len(p) == 0 {
      // If the caller wanted a zero byte read, return immediately
      // without trying (but after acquiring the readLock).
      // Otherwise syscall.Read returns 0, nil which looks like
      // io.EOF.
      // TODO(bradfitz): make it wait for readability? (Issue 15735)
      return 0, nil
   }
   if err := fd.pd.prepareRead(fd.isFile); err != nil {
      return 0, err
   }
   if fd.IsStream && len(p) > maxRW {
      p = p[:maxRW]
   }
   for {
      n, err := syscall.Read(fd.Sysfd, p)
      if err != nil {
         n = 0
         if err == syscall.EAGAIN && fd.pd.pollable() {
            if err = fd.pd.waitRead(fd.isFile); err == nil {
               continue
            }
         }

         // On MacOS we can see EINTR here if the user
         // pressed ^Z.  See issue #22838.
         if runtime.GOOS == "darwin" && err == syscall.EINTR {
            continue
         }
      }
      err = fd.eofError(n, err)
      return n, err
   }
}
```

## 超时管理

方法里被set的超时时间实质通过`addtimer`方法被添加到了runtime下的全局timer管理器里，这个timer管理器是用户态的，由被拆分出来的多个堆结构管理，当超时后会通过`netpollReadDeadline`、`netpollDeadline`、`netpollWriteDeadline`几个方法将休眠的goroutine唤醒。

因此可以简单理解Go的TCP超时机制就是通过单独实现了一个用户态的全局的timer管理器来主动唤醒的





## 网络轮询器 （epoll机制）

```
golang针对每个socket文件描述符建立的轮询机制 
golang的runtime在调度goroutine或者GC完成之后或者指定时间 调用epoll_wait获取所有产生IO事件的socket文件描述符
当然在runtime轮询之前 需要将socket文件描述符和当前的goroutine的相关信息加入epoll维护的数据结构中 并挂起当前goroutine 当IO就绪后 通过epoll返回的文件描述符和其中附带的goroutine信息 重新恢复goroutine
```

//go1.14.1 src/runtime/netpoll.go

- 轮询器的5个函数

```
初始化轮询器
func netpollinit()
边缘触发pd
func netpollopen(fd uintptr, pd * pollDesc) int32

func netpoll(delta int) gList
func netpollBreak()
func netpollIsPollDescriptor(fd uintptr) boll

```

- 

## 其他

- epoll标准伪代码

```
#include "lib/common.h"

#define MAXEVENTS 128

int main(int argc, char **argv) {
    int listen_fd, socket_fd;
    int n, i;
    int efd;
    struct epoll_event event;
    struct epoll_event *events;

    listen_fd = tcp_nonblocking_server_listen(SERV_PORT);

    efd = epoll_create1(0);
    if (efd == -1) {
        error(1, errno, "epoll create failed");
    }

    event.data.fd = listen_fd;
    event.events = EPOLLIN | EPOLLET;
    if (epoll_ctl(efd, EPOLL_CTL_ADD, listen_fd, &event) == -1) {
        error(1, errno, "epoll_ctl add listen fd failed");
    }

    /* Buffer where events are returned */
    events = calloc(MAXEVENTS, sizeof(event));

    while (1) {
        n = epoll_wait(efd, events, MAXEVENTS, -1);
        printf("epoll_wait wakeup\n");
        for (i = 0; i < n; i++) {
            if ((events[i].events & EPOLLERR) ||
                (events[i].events & EPOLLHUP) ||
                (!(events[i].events & EPOLLIN))) {
                fprintf(stderr, "epoll error\n");
                close(events[i].data.fd);
                continue;
            } else if (listen_fd == events[i].data.fd) {
                struct sockaddr_storage ss;
                socklen_t slen = sizeof(ss);
                int fd = accept(listen_fd, (struct sockaddr *) &ss, &slen);
                if (fd < 0) {
                    error(1, errno, "accept failed");
                } else {
                    make_nonblocking(fd);
                    event.data.fd = fd;
                    event.events = EPOLLIN | EPOLLET; //edge-triggered
                    if (epoll_ctl(efd, EPOLL_CTL_ADD, fd, &event) == -1) {
                        error(1, errno, "epoll_ctl add connection fd failed");
                    }
                }
                continue;
            } else {
                socket_fd = events[i].data.fd;
                printf("get event on socket fd == %d \n", socket_fd);
                while (1) {
                    char buf[512];
                    if ((n = read(socket_fd, buf, sizeof(buf))) < 0) {
                        if (errno != EAGAIN) {
                            error(1, errno, "read error");
                            close(socket_fd);
                        }
                        break;
                    } else if (n == 0) {
                        close(socket_fd);
                        break;
                    } else {
                        for (i = 0; i < n; ++i) {
                            buf[i] = rot13_char(buf[i]);
                        }
                        if (write(socket_fd, buf, n) < 0) {
                            error(1, errno, "write error");
                        }
                    }
                }
            }
        }
    }

    free(events);
    close(listen_fd);
}
```

- epoll 内核数据

```
typedef union epoll_data {
     void        *ptr;
     int          fd;
     uint32_t     u32;
     uint64_t     u64;
 } epoll_data_t;

 struct epoll_event {
     uint32_t     events;      /* Epoll events */
     epoll_data_t data;        /* User data variable */
 };
```

- accept 惊群

```
  惊群效应也有人叫做雷鸣群体效应，不过叫什么，简言之，惊群现象就是多进程（多线程）在同时阻塞等待同一个事件的时候（休眠状态），如果等待的这个事件发生，那么他就会唤醒等待的所有进程（或者线程），但是最终却只可能有一个进程（线程）获得这个时间的“控制权”，对该事件进行处理，而其他进程（线程）获取“控制权”失败，只能重新进入休眠状态，这种现象和性能浪费就叫做惊群
上下文切换（context  switch）过高会导致cpu像个搬运工，频繁地在寄存器和运行队列之间奔波，更多的时间花在了进程（线程）切换，而不是在真正工作的进程（线程）上面。直接的消耗包括cpu寄存器要保存和加载（例如程序计数器）、系统调度器的代码需要执行。间接的消耗在于多核cache之间的共享数据

```



## Refer

```
https://www.cnblogs.com/findumars/p/5624958.html
https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/
https://www.jianshu.com/p/3ff0751dfa04
```

