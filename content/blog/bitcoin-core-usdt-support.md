---
title: "Userspace, Statically Defined Tracing support for Bitcoin Core"
date: 2021-08-30T09:00:00+02:00
authors:
  - name: "0xB10C"
    github: "0xB10C"
    twitter: "0xB10C"
tags:
  - "Bitcoin Core"
  - "USDT tracepoints"
appearedfirston:
  label: b10c.me
  url: "https://b10c.me/blog/008-bitcoin-core-usdt-support/"
images:
  - /post-data/usdt-support-core/header.png
---

_This report updates on what 0xB10C, [Coinbase Crypto Community Fund grant recipient],
has been working on over the first half of his year-long Bitcoin development grant.
This specifically covers his work on Userspace, Statically Defined Tracing support
for Bitcoin Core. This report was published on [0xB10Cs blog] and the [Coinbase blog] too._

[Coinbase Crypto Community Fund grant recipient]: https://blog.coinbase.com/announcing-our-first-bitcoin-core-developer-grants-3d88559db068
[Coinbase blog]: https://blog.coinbase.com/userspace-statically-defined-tracing-support-for-bitcoin-core-e4076cd3e07
[0xB10Cs blog]: https://b10c.me/blog/008-bitcoin-core-usdt-support/

<!--more-->

The reference implementation to the Bitcoin protocol rules, Bitcoin Core, is
the most widely used software to interact with the Bitcoin network. Bitcoin
Core is, however, a black box to most users. While information can be queried
via the RPC interface or searched in the debug log, there is no defined interface
for real-time insights into process internals. Yet, some users could benefit
from more observability into their node. Hobbyists and companies running Bitcoin
Core in production want to include their nodes in their real-time monitoring.
Developers need visibility into test deployments to evaluate, review, debug,
and benchmark changes. Researchers want to observe and analyze the behavior
of nodes on the peer-to-peer network. Exchanges and other services handling
large sums of bitcoin want to detect attacks and other anomalies early.


### Peeking inside with Userspace, Statically Defined Tracing

<!--Intro eBPF and how it works-->

The [eBPF] technology present in the Linux kernel can be used for
observability into userspace applications. The technology allows
running a small, sandboxed program in the Linux kernel, which can hook
into predefined tracepoints in running processes. Once hooked into a
tracepoint, the program is executed each time the tracepoint is reached.
Tracepoints can pass data, for example, application state. Tracing scripts
can further process the data. The practice of hooking into tracepoints in
userspace applications is known as Userspace, Statically Defined Tracing
(USDT). For example, these tracepoints are also included in PostgreSQL,
MySQL, Python, NodeJS, Ruby, PHP, and libraries like libc, libpthread,
and libvirt.

[eBPF]: https://ebpf.io

<!--Why eBPF, in the way we deploy it, is a good fit for Bitcoin Core-->

The static tracepoints can be leveraged by Bitcoin Core users wishing for
more insights into their node. Adding USDT support [did not require intrusive
changes], and no custom tooling had to be written. When not used, the performance
impact of the tracepoints is minimal to non-existent. Only privileged processes
can hook into the tracepoints, no information leaks to other processes on the host.
These properties make Userspace, Statically Defined Tracing a good fit for Bitcoin
Core.

[did not require intrusive changes]: https://github.com/bitcoin/bitcoin/pull/19866


<!-- example tracepoint -->

For example, I [placed two tracepoints] in the peer-to-peer message handling
code of Bitcoin Core. For each inbound and outbound P2P message, the tracepoints
pass information about the peer, the connection, and the message. This data can
be filtered and processed by tracing scripts. As a demo, I have built a P2P Monitor
that shows the communication between two peers in real-time. Users can find this
script alongside other [USDT examples] in the `contrib/tracing/` directory of the
Bitcoin Core repository.

{{< tweet 1396889721859715077 >}}

[placed two tracepoints]: https://github.com/bitcoin/bitcoin/commit/4224dec22baa66547303840707cf1d4f15a49b20
[USDT examples]: https://github.com/bitcoin/bitcoin/tree/master/contrib/tracing

### Use-cases for Userspace, Statically Defined Tracing

