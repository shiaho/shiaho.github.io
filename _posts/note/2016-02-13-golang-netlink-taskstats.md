---
layout: post
title: "golang 通过 netlink 获取 taskstats"
tagline: "Supporting tagline"
description: ""
category: note
tags: [golang, linux, moniter]
wordpress_id: bf7ef145bff61ef14b3ca8838204720a
---
{% include JB/setup %}

## 开始
开始了解 **taskstate** 的目的是为了给项目[hickwall](javascript:void(0);)的agent增加 获取io read/write最高的top进程id功能，这个信息是可以通过读取proc里面的io来获取，但是考虑到要读取所有进程proc文件同样有很大的io消耗，而初衷是为了定位io消耗大的进程以此来发现问题，agent自身的io不能太大，所以暂时放弃这种方式。后来看到centos下有一个 `iotop` 命令，查看其源码 发现是使用netlink来获取的，于是决定采用netlink来获取。


Netlink是一种特殊的socket，用来实现用户态进程与内核的信息通信。可以通过`AF_NETLINK`来创建一个netlink套接字(socket),netlink协议由多个子协议组成目前包括:

 * NETLINK_ROUTE
 * NETLINK_W1
 * NETLINK_USERSOCK
 * NETLINK_FIREWALL
 * NETLINK_INET_DIAG
 * NETLINK_NFLOG
 * NETLINK_XFRM
 * NETLINK_SELINUX
 * NETLINK_ISCSI
 * NETLINK_AUDIT
 * NETLINK_FIB_LOOKUP
 * NETLINK_CONNECTOR
 * NETLINK_IP6_FW
 * NETLINK_DNRTMSG
 * NETLINK_KOBJECT_UEVENT
 * NETLINK_GENERIC
 * NETLINK_CRYPTO

taskstats属于`NETLINK_GENERIC`子协议。NETLINK_GENERIC有特殊的请求头 `GenlMsghdr`

~~~golang
type GenlMsghdr struct {
	Cmd      uint8
	Version  uint8
	Reserved uint16
}
~~~


在 `/usr/include/linux/taskstats.h` 有关于 taskstate的定义，首先先将其中的定义转换为go的定义

~~~golang
const (
	TS_COMM_LEN = 32
)

