# Make syzkaller a state-based guided fuzzer

## Goal
Make the syzkaller as a kernel-state-awareness fuzzer or state-based guided fuzzer. The fuzzer should collect the progs which hit the same code coverage but with different kernel data state. Currently syzkaller only collect coverage information.  

I wonder if it's effective that make syzkaller more kernel-state-awareness.
I finish collecting the some socket state and feedback to syzkaller currently. Use the coverage signal interface.  

## What is the improvement

### Types of branch  
1. A condition directly determined by kernel function parameters. Without any impact from other syscalls. In other words, it can be easily covered by mutating a single syscall.
In this [example](https://elixir.bootlin.com/linux/v4.20/source/net/ipv4/tcp.c#L1188), msg_flags is a branch-relative parameters which specified by input of syscall 'sendmsg'.

2. A condition determined by kernel function parameters' historical state.
In first [example](https://elixir.bootlin.com/linux/v4.20/source/net/ipv4/tcp.c#L1189), sk_state is a historical state which can be changed after calling listen/connect... In second [one](https://elixir.bootlin.com/linux/v4.20/source/net/ipv4/tcp.c#L1231), repair_queue is changed after calling setsockopt.

3. A condition determined by a local variable that can be changed in kernel function.
In this [example](https://elixir.bootlin.com/linux/v4.20/source/net/ipv4/tcp.c#L1346), local variable merge is changed by this [line](https://elixir.bootlin.com/linux/v4.20/source/net/ipv4/tcp.c#L1330).  

### Which is not easy to be covered

First one can be easily covered by syzkaller if powerful syscalls scriptions have been written. Collect function's input as feedbacl helps little coverage. Even though there are several paramters.

The second one, need time to explore, especial nested condition. For example, in tcp-ipv6 testing, we should not assume that setsockopt/getsockopt/close/shutdown... have no impact on calling sendmsg. Enable too much syscalls will waste much time on exploring their coverage( Original syzkaller do this). Actually, it has no impact on sendmsg unless it trigger a special state for sendmsg. Collecting useful state before calling sendmsg, without collecting any coverage signal of other kernel functions could be more effective. I implement this by using ebpf feedback and coverage filter. And it get a great improvement.

The third one need time to explore too. But it can't be solved by using ebpf feedback. ebpf know nothing about the internal of kernel function. I think fault-injection is a way that can help it. Kernel have a general framework to do function-ret-fault-injection. But it can't attach to inline function. ebpf use this framework also. It has much work to do with supporting a specified fault-injection in syzkaller.

### Result
It got a great improvement in the second type of branch. [Here]() is a example for tcp-ipv6. It can cover some branch with constrain like "tp->repair", "tp->repair_queue == TCP_*_QUEUE", "sk->sk_state == TCP_CLOSE". All of these branch need more time to explore in original syzkaller.

## Usage  

### Patch  
You need patch original syzkaller. Note 0002-Attach-weight-to-prog.patch only attach the weight to prog. Without using weight.
```  
git checkout 12365b99
git apply *.patch
```
### Gobpf  
Run:  
```  
go build pipe_monitor.go
```
Scp it to VM:/root/pipe_monitor.  

### Run state-base syzkaller
Just run syz-manager as original syzkaller.

## What can you customize  

## Code and features  
1. [0001-Add-ebpf-feedback.patch](0001-Add-ebpf-feedback.patch): run a ebpf monitor before execute_one, read pipe memory to get kernel socket state  
2. [pipe_monitor.go](pipe_monitor.go): load a ebpf text, monitor the socket state, feedback to syzkaller by using pipe memory. But it can't trace the historical state of a specific socket.  
3. [0002-Attach-weight-to-prog.patch](0002-Attach-weight-to-prog.patch): These patch attach weight to syzkaller prog. The weight is from state signal or coverage information. But it's not used in syz fuzz strategy.  

* These patch base on upstream syzkaller: 12365b99  
More detail refer to the code comments. 

### ebpf, kernel data type

ebpf text in ebpf/ebpftext.go is the only one can be modified as your will. You can get any data you want by writing ebpf by yourself.  In my case, I fill a uint64_t with socket state. Actually I try to separate them to two 32-bit. The high-32-bit is general historical socket state( type, flags, state ...). The low-32-bit is branch-related function argument and function id. The patched executor will calculate:  
```  
sig = hash(((state>>32)&0xffffffff))^(state&0xffffffff)
```  
which looks like a coverage signal

### kernel socket state

parse/parse.go is only for making the socket state readable. Modify it refer to you ebpf text as your will. Only for execprog. Now it's discarded.

## Example
pipe_monitor can run well with patched syzkaller. Without any different compare to original syzkaller's using. But you need write your ebpf to collect state.  
[Here](tcp-ipv6/test.md) is a example show the effect of ebpf state feedback compare to original syzkaller.

This is a example run patched execproc( patch_for_syz_executor.patch) with shm_monitor.
```  
# bin/linux_amd64/syz-execprog -executor=./bin/linux_amd64/syz-executor -cover=1 -threaded=0 -procs=1 --debug 06def8c2645148cb881aff26e222a886db9e1d72 
2019/01/09 04:34:14 parsed 1 programs
2019/01/09 04:34:14 executed programs: 0
spawned loop pid 7530
...
shm_monitor start ...
...
spawned worker pid 7799
...
2019/01/09 04:34:23 Monitoring the process 7799
2019/01/09 04:34:23 Waiting for signal
#0 [8476ms] -> socket$inet6_tcp(0xa, 0x1, 0x0)
#0 [8482ms] -> bind$inet6(0x3, 0x20000100, 0x1c)
#0 [8483ms] -> recvfrom$inet6(0x3, 0x20000280, 0x67, 0x20, 0x20000080, 0x1c)
#0 [8483ms] -> sendto$inet6(0x3, 0x200001c0, 0x0, 0x20000000, 0x200000c0, 0x1c)
#0 [8485ms] -> setsockopt$inet6_tcp_TCP_CONGESTION(0x3, 0x6, 0xd, 0x20000140, 0x3)
#0 [8486ms] -> sendto$inet6(0x3, 0x20000340, 0x982b5b491b7ad2c5, 0x41, 0x20000040, 0x16)
#0 [8492ms] -> recvfrom$inet6(0x3, 0x20000180, 0xfc, 0x100, 0x20000000, 0x1c)
Before send SIGUSR1 signal
...
2019/01/09 04:34:23 Socket state handle start ...
2019/01/09 04:34:23 7 signals in statelist
2019/01/09 04:34:23 8 rawSignals got!
2019/01/09 04:34:23 :8 covered
2019/01/09 04:34:23 :0 covered
2019/01/09 04:34:23 :8 covered
2019/01/09 04:34:23 Write signal:65808, byteX
...
...
2019/01/09 04:34:23 :8 covered
2019/01/09 04:34:23 :0 covered
2019/01/09 04:34:23 :8 covered
2019/01/09 04:34:23 Write signal:65808, byteX
2019/01/09 04:34:23 :0 covered
2019/01/09 04:34:23 SOCK_STREAM:2 covered
2019/01/09 04:34:23 SS_CONNECTED:3 covered
2019/01/09 04:34:23 Write signal:66320, byte: c
2019/01/09 04:34:23 :0 covered
2019/01/09 04:34:23 SOCK_STREAM:2 covered
2019/01/09 04:34:23 SS_CONNECTED:3 covered
2019/01/09 04:34:23 Write signal:66320, byte: c
2019/01/09 04:34:23 :0 covered
2019/01/09 04:34:23 SOCK_STREAM:2 covered
2019/01/09 04:34:23 SS_CONNECTED:3 covered
2019/01/09 04:34:23 Write signal:66320, byte: c
2019/01/09 04:34:23 Write signal:ffffffff, byte:����
...
2019/01/09 04:34:24 result: failed=false hanged=false err=<nil>

2019/01/09 04:34:24 executed programs: 1
...
```  