I list some use-cases for Userspace, Statically Defined Tracing
I have thought about or worked on. With only three tracepoints merged,
there is plenty of room for developers to add new tracepoints and
get creative with tracing scripts. [Issue #20981] contains discussion
and ideas for additional tracepoints that can be implemented.

[Issue #20981]: https://github.com/bitcoin/bitcoin/issues/20981


Researchers and developers can use the P2P message tracepoints to
monitor P2P network anomalies in real-time. One example could be
detecting the recent `addr` message flooding as reported in this
[bitcointalk.org post]. The messages were announcing random IP
addresses not belonging to nodes on the Bitcoin network. The flooding
has been [covered][paper] in detail by Grundmann and Baumstark. They
discuss that the attacker could obtain the number of connected peers and
learn about other addresses, including Tor addresses, the node is listening
on. This would reduce the privacy of the node operator. It's important to
stay vigilant to these attacks, discuss them, and then, if needed, react
to them.

[bitcointalk.org post]: https://bitcointalk.org/index.php?topic=5348856.0
[paper]: https://arxiv.org/abs/2108.00815

Similarly, I have been instrumenting the Bitcoin Core network address manager
with tracepoints. The addrman keeps track of gossiped network addresses for
potential outbound peers connections a node makes. It's designed to be resiliant
against [Eclipse Attacks], where a node only has connections to peers controlled
by the attacker. The attacker can choose which information to feed to the node,
enabling, for example, double-spending attacks. Information about the addresses
in the addrman might help detect the build-up of an eclipse attack when combined
with other data.

Additionally, these addrman tracepoints can be helpful during debugging and
code review. To showcase this, I build a tool that visualizes the addresses
in the addrman data structure based on the data submitted to the tracepoints.

{{< tweet 1407019872681332742 >}}

[Eclipse Attacks]: https://cs-people.bu.edu/heilman/eclipse/

<!-- prometheus monitoring -->

A Prometheus metric exporter can also build on top of the tracepoints
without requiring additional code in Bitcoin Core. There already exist
RPC-based Prometheus exporters and projects like [Statoshi]. However,
RPC-based exporters are limited by the information exposed via the RPC
interface, and Statoshi is large a patch-set that requires maintenance
on each Bitcoin Core release. I have published an experimental USDT-based
exporter called [bitcoind-observer] that hooks into the three currently
merged tracepoints and serves metrics in the Prometheus format. The exporter
can be used by everyone currently running a Bitcoin Core node compiled with
USDT support. A demo is available on [bitcoind.observer]. I've recently used
this to benchmark the bandwidth usage of [an implementation] of [Erlay: 
Bandwidth-Efficient Transaction Relay for Bitcoin]. Initial results can be
found [here][erlay-results].

[Statoshi]: https://statoshi.info/
[bitcoind-observer]: https://github.com/0xb10c/bitcoind-observer
[bitcoind.observer]: https://bitcoind.observer
[an implementation]: https://github.com/bitcoin/bitcoin/pull/21515
[Erlay: Bandwidth-Efficient Transaction Relay for Bitcoin]: https://arxiv.org/abs/1905.10518
[erlay-results]: https://github.com/naumenkogs/txrelaysim/issues/8#issuecomment-903255752

The already existing tracepoint `validation:block_connected` can be used
to benchmarking block validation. This allows, for example, to compare
the initial block download performance between different patches and can
aid in detecting performance improvements and regressions. For example,
the [bitcoinperf] project might benefit from such tracepoints. I've used
the tracepoint to [benchmark] Martin Ankerls pull request [#22702]. If
merged, the changes he proposes would result in a substantial block
validation speed up and reduction in memory usage.

[bitcoinperf]: https://github.com/chaincodelabs/bitcoinperf
[benchmark]: https://github.com/bitcoin/bitcoin/pull/22702#issuecomment-900662089
[#22702]: https://github.com/bitcoin/bitcoin/pull/22702

### Next steps

I will collect further ideas for tracepoints and implement them alongside
example tracing scripts and more tooling. This will also involve
communicating with other Bitcoin and Bitcoin Core developers about
which tracepoints could be helpful in their projects. An example is
Antoine Riard's [cross-layer anomaly detection watchdog] which he
initially proposed as a new, internal module to Bitcoin Core. However,
many of the required events and metrics can be collected by hooking into
tracepoints. This means the watchdog could be an external runtime, which
would speed up the watchdog development and requires less code and
maintenance on the Bitcoin Core side.

[cross-layer anomaly detection watchdog]: https://github.com/bitcoin/bitcoin/pull/18987

If everything goes according to plan, the v23.0 release of Bitcoin Core,
expected in early 2022, will include the first set of tracepoints. A
goal is to enable USDT support in release builds by default, which still needs
some work. Additionally, the tracepoint API should be semi-stable and thus
needs testing.


---

**In short**: I have been adding tracepoints to Bitcoin Core that users can hook
into to get insights into the internal state. The tracepoints are based on Linux
kernel technology and do not require intrusive changes or custom tooling. The
groundwork is done. Now further tracepoints can be added, and tooling can be written.

