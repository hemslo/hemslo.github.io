---
layout: page
title: eBPF Summit 2022 CTF Challenge 2 Writeup
permalink: /ebpf-summit-2022-ctf-challenge-2-writeup/
---

## 0x00 Introduction

This is the writeup for the second challenge of eBPF Summit 2022 CTF.
The challenge details can be found [here](https://github.com/isovalent/eBPF-Summit-2022-CTF#challenge-2).
The environment is a Ubuntu 22.04 VM with `k3s` and `cilium` installed.
The challenges are deployed as a kubernetes pod.

## 0x01 Information Gathering

The first step is to find out what's running in the cluster.

Note: if you shell into the VM, you need to set `KUBECONFIG` before using `kubectl`.

```shell
$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

```shell
$ kubectl get all -n challenge-2
NAME                               READY   STATUS    RESTARTS   AGE
pod/challenge-2-7f9ccf4686-vc4ch   1/1     Running   0          22m

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/challenge-2   1/1     1            1           22m

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/challenge-2-7f9ccf4686   1         1         1       22m
```

There is only one pod running, let's take a look at it.

```shell
$ kubectl describe pod/challenge-2-7f9ccf4686-vc4ch -n challenge-2

...
Init Containers:
  ctfinit:
    Container ID:   containerd://5cb67fadd957fcfef571e6d3de06d431c340e55e572d13b9d02143d7df0d8bb9
    Image:          ghcr.io/lizrice/ctfinit:map
    Image ID:       ghcr.io/lizrice/ctfinit@sha256:f24a2c9942e6cbe0ab084a2f8a42abed347f798f9752cc469dd3149918eeeb04
...
Containers:
  ctfapp:
    Container ID:   containerd://b7f4fcc56405f4205544adcfd56a17e47c6e46ad3cf087d73c7b9b17a1e2513b
    Image:          ghcr.io/lizrice/ctfapp:map
    Image ID:       ghcr.io/lizrice/ctfapp@sha256:9aef2e2c1bbf49baf8b55b012edc595f369899c438de8f6415325c4b381b99bd
    Port:           <none>
    Host Port:      <none>
```

There are two containers in the pod, `ctfinit` and `ctfapp`.
Let's check the logs.

```shell
$ kubectl logs pod/challenge-2-7f9ccf4686-vc4ch -n challenge-2 -c ctfinit

Init complete
```

```shell
$ kubectl logs pod/challenge-2-7f9ccf4686-vc4ch -n challenge-2 -c ctfapp

libbpf: loading object 'map_bpf' from buffer
libbpf: elf: section(3) tp/syscalls/sys_enter_fchmodat, size 392, link 0, flags 6, type=1
libbpf: sec 'tp/syscalls/sys_enter_fchmodat': found program 'hello' at insn offset 0 (0 bytes), code size 49 insns (392 bytes)
libbpf: elf: section(4) .reltp/syscalls/sys_enter_fchmodat, size 64, link 13, flags 40, type=9
libbpf: elf: section(5) .maps, size 32, link 0, flags 3, type=1
libbpf: elf: section(6) .rodata, size 53, link 0, flags 2, type=1
libbpf: elf: section(7) license, size 4, link 0, flags 3, type=1
libbpf: license of map_bpf is GPL
libbpf: elf: section(8) .BTF, size 1398, link 0, flags 0, type=1
libbpf: elf: section(10) .BTF.ext, size 384, link 0, flags 0, type=1
libbpf: elf: section(13) .symtab, size 240, link 1, flags 0, type=2
libbpf: looking for externs among 10 symbols...
libbpf: collected 0 externs total
libbpf: map 'ctf_flag': at sec_idx 5, offset 0.
libbpf: map 'ctf_flag': found type = 1.
libbpf: map 'ctf_flag': found key [8], sz = 4.
libbpf: map 'ctf_flag': found value [11], sz = 12.
libbpf: map 'ctf_flag': found max_entries = 10240.
libbpf: map 'map_bpf.rodata' (global data): at sec_idx 6, offset 0, flags 480.
libbpf: map 1 is "map_bpf.rodata"
libbpf: sec '.reltp/syscalls/sys_enter_fchmodat': collecting relocation for section(3) 'tp/syscalls/sys_enter_fchmodat'
libbpf: sec '.reltp/syscalls/sys_enter_fchmodat': relo #0: insn #27 against 'ctf_flag'
libbpf: prog 'hello': found map 0 (ctf_flag, sec 5, off 0) for insn #27
libbpf: sec '.reltp/syscalls/sys_enter_fchmodat': relo #1: insn #31 against '.rodata'
libbpf: prog 'hello': found data map 1 (map_bpf.rodata, sec 6, off 0) for insn 31
libbpf: sec '.reltp/syscalls/sys_enter_fchmodat': relo #2: insn #36 against '.rodata'
libbpf: prog 'hello': found data map 1 (map_bpf.rodata, sec 6, off 0) for insn 36
libbpf: sec '.reltp/syscalls/sys_enter_fchmodat': relo #3: insn #42 against '.rodata'
libbpf: prog 'hello': found data map 1 (map_bpf.rodata, sec 6, off 0) for insn 42
libbpf: map 'ctf_flag': created successfully, fd=4
libbpf: map 'map_bpf.rodata': created successfully, fd=5
```

From the log we can find several interesting things:

  * The `ctfapp` container is running a BPF program.
  * The BPF program is using
    * a map named `ctf_flag`.
    * a prog named `hello`.
    * a hook named `syscalls/sys_enter_fchmodat`.

Inspect them one by one from VM.

```shell
$ sudo bpftool map show
...
221: hash  name summit2022  flags 0x0
	key 4B  value 20B  max_entries 4  memlock 4096B
225: array  name packetdr.rodata  flags 0x480
	key 4B  value 21B  max_entries 1  memlock 4096B
	btf_id 295  frozen
229: hash  name ctf_flag  flags 0x0
	key 4B  value 12B  max_entries 10240  memlock 163840B
	btf_id 301
231: array  name map_bpf.rodata  flags 0x480
	key 4B  value 53B  max_entries 1  memlock 4096B
	btf_id 301  frozen
```

```shell
$ sudo bpftool map dump name ctf_flag
[]
```

It's empty. Any other clues?

```shell
$ sudo bpftool map dump name summit2022
key:
01 00 00 00
value:
52 75 6e 20 63 68 6d 6f  64 20 6f 6e 20 61 20 66
69 6c 65 20
Found 1 element
```

Decode those hex values. I usually use [pwntools](https://github.com/Gallopsled/pwntools) in ipython.

```python
>>> from pwn import *
>>> unhex('52 75 6e 20 63 68 6d 6f  64 20 6f 6e 20 61 20 66 69 6c 65 20'.replace(' ', ''))
b'Run chmod on a file '
```

TIL another trick from [eBPF 2022 Summit - Day 2](https://youtu.be/a3AwA1VdohU?t=9534)

```shell
$ xxd -r -p
52 75 6e 20 63 68 6d 6f  64 20 6f 6e 20 61 20 66 69 6c 65 20
Run chmod on a file
```

Looks like we got an instruction. Before we try it, let's have a look at prog `hello`.

```shell
$ sudo bpftool prog show | grep hello
1310: tracepoint  name hello  tag b87e7017b07e3151  gpl
$ sudo bpftool prog dump x name hello
int hello(void * ctx):
; int hello(void *ctx)
   0: (b7) r1 = 0
; value[5] = 0;
   1: (73) *(u8 *)(r10 -21) = r1
   2: (b7) r1 = 68
; value[3] = 68;
   3: (73) *(u8 *)(r10 -23) = r1
   4: (b7) r1 = 45
; value[2] = 45;
   5: (73) *(u8 *)(r10 -24) = r1
   6: (b7) r1 = 50
; value[4] = 50;
   7: (73) *(u8 *)(r10 -22) = r1
; value[1] = 50;
   8: (73) *(u8 *)(r10 -25) = r1
   9: (b7) r1 = 82
; value[0] = 82;
  10: (73) *(u8 *)(r10 -26) = r1
  11: (b7) r1 = 1
; __u32 key = 1;
  12: (63) *(u32 *)(r10 -32) = r1
  13: (bf) r1 = r10
;
  14: (07) r1 += -20
; bpf_get_current_comm(&comm, sizeof(comm));
  15: (b7) r2 = 20
  16: (85) call bpf_get_current_comm#180688
; if (comm[0] == 'c' && comm[1] == 'h' && comm[4] == 'd') {
  17: (71) r1 = *(u8 *)(r10 -20)
; if (comm[0] == 'c' && comm[1] == 'h' && comm[4] == 'd') {
  18: (55) if r1 != 0x63 goto pc+16
  19: (71) r1 = *(u8 *)(r10 -19)
  20: (55) if r1 != 0x68 goto pc+14
  21: (71) r1 = *(u8 *)(r10 -16)
  22: (55) if r1 != 0x64 goto pc+12
  23: (bf) r2 = r10
;
  24: (07) r2 += -32
  25: (bf) r3 = r10
  26: (07) r3 += -26
; bpf_map_update_elem(&ctf_flag, &key, &value, 0);
  27: (18) r1 = map[id:229]
  29: (b7) r4 = 0
  30: (85) call htab_map_update_elem#213200
; bpf_printk("The flag is now available\n");
  31: (18) r1 = map[id:231][0]+0
  33: (b7) r2 = 27
  34: (85) call bpf_trace_printk#-73936
; cgroup_id = bpf_get_current_cgroup_id();
  35: (85) call bpf_get_current_cgroup_id#182432
; bpf_printk("Cgroup: %d", cgroup_id);
  36: (18) r1 = map[id:231][0]+27
  38: (b7) r2 = 11
  39: (bf) r3 = r0
  40: (85) call bpf_trace_printk#-73936
; pid = bpf_get_current_pid_tgid();
  41: (85) call bpf_get_current_pid_tgid#180224
; bpf_printk("Pid / tgid: %d", pid);
  42: (18) r1 = map[id:231][0]+38
  44: (b7) r2 = 15
  45: (bf) r3 = r0
  46: (85) call bpf_trace_printk#-73936
; return 0;
  47: (b7) r0 = 0
  48: (95) exit
```

By reading the code, we can see the main logic for this tracepoint is:
  * Check if the process name is `ch**d`.
  * If yes, update the map `ctf_flag` with key `1` and value `82 50 45 68 50 0`.

```python
>>> list(map(chr, [82,50,45,68,50,0]))
['R', '2', '-', 'D', '2', '\x00']
```

So we actually got the flag already, but let's keep playing.


### 0x02 Exploit

```shell
$ touch a.txt
$ chmod 777 a.txt
```

let's take a look at logs.

```shell
$ sudo bpftool prog tracelog
...
           chmod-6907    [001] d...1  5219.880346: bpf_trace_printk: The flag is now available

           chmod-6907    [001] d...1  5219.880355: bpf_trace_printk: Cgroup: 4089
           chmod-6907    [001] d...1  5219.880356: bpf_trace_printk: Pid / tgid: 6907
...
```

We can see the flag is available now. Let's read it.

```shell
$ sudo bpftool map dump name ctf_flag
[{
        "key": 1,
        "value": {
            "flag": "R2-D2"
        }
    }
]
```

Case closed.

![image](/assets/images/ebpf-summit-ctf-2.png)

### 0x03 Conclusion

This is an easy challenge, but it covers the basic usage of eBPF.
Especially, the handy tool `bpftool`,
the best learning resource is this [youtube video](https://youtu.be/1EOLh3zzWP4) by it's creator Quentin Monnet.

Now we proceed to the next challenge.
