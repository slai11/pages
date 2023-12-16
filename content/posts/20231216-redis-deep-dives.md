---
title: "20231216 Redis Deep Dives"
date: 2023-12-16T13:38:23+08:00
---

I've spent 2023 working on Redis and how the monolith use Redis at GitLab. Some issues I've worked on are simplifying the provisioning process, updating the monolith to be compatible with Redis Cluster and live migrations of data from standalone Redis to a Redis Cluster for various workloads.

Looking at the natural progression of things, I'll probably be involved in scaling the Redis Clusters and automating/democratising the process next year.

But I even then, I don't think I really know that much about Redis. I don't know how `SCAN` actually work in the server-side, how replication works, etc.

So without further ado, I'm going to document my foray into `dictScan`:

### Other readings

I tried reading these few blog posts which gave me a really good idea but I found it hard to really connect what I'm reading with the C code.

- https://engineering.q42.nl/redis-scan-cursor/
- https://www.freecodecamp.org/news/redis-hash-table-scan-explained-537cc8bb9f52?ref=engineering.q42.nl

I started up my `redis-server` and seeded it with 100 keys using a simple hacky bash script:

```bash
#!/bin/bash
for i in {1..100}; do
    key=$(LC_CTYPE=C tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 10)
    value=$(LC_CTYPE=C tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 20)
    redis-cli SET $key $value
done
```

I ran `redis-cli scan 0` and continued iterating along each cursor but I found myself asking questions like "how are each returned cursor linked to the next?".

```
➜ redis-cli scan 0
1) "48"
2)  1) "\xea\xe4G\xe3H\xfdKoaQ"
    2) "f0E\xbeI\xcf\xca\xd6\xed\xf5"
    3) "\xfa\xe80\xbd6\xc6\xd89aQ"
    4) "\xe9\xdcUm\xe4\xc5\xbdkQ\xc3"
    5) "\xc8Ka\xbd\xb9\xd8w\xebE4"
    6) "\xd34\xc4\xd3\xdb\xfc\xd3\xe1S\xe3"
    7) "\xea\xf4p\xf1S\xf2gqT\xe3"
    8) "\xc4Os\xeaip\xbe\xea\xebC"
    9) "Z\xccX\xd2\xc6JP\xe8\xe3\xfc"
   10) "\xefz1W\xfbG\xc2\xeak\xc4"
   11) "\xdb\xff4\xfdsar\xf1jc"
   12) "\xddM\xf5X\xfd\xf6dJwN"
   13) "\xfaoNH\xd5\xc0\xf4u8E"
   14) "\xdasD\xd9\xe7\xce\xfdFEB"
```

The sequence of cursors for full iteration:

```
0
48
76
114
22
65
69
45
123
31
0
```

I converted each number to binary and tried to step through the `dictScan` method blindly. The segment below covers the cursor-advancement logic. `m0` is the mask which is `1111111` or `2^7 - 1` since the dict size needs at least 128 to house 100 keys.

```c
/* Set unmasked bits so incrementing the reversed cursor
 * operates on the masked bits */
v |= ~m0;

/* Increment the reverse cursor */
v = rev(v);
v++;
v = rev(v);

```

But how did 0 become 48? 0 is a unsigned int so 64 bits. Skipping all the messy calculations, shouldn't you end up with 64?

```
v = 0 OR 18446744073709551488 // bitwise NOT of 127 11..10000000

v = rev(v) // we get 144115188075855871 which is 000000011...1 (too many 1s)
v++ // 144115188075855872
v = rev(v) // 64 which is 111111
```

### Figuring it out

Slightly annoyed, I pulled the Redis repository and built my own Redis using the `make` command. To my pleasant surprise, instrumenting the `dictScan` function was easy. Just include the `server.h` file and start peppering the function with `serverLog(LL_NOTICE, ...);`.

Long text ahead, but the TLDR is that my understanding was partially right. The Redis server runs many `dictScan`, hence the cursor I got back is not what I calculated.

```
8249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---Start---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v size: 8
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v before OR: 0
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after OR: 18446744073709551488
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] mask: 127
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after reversed: 144115188075855871
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after increment: 144115188075855872
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] output v: 64
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---END---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---Start---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v size: 8
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v before OR: 64
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after OR: 18446744073709551552
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] mask: 127
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after reversed: 288230376151711743
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after increment: 288230376151711744
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] output v: 32
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---END---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---Start---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v size: 8
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v before OR: 32
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after OR: 18446744073709551520
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] mask: 127
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after reversed: 432345564227567615
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after increment: 432345564227567616
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] output v: 96
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---END---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---Start---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v size: 8
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v before OR: 96
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after OR: 18446744073709551584
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] mask: 127
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after reversed: 576460752303423487
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after increment: 576460752303423488
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] output v: 16
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---END---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---Start---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v size: 8
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v before OR: 16
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after OR: 18446744073709551504
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] mask: 127
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after reversed: 720575940379279359
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after increment: 720575940379279360
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] output v: 80
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---END---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---Start---
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v size: 8
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v before OR: 80
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after OR: 18446744073709551568
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] mask: 127
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after reversed: 864691128455135231
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] v after increment: 864691128455135232
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] output v: 48
28249:M 16 Dec 2023 15:36:02.546 * [scan_debug] ---END---
```

But wait, it ran 6 `dictScan` but returned 14 keys. I'll probably look into that another time.

### So what?

I've not fully understood scans since there is also a whole chunk of comment dedicated to table resizing, which is the real power of `SCAN`, since it guarantees that you will visit all keys even during table resizing (adding and removing keys during the process of iterating).

But that will be another post.
