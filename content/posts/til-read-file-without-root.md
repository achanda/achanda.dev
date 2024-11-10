+++
title = 'TIL: Read sysfs without root access in Linux'
date = 2024-11-10T14:48:48-06:00
draft = false
readTime = true
tags = ['linux']
+++

Typically an user needs root access to be able to read sysfs data. But such root access might not be possible
in all case; an example being Postgres which cannot be run as root. I recently learned that it is possible
to set an extra [capability](https://man7.org/linux/man-pages/man7/capabilities.7.html) to executables 
such that it can read sysfs while not being root.

I wrote a simple C program to read a specific sysfs entry

```
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>

#define SYSFS_PATH "/sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/energy_uj"
#define BUFFER_SIZE 32

int main() {
    int fd;
    char buffer[BUFFER_SIZE];
    ssize_t bytes_read;
    unsigned long long energy_uj;

    // Open the sysfs file
    fd = open(SYSFS_PATH, O_RDONLY);
    if (fd == -1) {
        fprintf(stderr, "Error opening file: %s\n", strerror(errno));
        return 1;
    }

    // Read the contents of the file
    bytes_read = read(fd, buffer, BUFFER_SIZE - 1);
    if (bytes_read == -1) {
        fprintf(stderr, "Error reading file: %s\n", strerror(errno));
        close(fd);
        return 1;
    }

    // Null-terminate the buffer
    buffer[bytes_read] = '\0';

    // Close the file descriptor
    close(fd);

    // Convert the string to an unsigned long long
    energy_uj = strtoull(buffer, NULL, 10);

    // Print the result
    printf("Energy (µJ): %llu\n", energy_uj);

    return 0;
}
```

When I compile and run it as my regular non-root user, I get this

```
abhishek@guest:~$ id -u
1002
abhishek@guest:~$ cc foo.c
abhishek@guest:~$ ./a.out
Error opening file: Permission denied
```

But I can set a specific capability on the generated binary

```
abhishek@guest:~$ sudo setcap cap_dac_read_search=+ep a.out
[sudo] password for abhishek:
abhishek@guest:~$ echo $?
0
```

And then the access works

```
abhishek@guest:~$ ./a.out
Energy (µJ): 128875379306
```
