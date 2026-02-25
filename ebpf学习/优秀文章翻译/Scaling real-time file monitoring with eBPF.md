File integrity monitoring (FIM) helps teams detect unauthorized changes to sensitive files and is a critical part of any security posture. Yet building an FIM system that works reliably across modern, large-scale infrastructure is harder than it looks.

When we set out to build an FIM system that could handle the realities of Datadog’s product environments, we quickly realized that existing approaches weren’t going to cut it. Periodic filesystem scans, for example, seem simple and reliable on paper. In practice, they miss the kinds of events we care about most: If an attacker tampers with a file and reverts the change before the next scan, it’s as if nothing ever happened. Even when a scan does catch something, it only tells us that a file changed—not how it changed, why it changed, or who changed it. For an engineering team focused on giving security teams actionable insights, that lack of visibility was a dealbreaker.

Even legacy event-based Linux monitoring technologies come with significant drawbacks. `inotify` lacks the necessary system-level context to correlate file events with processes and containers, while `auditd`—although more comprehensive—often struggles with high performance overhead and scalability under heavy system loads.

To get to the level of context and scalability we needed, we turned to [eBPF](https://ebpf.io/). It gave us a way to observe real-time file activity directly from the kernel, without sacrificing stability or safety. With eBPF, we could see not only that a file was changed but also which process triggered the change and what container it ran in, providing us with meaningful insights for security investigations.

That visibility came at a cost. We were now dealing with an overwhelming volume of data: more than **10 billion file-related events per minute** across all of Datadog’s infrastructure on a regular Friday afternoon. Processing that stream, without dropping events or degrading host performance, became one of the hardest technical challenges we had to solve.

In this post, we’ll share how we tackled that challenge, including:

- Why eBPF gave us the observability we needed where other tools fell short
- How we scaled to billions of events per minute without overwhelming our Agents or backend
- The techniques we used to pre-filter 94% of events directly in the kernel, without losing important signals

## Managing load at the edge: Scaling eBPF-powered file monitoring

Once we unlocked deep visibility into filesystem activity with eBPF, we faced another challenge: scale. Collecting every file event across Datadog’s infrastructure generated an overwhelming amount of data, far more than we could reasonably send to the backend.

Each file activity event carries critical context, such as the process that triggered it, the container it ran in, and other metadata. Once serialized, that context and file information weighs in at roughly **5 KB per event**. At our scale—with more than **10 billion events per minute**—sending everything upstream would mean pushing multiple terabytes of data per second across the network. Most of these events wouldn’t match any detection rules and would ultimately be discarded, making the processing and storage cost unjustifiable.

The challenge wasn’t limited to the backend. On the Agent side, serializing and transmitting such a massive stream would spike CPU and memory usage, increasing the risk of dropped events—the exact opposite of what we wanted. Sustaining hundreds of megabits per second of outbound traffic per host just for security monitoring simply wasn’t viable.

To address this, we introduced **Agent-side rules**. Instead of sending everything upstream, we matched file activity against a set of rules locally on each host. This allowed us to discard noise early and only send the events that mattered for security investigations. By filtering at the edge, we drastically reduced the amount of data we needed to serialize and transmit—down to roughly **one million events per minute** across all infrastructure—while maintaining full detection coverage.



![image-20260225113427710](./picture/image-20260225113427710.png)

*The Datadog Agent filters events locally before forwarding relevant ones to the backend for detection and notification.*

## Filtering 94% of events at the kernel level

When we set out to build our file integrity monitoring solution using eBPF, we quickly realized how tightly performance and coverage were linked. At the Agent level, eBPF gave us a powerful way to observe system activity without interrupting it, but that also meant we had to keep up with a nonstop stream of events. In a Linux environment, where nearly everything is treated as a file, the volume of system calls and file-related events was massive. Handling that flood of data in real time became one of the core engineering challenges we had to solve.

### A simplified eBPF FIM architecture

At first glance, the architecture behind a basic eBPF-based FIM solution isn’t all that complicated. The Agent loads eBPF programs that hook into key points across the system to observe activity. These programs push data into a ring buffer, and from there, the Agent reads and analyzes each event. Simple enough—on paper.

![image-20260225113625059](./picture/image-20260225113625059.png)

*eBPF programs collect filesystem activity in kernel space and send it to the Datadog Agent’s rule engine through a ring buffer.*

### The scale problem

Reality hit fast. Some of our more sensitive workloads generate **up to 5,000 relevant syscalls per second**. These aren’t background noise—they’re security-related events that need to be analyzed. The Agent wasn’t just parsing thousands of events per second; it had to do so without dropping any and without leaving a noticeable performance footprint on the system. Processing that kind of volume isn’t the hard part—doing it with zero impact while maintaining full security coverage is where the real challenge began.

Our first benchmarks made the scale problem clear. On some hosts, the Agent struggled to keep up. The stream of events flowing through the ring buffer could outpace the Agent’s ability to process them, leading to dropped events. Every missed event meant a potential blind spot in our security coverage—something we couldn’t afford in a system designed to detect tampering.

### Moving logic closer to the kernel

As we analyzed performance bottlenecks, one thing became clear: To keep up with high event volumes and maintain coverage, we needed to rethink where our logic lived. We made a key architectural shift—moving as much of the event evaluation as possible into our eBPF programs. This allowed us to filter out irrelevant events directly in the kernel, drastically reducing the amount of data being pushed through the ring buffer. Only the events that truly mattered made it to user space.

That optimization gave our Agent a critical edge. Even when preempted by the kernel scheduler, it could still perform a second, deeper evaluation before deciding whether to send an event to the backend. It was a huge win, but it came with its own set of challenges.

## Approvers and discarders: In-kernel prefiltering

As powerful as eBPF is, it’s intentionally constrained to ensure kernel stability. These limits, especially around computation, protect the system from runaway programs, but they also make it difficult to run complex logic. This challenge becomes even more pronounced on older kernels without the latest eBPF capabilities.

To work within these limits, we adopted a two-stage evaluation model:

1. **In-kernel filtering:** A lightweight phase that handles quick decisions with minimal overhead
2. **User-space evaluation:** A deeper phase that performs the full evaluation—rich with context, correlations, and the kind of logic that would be impossible (or unsafe) to run in the kernel

To make this in-kernel filtering more effective, we introduced two core concepts: approvers and discarders.

- **Approvers** allow events that match certain criteria to pass through.
- **Discarders** explicitly filter out noise early.

Together, they provide a precise and efficient way to manage event flow without overwhelming the Agent or the system it’s running on.

### How approvers work

Approvers are static filters generated when we compile our detection rules. By analyzing each rule’s conditions, we can identify patterns or specific values that make an event worth a deeper look in user space.

For example:

```
open.file.path == "/etc/passwd" && open.flags & O_CREAT > 0
```

In this case, the `passwd` file name is a meaningful value. We define it as an approver and pass it into the kernel using eBPF maps. These maps enable fast, low-overhead filtering within the kernel, ensuring that only relevant events are forwarded to user space for further analysis.

### How discarders complement them

Of course, it doesn’t stay simple for long. One rule is easy, but when you’re managing hundreds, each with different fields and conditions across different syscalls, it quickly becomes a complex optimization problem.

Depending on the rule expression, the rule engine may not always be able to generate approvers. For example:

```
open.file.path == "/etc/*"
```

Here, we can’t extract a specific filename to use as a static filter, since the `*` wildcard matches any file under `/etc/`. There’s no concrete value to approve ahead of time.

That’s why we also introduced discarders, a dynamic counterpart to approvers. Discarders are created at runtime by the rule engine. If it determines that a particular event value will never match any rule, it marks that value as a candidate for in-kernel filtering.

Continuing with the previous example, we can confidently say that any file access under `/tmp` will never match the rule targeting `/etc/*`. In that case, `/tmp` becomes a discarder. These discarders are injected into the kernel via LRU eBPF maps, allowing us to filter them efficiently while keeping memory usage under control.

Like approvers, discarders start simple. But in a real-world environment with a large and evolving ruleset, figuring out what can safely be discarded requires non-trivial algorithms.

![image-20260225114919023](./picture/image-20260225114919023.png)

*Approvers and discarders work together in the kernel to filter out irrelevant file events before passing only meaningful ones to the Datadog Agent for deeper analysis.*

Together, approvers and discarders allow us to **pre-filter up to 94% of events directly in the kernel**. That means fewer events to process in user space, dramatically lower CPU usage, and, most importantly, no dropped events. This layered, adaptive filtering model is what makes our FIM solution both high-performance and high-coverage, even under heavy workloads.

## Beyond detection: Enriching FIM with context

Building file integrity monitoring on top of eBPF has been both a challenge and an opportunity. By moving beyond traditional approaches like periodical scans or `inotify`, we gained the ability to capture real-time, high-fidelity insights into file activity, even in short-lived processes and ephemeral containers. Tackling the sheer volume of data forced us to innovate with in-kernel filtering, Agent-side rules, and the concepts of approvers and discarders, which together made it possible to preserve both performance and coverage at Datadog scale.

And FIM is only the beginning. Detecting that a file has changed is important, but understanding why it changed, which process or user was behind it, and what else was happening on the system at the time is what turns raw events into actionable security insights. Our next step is to continue enriching FIM with as much context as possible, so security teams don’t just know that something happened but have the full story they need to investigate and respond effectively.