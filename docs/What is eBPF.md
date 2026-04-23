# What is eBPF
## Introduction
Modifying the Linux kernel is complicated because you either have to contribute to the project (which can take years to be accepted and must address a need that others also have), or create and maintain a kernel module (.ko), which can cause system instability.

Linux 2.2 delivered a new feature: BPF (Berkeley Packet Filter), which allows you to safely run code in the kernel. It was used for packet filtering, as in tools like tcpdump (the program can see packets before the kernel processes them).

Linux 3.18 introduced eBPF (extended Berkeley Packet Filter), designed to support more than just network filtering: tracing, monitoring, and security.

This is very convenient, and it addresses a problem that Windows has not managed to solve. Early versions of Windows antivirus software hooked system calls directly, which could lead to crashes. To prevent this, Windows simply blocked any kernel interaction from unsigned drivers (Kernel Patch Protection (KPP)), forcing security tool developers to operate in user space at the same level as malware which is not very convenient. *Fun fact: Microsoft is currently developing 'eBPF for Windows' to bring these exact capabilities to their ~~spyware~~ OS.*

Linux, on the other hand, gives us a gateway into the kernel.

## Example of applications that use eBPF
### bpftrace
You can add an event listener on a specific tracepoint like "sys_enter_openat", which triggers when the kernel executes the "openat" syscall :
```bash
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'
```
```
cat yolo.txt
```
```
[...]
/home/askeladd/yolo.txt
```

*documentation : https://bpftrace.org/tutorial-one-liners#lesson-4-syscall-counts-by-process*

## How to create eBPF program
### Using Go and C
```
sudo apt install clang llvm libbpf-dev linux-headers-$(uname -r)
```
```
clang : compiler using the llvm toolchain
llvm : set of modular compiler tools and libraries
libbpf-dev : provides development files needed to build programs that interact with the Linux kernel using eBPF
linux-headers : the header files needed to compile kernel modules for the currently running Linux kernel
```

We will use Go as an intermediate to load our C programm in eBPF. The ``cilium/ebpf`` packages contains the tools ``bpf2go`` that we will use to generate our Go program. (*Creators of Cilium have open-sourced their internal library which allows communication with the Linux bpf() syscall.*)

We can't just compile our C program and send it to the kernel. We are going to "translate" it in bytecode (.o), and trigger the ``bpf`` syscall with the Go loader.

The Kernel will then analyse the code (with a tool called verifier) to ensure that it will not crash the system and run it.

