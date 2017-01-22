---
description: Review of security vulnerabilities Docker mitigated
keywords: Docker, Docker documentation,  security, security non-events
title: Docker security non-events
---

此页列出了使用因使用Docker而避免的安全缺陷：进程运行在容器里而没有受到未修复的安全缺陷的影响。这里假设容器运行的时候没有添加额外的capabilities，并且没有使用`--privileged`

下面的列表远不是全部，只是少数我们所关注的公开披露的缺陷。很大程度上，还有更多没被报道的缺陷。幸运的是，自动Docker默认采用了apparmor、seccomp以及drop capabilities等安全手段，未知的缺陷也和已知的缺陷一样不会造成太大问题。

规避的漏洞:

* [CVE-2013-1956](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2013-1956),
[1957](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2013-1957),
[1958](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2013-1958),
[1959](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2013-1959),
[1979](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2013-1979),
[CVE-2014-4014](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-4014),
[5206](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-5206),
[5207](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-5207),
[7970](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7970),
[7975](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-7975),
[CVE-2015-2925](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-2925),
[8543](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-8543),
[CVE-2016-3134](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-3134),
[3135](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-3135) 等:

允许用户访问原先只有root才能访问的系统调用，比如`mount()`，以此所引入的非特权用户命名空间导致了非特权用户的攻击面激增。所有这些CVEs都是引入用户命名空间导致的安全缺陷的例子。Docker可以使用用户命名空间设置容器，但是使用了默认seccomp profile禁止容器内的进程创建嵌套的命名空间，避免了如下的缺陷被利用：

* [CVE-2014-0181](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0181),
[CVE-2015-3339](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-3339):
这些漏洞要求存在setuid的二进制文件。Docker使用`NO_NEW_PRIVS `进程标志和其他机制禁用了容器内setuid二进制文件。
* [CVE-2014-4699](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-4699):
`ptrace()`里的一个提权漏洞，Docker使用apparmor，seccomp和Drop `CAP_PTRACE`禁用了容器内的`ptrace()`。三倍保护！
* [CVE-2014-9529](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-9529):
通过一系列精心设计的`keyctl()`调用导致内核DoS/内存崩溃。Docker使用seccomp在容器内禁用了`keyctl()`
* [CVE-2015-3214](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-3214),
[4036](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-4036): 这些通用虚拟化驱动程序里的漏洞允许guest OS用户在宿主机系统里执行代码。要利用这些漏洞，需要在guest OS里拥有访问虚拟化设备的权限。对于不带`--privileged`容器，Docker隐藏了对这些设备的直接访问。有意思的是，这个例子说明容器被虚拟机更安全，而不是通常认为的虚拟机比容器更安全。
* [CVE-2016-0728](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-0728):
通过精心设计的`keyctl()`调用引起的Use-after-free提权。Docker使用seccomp profile禁用了容器内的`keyctl()`
* [CVE-2016-2383](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-2383):
eBPF（内核里的DSL，用于表达类似seccomp过滤器）中的一个漏洞允许读取任意内核内存。`bpf()`在容器里被Docker用`seccomp`禁用了。
* [CVE-2016-3134](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-3134),
[4997](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-4997),
[4998](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-4998):
setsockopt里有一个漏洞，在使用选项`IPT_SO_SET_REPLACE`, `ARPT_SO_SET_REPLACE`, 和
`ARPT_SO_SET_REPLACE`时会导致内存崩溃，本地提权。Docker默认通过Drop `CAP_NET_ADMIN`禁用了这几个参数。


没有避免的漏洞:

* [CVE-2015-3290](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-3290),
[5157](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2015-5157): 
内核里的不可屏蔽中断处理允许提权，在容器里也也能利用这个漏洞，因为没禁用系统调用`modify_ldt()`
* [CVE-2016-5195](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2016-5195):
Linux内核里内存子系统在处理copy-on-write时存在竞态条件，会破坏私有只读内存映射，导致本地非特权用户拥有写只读内存的权限。也就是“脏牛”漏洞。*部分避免:* 在一些操作系统上，通过组合使用seccomp过滤掉`ptrace`和挂载只读`/proc/self/mem`可以避免这个漏洞。
