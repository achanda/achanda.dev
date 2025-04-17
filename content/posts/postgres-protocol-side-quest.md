+++
title = 'Postgres Protocol and a Side Quest'
date = 2025-04-10T22:24:49-05:00
draft = false
readTime = true
tags = ['postgres']
+++

I was watching Heikki Linnakangas's [talk](https://www.youtube.com/watch?v=FBPubrwGKhI) on the Postgres protocol. I wanted to 
follow along with and see the protocol in action for myself. I ran into a problem here: my ostgres server rejected all non secure
connections. And with TLS enabled, I could not get wireshark to decode postgres protocol messages because of encryption.

Now, wireshark supports decoding TLS connections using a [key log file](https://wiki.wireshark.org/TLS#key-log-format). Looking
through documentation I realized that there is no way for a client to  log the TLS keys using psql. So the natural next step
was to add that support and try to get that accepted upstream in postgres. Once I understood how this could be implemented, this
became my side quest.

This [patch](https://github.com/postgres/postgres/commit/2da74d8d6400975bf73fb0df97c3943ad3ed9a36) was merged on 3rd April 2025. 
This allows a client to pass in a file name where psql will dump encryption secrets. Here is how to use it

```
psql "postgresql://user:pass@server:5432/postgres?sslmode=require&sslkeylogfile=/home/abhishek/foo.txt"
psql (18devel, server 17.4)
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: postgresql)
Type "help" for help.

postgres=>
```

And now when I configure Wireshark to read that file, it can successfully decode the postgres protocol

![wireshark](wireshark.png)

Happy debugging the postgres protocol!