type Taskstats struct {

						    //1) Common and basic accounting fields:
						    /* The version number of this struct. This field is always set to
						     * TAKSTATS_VERSION, which is defined in <linux/taskstats.h>.
						     * Each time the struct is changed, the value should be incremented.
						     */
	Version uint16;

						    /* The exit code of a task. */
	Ac_exitcode uint32;	/* Exit status */

							/* The accounting flags of a task as defined in <linux/acct.h>
							* Defined values are AFORK, ASU, ACOMPAT, ACORE, and AXSIG.
							*/
	Ac_flag uint8;		/* Record flags */

							/* The value of task_nice() of a task. */
	Ac_nice uint8;		/* task_nice */

							//2) Delay accounting fields:
							/* Delay accounting fields start
							 *
							 * All values, until the comment "Delay accounting fields end" are
							 * available only if delay accounting is enabled, even though the last
							 * few fields are not delays
							 *
							 * xxx_count is the number of delay values recorded
							 * xxx_delay_total is the corresponding cumulative delay in nanoseconds
							 *
							 * xxx_delay_total wraps around to zero on overflow
							 * xxx_count incremented regardless of overflow
							 */

							/* Delay waiting for cpu, while runnable
							 * count, delay_total NOT updated atomically
							 */
	Cpu_count uint64;
	Cpu_delay_total uint64;

							/* Following four fields atomically updated using task->delays->lock */

							/* Delay waiting for synchronous block I/O to complete
							 * does not account for delays in I/O submission
							 */
	Blkio_count uint64;
	Blkio_delay_total uint64;

							/* Delay waiting for page fault I/O (swap in only) */
	Swapin_count uint64;
	Swapin_delay_total uint64;

							/* cpu "wall-clock" running time
							 * On some architectures, value will adjust for cpu time stolen
							 * from the kernel in involuntary waits due to virtualization.
							 * Value is cumulative, in nanoseconds, without a corresponding count
							 * and wraps around to zero silently on overflow
							 */
	Cpu_run_real_total uint64;

							/* cpu "virtual" running time
							 * Uses time intervals seen by the kernel i.e. no adjustment
							 * for kernel's involuntary waits due to virtualization.
							 * Value is cumulative, in nanoseconds, without a corresponding count
							 * and wraps around to zero silently on overflow
							 */
	Cpu_run_virtual_total uint64;
							/* Delay accounting fields end */
							/* version 1 ends here */

							/* The name of the command that started this task. */
	Ac_comm [TS_COMM_LEN]byte;	/* Command name */

							/* The scheduling discipline as set in task->policy field. */
	Ac_sched uint64;		/* Scheduling discipline */

	Ac_pad [3]uint8;
	Ac_uid uint32;			/* User ID */
	Ac_gid uint32;			/* Group ID */
	Ac_pid uint32;			/* Process ID */
	Ac_ppid uint32;		/* Parent process ID */

							/* The time when a task begins, in [secs] since 1970. */
	Ac_btime uint32;		/* Begin time [sec since 1970] */

							/* The elapsed time of a task, in [usec]. */
	Ac_etime uint64;		/* Elapsed time [usec] */

							/* The user CPU time of a task, in [usec]. */
	Ac_utime uint64;		/* User CPU time [usec] */

							/* The system CPU time of a task, in [usec]. */
	Ac_stime uint64;		/* System CPU time [usec] */

							/* The minor page fault count of a task, as set in task->min_flt. */
	Ac_minflt uint64;		/* Minor Page Fault Count */

							/* The major page fault count of a task, as set in task->maj_flt. */
	Ac_majflt uint64;		/* Major Page Fault Count */

							//3) Extended accounting fields
							/* Extended accounting fields start */

							/* Accumulated RSS usage in duration of a task, in MBytes-usecs.
							 * The current rss usage is added to this counter every time
							 * a tick is charged to a task's system time. So, at the end we
							 * will have memory usage multiplied by system time. Thus an
							 * average usage per system time unit can be calculated.
							 */
	Coremem uint64;		/* accumulated RSS usage in MB-usec */

							/* Accumulated virtual memory usage in duration of a task.
							* Same as acct_rss_mem1 above except that we keep track of VM usage.
							*/
	Virtmem uint64;		/* accumulated VM usage in MB-usec */

							/* High watermark of RSS usage in duration of a task, in KBytes. */
	Hiwater_rss uint64;		/* High-watermark of RSS usage */

							* High watermark of VM  usage in duration of a task, in KBytes. */
	Hiwater_vm uint64;		/* High-water virtual memory usage */

							/* The following four fields are I/O statistics of a task. */
	Read_char uint64;		/* bytes read */
	Write_char uint64;		/* bytes written */
	Read_syscalls uint64;		/* read syscalls */
	Write_syscalls uint64;		/* write syscalls */

							/* Extended accounting fields end */

							/4) Per-task and per-thread statistics
	Read_bytes uint64;             /* bytes of read I/O */
	Write_bytes uint64;            /* bytes of write I/O */
	Cancelled_write_bytes uint64;  /* bytes of cancelled write I/O */
	Nvcsw uint64;			/* Context voluntary switch counter */
	Nivcsw uint64;			/* Context involuntary switch counter */

							//5) Time accounting for SMT machines
	Ac_utimescaled uint64;		/* utime scaled on frequency etc */
	Ac_stimescaled uint64;		/* stime scaled on frequency etc */
	Cpu_scaled_run_real_total uint64; /* scaled cpu_run_real_total */

							//6) Extended delay accounting fields for memory reclaim
							/* Delay waiting for memory reclaim */
	Freepages_count uint64;
	Freepages_delay_total uint64;
}

const (
	TASKSTATS_CMD_UNSPEC = iota /* Reserved */
	TASKSTATS_CMD_GET           /* user->kernel request/get-response */
	TASKSTATS_CMD_NEW           /* kernel->user event */
	__TASKSTATS_CMD_MAX
)

const (
	TASKSTATS_TYPE_UNSPEC    = iota /* Reserved */
	TASKSTATS_TYPE_PID              /* Process id */
	TASKSTATS_TYPE_TGID             /* Thread group id */
	TASKSTATS_TYPE_STATS            /* taskstats structure */
	TASKSTATS_TYPE_AGGR_PID         /* contains pid + stats */
	TASKSTATS_TYPE_AGGR_TGID        /* contains tgid + stats */
	__TASKSTATS_TYPE_MAX
)

