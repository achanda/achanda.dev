+++
title = 'pg_power: initialization and basic setup'
date = 2024-12-19T20:46:06-06:00
draft = false
readTime = true
tags = ['linux', 'postgres']
+++

I have been playing around with the [`powercap framework`](https://www.kernel.org/doc/html/next/power/powercap/powercap.html). I wrote
a postgres extension that shows the energy usage of a query. Postgres has a hook mechanism that allows an extension
to override the default executor. This implementation is very simple: the extension records the current energy reading when a query starts
and then calls the actual executor that runs the query. When the query finishes, a second hook records the current energy reading. The overall
energy usage of this query is the difference between the two values.

The code is here [`pg_power.c`](https://github.com/achanda/pg_power/blob/829480ee2103e117e941e8f0cfd94ac6547d786e/pg_power.c) and here is a
section by section breakdown:

We start with a standard set of includes and the magic block

```C
#include "postgres.h"
#include "fmgr.h"
#include "executor/executor.h"

PG_MODULE_MAGIC;
```

Next we need to define a few statically scoped vars to save the actual executor functions and the energy calculation

```C
static ExecutorStart_hook_type prev_ExecutorStart = NULL;
static ExecutorEnd_hook_type prev_ExecutorEnd = NULL;

static unsigned long long start = 0;
```

We can now define our hooks. `custom_ExecutorStart` will be called when a query starts and `custom_ExecutorEnd` when the query ends.

```C
static void custom_ExecutorStart(QueryDesc *queryDesc, int eflags)
{
    start = read_energy_uj();
    elog(LOG, "Query execution starting: %lld", start);

    if (prev_ExecutorStart)
        prev_ExecutorStart(queryDesc, eflags);
    else
        standard_ExecutorStart(queryDesc, eflags);
}

static void custom_ExecutorEnd(QueryDesc *queryDesc)
{
    unsigned long long current = read_energy_uj();
    elog(LOG, "Query execution completed: %lld", current-start);

    if (prev_ExecutorEnd)
        prev_ExecutorEnd(queryDesc);
    else
        standard_ExecutorEnd(queryDesc);
}
```

The `read_energy_uj` function is a helper that reads the current energy usage from the `powercap` framework.

```C
unsigned long long read_energy_uj() {
    const char* file_path = "/sys/devices/virtual/powercap/intel-rapl/intel-rapl:0/energy_uj";
    FILE* file = fopen(file_path, "r");

    if (file == NULL) {
        elog(LOG, "[ERROR] Failed to open energy file: %s\n", file_path);
        exit(EXIT_FAILURE);
    }

    unsigned long long energy_value;
    int result = fscanf(file, "%llu", &energy_value);

    if (result != 1) {
        elog(LOG, "[ERROR] Failed to read energy value from file: %s\n", file_path);
        fclose(file);
        exit(EXIT_FAILURE);
    }

    fclose(file);
    return energy_value;
}
```

The final few things are the functions that are called when the extension is loaded and unloaded. Here we setup
our hook overrides and set those back when exiting

```C
void _PG_init(void)
{
    prev_ExecutorStart = ExecutorStart_hook;
    ExecutorStart_hook = custom_ExecutorStart;
    prev_ExecutorEnd = ExecutorEnd_hook;
    ExecutorEnd_hook = custom_ExecutorEnd;
}

void _PG_fini(void)
{
    ExecutorStart_hook = prev_ExecutorStart;
    ExecutorEnd_hook = prev_ExecutorEnd;
}
```

One problem here is that an application needs root permissions to be able to read sysfs. However, postgres will refuse
to start up if run as root. A workaround here is to use the capability system to selectively grant read permissions
to the postgres binary. Since the extension is loaded inside postgres, it will inherit that capability and be able to
read sysfs. This approach is described ['here'](https://achanda.dev/posts/til-read-file-without-root/).

Granting read capability to postgres will look like this:

```
sudo setcap cap_dac_read_search=+ep /usr/lib/postgresql/14/bin/postgres
```

A sample session should log the energy usage of each query:

```
abhishek@guest:~$ /usr/lib/postgresql/14/bin/postgres --config-file=$(pwd)/pgdata/postgresql.conf -D $(pwd)/pgdata -k $(pwd)/pgdata
2024-12-18 03:13:00.490 GMT [1751699] LOG:  starting PostgreSQL 14.13 (Ubuntu 14.13-0ubuntu0.22.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
2024-12-18 03:13:00.490 GMT [1751699] LOG:  listening on IPv6 address "::1", port 5432
2024-12-18 03:13:00.490 GMT [1751699] LOG:  listening on IPv4 address "127.0.0.1", port 5432
2024-12-18 03:13:00.492 GMT [1751699] LOG:  listening on Unix socket "/home/abhishek/pgdata/.s.PGSQL.5432"
2024-12-18 03:13:00.494 GMT [1751700] LOG:  database system was shut down at 2024-12-18 03:12:28 GMT
2024-12-18 03:13:00.498 GMT [1751699] LOG:  database system is ready to accept connections
2024-12-18 03:13:09.611 GMT [1751715] LOG:  Query execution starting: 69877598944
2024-12-18 03:13:09.611 GMT [1751715] STATEMENT:  select 1+1;
2024-12-18 03:13:09.611 GMT [1751715] LOG:  Query execution completed: 3785
2024-12-18 03:13:09.611 GMT [1751715] STATEMENT:  select 1+1;
^C2024-12-18 03:13:21.466 GMT [1751699] LOG:  received fast shutdown request
2024-12-18 03:13:21.469 GMT [1751699] LOG:  aborting any active transactions
2024-12-18 03:13:21.469 GMT [1751715] FATAL:  terminating connection due to administrator command
2024-12-18 03:13:21.471 GMT [1751699] LOG:  background worker "logical replication launcher" (PID 1751706) exited with exit code 1
2024-12-18 03:13:21.472 GMT [1751701] LOG:  shutting down
2024-12-18 03:13:21.481 GMT [1751699] LOG:  database system is shut down
```

Note that powercap records the energy usage of the whole system, so each measurement will include all background
processes. But since the background noise should be same for both measurements, those should cancel out when subtracted.

> [CAUTION]
> Do not grant the postgres binary extra capabilities in production, or anywhere near production. This opens up the door
> to a wide range of vulnerabilities.
