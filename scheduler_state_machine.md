Schedulers state machine
========================

# Introduction

This document jumps right into scheduler implementation details. If you've never
read scheduler code before, you might benefit from talking to someone on the lab
team to get context first. You can also check out the docstrings in the 
[scheduler code] (https://chromium-review.googlesource.com/#/c/174080/).

# Cross section of db writes

None of what follows will make much sense if you've never thought about how jobs run 
in the lab:
* A job is inserted into a queue
* Till the scheduler finds a host with the required capabilites the job remains queued.
  * A host matching the capabilities is found and leased to the job.
* The scheduler checks the host 
  * Clean the environment, chrome/ue/system services are alive, disk/network ok.
* The scheduler configures the good host (os, firmware etc).
* The job runs on the host.
* The scheduler gathers results.
  * The host is released
* The scheduler parses logs to figure out if the job passed.
  * The job is complete
 
![Scheduler layers] (https://cloud.githubusercontent.com/assets/3627706/5236140/564a510c-77db-11e4-9ff3-cda23bef3923.jpg "Scheduler layers")