const (
	TASKSTATS_CMD_ATTR_UNSPEC = iota
	TASKSTATS_CMD_ATTR_PID
	TASKSTATS_CMD_ATTR_TGID
	TASKSTATS_CMD_ATTR_REGISTER_CPUMASK
	TASKSTATS_CMD_ATTR_DEREGISTER_CPUMASK
	__TASKSTATS_CMD_ATTR_MAX
)
~~~

##实现

有了taskstate的golang版定义，就可以通过 syscall.socket 来获取taskstate信息了

~~~golang
import (
	"encoding/binary"
	"github.com/hkwi/nlgo"
	"unsafe"
	"syscall"
	"fmt"
)

type MSG struct {
	Len  uint16
	Type uint16
	Pid  uint32
}

type Attr struct {
	Type int
	Data []byte
}

func GetTaskStats(nlsk *nlgo.NlSock, p int) (t *Taskstats) {
	nlsk.Flags |= nlgo.NL_NO_AUTO_ACK
	const familyID  =  22
	m := &MSG{
		Len:  8,
		Type: TASKSTATS_CMD_ATTR_PID,
		Pid:  uint32(p),
	}

	hdr := (*[nlgo.SizeofGenlMsghdr]byte)(unsafe.Pointer(&nlgo.GenlMsghdr{
		Cmd:     TASKSTATS_CMD_GET,
		Version: 0,
	}))[:]
	req := ((*[8]byte)(unsafe.Pointer(m)))[:]
	length := 4
	pad := ((length + 4 - 1) & (^3)) - length
	for i := 0; i < pad; i++ {
		req = append(req, 0)
	}
	hdr = append(hdr, req...)
	nlgo.NlSendSimple(nlsk, familyID, 1, hdr[:])

	func() error {
		for {
			buf := make([]byte, 16384)
			if nn, _, err := syscall.Recvfrom(nlsk.Fd, buf, 0); err != nil {
				return err
			} else if nn > len(buf) {
				return nlgo.NLE_MSG_TRUNC
			} else {
				buf = buf[:nn]
			}
			if msgs, err := syscall.ParseNetlinkMessage(buf); err != nil {
				return err
			} else {
				for _, msg := range msgs {

					switch msg.Header.Type {
					case syscall.NLMSG_DONE:
						return nil
					case syscall.NLMSG_ERROR:
						return fmt.Errorf("NlMsgerr=%s", nlgo.NlMsgerr(msg))
					case 22:
						genl := (*nlgo.GenlMsghdr)(unsafe.Pointer(&msg.Data[0]))
						_ = genl
						attrs := parse_attributes(msg.Data[nlgo.GENL_HDRLEN:])
						for _, attr := range attrs {
							if attr.Type == TASKSTATS_TYPE_AGGR_PID {
								attrs = parse_attributes(attr.Data)
								break
							}
						}
						for _, attr := range attrs {
							if attr.Type == TASKSTATS_TYPE_STATS {
								_ = uint32(*(*uint32)(unsafe.Pointer(&attr.Data[248])))
								t = (*Taskstats)(unsafe.Pointer(&attr.Data[0]))
								break
							}
						}
						return nil
					default:
						return fmt.Errorf("unexpected NlMsghdr=%s", msg.Header)
					}
				}
			}
		}
	}()
	return
}


func parse_attributes(data []byte) map[int]Attr {
	attrs := make(map[int]Attr, 0)

	for len(data) != 0 {

		attr_len := binary.LittleEndian.Uint16(data[0:2])

		attr_type := binary.LittleEndian.Uint16(data[2:4])

		attrs[int(attr_type)] = Attr{
			int(attr_type),
			data[4:attr_len],
		}

		data = data[nlgo.NLMSG_ALIGN(int(attr_len)):]
	}
	return attrs
}
~~~

这里借用了 `github.com/hkwi/nlgo` 库的创建套接字和请求头的定义，这部分完全自己实现其实也很容易。`github.com/hkwi/nlgo` 中没有和taskstats相关的定义，所以这部分需要自已做。

## golang便捷的兼容C结构体的encode和decode
...

## C中的 __attribute__((aligned(8)));
...



