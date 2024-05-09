+++
title = 'Measuring energy usage of a function in C'
date = 2024-05-07T16:43:04-05:00
draft = false
+++


I recently needed a way to measure how much energy a piece of code consumes. This is easy to do in Linux using the perf utility assuming the underlying kernel has been built with appropriate flags. This looks something like this

```sh
$ sudo perf stat -a -e power/energy-pkg/ ls -la /usr
total 116
drwxr-xr-x  14 root root  4096 Mar 28 02:08 .
drwxr-xr-x  19 root root  4096 Apr 16 16:02 ..
drwxr-xr-x   2 root root 36864 May  7 15:05 bin
drwxr-xr-x   2 root root  4096 Apr 18  2022 games
drwxr-xr-x  37 root root 16384 May  7 15:51 include
drwxr-xr-x  88 root root  4096 May  7 15:05 lib
drwxr-xr-x   2 root root  4096 Mar 28 02:08 lib32
drwxr-xr-x   2 root root  4096 Apr 19 06:14 lib64
drwxr-xr-x   9 root root  4096 Apr 16 18:02 libexec
drwxr-xr-x   2 root root  4096 Mar 28 02:08 libx32
drwxr-xr-x  10 root root  4096 Mar 28 02:08 local
drwxr-xr-x   2 root root 20480 May  7 15:20 sbin
drwxr-xr-x 119 root root  4096 May  7 15:05 share
drwxr-xr-x   7 root root  4096 Apr 20 06:56 src

 Performance counter stats for 'system wide':

              0.03 Joules power/energy-pkg/

       0.000777146 seconds time elapsed
```

This tells me that the `ls` command consumed 0.03 Joules of energy and ran for 0.00078 seconds. I needed to get similar data for a C function that moght be surrounded by code that I do not need to measure. Something like this:

```
start_measurement();
function_call_one();
end_measurement();

start_measurement();
function_call_two();
end_measurement();
```

Here the two function calls need to be in the same translation unit, so I cannot seperate them and then use the perf tool to measure energy consumption.

