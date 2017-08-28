Catfs is a caching filesystem written in Rust.

[![Build Status](https://travis-ci.org/kahing/catfs.svg?branch=master)](https://travis-ci.org/kahing/catfs)
[![Crates.io](https://img.shields.io/crates/v/catfs.svg)](https://crates.io/crates/catfs)
[![Crates.io Downloads](https://img.shields.io/crates/d/catfs.svg)](https://crates.io/crates/catfs)

# Overview

Catfs allows you to have cached access to another (possibily remote)
filesystem. Caching semantic is read-ahead and write-through (see
[Current Status](#current-status)). Currently it only provides a data
cache and all metadata operations hit the source filesystem.

Catfs is ALPHA software. Don't use this if you value your data.

# Installation

* On Linux, install via
  [pre-built binaries](https://github.com/kahing/catfs/releases/). You
  may also need to install fuse-utils first.

* Or build from source which requires [Cargo](http://doc.crates.io/).

```ShellSession
:~/catfs$ cargo install catfs
$ # optimized binary now in $HOME/.cargo/bin/catfs
```

# Usage

Catfs requires extended attributes (xattr) to be enabled on the
filesystem where files are cached to. Typically this means you need to
have `user_xattr` mount option turned on.

```ShellSession
$ catfs <from> <to> <mountpoint>
```

Catfs will expose files in `<from>` under `<mountpoint>`, and cache
them to `<to>` as they are accessed. You can use `--free` to control
how much free space `<to>`'s filesystem has.

# Benchmark

Compare using catfs to cache sshfs vs sshfs only. Topology is
laptop - 802.11n - router - 1Gbps wired - desktop. Laptop has SSD
whereas desktop has spinning rust.

![Benchmark result](/bench/bench.catfs_vs_sshfs.png?raw=true "Benchmark")

Compare running catfs with two local directories on the same
filesystem with direct access. This is not a realistic use case but
should give you an idea of the worst case slowdown.

![Benchmark result](/bench/bench.png?raw=true "Benchmark")

Write is twice as slow as expected since we are writing twice the
amount.

<a name="runbenchmark"></a>
To run the benchmark, do:

```ShellSession
$ sudo docker run -e SSHFS_SERVER=user@host --rm --privileged --net=host -v $PWD/target:/root/catfs/target kahing/catfs-bench
 # result is written to $PWD/target
```

The docker container will need to be able to ssh to `user@host`. Typically I arrange that by mounting the ssh socket from the host

```ShellSession
$ sudo docker run -e SSHFS_OPTS="-o ControlPath=/root/.ssh/sockets/%r@%h_%p -o ControlMaster=auto -o StrictHostKeyChecking=no -o Cipher=arcfour user@host:/tmp" -e SSHFS_SERVER=user@host --rm --privileged --net=host -v $HOME/.ssh/sockets:/root/.ssh/sockets  -v $PWD/target:/root/catfs/target kahing/catfs-bench
```

# License

Copyright (C) 2017 Ka-Hing Cheung

Licensed under the Apache License, Version 2.0

# Current Status

Catfs is ALPHA software. Don't use this if you value your data.

Entire file is cached if it's open for read, even if nothing is
actually read.

Data is first written to the cache and the entire file is always
written back to the original filesystem on `close()`, so effectively
it's a write-through cache. Note that even changing one byte will
cause the entire file to be re-written.

Paging in/writeback are done in background threads. All other requests
are serviced on the same thread, so many operations could block each
other.

## TODO

* move all operations to background threads

# References

* Catfs is designed to work with [goofys](https://github.com/kahing/goofys/)
* [FS-Cache](https://www.kernel.org/doc/Documentation/filesystems/caching/fscache.txt)
  provides caching for some in kernel filesystems but doesn't support
  other FUSE filesystems.
* Other similar fuse caching filesystems, no idea about their completeness:
  * [fuse-cache](https://sourceforge.net/projects/fuse-cache/)
  * [mcachefs](https://github.com/Doloops/mcachefs)
  * [pcachefs](https://github.com/ibizaman/pcachefs)
