> [!NOTE] Module 3 — Assignment 
> **Problem Statement:** 
> You are hosting a system network, where you have two processes running in background. First one is a process to monitor the uptime of each system and second one is a process that keeps track of all processes running in the working environment. The priority value of the first process is 8. The priority value of the second process is lower than the first process. Use 'renice' to change the priority.


1. Finding the Process IDs (PIDs) using `ps` command

```bash
ps aux | grep uptime-monitor 
ps aux | grep process-tracker
```
*We assume the name of the processes are uptime-monitor and process-tracker*

2. Using `renice` to Change the Priority
- *We assume the PID for the first process (uptime monitor) is 1234 and its priority is 8.
- *We assume the PID for the second process (process tracker) is 5678 and we want to set its priority lower than the first one.
```bash
sudo renice 9 5678
```

Now the process with PID 5678 has a lower priority compared to the uptime monitor process with PID 1234 with a priority of 8.

