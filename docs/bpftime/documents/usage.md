# Usage

It's at a early stage and may contain bugs on more platforms and eBPF programs. We are working on to improve the stability and compatibility. If you find any bugs, please feel free to open an issue, thanks!

## Table of Contents

- [Usage](#usage)
  - [Table of Contents](#table-of-contents)
  - [Uprobe and uretprobe](#uprobe-and-uretprobe)
  - [Syscall tracing](#syscall-tracing)
  - [Run with LD\_PRELOAD directly](#run-with-ld_preload-directly)
  - [Run with JIT enabled](#run-with-jit-enabled)
  - [Run with kernel eBPF](#run-with-kernel-ebpf)
  - [Control Log Level](#control-log-level)

## Uprobe and uretprobe

With `bpftime`, you can build eBPF applications using familiar tools like clang and libbpf, and execute them in userspace. For instance, the `malloc` eBPF program traces malloc calls using uprobe and aggregates the counts using a hash map.

You can refer to [documents/build-and-test.md](build-and-test.md) for how to build the project.

To get started, you can build and run a libbpf based eBPF program starts with `bpftime` cli:

```console
make -C example/malloc # Build the eBPF program example
bpftime load ./example/malloc/malloc
```

In another shell, Run the target program with eBPF inside:

```console
$ bpftime start ./example/malloc/test
Hello malloc!
malloc called from pid 250215
continue malloc...
malloc called from pid 250215
```

You can also dynamically attach the eBPF program with a running process:

```console
$ ./example/malloc/test & echo $! # The pid is 101771
[1] 101771
101771
continue malloc...
continue malloc...
```

And attach to it:

```console
$ sudo bpftime attach 101771 # You may need to run make install in root
Inject: "/root/.bpftime/libbpftime-agent.so"
Successfully injected. ID: 1
```

You can see the output from original program:

```console
$ bpftime load ./example/malloc/malloc
...
12:44:35 
        pid=247299      malloc calls: 10
        pid=247322      malloc calls: 10
```

Alternatively, you can also run our sample eBPF program directly in the kernel eBPF, to see the similar output:

```console
$ sudo example/malloc/malloc
15:38:05
        pid=30415       malloc calls: 1079
        pid=30393       malloc calls: 203
        pid=29882       malloc calls: 1076
        pid=34809       malloc calls: 8
```

## Syscall tracing

An example can be found at [benchmark/hash_maps](../benchmark/hash_maps/).

Build the example:

```sh
make -C benchmark/hash_maps
```

Start server:

```sh
$ sudo ~/.bpftime/bpftime load benchmark/hash_maps/opensnoop
[2023-10-01 16:46:43.409] [info] manager constructed
[2023-10-01 16:46:43.409] [info] global_shm_open_type 0 for bpftime_maps_shm
[2023-10-01 16:46:43.410] [info] Closing 3
[2023-10-01 16:46:43.411] [info] mmap64 0
[2023-10-01 16:46:43.411] [info] Calling mocked mmap64
[2023-10-01 16:46:43.411] [info] Closing 3
[2023-10-01 16:46:43.411] [info] Closing 3
[2023-10-01 16:46:43.423] [info] Closing 3
[2023-10-01 16:46:43.423] [info] Closing 3
```

Start victim:

```console
$ sudo ~/.bpftime/bpftime start -s benchmark/hash_maps/victim
[2023-10-01 16:46:58.855] [info] Entering new main..
[2023-10-01 16:46:58.855] [info] Using agent /root/.bpftime/libbpftime-agent.so
[2023-10-01 16:46:58.856] [info] Page zero setted up..
[2023-10-01 16:46:58.856] [info] Rewriting segment from 559a839b4000 to 559a839b5000
[2023-10-01 16:46:58.859] [info] Rewriting segment from 7f130aa22000 to 7f130ab9a000
[2023-10-01 16:46:59.749] [info] Rewriting segment from 7f130acc3000 to 7f130adb0000
[2023-10-01 16:47:00.342] [info] Rewriting segment from 7f130ae9c000 to 7f130afcd000
[2023-10-01 16:47:01.072] [info] Rewriting segment from 7f130b125000 to 7f130b1a3000
.....
[2023-10-01 16:47:02.084] [info] Attach successfully
[2023-10-01 16:47:02.084] [info] Transformer exiting..

Opening test.txt..
VICTIM: get fd 3
VICTIM: closing fd
Opening test.txt..
VICTIM: get fd 3
VICTIM: closing f
```

## Run with LD_PRELOAD directly

If the command line interface is not enough, you can also run the eBPF program with `LD_PRELOAD` directly.

The command line tool is a wrapper of `LD_PRELOAD` and can work with `ptrace` to inject the runtime shared library into a running target process.

Run the eBPF tool with libbpf:

```sh
LD_PRELOAD=build/runtime/syscall-server/libbpftime-syscall-server.so example/malloc/malloc
```

Start the target program to trace:

```sh
LD_PRELOAD=build/runtime/agent/libbpftime-agent.so example/malloc/victim
```

## Run with JIT enabled

If the performance is not good enough, you can try to enable JIT. The JIT will be enabled by default in the future after more tests.

Set `BPFTIME_USE_JIT=true` in the server to enable JIT, for example, when running the server:

```sh
LD_PRELOAD=~/.bpftime/libbpftime-syscall-server.so BPFTIME_USE_JIT=true example/malloc/malloc
```

The default behavior is using ubpf JIT, you can also use LLVM JIT by compile with LLVM JIT enabled. See [documents/build-and-test.md](build-and-test.md) for more details.

## Run with kernel eBPF

You can run the eBPF program in userspace with kernel eBPF in two ways. The kernel must have eBPF support enabled, and kernel version should be higher enough to support mmap eBPF map.

1. with the shared library `libbpftime-syscall-server.so`, for example:

```sh
BPFTIME_NOT_LOAD_PATTERN=start_.* BPFTIME_RUN_WITH_KERNEL=true LD_PRELOAD=~/.bpftime/libbpftime-syscall-server.so example/malloc/malloc
```

2. Using daemon mode, see <https://github.com/eunomia-bpf/bpftime/tree/master/daemon>

## Control Log Level

Set `SPDLOG_LEVEL` to control the log level dynamically, for example, when running the server:

```sh
SPDLOG_LEVEL=Debug LD_PRELOAD=~/.bpftime/libbpftime-syscall-server.so example/malloc/malloc
```