I started with looking at the source code for [perf-stat](https://github.com/torvalds/linux/blob/dccb07f2914cdab2ac3a5b6c98406f765acab803/tools/perf/builtin-stat.c) but I was lost! This command takes in a whole bunch of options, the code was hard to follow. I googled `How perf works` and found Julia Evans' [blog post](https://jvns.ca/blog/2016/03/12/how-does-perf-work-and-some-questions/) titled exactly that. That post pointed me to the system call [perf_event_open](https://man7.org/linux/man-pages/man2/perf_event_open.2.html). The man page has an example code at the end which gets total number of instructions executed for a printf call, while that is useful I still do not know what particular parameters I need to use to get energy info. At this point I decided to `strace` the `perf stat` command to see how it was using `perf_event_open`.

```sh
$ sudo strace -v -s 100000 perf stat -a -e power/energy-pkg/ ls -la /usr > output.txt 2>&1
```

Grepping for the word `perf` in output.txt revealed three calls to `perf_event_open`

```sh
$ rg perf_event_open output.txt
2161:perf_event_open({type=0x17 /* PERF_TYPE_??? */, size=PERF_ATTR_SIZE_VER7, config=0x2, sample_period=0, sample_type=PERF_SAMPLE_IDENTIFIER, read_format=PERF_FORMAT_TOTAL_TIME_ENABLED|PERF_FORMAT_TOTAL_TIME_RUNNING, disabled=1, inherit=1, pinned=0, exclusive=0, exclude_user=0, exclude_kernel=0, exclude_hv=0, exclude_idle=0, mmap=0, comm=0, freq=0, inherit_stat=0, enable_on_exec=0, task=0, watermark=0, precise_ip=0 /* arbitrary skid */, mmap_data=0, sample_id_all=0, exclude_host=0, exclude_guest=1, exclude_callchain_kernel=0, exclude_callchain_user=0, mmap2=0, comm_exec=0, use_clockid=0, context_switch=0, write_backward=0, namespaces=0, ksymbol=0, bpf_event=0, aux_output=0, cgroup=0, text_poke=0, build_id=0, inherit_thread=0, remove_on_exec=0, sigtrap=0, wakeup_events=0, config1=0, config2=0, sample_regs_user=0, sample_regs_intr=0, aux_watermark=0, sample_max_stack=0, aux_sample_size=0, sig_data=0}, -1, 0, -1, PERF_FLAG_FD_CLOEXEC) = -1 EINVAL (Invalid argument)
2162:perf_event_open({type=0x17 /* PERF_TYPE_??? */, size=PERF_ATTR_SIZE_VER7, config=0x2, sample_period=0, sample_type=PERF_SAMPLE_IDENTIFIER, read_format=PERF_FORMAT_TOTAL_TIME_ENABLED|PERF_FORMAT_TOTAL_TIME_RUNNING, disabled=1, inherit=1, pinned=0, exclusive=0, exclude_user=0, exclude_kernel=0, exclude_hv=0, exclude_idle=0, mmap=0, comm=0, freq=0, inherit_stat=0, enable_on_exec=0, task=0, watermark=0, precise_ip=0 /* arbitrary skid */, mmap_data=0, sample_id_all=0, exclude_host=0, exclude_guest=1, exclude_callchain_kernel=0, exclude_callchain_user=0, mmap2=0, comm_exec=0, use_clockid=0, context_switch=0, write_backward=0, namespaces=0, ksymbol=0, bpf_event=0, aux_output=0, cgroup=0, text_poke=0, build_id=0, inherit_thread=0, remove_on_exec=0, sigtrap=0, wakeup_events=0, config1=0, config2=0, sample_regs_user=0, sample_regs_intr=0, aux_watermark=0, sample_max_stack=0, aux_sample_size=0, sig_data=0}, -1, 0, -1, 0) = -1 EINVAL (Invalid argument)
2163:perf_event_open({type=0x17 /* PERF_TYPE_??? */, size=PERF_ATTR_SIZE_VER7, config=0x2, sample_period=0, sample_type=PERF_SAMPLE_IDENTIFIER, read_format=PERF_FORMAT_TOTAL_TIME_ENABLED|PERF_FORMAT_TOTAL_TIME_RUNNING, disabled=1, inherit=1, pinned=0, exclusive=0, exclude_user=0, exclude_kernel=0, exclude_hv=0, exclude_idle=0, mmap=0, comm=0, freq=0, inherit_stat=0, enable_on_exec=0, task=0, watermark=0, precise_ip=0 /* arbitrary skid */, mmap_data=0, sample_id_all=0, exclude_host=0, exclude_guest=0, exclude_callchain_kernel=0, exclude_callchain_user=0, mmap2=0, comm_exec=0, use_clockid=0, context_switch=0, write_backward=0, namespaces=0, ksymbol=0, bpf_event=0, aux_output=0, cgroup=0, text_poke=0, build_id=0, inherit_thread=0, remove_on_exec=0, sigtrap=0, wakeup_events=0, config1=0, config2=0, sample_regs_user=0, sample_regs_intr=0, aux_watermark=0, sample_max_stack=0, aux_sample_size=0, sig_data=0}, -1, 0, -1, 0) = 3
```

But how do we figure out what values to use for `type` and `config`? Scrolling up from the `perf_event_open` call I found a series of `openat` calls to various files. Here are some that matches the string `energy-pkg`

```sh
743:openat(AT_FDCWD, "/sys/bus/event_source/devices/power/events/energy-pkg", O_RDONLY) = 5
747:openat(AT_FDCWD, "/sys/bus/event_source/devices/power/events/energy-pkg.unit", O_RDONLY) = 6
750:openat(AT_FDCWD, "/sys/bus/event_source/devices/power/events/energy-pkg.scale", O_RDONLY) = 6
754:openat(AT_FDCWD, "/sys/bus/event_source/devices/power/events/energy-pkg.per-pkg", O_RDONLY) = -1 ENOENT (No such file or directory)
755:openat(AT_FDCWD, "/sys/bus/event_source/devices/power/events/energy-pkg.snapshot", O_RDONLY) = -1 ENOENT (No such file or directory)
```

So it looks like `perf` is reading those files to get the necessary configuration for the `perf_event_open` call. [Here](https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-bus-event_source-devices-events) are the kernel docs for those files, that further validated my suspicion.


Here are the contents of those files on my box

```sh
$ sudo cat /sys/bus/event_source/devices/power/events/energy-pkg
event=0x02
$ sudo cat /sys/bus/event_source/devices/power/events/energy-pkg.unit
Joules
$ sudo cat /sys/bus/event_source/devices/power/events/energy-pkg.scale
2.3283064365386962890625e-10
```

Poking around in that directory, I found another file that looked important

```sh
$ sudo cat /sys/bus/event_source/devices/power/type
23
```
At this point, I was pretty confident I have all the info I need to use this syscall to measure power consumption. This is what I came up with

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/syscall.h>
#include <linux/perf_event.h>
#include <asm/unistd.h>

#define PERF_EVENT_OPEN_SYSCALL_NR __NR_perf_event_open

int main() {
    int type = 0x17; // 23 in hex
    long config = 0x02; // event code in hex
    
    // Set up the perf_event_attr struct
    struct perf_event_attr attr;
    memset(&attr, 0, sizeof(attr));
    attr.type = type;
    attr.size = sizeof(struct perf_event_attr);
    attr.config = config;

    // Open the perf event
    long fd = syscall(PERF_EVENT_OPEN_SYSCALL_NR, &attr, -1, 0, -1, 0);
    if (fd == -1) {
        perror("perf_event_open");
        return 1;
    }

    // Read the initial counter value
    long int counter_before;
    if (read(fd, &counter_before, sizeof(long int)) != sizeof(long int)) {
        perror("read");
        close(fd);
        return 1;
    }

    // Wait for 10 seconds
    printf("Measuring energy usage for 10 seconds...\n");
    sleep(10);

    // Read the final counter value
    long int counter_after;
    if (read(fd, &counter_after, sizeof(long int)) != sizeof(long int)) {
        perror("read");
        close(fd);
        return 1;
    }

    // Calculate the energy usage
    double scale = 2.3283064365386962890625e-10; // from energy-cores.scale
    double energy = (counter_after - counter_before) * scale;
    printf("Energy usage: %.9f Joules\n", energy);

    // Close the perf event
    close(fd);
    return 0;
}
```

Running this is simple

```sh
$ cc perf.c && sudo ./a.out
Measuring energy usage for 10 seconds...
Energy usage: 10.955932617 Joules
```