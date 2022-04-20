- [从源代码角度看epoll在Go中的使用](https://zhuanlan.zhihu.com/p/108509080?utm_source=qq)
- [从源代码角度看epoll在Go中的使用](https://zhuanlan.zhihu.com/p/109559267)
- [网络轮询器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/)
- [Go netpoller 原生网络模型之源码全面揭秘](https://mp.weixin.qq.com/s?__biz=MzAxMTA4Njc0OQ==&mid=2651443085&idx=3&sn=2c1ed8474bc7fed68b519ce9e5f5e0b0&scene=21#wechat_redirect)
- [深入理解select、poll和epoll及区别](https://blog.csdn.net/wteruiycbqqvwt/article/details/90299610)
- [select、poll、epoll之间的区别总结[整理] + 知乎大神解答](https://blog.csdn.net/qq546770908/article/details/53082870?spm=1001.2101.3001.6650.1&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_aa&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1.pc_relevant_aa&utm_relevant_index=1)

Go提供了功能完备的标准网络库：net包，net包的实现相当之全面，http\tcp\udp均有实现且对用户提供了简单友好的使用接口。在Linux系统上Go使用了epoll来实现net包的核心部分

对于服务端程序而言，主要流程是Listen->Accept->Send/Write，客户端主要流程Connect->Send/Write，本文以这两个流程深入分析net包在Go中是如何实现的。

### Listen
监听方法是在ListenConfig结构中的Listen方法实现的(net/dial.go)：

$默认的tcp连接是15s

```
func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
	addrs, err := DefaultResolver.resolveAddrList(ctx, "listen", network, address, nil)
	if err != nil {
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: nil, Err: err}
	}
	sl := &sysListener{
		ListenConfig: *lc,
		network:      network,
		address:      address,
	}
	var l Listener
	la := addrs.first(isIPv4)
	switch la := la.(type) {
	case *TCPAddr:
		l, err = sl.listenTCP(ctx, la)
	case *UnixAddr:
		l, err = sl.listenUnix(ctx, la)
	default:
		return nil, &OpError{Op: "listen", Net: sl.network, Source: nil, Addr: la, Err: &AddrError{Err: "unexpected address type", Addr: address}}
	}
	if err != nil {
		return nil, &OpError{Op: "listen", Net: sl.network, Source: nil, Addr: la, Err: err} // l is non-nil interface containing nil pointer
	}
	return l, nil
}
```
$在`Listen`函数实现中，两个关键流程是`DefaultResolver.resolveAddrList`和`listenTCP/listenUnix`。
其中`DefaultResolver.resolveAddrList`使用提示根据协议的不同解析 addr 并返回地址列表。当 error 为 nil 时，结果至少包含一个地址。
在`listenTCP/listenUnix`中,以前者为例，首先创建[Socket(什么是Socket)](https://blog.csdn.net/qq_35745940/article/details/118461845)

在这里有创建[fd(什么是fd)](https://blog.csdn.net/michaelwoshi/article/details/105759976)
```
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
    // 调用各平台对应的socket api创建socket
    s, err := sysSocket(family, sotype, proto)
    if err != nil {
        return nil, err
    }
    // 设置socket选项
    if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }
    // 创建fd
    if fd, err = newFD(s, family, sotype, net); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }

    // 监听
    if laddr != nil && raddr == nil {
        switch sotype {
        case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
            // TCP
            if err := fd.listenStream(laddr, listenerBacklog(), ctrlFn); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        case syscall.SOCK_DGRAM:
            // UDP
            if err := fd.listenDatagram(laddr, ctrlFn); err != nil {
                fd.Close()
                return nil, err
            }
            return fd, nil
        }
    }
    // 发起连接，非listen socket会走到这里来
    if err := fd.dial(ctx, laddr, raddr, ctrlFn); err != nil {
        fd.Close()
        return nil, err
    }
    return fd, nil
}
```

socket函数主要流程：新建socket-->设置socket option-->创建fd-->进入监听逻辑。sysSocket根据平台有不同实现，windows实现在socket_windows.go中，linux实现则在sock_cloexec.go中，本文重点分析在linux平台上的实现(net/sock_cloexec.go):
`listenStream`主要做了以下几件事：
1. 检查未完成连接和已完成连接两个队列是否超出系统预设。
2. 调用socket bind。
3. 调用socket listen。
4. 初始化fd。
5. 调用socket getsockname。

Linux平台上，系统提供了[五种IO模型](https://blog.csdn.net/qq_42052956/article/details/111994266)

阻塞IO、非阻塞IO、IO多路复用、信号驱动IO和异步IO，对应到内核层面提供的用户接口即select、poll和epoll。Go net包是基于epoll进行封装的，基本模型结合了epoll和Go语言的优势：epoll+goroutine，这样达到异步且高并发。




