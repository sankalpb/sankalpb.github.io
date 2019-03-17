---
title: Linux OOM Killer Policy and Knobs
category: [linux]
---
<div id="table-of-contents">
<h2>Table of Contents</h2>
<div id="text-table-of-contents">
<ul>
<li><a href="#sec-1">1. Linux OOM killer policy and knobs</a>
<ul>
<li><a href="#sec-1-1">1.1. Further reading</a></li>
</ul>
</li>
</ul>
</div>
</div>

# Linux OOM killer policy and knobs<a id="sec-1" name="sec-1"></a>

-  /proc/\<pid\>/oom_adj -  The possible values of oom_adj rangae from -17 to +15. The higher the score, more likely the associated process is to be killed by OOM-killer. If oom_adj is set to -17, the process is not considered for OOM-killing.
-  /proc/\<pid\>/oom_score - badness score. The more memory the process uses, the higher the score. The longer a process is alive in the system, the smaller the score. Half of each child's memory size is added to the parent's score if they do not share the same memory. Thus forking servers are the prime candidates to be killed.
-  If the task has nice value above zero, its score doubles.
- Superuser or direct hardware access tasks (CAP_SYS_ADMIN, CAP_SYS_RESOURCE or CAP_SYS_RAWIO) have their score divided by 4. This is cumulative, i.e., a super-user task with hardware access would have its score divided by 16.
-  If OOM condition happened in one cpuset and checked task does not belong to that set, its score is divided by 8.
- The resulting score is multiplied by two to the power of oom_adj (i.e. points \<\<= oom_adj when it is positive and points \>\>= oom_adj otherwise).

## Further reading<a id="sec-1-1" name="sec-1-1"></a>

<https://lwn.net/Articles/317814/>
