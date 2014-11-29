Schedulers state machine
========================

# Introduction

This document jumps right into scheduler implementation details. If you've never
read scheduler code before, you might benefit from talking to someone on the lab
team to get context first. You can also check out the docstrings in the 
[scheduler code] (https://chromium-review.googlesource.com/#/c/174080/).

# Cross section of db writes

None of what follows will make much sense if you've never thought about how jobs run 
in the lab.

To start off with, a job is inserted as a row in the database in the Queued
state, without a host (all jobs and hosts have a status field associated with
them, if you've never looked at the chromeos_autotest_db, this would be a good
time to do so. 

What follows are the state transitions the job and host go through before we can
call a job successfully completed, but before I delve into the mechanics of the
scheduler it makes sense to define what we mean by 'running' a job, since it
doesn't simply mean to ./execute the control file on a dut.

* Till we find a host with the required capabilites the job remains queued.
  * A host matching the capabilities is found and leased
* We check the host 
  * Clean the environment, chrome/ue/system services are alive, disk/network ok.
* We configure the good host (os, firmware etc).
* The job runs on the host.
* We gather results.
  * The host is released
* We parse logs to figure out if the job passed.
  * The job is complete
 
![Scheduler layers] (https://cloud.githubusercontent.com/assets/3627706/5236140/564a510c-77db-11e4-9ff3-cda23bef3923.jpg "Scheduler layers")


