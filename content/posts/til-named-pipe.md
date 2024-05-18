+++
title = 'TIL: Getting the current capacity of a named pipe in Linux'
date = 2024-05-17T21:43:58-05:00
draft = false
+++

A named pipe (also known as FIFO) is a IPC channel, it allows one process to write and another to read. Like all things
in Linux, it is implemented as a file like entity using either [`mkfifo`](https://man7.org/linux/man-pages/man3/mkfifo.3.html) or [`pipe`](https://man7.org/linux/man-pages/man2/pipe.2.html) system calls.

Pipes on Linux have a limited size, typically set to 65536 bytes on modern systems. I recently came across a situation where I needed to know the current remaining capacity of a given pipe without consuming data from it, while one process is writing to it and another is reading from it. I found that the `ioctl` system call takes in a flag `FIONREAD` that makes it read the count of unread bytes in a given pipe, identified by the file descriptor of either ends. The count is placed in an int buffer that `ioctl` takes in.

The relevant section in https://man7.org/linux/man-pages/man7/pipe.7.html reads

```
The following ioctl(2) operation, which can be applied to a file descriptor that refers to 
either end of a pipe, places a count of the number of unread bytes in the pipe in the int 
buffer pointed to by the final argument of the call:

ioctl(fd, FIONREAD, &nbytes);

The FIONREAD operation is not specified in any standard, but is provided on many
implementations.
```

We assume that the name of the pipe is already known, and the reader and writer are active. The resulting python code looks like this:

```python
import fcntl
import os
import termios

fd = os.open(pipe_name, os.O_RDONLY | os.O_NONBLOCK)
buf = array.array('i', [0])
fcntl.ioctl(fd, termios.FIONREAD, buf)
current_size = buf[0]
remaining_capacity = 65536 - current_size
```

Shout out to my colleague [Nicolas Thauvin](https://github.com/orgrim) for pointing me in this direction.
