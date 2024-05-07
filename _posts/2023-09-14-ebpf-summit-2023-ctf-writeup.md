---
layout: page
title: eBPF Summit 2023 CTF Writeup
permalink: /ebpf-summit-2023-ctf-writeup/
description: |
  Summary of the eBPF Summit 2023 CTF where participants needed to identify
  and kill a specific process reading /etc/passwd to reveal a flag.
---

## 0x00 Introduction

This is the writeup for eBPF Summit 2023 CTF.
The challenge details can be found [here](https://github.com/isovalent/eBPF-Summit-2023-CTF).
The environment is a Ubuntu 22.04 VM.

The goal is to find the process that is reading `/etc/passwd`,
kill that process then the flag will be revealed at `/ebpf.summit`.

## 0x01 Information Gathering

All commands are executed as `root`.

Check current `/ebpf.summit` content :

```shell
cat /ebpf.summit
```

```text
I've been in your kernel for [22.109948 seconds]
```

Check which process is reading `/etc/passwd` :

```shell
lsof /etc/passwd
```

No output, either the process is not holding the fd or it's invisible.

Check if the process in provision is still running:

```shell
ps aux | grep ebpf
```

```text
root       12474  0.0  0.0   6420  1844 pts/1    S+   08:10   0:00 grep --color=auto ebpf
```

Nope, can't find anything.

To monitor it in real time, we can use traditional tools like `inotifywait` or `fatrace`.
But since this is an eBPF challenge, let's use some eBPF tools.

Usually [BCC](https://github.com/iovisor/bcc) or [bpftrace](https://github.com/iovisor/bpftrace)
are my first choice since they are easy to use and have a lot of examples.
In this environment, [Tetragon](https://tetragon.cilium.io/) is also pre-installed, another great tool.
And `bpftool` to further investigate the eBPF programs.

Install tools first:

```shell
apt install bpftrace bpfcc-tools linux-headers-$(uname -r)
```

Check which process is opening `/etc/passwd` using `opensnoop`:

```shell
opensnoop-bpfcc | grep "/etc/passwd"
```

```text
2372   ebpf.summit.202    18   0 /etc/passwd
```

Or a one line `bpftrace` script:

```shell
bpftrace -e 'tracepoint:syscalls:sys_enter_openat /str(args->filename) == "/etc/passwd"/ { printf("%d %s \n", pid, comm); }'
```

```text
Attaching 1 probe...
2372 ebpf.summit.202
```

## 0x02 Exploit

Kill the process

```shell
kill 2372
```

Check the flag

```shell
cat /ebpf.summit
```

```text
You purged the computers of the malware - and not a second too late. Congratula
tions! The location of the base remains a secret. Maybe not for long though, wh
ile everyone was focusing on the computers, Bajeroff Lake, the traitor, managed
 to escape from his cell and stole a shuttle to escape the base. On the radars,
you only see him jump into hyperspace. There's no doubt your paths will cross a
gain one day. Before that, you'll take a day or three off to enjoy a well-deser
ved rest. How about checking in on your giant bees, for a change?

Oh wait, they're just calling all hands on deck: a Rebel squadron fell into an
ambush and is fighting their way out... You'll relax another week!

-------------------------------------------------------------------------------

Thanks for playing the eBPF Summit 2023 Capture the Flag, paste the below code
in the CTF channel on the eBPF Slack!

GJ4xMFOoFRSFES5XFFq7MFO8LKZtnJ9trJ46pvOeMKWhMJjtMz4lVSf0Zl9mZwtmBQLtp7Iwo70xp65X
```

The same method can be used for both easy and hard mode, I haven't checked their differences yet.

Update: Revealed in [eCHO Episode 107: eBPF Summit CTF Walkthrough](https://www.youtube.com/watch?v=PYeRuRtI55M).
Easy mode can expose pid in system log

```shell
$ dmesg
[  523.951058] ebpf.summit.202[3385] is installing a program with bpf_probe_write_user helper that may corrupt user memory!
```

While hard mode will clear system log.

I actually solved it before the event started so I can focus on summit talks.
However, when approaching to the end of the event, I saw a message from host:

```text
For glory points, has anyone cracked the code? 
```

What? That code has some hidden meanings?

It looks like a base64 string, but decode result doesn't make any sense.

It's 5 AM, too tired to do cryptography analysis, so I took a shortcut, reverse engineer the challenge binary.

Load [binary](https://github.com/isovalent/eBPF-Summit-2023-CTF/tree/main/bin) into my favorite RE tool [Cutter](https://cutter.re/),
use `Ghidra` to decompile the `main` function.

```c
            auVar19 = runtime.stringtoslicebyte
                                ((int64_t)&stack0xfffffffffffffd78, auVar19._0_8_, iVar14, arg_8h, arg_10h, 
                                 in_stack_fffffffffffffca8);
            auVar19 = encoding/base64.(*Encoding).EncodeToString
                                (_encoding/base64.StdEncoding, auVar19._0_8_, auVar19._8_8_, iVar14, arg_8h, arg_10h, 
                                 in_stack_fffffffffffffca8, arg_20h);
            auVar19 = main.rot13rot5(auVar19._0_8_, auVar19._8_8_, arg_8h, arg_10h);
```

Ok, the flag is encoded by `base64` then `rot13rot5` ([rot18](https://en.wikipedia.org/wiki/ROT13#Variants)).

To decode it, can use online decoders, or life is short, use python:

```python
import string
import base64

rot18 = str.maketrans(
    {
        c: string.ascii_lowercase[i - 13] for i, c in enumerate(string.ascii_lowercase)
    } | {
        c: string.ascii_uppercase[i - 13] for i, c in enumerate(string.ascii_uppercase)
    } | {
        c: string.digits[i - 5] for i, c in enumerate(string.digits)
    }
)


def decode(s):
    return base64.b64decode(s.translate(rot18)).decode()
```

```python
>>> decode('GJ4xMFOoFRSFES5XFFq7MFO8LKZtnJ9trJ46pvOeMKWhMJjtMz4lVSf0Zl9mZwtmBQLtp7Iwo70xp65X')
"Mode [HARD]\nI've was in your kernel for [93.328386 seconds]\n"
```

Knowing how it’s generated, I can craft an impossible but valid flag:

```python
def encode(s):
    return base64.b64encode(s.encode()).decode().translate(rot18)
```

```python
>>> encode("Mode [HARD]\nI've was in your kernel for [0.000000 seconds]\n")
'GJ4xMFOoFRSFES5XFFq7MFO8LKZtnJ9trJ46pvOeMKWhMJjtMz4lVSfjYwNjZQNjZPOmMJAiozEmKDb='
```

On second thought, is there a way to achieve the impossible without cheating?

How to kill the process when it's accessing `/etc/passwd`?
It's doable with eBPF, one handy tool is `Tetragon`.
Can use a similar policy like [Synchronously stopping the process](https://tetragon.cilium.io/docs/getting-started/try-tetragon-linux/#synchronously-stopping-the-process).

```yaml
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: "kill-passwd-access"
spec:
  kprobes:
  - call: "sys_openat"
    syscall: true
    args:
    - index: 0
      type: "int"
    - index: 1
      type: "string"
    selectors:
    - matchArgs:
      - index: 1
        operator: "Equal"
        values:
        - "/etc/passwd\0"
      matchActions:
      - action: Signal
        argSig: 15
 ```

Start tetragon with this policy:

```shell
tetragon --tracing-policy tracing_policy.yaml
```

Then start the challenge:

```shell
HARDMODE=true /tmp/ebpf.summit.2023
```

Check the flag in `/ebpf.summit`:

```text
GJ4xMFOoFRSFES5XFFq7MFO8LKZtnJ9trJ46pvOeMKWhMJjtMz4lVSfjYwNjZQN8APOmMJAiozEmKDb=
```

Decode:

```python
>>> decode('GJ4xMFOoFRSFES5XFFq7MFO8LKZtnJ9trJ46pvOeMKWhMJjtMz4lVSfjYwNjZQN8APOmMJAiozEmKDb=')
"Mode [HARD]\nI've was in your kernel for [0.000074 seconds]\n"
```

That's close enough!

## 0x03 Additional Analysis

One thing about this challenge is that process is not visible in `ps` output, otherwise it's too easy to find and kill.
How is this achieved?

Hide a process is not a new trick, for example [libprocesshider](https://github.com/gianlucaborello/libprocesshider).
eBPF offensive capabilities are also getting more attention,
like [bad-bpf](https://github.com/pathtofile/bad-bpf) or [boopkit](https://github.com/krisnova/boopkit).

`getdents` can be hooked to hide a process, example [Pid-Hide](https://github.com/pathtofile/bad-bpf#pid-hide).

To see if this is the case, check the eBPF programs:

```shell
bpftool prog show
```

There are some matching ones:

```text
27: tracepoint  name handle_getdents  tag 5fa666123368aa3b  gpl
        loaded_at 2023-09-14T13:55:09+0000  uid 0
        xlated 296B  jited 173B  memlock 4096B  map_ids 37,38
        btf_id 100
28: tracepoint  name handle_getdents  tag 3da1705dc803ad56  gpl
        loaded_at 2023-09-14T13:55:09+0000  uid 0
        xlated 1784B  jited 1274B  memlock 4096B  map_ids 38,39,40,37,41,42
        btf_id 105
29: tracepoint  name handle_getdents  tag 97c6fdabf49fbe34  gpl
        loaded_at 2023-09-14T13:55:09+0000  uid 0
        xlated 520B  jited 295B  memlock 4096B  map_ids 42
        btf_id 106
```

Inspect them to confirm:

```shell
bpftool prog dump x id 27 | grep getdents
```

```text
int handle_getdents_enter(struct trace_event_raw_sys_enter * ctx):
; int handle_getdents_enter(struct trace_event_raw_sys_enter *ctx)
```

```shell
bpftool prog dump x id 28 | grep getdents
```

```text
int handle_getdents_exit(struct trace_event_raw_sys_exit * ctx):
; int handle_getdents_exit(struct trace_event_raw_sys_exit *ctx)
```

```shell
bpftool prog dump x id 29 | grep getdents
```

```text
int handle_getdents_patch(struct trace_event_raw_sys_exit * ctx):
```

It can fool `ps` because `ps` is using `getdents` to get processes info.
But it can't avoid detection from other eBPF tools. Or can it? Sounds familiar?

> You Dare Use My Own Spells Against Me

That's another story for another day.

## 0x04 Conclusion

This is a relatively easy challenge compared to previous years.
Very friendly for beginners to get started with eBPF.
It also contains a lot of fun elements, many thanks to organizers.
Looking forward to next year's challenge!
