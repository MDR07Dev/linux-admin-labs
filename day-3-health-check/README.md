PURPOSE:

 - To simulate, identify, and safely manage a runaway CPU-consuming process on the ASC platform host — covering process identification, signal handling, priority adjustment, and disconnect-resilient process execution.

Process Identification:

 - Spun up a CPU-consuming process and confirmed it using live resource data (CPU%, runtime) rather than assumption.
  __Commands used: yes > /dev/null &, top_

SIGTERM Test:

 - Sent a graceful termination signal to the process and confirmed it exited cleanly. A redundant force-kill attempt correctly errored, proving nothing was left to kill.
   __Commands used: kill <pid>, ps -o pid,ni,pri,comm -p <pid>, kill -9 <pid>_

SIGKILL Test:

 - Sent a force-kill signal to a separate running process and confirmed termination with before/after process checks.
   __Commands used: ps -o pid,ni,pri,comm -p <pid>, kill -9 <pid>, ps -o pid,ni,pri,comm -p <pid>_

Priority Adjustment (renice):

 - Adjusted a running process's scheduling priority as a non-destructive alternative to killing it, confirmed via before/after niceness and priority values.
   __Commands used: ps -o pid,ni,pri,pcpu,comm -p <pid>, renice +10 -p <pid>, ps -o pid,ni,pri,pcpu,comm -p <pid>_

Disconnect Survival (nohup):

 - Started a long-running task with nohup, closed the originating SSH session entirely, reconnected with a fresh session, and confirmed the process was still alive and detached from any terminal.
   __Commands used: nohup sleep 200 > /tmp/sleep.log 2>&1 &, jobs -l, ps -p <pid>_

Concept Note:

 - A closing shell sends SIGHUP to its background children by default; nohup makes a process ignore SIGHUP — this is why the process survived disconnect while a plain background (&) job would not have.

Status: Closed
