+++
title = 'TIL: Looking at RocksDB Events'
date = 2024-12-22T21:47:47-06:00
draft = false
readTime = true
tags = ['rocksdb']
+++

RocksDB has an [`eventing system`](https://github.com/facebook/rocksdb/wiki/EventListener) that clients can use to listen on
specific events within the DB. These events are written to the log file when RocksDB runs. The name of the log file is `LOG`
by default and is located in the database directory.

The whole log file is not in json format, so we need to grep to convert the output to json. Here is a bash one-liner that
continuously tails the log file and prints out the events.

```
tail -f db/LOG | grep --line-buffered -oP '{.*}' | jq --unbuffered .
```
