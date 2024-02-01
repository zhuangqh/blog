所谓容器引擎，比如 docker，最核心做的一个事情就是启动容器进程：从 containerd fork 出一个 containerd-shim 进程，再由 shim进程 fork 出真正的应用进程。

而 golang 最核心的概念就是 GMP线程模型，通常开发者是接触不到真正的系统线程的，取而代之的是更为轻量的 Go routine，系统线程则被 go runtime 封装管理起来。

例如，开发者可以通过以下代码，使用exec 系统库，fork 出一个 sleep 进程。这背后会调用 os.StartProcess 接口，这是你在 go里接触到的最底层的 fork 接口了，go runtime 会通过系统调用执行 clone，在子进程执行 exec，并处理好信号屏蔽，go routine 调度等逻辑。

```
func startTopProcess() error {
	cmd := exec.Command("sleep", "2s")
	var b bytes.Buffer
	cmd.Stdout = &b
	cmd.Stderr = &b
	if err := cmd.Run(); err != nil {
		return err
	}
	return nil
}
```

在单机上 50 并发创建容器时，go fork 单次耗时竟长达 100ms 。在追求极致的路上，单次容器创建花在这上面的100ms 时间已经很长了。
## ForkLock 之痛

深入去看 os.StartProcess 的代码，映入眼帘的就是这个全局读写锁ForkLock。在 forkexec 过程中会全程锁住，那 50 并发fork就变成串行的，明显会慢呀。

```
func forkExec(argv0 string, argv []string, attr *ProcAttr) (pid int, err error) {
...
	// Acquire the fork lock so that no other threads
	// create new fds that are not yet close-on-exec
	// before we fork.
	ForkLock.Lock()

	// Allocate child status pipe close on exec.
	if err = forkExecPipe(p[:]); err != nil {
		ForkLock.Unlock()
		return 0, err
	}

	// Kick off child.
	pid, err1 = forkAndExecInChild(argv0p, argvp, envvp, chroot, dir, attr, sys, p[1])
	if err1 != 0 {
		Close(p[0])
		Close(p[1])
		ForkLock.Unlock()
		return 0, Errno(err1)
	}
	ForkLock.Unlock()
```

查阅注释发现，forklock 的引入是为了避免 clone 过程中，子进程错误继承了父进程的 fd。比如 pipe 的写端被子进程继承，又没有关闭，那么父进程的读端就可能卡住。虽然 linux 有 CloseOnExec的 flag，能让打开的 fd，在被子进程继承后，由内核自动关闭。但是 unix 规范里并没有这个定义，比如 macos，或者 linux 2.6 以下版本，都是不支持在 open fd 的时候就设置 closeonexec 的。

所以最终的方案就变成了现在这样，在 openfd 的时候会拿 forklock 的读锁，fork 的时候拿 forklock 的写锁，以此实现两个操作的互斥。但这里带来了另一个问题，fork 和 fork 间也变成互斥的，所以时延高了。

我们做了个实验，将这两个锁反转，让 openfd 拿写锁，fork 拿读锁。fork 的时延立马下来了，50 并发 max 只要 20ms，说明我们的分析是对的。但这也解决不了问题，open fd 间变成互斥的了。

## 异或锁

大致明确了问题所在，查阅社区的 issue，其实很早就有相关的讨论。最早是 gitlab 的研发团队在 2013 年提出来的，他们的优化方向是将fork 换成了更轻量的CLONE_VFORK 和 CLONE_VM，带来了非常明显的性能提升，但是代码直到 2018 年才合入。详情请看 [How a fix in Go 1.9 sped up our Gitaly service by 30x | GitLab](https://about.gitlab.com/blog/2018/01/23/how-a-fix-in-go-19-sped-up-our-gitaly-service-by-30x/)

18 年后陆陆续续又有几个关于 forklock 的零碎讨论，后面又不了了之。我发现这个问题后，又激活了这块的讨论。

其实在 linux 高版本下是可以去掉这个 lock 的，现在的 forklock 是一种兼容性考虑，本质的需求是将 openfd 和 fork 互斥，但是不应该让 openfd之间或者 fork 之间互斥，这有点类似 “异或”的逻辑。

最后的方案是这样的：fork 继续用写锁，但是引入一个计数器，fork 获取写锁的时候，如果发现写锁已经被获取，则计数加一，直接成功。释放写锁的时候，计数减一，如果计数为 0，则真正释放写锁。同时会检查读锁的等待情况，让出写锁，防止读锁饿死。
![[gofork.png]]
这样子 fork 和 openfd 能交替执行，相同的操作是可以并行的。经过优化后，50 并发下 fork max 只有 20ms+。并发越高，收益会越明显。
