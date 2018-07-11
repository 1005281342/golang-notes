# netpoll

hook_unix.go :

```go
    // Placeholders for socket system calls.
    socketFunc        func(int, int, int) (int, error)  = syscall.Socket
    connectFunc       func(int, syscall.Sockaddr) error = syscall.Connect
    listenFunc        func(int, int) error              = syscall.Listen
    getsockoptIntFunc func(int, int, int) (int, error)  = syscall.GetsockoptInt
```

这些 hook 主要是为了能够写测试，在测试代码中，socketFunc，connectFunc ... 都会被替换成测试专用函数，main_unix_test.go:

```go
func installTestHooks() {
    socketFunc = sw.Socket
    poll.CloseFunc = sw.Close
    connectFunc = sw.Connect
    listenFunc = sw.Listen
    poll.AcceptFunc = sw.Accept
    getsockoptIntFunc = sw.GetsockoptInt

    for _, fn := range extraTestHookInstallers {
        fn()
    }
}
```

用这种全局函数 hook，或者叫注册表的方式，可以实现类似于面向对象中的 interface 功能。不过因为不同平台提供的网络编程函数差别有些大，所以这里这些全局网络函数也就只是用来方便测试。

```go
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

```go
func (fd *netFD) init() error {
    return fd.pfd.Init(fd.net, true)
}
```

```mermaid
graph LR

ListenTCP --> listenTCP
listenTCP --> internetSocket
internetSocket --> socket
socket --> listenStream
```

```go
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

```go
func listenTCP(ctx context.Context, network string, laddr *TCPAddr) (*TCPListener, error) {
    fd, err := internetSocket(ctx, network, laddr, nil, syscall.SOCK_STREAM, 0, "listen")
    if err != nil {
        return nil, err
    }
    return &TCPListener{fd}, nil
}
```

```go
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string) (fd *netFD, err error) {
    if (runtime.GOOS == "windows" || runtime.GOOS == "openbsd" || runtime.GOOS == "nacl") && mode == "dial" && raddr.isWildcard() {
        raddr = raddr.toLocal(net)
    }
    family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
    return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr)
}
```

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
```