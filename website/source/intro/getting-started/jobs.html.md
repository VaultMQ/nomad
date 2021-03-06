---
layout: "intro"
page_title: "Jobs"
sidebar_current: "getting-started-jobs"
description: |-
  Learn how to submit, modify and stop jobs in Nomad.
---

# Jobs

Jobs are the primary configuration that users interact with when using
Nomad. A job is a declarative specification of tasks that Nomad should run.
Jobs have a globally unique name, one or many task groups, which are themselves
collections of one or many tasks.

The format of the jobs is [documented here](/docs/jobspec/index.html). They
can either be specified in [HCL](https://github.com/hashicorp/hcl) or JSON,
however we recommend only using JSON when the configuration is generated by a machine.

## Running a Job

To get started, we will use the [`init` command](/docs/commands/init.html) which
generates a skeleton job file:

```
$ nomad init
Example job file written to example.nomad

$ cat example.nomad

# There can only be a single job definition per file.
# Create a job with ID and Name 'example'
job "example" {
	# Run the job in the global region, which is the default.
	# region = "global"
...
```

In this example job file, we have declared a single task 'redis' which is using
the Docker driver to run the task. The primary way you interact with Nomad
is with the [`run` command](/docs/commands/run.html). The `run` command takes
a job file and registers it with Nomad. This is used both to register new
jobs and to update existing jobs.

We can register our example job now:

```
$ nomad run example.nomad
==> Monitoring evaluation "26cfc69e"
    Evaluation triggered by job "example"
    Allocation "8ba85cef" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "26cfc69e" finished with status "complete"
```

Anytime a job is updated, Nomad creates an evaluation to determine what
actions need to take place. In this case, because this is a new job, Nomad has
determined that an allocation should be created and has scheduled it on our
local agent.

To inspect the status of our job we use the [`status` command](/docs/commands/status.html):

```
$ nomad status example
ID          = example
Name        = example
Type        = service
Priority    = 50
Datacenters = dc1
Status      = running
Periodic    = false

==> Evaluations
ID        Priority  Triggered By  Status
26cfc69e  50        job-register  complete

==> Allocations
ID        Eval ID   Node ID   Task Group  Desired  Status
8ba85cef  26cfc69e  171a583b  cache       run      running
```

Here we can see that our evaluation that was created has completed, and that
it resulted in the creation of an allocation that is now running on the local node.

An allocation represents an instance of Task Group placed on a node. To inspect
an Allocation we use the [`alloc-status` command](/docs/commands/alloc-status.html):

```
$ nomad alloc-status 8ba85cef
ID              = 8ba85cef
Eval ID         = 26cfc69e
Name            = example.cache[0]
Node ID         = 58d69d9d
Job ID          = example
Client Status   = running
Evaluated Nodes = 1
Filtered Nodes  = 0
Exhausted Nodes = 0
Allocation Time = 27.704µs
Failures        = 0

==> Task "redis" is "running"
Recent Events:
Time                   Type      Description
15/03/16 15:40:57 PDT  Started   Task started by client
15/03/16 15:40:00 PDT  Received  Task received by client

==> Status
Allocation "4b5f832c" status "running" (0/1 nodes filtered)
  * Score "58d69d9d-0015-2c69-e9ba-cc9ee476bb6d.binpack" = 1.580850

==> Task Resources
Task: "redis"
CPU  Memory MB  Disk MB  IOPS  Addresses
500  256        300      0     db: 127.0.0.1:52004
```

To inspect the file system of a running allocation, we can use the [`fs`
command](/docs/commands/fs.html):

```
$ nomad fs ls 8ba85cef alloc/logs
Mode        Size    Modfied Time           Name
-rw-rw-r--  0 B     15/03/16 15:40:56 PDT  redis.stderr.0
-rw-rw-r--  2.3 kB  15/03/16 15:40:57 PDT  redis.stdout.0

$ nomad fs cat 8ba85cef alloc/logs/redis.stdout.0
 1:C 15 Mar 22:40:57.188 # Warning: no config file specified, using the default config. In order to specify a config file use redis-server /path/to/redis.conf
```

## Modifying a Job

The definition of a job is not static, and is meant to be updated over time.
You may update a job to change the docker container, to update the application version,
or to change the count of a task group to scale with load.

For now, edit the `example.nomad` file to uncomment the count and set it to 3:

```
# Control the number of instances of this group.
# Defaults to 1
count = 3
```

Once you have finished modifying the job specification, use `nomad run` to
push the updated version of the job:

```
$ nomad run example.nomad
==> Monitoring evaluation "127a49d0"
    Evaluation triggered by job "example"
    Allocation "8ab24eef" created: node "171a583b", group "cache"
    Allocation "f6c29874" created: node "171a583b", group "cache"
    Allocation "8ba85cef" modified: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "127a49d0" finished with status "complete"
```

Because we set the count of the task group to three, Nomad created two
additional allocations to get to the desired state. It is idempotent to
run the same job specification again and no new allocations will be created.

Now, let's try to do an application update. In this case, we will simply change
the version of redis we want to run. Edit the `example.nomad` file and change
the Docker image from "redis:latest" to "redis:2.8":

```
# Configure Docker driver with the image
config {
    image = "redis:2.8"
}
```

This time we have not changed the number of task groups we want running,
but we've changed the task itself. This requires stopping the old tasks
and starting new tasks. Our example job is configured to do a rolling update via
the `stagger` attribute, doing a single update every 10 seconds. Use `run` to push the updated
specification now:

```
$ nomad run example.nomad
==> Monitoring evaluation "ebcc3e14"
    Evaluation triggered by job "example"
    Allocation "9a3743f4" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "ebcc3e14" finished with status "complete"
==> Monitoring evaluation "b508d8f0"
    Evaluation triggered by job "example"
    Allocation "926e5876" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "b508d8f0" finished with status "complete"
==> Monitoring next evaluation "ea78c05a" in 10s
==> Monitoring evaluation "ea78c05a"
    Evaluation triggered by job "example"
    Allocation "3c8589d5" created: node "171a583b", group "cache"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "ea78c05a" finished with status "complete"
```

We can see that Nomad handled the update in three phases, only updating a single task
group in each phase. The update strategy can be configured, but rolling updates makes
it easy to upgrade an application at large scale.

## Stopping a Job

So far we've created, run and modified a job. The final step in a job lifecycle
is stopping the job. This is done with the [`stop` command](/docs/commands/stop.html):

```
$ nomad stop example
==> Monitoring evaluation "fd03c9f8"
    Evaluation triggered by job "example"
    Evaluation status changed: "pending" -> "complete"
==> Evaluation "fd03c9f8" finished with status "complete"
```

When we stop a job, it creates an evaluation which is used to stop all
the existing allocations. This also deletes the job definition out of Nomad.
If we try to query the job status, we can see it is no longer registered:

```
$ nomad status example
No job(s) with prefix or id "example" found
```

If we wanted to start the job again, we could simply `run` it again.

## Next Steps

Users of Nomad primarily interact with jobs, and we've now seen
how to create and scale our job, perform an application update,
and do a job tear down. Next we will add another Nomad
client to [create our first cluster](cluster.html)

