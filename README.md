# Exemplar Release

This exemplar [BOSH][bosh] release is intended as a collection of recommended practices for authoring a BOSH release,
with a particular focus on jobs. Not all advice here should be taken verbatim for all releases, so we have made an
attempt to explain the reasoning behind all recommendations so that the conscientious release author can adapt the
recommendations to their unique circumstances.

[bosh]: https://bosh.io

<!-- toc -->

- [Order Independence](#order-independence)
- [BOSH Job Lifecycle](#bosh-job-lifecycle)
  * [Starting](#starting)
    + [Pre-Start](#pre-start-docshttpsboshiodocspre-starthtml)
    + [Monit Start](#monit-start)
      - [Logging](#logging)
    + [Post-Start](#post-start-docshttpsboshiodocspost-starthtml)
    + [Post-Deploy](#post-deploy-docshttpsboshiodocspost-deployhtml)
  * [Stopping](#stopping)
    + [Drain](#drain-docshttpsboshiodocsdrainhtml)
    + [Monit Stop](#monit-stop)
- [Template Advice](#template-advice)
  * [ERB](#erb)
  * [bash](#bash)
- [Monit Advice](#monit-advice)
- [Exporting Logs](#exporting-logs)

<!-- tocstop -->

## Order Independence

The primary goals of a BOSH job are to maximize availability, remove the need for manual operation, and contribute to
quick deployments. To accomplish this, it is important to understand the BOSH deployment lifecycle and what your job
should be doing to take advantage of it.

A process should be able to start and stay running without dependencies being ready. Only once its required dependencies
(those needed to respond to requests) are reporting as healthy should it report as healthy. This allows jobs to be
started in any order and then propagate health upwards through their dependents when they're ready. We think removing
this burden of job ordering from the operator reduces complexity and increases system reliability.

On the other hand, stopping can follow a similar but reversed pattern. If your service is being load-balanced then you
can use a signal in your drain script to mark yourself as unhealthy. After a period of time or after no more requests
are detected you can stop the process completely. If you aren't behind a load balancer then you will have to find
another way to tell upstream components about your imminent demise.

## BOSH Job Lifecycle

Every BOSH job goes through a specific lifecycle for starting and stopping. The BOSH website has a great
[overview](https://bosh.io/docs/job-lifecycle.html) of the lifecycle. You may wish to review this before reading on to
our recommendations for the individual parts of the lifecycle.

### Starting

#### Pre-Start ([docs](https://bosh.io/docs/pre-start.html))

Pre-start scripts are run before BOSH hands off control to Monit, so will not necessarily run every time a job starts
(e.g. if a VM is rebooted outside of BOSH's control). However, pre-start scripts will run at least once for every new
release version that is deployed, and have no timeout (in contrast to a start script). Therefore, pre-start scripts
can best be used for performing lengthy set-up of persistent state that must be done for a new version of a release,
such as database migrations. However, because the pre-start script does not necessarily run in every VM your job
will run in, do not perform any work in temporary directories such as `/var/vcap/sys/run` (see [VM Configuration
Locations][vm-config-loc] for the list of temporary directories).

[vm-config-loc]: https://bosh.io/docs/vm-config.html

In general, a pre-start script should not be necessary and the start script should be sufficient. If you find a
pre-start script is necessary, keep the above caveats in mind.

#### Monit Start

The start script has two main responsibilities: writing the process ID (PID) to a pidfile and starting the main process.
All setup required for the process to run (that has not been done in the pre-start script) must be done at this time.
Monit places a short timeout on the pidfile being written, so the work done in the start script should be focused on
executing your process and getting it healthy quickly.

**Note**: This script will likely run many times, so ensure that it is idempotent (that it can be run repeatedly without
causing the system to enter a bad state)

The general workflow of a start script:

1. Create a log directory for log persistence in `/var/vcap/sys/log/<job name>`
   - Idempotency recommendation: Do not fail if the log directory already exists

2. Create a run directory to contain the pidfile in `/var/vcap/sys/run/<job name>`
   - Idempotency recommendation: Do not fail if the run directory already exists

3. Change ownership permissions on the log and run directories to user vcap and group vcap

4. Write the process ID (in BASH, this is `$$`) your pidfile in `$RUN_DIR/<task name>.pid`
   - Idempotency recommendation: Your start script may be called more than once without stopping any existing processes.
     Therefore, you should only write to this file if it does not exist and contain the PID of a running process. If it
     contains a running process, assume that you should exit.
   - **Note**: This can be achieved simply using [sipid][sipid]'s `sipid claim` command.

5. `exec` your process, which will execute it in the context of your start script, replacing it. Use `chpst` to change
   the running user to `vcap` so your process does not run as root. This looks like
   `exec chpst -u vcap:vcap /var/vcap/packages/<package name>/bin/<executable>`.
  - Avoid using monit to change the ownership of the process. Depend on monit features sparingly, as Monit is not
    guaranteed to exist in future BOSH releases.

**Note**: The start script is executed as root. Do not assume you can only break your own process.

**Note**: If your process cannot start quickly, consider moving long-running tasks to pre-start or post-start.

##### Logging

Job logs should be put in the `/var/vcap/sys/log/<job>` directory. You can redirect the logs from your server (if
they're output on `stdout` and `stderr`) by using standard redirection:

```
exec ... \
  >> "/var/vcap/sys/log/<job>/<process>.out.log" \
  2>> "/var/vcap/sys/log/<job>/<process>.err.log"
```

If your process takes log locations as parameters, use that instead:

```
exec ... \
  --out-log-file "/var/vcap/sys/log/<job>/<process>.out.log" \
  --err-log-file "/var/vcap/sys/log/<job>/<process>.err.log"
```
For detail on forwarding logs to external locations, please see [Exporting Logs](#exporting-logs).

#### Post-Start ([docs](https://bosh.io/docs/post-start.html))

Post-start is useful for custom health-checks to ensure you job has started correctly. For example, if your process
starts quickly, but takes time to discover services or connect to its backend, you may wish to use post-start to query
the readiness of your job to start handling requests. If a post-start script is provided, BOSH will not consider a job
to be ready until it has exited successfully.

#### Post-Deploy ([docs](https://bosh.io/docs/post-deploy.html))

The authors have never seen this used. It may be useful for checking the health of an entire deployment. If you find it
useful, please [let us know][contact-us].

### Stopping

Job is unmonitored before any stop scripts can run, so you can safely exit in either `drain` or `monit stop` without the
job becoming listed as unhealthy.

#### Drain ([docs](https://bosh.io/docs/drain.html))

Drain scripts are optional hooks into the BOSH job lifecycle that are run before stopping the job via Monit. They
are typically used for services which must perform some work before being shut down, for example flushing a request
queue or evacuating containers from a Diego cell. As a rule of thumb, if `monit stop`ping your job could cause dropped
connections or a lack of availability, a drain script should be used to prevent this. Most commonly, your drain script
will send a request to a drain endpoint on your process and wait for it to return rather than implementing the drain
behavior itself.

A concrete example is the [gorouter][gorouter], which has a configurable `drain_wait` parameter. When non-zero,
gorouter's drain script will instruct gorouter to report itself as unhealthy to its load-balancer with the intent of
being removed from the balanced instance group before shutting down and rejecting requests. When `monit stop` is called,
the router will already be receiving no connections, so will not drop connections when it is shut down quickly. This is
the [lame duck][lame-duck] pattern.

[gorouter]: https://github.com/cloudfoundry/gorouter
[lame-duck]: https://landing.google.com/sre/book/chapters/load-balancing-datacenter.html#robust_approach_lame_duck

Drain scripts have no timeout, so should take whatever time necessary to block on any draining work. One may, however,
run the risk of writing a drain script that never finishes and blocks a deployment. A well-written drain script is
guaranteed to finish, such as by adding your own sensible timeout around draining work (e.g. 10 minutes).

When your draining process has completed, your drain script should output a "0" to STDOUT to inform BOSH that draining
is complete.

Open question: If your job needs to wait for another job to drain first, what is the best way to block on that? If you
have any ideas, please [let us know][contact-us].

#### Monit Stop

The stop program in your job's `monit` file is executed after the drain script (if present) has finished running. It can
also be run directly by an operator if they execute `monit stop <job>` on the machine. By default, there is a 30 second
timeout on this script completing. Monit will assume scripts taking longer than this have failed. We do not recommend
changing this value if you need more time. Instead, you should do all the work necessary in your `drain` script such
that your service can shutdown quickly (where "quickly" generally means in under 10 seconds).

If you're not using drain then we recommend that you send `SIGTERM` to your process which should cause it to start
shutting down. If your process shuts down at this point then you're good to go. If for some reason your process locks up
or is unable to exit for some other reason then you should send `SIGKILL` before the timeout. If your language runtime
supports dumping the stacks of all running threads on `SIGQUIT` (Go and Java do) then you can send that signal just
before the `SIGKILL` to aid debugging why the process is stuck. If you are not using a `SIGKILL` respecting runtime then
adding this functionality to your own program normally isn't difficult.

Instead of writing this pattern in `bash` you can use our `sipid kill` command which will handle all of the details of
killing a process within a timeout for you.

If you're using drain to kill the process then your process may already be shut down by the time that `monit stop` is
called. In this case we do not need to do anything further in the stop executable.

## Template Advice

### ERB

We advise avoiding any ERB in your control scripts wherever possible. The control flow of starting and stopping your
program should be deterministic and simple. All ERB should be relegated to static configuration files so that properties
can be interpolated.

Any template in your job can use ERB (even the monit files!). While this is extremely powerful it can be very difficult
to understand and maintain complex templates.

### bash

The [ShellCheck][shellcheck] tool can be used to find common errors in your `bash` scripts. This yet another reason to
keep ERB out of your scripts!

[shellcheck]: https://github.com/koalaman/shellcheck

## Monit Advice

Try to keep your Monit configuration as simple as possible. The BOSH team is planning on removing Monit soon - do not
couple yourself to it! This includes such changes as monitoring memory and changing away from the default timeout.

You must specify `group vcap` in your `monit` file because the agent [uses this tag][agent-monit-group] to find
processes it should be managing on the machine.

[agent-monit-group]: https://github.com/cloudfoundry/bosh-agent/blob/5beacb106a67e403a15e9926b6ee39006d774d31/jobsupervisor/monit_job_supervisor.go#L114

## Exporting Logs

BOSH itself does not handle forwarding logs off-system. If you have written your logs appropriately as described in
the [Logging](#logging) section, operators can choose the correct log-forwarding mechanism for their deployment using job co-location in the deployment manifest or BOSH [addons][addons]. Operators may choose to use something like [google-fluentd][google-fluentd] or [syslog-release][syslog-release]. If you are also responsible for providing a deployment manifest generation tool, you may wish to provide the option to add syslog-release forwarding to all components.

[addons]: https://bosh.io/docs/runtime-config.html#addons
[google-fluentd]: https://github.com/cloudfoundry-community/stackdriver-tools#deploying-host-logging
[syslog-release]: https://github.com/cloudfoundry/syslog-release
[loggregator]: https://github.com/cloudfoundry/loggregator


### Metron Agent

`metron_agent` has functionality to forward syslogs, but this has been superseded by `syslog-release`. `metron_agent`
should only be used to forward logs to loggregator.

### The Ol' `tee` and `logger` Approach

Many releases currently include complex setup to forward logs to both `/var/vcap/sys/log` as well as syslog. This
is not necessary if operators use one of the log-forwarding options mentioned above. Drawbacks of the tee and logger approach include starting multiple processes for every line of logs, fairly subtle behavior that can easily break or security vulnerabilities. If your release has any code like the following, please remove it and follow the recommendations above:

```
DO NOT USE THIS CODE

exec > \
  >(
    tee -a >(logger -p user.info -t vcap.$(basename $0).stdout) | \
      awk -W interactive '{ gsub(/\\n/, ""); system("echo -n [$(date +\"%Y-%m-%d %H:%M:%S%z\")]"); print " " $0 }' \
      >> /var/vcap/sys/log/<job>/<process>.out.log
  )
exec 2> \
  >(
    tee -a >(logger -p user.error -t vcap.$(basename $0).stderr) | \
      awk -W interactive '{ gsub(/\\n/, ""); system("echo -n [$(date +\"%Y-%m-%d %H:%M:%S%z\")]"); print " " $0 }' \
      >> /var/vcap/sys/log/<job>/<process>.err.log
  )

DO NOT USE THIS CODE
```

<sub>If you must use this, you should understand it. The `exec` calls redirect `STDOUT` and `STDERR` respectively,
sending them to a sub-shell that calls `tee`. `tee` splits the output to 1) syslog via `logger` and 2) the BOSH log
directory (but not before appending timestamps with `awk`).</sub>

<!-- Global Links -->

[contact-us]: https://github.com/cloudfoundry/exemplar-release/pulls
[sipid]: https://github.com/cloudfoundry/sipid