So we write our code in C. We translate it into bytecode using the ``go generate`` command (because in ``main.go`` there's this line that will call the library: `//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf hello.c` on our program to generate the bytecode.)

Once that's done, we build our ``main.go`` (``go build -o ebpf-app``). Our compiled ``ebpf-app`` will retrieve the bytecode and send it to eBPF via the ``syscall``.

Init the Go project ; this will allow us to manage dependencies.
```
go mod init ebpf-hello
```

We add the two dependencies needed for bpf2go :
```
go get github.com/cilium/ebpf/cmd/bpf2go
go get github.com/cilium/ebpf
```

hello.c
```c
// tells Go to not compile this program (we only want the eBPF bytecode)
//go:build ignore

// kernel API for eBPF
#include <linux/bpf.h>

// macro to facilitate development (SEC comes from there)
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "GPL";

// define where the program is attached
// here we hook the execve that trigger when commands are launched
SEC("tracepoint/syscalls/sys_enter_execve")
int hello_world(void *ctx) {
    bpf_printk("Hello world from eBPF !");
    
    return 0;
}
```

main.go
```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"

	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/rlimit"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf hello.c

func main() {
	if err := rlimit.RemoveMemlock(); err != nil {
		log.Fatalf("fail to deactivate memlock limite: %v", err)
	}

	var objs bpfObjects
	if err := loadBpfObjects(&objs, nil); err != nil {
		log.Fatalf("fail to load the objets eBPF: %v", err)
	}
	defer objs.Close()

	tp, err := link.Tracepoint("syscalls", "sys_enter_execve", objs.HelloWorld, nil)
	if err != nil {
		log.Fatalf("tracepoint attachment fail: %v", err)
	}
	defer tp.Close()

	stopper := make(chan os.Signal, 1)
	signal.Notify(stopper, os.Interrupt, syscall.SIGTERM)
	<-stopper

}
```

Generate :
```
go generate
```
*Execute among others ``//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf hello.c``*


Compile the Go loader : 
```
go build -o ebpf-app
```

Run the loader that will push the hello.c opcodes in eBPF through bpf syscall :
```
sudo ./ebpf-app
```

We can see the print from the trace_pipe :
```
sudo cat /sys/kernel/debug/tracing/trace_pipe

konsole-90904   [002] ...21 22059.808197: bpf_trace_printk: Hello world from eBPF !
```

### Demo HIDS
Printing Hello-world was as fun as always but not very useful. What we are going to do now is trace every openat syscall (when a file is open) and filter it looking for sensitive access.

To do so we first need to understand the data structure of the tracepoint for openat.
```
sudo cat /sys/kernel/tracing/events/syscalls/sys_enter_openat/format

name: sys_enter_openat
ID: 659
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:int __syscall_nr; offset:8;       size:4; signed:1;
        field:int dfd;  offset:16;      size:8; signed:0;
        field:const char * filename;    offset:24;      size:8; signed:0;
        field:int flags;        offset:32;      size:8; signed:0;
        field:umode_t mode;     offset:40;      size:8; signed:0;

print fmt: "dfd: 0x%08lx, filename: 0x%08lx, flags: 0x%08lx, mode: 0x%08lx", ((unsigned long)(REC->dfd)), ((unsigned long)(REC->filename)), ((unsigned long)(REC->flags)), ((unsigned long)(REC->mode))
```

Here is the translation in C :
```c
struct openat_args {
    unsigned short common_type
    unsigned char common_flags
    unsigned char common_preempt_count
    int common_pid
    long syscall_nr;
    long dfd;
    const char *filename;
    long flags;
    long mode;
};
```

```c
SEC("tracepoint/syscalls/sys_enter_openat")
int demo_hids(struct openat_args *ctx) // use our struct
{
    char filename_buffer[256];

    long ret = bpf_probe_read_user_str(filename_buffer, sizeof(filename_buffer), ctx->filename);

    if (ret > 0) {
        bpf_printk("filename : %s\n", filename_buffer);
    }

    return 0;
}
```

We also need to change our loader :
```go
	tp, err := link.Tracepoint("syscalls", "sys_enter_openat", objs.DemoHids, nil)
	if err != nil {
		log.Fatalf("tracepoint attachment fail: %v", err)
	}
	defer tp.Close()
```

The issue here is that a lot of files are opened by the system, so we need to filter the interesting one.
```
sudo cat /sys/kernel/debug/tracing/trace_pipe

           <...>-18797   [002] ...21  2522.895339: bpf_trace_printk: filename : /etc/ld.so.cache

           <...>-18797   [002] ...21  2522.895382: bpf_trace_printk: filename : /lib/x86_64-linux-gnu/libc.so.6

           <...>-18797   [002] ...21  2522.895721: bpf_trace_printk: filename : /usr/lib/locale/locale-archive

           <...>-18797   [002] ...21  2522.895791: bpf_trace_printk: filename : /etc/passwd

```

So we will compare filename to a list, however the eBPF verifier is pretty tricky to pass and do not allow any approximations that could destabilize the kernel. For example, if the maximum file size is unknown, it will not accept the file because there is no guarantee that the loop is not infinite.

So we're going to have to use 3 things that are a bit different from classic C.

eBPF rejects dynamic loops because it must prove that the program terminates. Using ``#pragma unroll`` tells the compiler to fully unroll the loop. 
```
Exemple (pseudo code):

#pragma unroll
for i in range(3):
	print(3)

Become :

print(3)
print(3)
print(3)
```

Similarly ``static __always_inline``, tells the compiler to replace the function call with the code directly, which makes it easier to analyze (and anyway, if it's not happy it won't let the program through, so it's better to be nice to it).
```
Exemple (pseudo code):

static __always_inline
def do_a_flip(a):
	return a + 5

b = do_a_flip(6)

Become :

b = 6 + 5
```

And in order for him to be able to unroll the loops and the function, we need to know their maximum size in advance.
```
#define MAX_PATH_LEN 32
#define MAX_FILES 3
```

```c
//go:build ignore

#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "GPL";

struct openat_args {
    unsigned short common_type;
    unsigned char common_flags;
    unsigned char common_preempt_count;
    int common_pid;
    long syscall_nr;
    long dfd;
    const char *filename;
    long flags;
    long mode;
};

#define MAX_PATH_LEN 32
#define MAX_FILES 3

const char target_files[MAX_FILES][MAX_PATH_LEN] = {
    "/etc/shadow",
    "/etc/passwd",
    "/etc/hosts"
};

static __always_inline int match(char *filename_buffer, const char *candidate){
    #pragma unroll
    for (int i = 0; i < MAX_PATH_LEN; i++) { // for every character
        if (filename_buffer[i] != candidate[i]) { // if != stop and return 0
            return 0;
        }
        if (candidate[i] == '\0') { // if the string reaches the end of the loop, it means it's the same 
            return 1;
        }
    }
    return 1;
}

static __always_inline int is_targeted_file(char *filename_buffer){
    #pragma unroll
    for (int i = 0; i < MAX_FILES; i++) {
        if (match(filename_buffer, target_files[i])) {
            return 1;
        }
    }
    return 0;
}

SEC("tracepoint/syscalls/sys_enter_openat")
int demo_hids(struct openat_args *ctx)
{
    char filename_buffer[256];

    long ret = bpf_probe_read_user_str(filename_buffer, sizeof(filename_buffer), ctx->filename);

    if (ret > 0) {
        if (is_targeted_file(filename_buffer) == 1){
            bpf_printk("filename : %s\n", filename_buffer);
        }
    }

    return 0;
}
```

```
 accounts-daemon-900     [006] ...21  5062.959886: bpf_trace_printk: filename : /etc/shadow

 accounts-daemon-900     [006] ...21  5062.960157: bpf_trace_printk: filename : /etc/passwd

 accounts-daemon-900     [006] ...21  5062.961157: bpf_trace_printk: filename : /etc/passwd

 accounts-daemon-900     [006] ...21  5062.961198: bpf_trace_printk: filename : /etc/shadow

           <...>-25961   [005] ...21  5063.250447: bpf_trace_printk: filename : /etc/passwd

           <...>-25977   [000] ...21  5064.825606: bpf_trace_printk: filename : /etc/hosts
```

### Using Python

We can't run Python in the kernel... However, we can use it as a loader and I find it more practical than using Go :
```
sudo apt-get install python3-bpfcc bpfcc-tools libbpfcc
```

```py
from bcc import BPF

program = """
int hello_world(void *ctx) {
    bpf_trace_printk("eBPF very much\\n");
    return 0;
}
"""

b = BPF(text=program)

# attach the program to the execve event
execve_fn = b.get_syscall_fnname("execve")
b.attach_kprobe(event=execve_fn, fn_name="hello_world")

# read trace_pipe
try:
    b.trace_print()
except KeyboardInterrupt:
    exit()
```

```
sudo python3 hello.py 

b'           <...>-27917   [006] ...21  5617.110294: bpf_trace_printk: eBPF very much'
b''
b'              sh-27918   [001] ...21  5617.112760: bpf_trace_printk: eBPF very much'
b''
b'           <...>-27919   [006] ...21  5617.115964: bpf_trace_printk: eBPF very much'
```

**bpf_printk is great for debugging, but it's slow. In production, we send data to userland using Maps or Ring Buffer.**

## Maps
The map are a storage method used by eBPF program. For example, if we want to generate a metrics with the number of syscall "send_msg", we arren't going to use a classic variable to increment because it will be isolated (innaccessible in userland), potentially deleted by the Verifier (which optimizes the program) and is deleted between the eBPF call. However we can use maps, that allow in addition to avoiding all the problems mentioned above, allow to share the key-value table with userland.

```py
from bcc import BPF
import time

program = """
// here is our map, is name is counter, the key is 32 bits and the value can be 64 bits
BPF_HASH(counter, u32, u64); // 

struct sys_enter_sendmsg_args {
    unsigned long long unused;
    long __syscall_nr;
    long fd;
    void *msg;
    unsigned long flags;
};

int tracepoint__syscalls__sys_enter_sendmsg(struct sys_enter_sendmsg_args *args) {
    u32 key = 0; // the key is 0 (it can be any integer)
    u64 *value; // we save place in memory for our value

    value = counter.lookup(&key); // value point on the address of the value of key
    if (value) {
        (*value)++; // if the value have already been initiated we incremente it
    } else {
        u64 init = 1; // if not we init the value
        counter.update(&key, &init);
    }

    return 0;
}
"""

b = BPF(text=program)

while True:
    time.sleep(1)
    # thanks to the BCC library, we can retrieve the variable more easily in userland using :
    for k, v in b["counter"].items():
        print("Total sendmsg:", v.value)
```

```
$ sudo python3 main.py 
Total sendmsg: 57
Total sendmsg: 168
Total sendmsg: 174
Total sendmsg: 205
Total sendmsg: 311
```

*In production, we should use BPF_PERCPU_HASH (one map per CPU core to avoid collisions) or an operation like __sync_fetch_and_add(value, 1).*

## Ring Buffer
Maps are very good for statistics but the most efficient for tracing is ring buffer which implements the concept of last-in first-out and temporary queue (like Kafka).

```py
from bcc import BPF

program = """
// defines the structure of the data stored in our ring
struct event_t {
   u32 pid;
   char comm[16];
};

// create the ring and define the number of pages (linux manages memory by pages (4 kb)) that can be stored
BPF_RINGBUF_OUTPUT(events_ring, 8);

int hook_execve(struct pt_regs *ctx) {
   struct event_t *event;

   // save space in the buffer
   event = events_ring.ringbuf_reserve(sizeof(struct event_t));
   if (!event) {
      return 0;
   }

   // get the event data
   event->pid = bpf_get_current_pid_tgid() >> 32;
   bpf_get_current_comm(&event->comm, sizeof(event->comm));

   // put the data in the ring
   events_ring.ringbuf_submit(event, 0);

   return 0;
}
"""

b = BPF(text=program)

execve_fn = b.get_syscall_fnname("execve")
b.attach_kprobe(event=execve_fn, fn_name="hook_execve")


def callback(ctx, data, size):
   event = b["events_ring"].event(data)
   print(f"pid: {event.pid}, cmd: {event.comm.decode('ascii')}")

b["events_ring"].open_ring_buffer(callback)

while True:
    # ask the kernel if their is a new event (if so it will call the callback function)
    b.ring_buffer_poll()
```

## XDP
XDP (Express data path) is a specific network hook where you can attach a eBPF program. It allow to send/intercep and allow or not the trams.

All of this allow big optimisation will bypassing the kernel overhead of the linux network stack (interrupt handling, protocol parsing, memory allocation).

### XDP Fallback
Runs in the kernel, but after the classic network stack, useful for testing or monitoring (but not for filtering packets as they enter the system since that it is at the end of the process).

```python
from bcc import BPF

program = """
#include <uapi/linux/bpf.h>

int xdp_prog(struct xdp_md *ctx) {
    bpf_trace_printk("New trame retreive in XDP\\n");

    return XDP_PASS;
}
"""

b = BPF(text=program)
fn = b.load_func("xdp_prog", BPF.XDP)
b.attach_xdp("wlp4s0", fn, BPF.XDP_FLAGS_SKB_MODE)

try:
    b.trace_print()
except KeyboardInterrupt:
    b.remove_xdp(device, BPF.XDP_FLAGS_SKB_MODE)
    exit()
```

XDP_FLAGS_SKB_MODE = mode fallback

### XDP natif
The eBPF program runs in the network card driver before the packet enters the Linux network stack.

To switch to native mode, you must use the mode: XDP_FLAGS_DRV_MODE

It really depends on the network cards that need to be compatible.

### XDP offload
The program is loaded and runs directly on the network card. This guarantees maximum performance, but not all network cards support it. It is very useful for precisely managing large volumes of data (datacenters, telecom) and ensuring early filtering.

Most NIC are not compatible; it's mainly in cases of very advanced infrastructure (datacenters).