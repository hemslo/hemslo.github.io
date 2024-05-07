---
layout: page
title: eBPF Summit 2022 CTF Challenge 3 Writeup
permalink: /ebpf-summit-2022-ctf-challenge-3-writeup/
description: |
  A detailed writeup of the third challenge at the eBPF Summit 2022 CTF,
  focusing on troubleshooting network issues using Kubernetes and Cilium.
---

## 0x00 Introduction

This is the writeup for the third challenge of eBPF Summit 2022 CTF.
See [writeup for challenge 2](/ebpf-summit-2022-ctf-challenge-2-writeup/) for the previous setup.
The challenge details can be found [here](https://github.com/isovalent/eBPF-Summit-2022-CTF#challenge-3).

## 0x01 Information Gathering

The first step is to find out what's running in the cluster.

Note: if you shell into the VM, you need to set `KUBECONFIG` before using `kubectl`.

```shell
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

```shell
$ kubectl get all
NAME                               READY   STATUS    RESTARTS   AGE
pod/challenge-3-7465fc8b59-vwgwf   1/1     Running   0          3h

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   3h

NAME                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/challenge-3   1/1     1            1           3h

NAME                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/challenge-3-7465fc8b59   1         1         1       3h
```

There is only one pod running, let's take a look at it.

```shell
$ kubectl describe pod/challenge-3-7465fc8b59-vwgwf

...
Init Containers:
  ctfinit:
    Container ID:   containerd://20c218f7e87b78f947522ac7d5228282c45714562d5fa584ba2877f6d1b2651c
    Image:          ghcr.io/lizrice/ctfinit:drop
    Image ID:       ghcr.io/lizrice/ctfinit@sha256:70d741f7a58720c500348b7be70f5bed0d30bd4b63765404e85d651d061e5b49
...
Containers:
  ctfapp:
    Container ID:   containerd://19f4d67df58afe4889a504d11684a5696f9ce008955962bc05c317153038ef99
    Image:          ghcr.io/lizrice/ctfapp:drop
    Image ID:       ghcr.io/lizrice/ctfapp@sha256:78934f9a1d4b4a70ec736f067c4b19e347b442232a8870da898d03b019e3f252
    Port:           <none>
    Host Port:      <none>
...
```

Similar to the previous challenge, there are two containers running in the pod `ctfinit` and `ctfapp`.
Let's check the logs.

```shell
$ kubectl logs challenge-3-7465fc8b59-vwgwf -c ctfinit
Init complete
$ kubectl logs challenge-3-7465fc8b59-vwgwf -c ctfapp
I need internet access so I can learn about eBPF! But I'm getting errors:
get: Get "https://docs.cilium.io/en/stable/bpf/#xdp": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
I'll give you another update in around 30 seconds

I need internet access so I can learn about eBPF! But I'm getting errors:
get: Get "https://docs.cilium.io/en/stable/bpf/#xdp": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
I'll give you another update in around 30 seconds
```

Looks like the `ctfapp` container is trying to access the internet but failing.

Let's try to validate the assumption by running a shell in the container.

```shell
$ kubectl exec -ti challenge-3-7465fc8b59-vwgwf -c ctfapp -- sh
error: Internal error occurred: error executing command in container: failed to exec in container: failed to start exec "d87692ae7dd5dd7be67e5d3f8afc4264a9fe6637436d18c50745203b8069af75": OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH: unknown
```

Oops, no shell, let's try `kubectl debug`

```shell
$ kubectl debug -ti challenge-3-7465fc8b59-vwgwf --target ctfapp --image=busybox
$ wget https://docs.cilium.io/en/stable/bpf/#xdp
Connecting to docs.cilium.io (104.17.33.82:443)
```

No access to the internet, try the same command in VM:

```shell
$ wget https://docs.cilium.io/en/stable/bpf/#xdp
Resolving docs.cilium.io (docs.cilium.io)... 104.17.32.82, 104.17.33.82
Connecting to docs.cilium.io (docs.cilium.io)|104.17.32.82|:443... connected.
HTTP request sent, awaiting response... 200 OK
```

So the host can access the internet but the pod cannot.

Can it be some bpf prog blocking the traffic? Let's check some bpf logs.

```shell
$ sudo bpftool prog tracelog
...
     ksoftirqd/3-33      [003] d.s.1 14236.774115: bpf_trace_printk: Dropping TCP packet

     ksoftirqd/3-33      [003] d.s.1 14239.131882: bpf_trace_printk: Dropping TCP packet
```

Looks like there is something dropping packet. Is there any xdp prog?

```shell
$ sudo bpftool prog show | grep xdp
1301: xdp  name a85d49  tag fc274aabdac1d306  gpl
```

What is it?

```shell
$ sudo bpftool prog dump x id 1301
int a85d49(struct xdp_md * ctx):
; int a85d49(struct xdp_md *ctx)
   0: (b7) r0 = 0
; void *data = (void *)(long)ctx->data;
   1: (79) r2 = *(u64 *)(r1 +0)
; void *data_end = (void *)(long)ctx->data_end;
   2: (79) r1 = *(u64 *)(r1 +8)
; if (data + sizeof(struct ethhdr) > data_end)
   3: (bf) r3 = r2
   4: (07) r3 += 14
; if (data + sizeof(struct ethhdr) > data_end)
   5: (2d) if r3 > r1 goto pc+11
   6: (bf) r3 = r2
   7: (07) r3 += 34
   8: (2d) if r3 > r1 goto pc+8
   9: (b7) r0 = 2
; if (iph->protocol == IPPROTO_ICMP) {
  10: (71) r1 = *(u8 *)(r2 +23)
; if (iph->protocol == IPPROTO_ICMP) {
  11: (55) if r1 != 0x6 goto pc+5
; bpf_printk("Dropping TCP packet\n");
  12: (18) r1 = map[id:225][0]+0
  14: (b7) r2 = 21
  15: (85) call bpf_trace_printk#-73936
  16: (b7) r0 = 1
; }
  17: (95) exit
```

OK, this is the one blocking the traffic. Where is it attached?

```shell
$ ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000
    link/ether 52:55:55:d5:a0:ff brd ff:ff:ff:ff:ff:ff
    altname enp0s1
3: cilium_net@cilium_host: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c2:6c:b4:f1:82:c0 brd ff:ff:ff:ff:ff:ff
4: cilium_host@cilium_net: <BROADCAST,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether c6:93:c4:1a:1c:d4 brd ff:ff:ff:ff:ff:ff
5: cilium_vxlan: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 46:29:aa:ab:06:ff brd ff:ff:ff:ff:ff:ff
7: lxc_health@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ba:3a:b1:93:73:b7 brd ff:ff:ff:ff:ff:ff link-netnsid 0
9: lxc6deee4986637@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether d2:33:63:ce:63:10 brd ff:ff:ff:ff:ff:ff link-netns cni-2dce3bd9-38d2-cc9e-26d0-9115ed5611ea
11: lxc2fdeff222904@if10: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 2e:5f:c2:91:4b:ba brd ff:ff:ff:ff:ff:ff link-netns cni-52ff6a2f-66c3-b385-5ed0-f5e922a87859
13: lxc618a1255c457@if12: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ea:ad:91:a5:08:3d brd ff:ff:ff:ff:ff:ff link-netns cni-e1de76c3-6d1f-e7e0-639a-f7f99a27fc52
19: lxc99d1afd98d54@if18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether fe:93:ea:ff:53:56 brd ff:ff:ff:ff:ff:ff link-netns cni-e9934a9c-656a-2a38-e26a-579c46a82af8
21: lxcec181dd23b0d@if20: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether fa:f9:92:3b:83:9a brd ff:ff:ff:ff:ff:ff link-netns cni-f96d09b0-c1c4-dc19-2ff8-1ffdf0bfe0e1
23: lxc9fd8b423732d@if22: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether d6:b6:78:a3:74:ce brd ff:ff:ff:ff:ff:ff link-netns cni-fc740045-2b6d-5855-22a3-5545722ddf8d
25: lxc52fa0150481e@if24: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 12:83:88:f5:88:07 brd ff:ff:ff:ff:ff:ff link-netns cni-366802eb-f758-2697-1ffa-f4f5b2d9636b
```

It's not attached to any interface from host, let's see the other side of the veth pair.

```shell
$ sudo ip -all netns exec ip link show
...
netns: cni-52ff6a2f-66c3-b385-5ed0-f5e922a87859
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 52:eb:c4:6c:66:41 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    prog/xdp id 1301 tag fc274aabdac1d306 jited
...
```

Got you.

Note I did a shortcut by showing all namespaces, for precise tracing, we can also use `nsenter`.
First we need to know the pid of the container `ctfapp` in host.

```shell
$ sudo crictl inspect 19f4d67df58afe4889a504d11684a5696f9ce008955962bc05c317153038ef99
...
  "info": {
    "sandboxID": "10f228ff47aaa79b329e15797144ce347244333659d1a2e08c39d3e9688d3252",
    "pid": 10481,
...
```

Then we can use `nsenter` to enter the namespace of the container.

```shell
sudo nsenter -t 10481 -n ip link show
```

## 0x02 Exploit

Detach the xdp program from the interface.

```shell
sudo ip netns exec cni-52ff6a2f-66c3-b385-5ed0-f5e922a87859 bpftool net detach xdp dev eth0
```

Check again

```shell
$ sudo ip netns exec cni-52ff6a2f-66c3-b385-5ed0-f5e922a87859 ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdpgeneric qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 52:eb:c4:6c:66:41 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    prog/xdp id 1301 tag fc274aabdac1d306 jited
```

It's still there. Because it's `xdpgeneric`, we need to do this.

```shell
$ sudo ip netns exec cni-52ff6a2f-66c3-b385-5ed0-f5e922a87859 bpftool net detach xdpgeneric dev eth0
$ sudo ip netns exec cni-52ff6a2f-66c3-b385-5ed0-f5e922a87859 ip link show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
10: eth0@if11: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 52:eb:c4:6c:66:41 brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

Done! Let's check the log.

```shell
$ kubectl logs challenge-3-7465fc8b59-vwgwf -c ctfapp
The first flag is ALDERAAN
Now I've got internet access, but I'm distracted by looking at https://www.starwars.com. Make me focus on reading about Cilium at docs.cilium.io.
I'll give you another update in around 30 seconds
```

Now we get the first flag, the next part is to limit internet access to `docs.cilium.io` only.
That's the exact use case for [FQDN cilium network policy](https://docs.cilium.io/en/latest/security/dns/#gs-dns).
[Cilium Network Policy Editor](https://editor.cilium.io/) is a great tool to help writing the policy.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: ctf-policy
spec:
  endpointSelector: {}
  egress:
    - toEndpoints:
        - {}
    - toEntities:
        - cluster
    - toEndpoints:
        - matchLabels:
            io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: UDP
          rules:
            dns:
              - matchPattern: "*"
    - toFQDNs:
        - matchName: docs.cilium.io
```

Apply the policy.

```shell
kubectl apply -f ctf-policy.yaml
```

Then check the log again.

```shell
$ kubectl logs challenge-3-7465fc8b59-vwgwf -c ctfapp -f
I need internet access so I can learn about eBPF! But I'm getting errors:
get: Get "https://docs.cilium.io/en/stable/bpf/#xdp": context deadline exceeded (Client.Timeout exceeded while awaiting headers)
I'll give you another update in around 30 seconds```
```

Looks like it's blocked again. Due to how FQDN policy works, we need to wait for a while for the DNS cache to expire.
Because data plane only understand ip address, it needs to resolve the domain name to ip address first.
Once there is new DNS request, the ip cache will be refreshed.

Or we can just send the DNS request manually.
Back to the debug shell again.

```shell
$ # kubectl debug -ti challenge-3-7465fc8b59-vwgwf --target=ctfapp --image=busybox
$ nslookup docs.cilium.io
Server:  10.43.0.10
Address: 10.43.0.10:53

Non-authoritative answer:
Name: docs.cilium.io
Address: 104.17.32.82
Name: docs.cilium.io
Address: 104.17.33.82

$ wget https://docs.cilium.io
Connecting to docs.cilium.io (104.17.33.82:443)
wget: note: TLS certificate validation not implemented
Connecting to docs.cilium.io (104.17.33.82:443)
saving to 'index.html'

$ wget https://www.starwars.com
Connecting to www.starwars.com (149.135.80.25:443)
```

Then check the log.

```shell
$ kubectl logs challenge-3-7465fc8b59-vwgwf -c ctfapp -f
...
The second flag is DANTOOINE
```

Case closed.

![image](/assets/images/ebpf-summit-ctf-3.png)

## 0x03 Conclusion

This challenge is a little harder than the previous one, it requires some knowledge about XDP and Cilium network policy.
There are 2 good eCHO episodes covering both.

* [eCHO episode 13: XDP Hands-on Tutorial](https://youtu.be/YUI78vC4qSQ)
* [eCHO Episode 43: Deep dive on FQDN Policy](https://youtu.be/iJ98HRZi8hM)
