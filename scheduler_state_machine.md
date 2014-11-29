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

* Till we find a host with the required capabilites (board, cpu, sensors etc as
  specified by the job) the job remains queued.
* ----- a host matching the capabilities is leased
* We double check the host (clean the environment, chrome/ue/system services are
  alive, disk/network interfaces are ok).
* We configure the good host (os, firmware etc). Don't worry about bad hosts
  for now.
* The job runs on the host.
* We gather results. Assume it didn't abort/fall off the network for now.
* ----- the host is released
* We parse logs to figure out if the job passed. Assume no crashes etc.
* ----- the job is complete
 
[Scheduler layers] ()
So what you're about to read about is a job going through:
Prejob, job, Postjob, Postjob, each of which has 3 sections: a prolog, body
and epilog. I don't really know how to describe the state transitions and
give you context on what code drives the machine, without just unrolling
a few scheduler ticks in front of you.

# New job
* The scheduler doesn't even see this job till it acquires a host.

# Host scheduler
* Find a host (if you're trying to follow source, ignore all the magic in
  rdb.* for now).
* Queue a reset against the host
    - scheduleprejobtask
* Set active bit on job
--- No host or hqe state changes till now, a new task is queued and the hqe
is marked active to remove it from the input job queue. This touches
afe_special_tasks and afe_host_queue_entries.
    HQE     : Queued
    HOST    : Ready

# Scheduler
* New task is detected, the scheduler still doesn't care about the job. The
  schedulers goal right now is to verify that the host associated with the
  reset is usable, even though it doesn't know what's going to use it.
    - schedulespecialtask
* An agent (a prejob task) is created for the reset. Remember that any agent
  will invoke the curiosity of handle_agents.

% PREJOB
* The prolog of the task is executed through handle agents.
--- is_active and time_started are set on the row in afe_special_tasks.
    HQE     : Resetting
    HOST    : Resetting
* Handle agents goes on to execute the agents tick, queueing up actions
  against a drone.
* The autoserv runs on the DUT and exits, writing an exit code in its pidfile.
* Handle agents is continuously polling this pidfile and notices that the job
  is done.
    - BaseAgentTask epilog
        - SpecialAgentTask cleanup
---is_active, is_complete, success, time_finished set on the task.
Above set via AFE MODELS (if theis phrase doesn't make sense to you, ignore it;
otoh if you're wondering how, it is done through task.finish()).
    HQE     : Resetting
    HOST    : Resetting
* The epilog of the reset task is invoked.
    - on_pending
    HQE     : Pending
    HOST    : Pending
    - job.run
    HQE     : Starting
    HOST    : Pending
% END PREJOB (assume there is no provision and everything runs smoothly)

* Scheduler finally notices the job because it is in Starting
    - schedulerunning hqes
* Scheduler converts the line in afe_host_queue_entries into an agent
  (either a QueueTask or a PostJobTask, we're done with PrejobTasks).
  Again, remember that this will invoke the curiosity of handle_agents.
    HQE     : Starting
    HOST    : Pending
* Note that at this point there are only 2 allowed host statuses, Pending
  and Running (don't worry about Running for now).

% QUEUE TASK (aka the job)
* The Queue Task prolog is executed by handle agents.
    HQE     : Running
    HOST    : Running
* The job runs on a drone and finishes, handle agents notices the exit
  code because it's polling the pidfile via ssh.
    - BaseAgentTask finish
        - AbstractQueueTask epilog
            - QueueTask epilog
    (yes the queue task has en epilog but the way it's invoked is a little
     long winded).
    HQE     : Gathering
    HOST    : Running
% END QUEUE TASK
* Scheduler notices a HQE with status=Gathering
* Note that post job tasks aren't backed by rows in the db like prejob tasks.
  This is because they don't really convey important information when they fail
  (a provision failing may mean a bad dut, a parsing fail means parsing failed).
* An agent is created for the postjob task (we're done with prejob and queue
  tasks now).

% POST JOB TASK: GATHERING
* Gathering prolog runs, the only allowable host state is Running now.
* Gathering epilog runs, gathering is pretty boring when there're no crashes.
    - BaseAgentTask _parse_results
---active=0 on HQE, but it is still not complete.
    HQE     : Parsing
    HOST    : Ready (no more host to worry about)
* Sidenote: Host scheduler will pick this host up for the next job. Note that
  it needs both th Ready state and the active=0 to do so.
% END POST JOB TASK: GATHERING

* Scheduler notices an HQE in parsing, and creates an agent for the parsing job.
* This is a FinalReparseTask.

% POST JOB TASK: PARSING
* Parsing prolog runs, there is a backpressure limit here of max_parse_processes
  so we don't clog up job run times with parsing jobs.
* Parsing runs if it isn't in rate limit mode.
* Parsing epilog runs.
    -archive_results
---HQE gets the complete bit and finished_on set.
    HQE     : Completed (no more hqe to worry about)
    HQE     : Completed (no more hqe to worry about)
% END POST JOB TASK: PARSING

