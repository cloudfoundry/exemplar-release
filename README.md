# Exemplar Release

This exemplar [BOSH](https://bosh.io) release is intended as a collection of recommended practices for authoring a
BOSH release, with a particular focus on jobs. Not all advice here should be taken verbatim for all releases, so we
have made an attempt to explain the reasoning behind all recommendations so that the conscientious release author can
adapt the recommendations to their unique circumstances.

## BOSH Job Lifecycle

Every BOSH job goes through a specific lifecycle for starting and stopping. Bosh.io has a great
[overview](https://bosh.io/docs/job-lifecycle.html) of the lifecycle. You may wish to review this before reading on
to our recommendations for the individual parts of the lifecycle.

### Starting

#### Pre-Start ([docs](https://bosh.io/docs/pre-start.html))

Pre-start scripts are run before BOSH hands off control to Monit, so will not necessarily run every time a job starts
(e.g. if a VM is rebooted outside of BOSH's control). However, pre-start scripts will run at least once for every new
release version that is deployed, and have no timeout (in contrast to a start script). Therefore, pre-start scripts
can best be used for performing lengthy set-up of persistent state that must be done for a new version of a release,
such as database migrations. However, because the pre-start script does not necessarily run in every VM your job will
run in, do not perform any work in temporary directories such as `/var/vcap/sys/run` (see
[VM Configuration Locations](https://bosh.io/docs/vm-config.html) for the list of temporary directories).

In general, a pre-start script should not be necessary and the start script should be sufficient. If you find a
pre-start script is necessary, keep the above caveats in mind.

#### Monit Start

The start script has two main responsibilities: writing the process ID (PID) to a a pidfile and starting the main
process. All setup required for the process to run (that has not been done in the pre-start script) must be done
at this time. Monit places a short timeout on the pidfile being written, so the work done in the start script should
be focused on executing your process and getting it healthy quickly.

**Note**: This script will likely run many times, so ensure that it is idempotent (that it can be run repeatedly without
causing the system to enter a bad state)

The general workflow of a start script:

1. Create a log directory for log persistence in `/var/vcap/sys/log/<job name>`
   - Idempotency recommendation: Do not fail if the log directory already exists
1. Create a run directory to contain the pidfile in `/var/vcap/sys/run/<job name>`
   - Idempotency recommendation: Do not fail if the run directory already exists
1. Change ownership permissions on the log and run directories to vcap:vcap
1. Write the process ID (in BASH, this is `$$`) your pidfile in `$RUN_DIR/<task name>.pid`
   - Idempotency recomendation: Your start script may be called more than once without stopping any existing processes.
     Therefore, you should only write to this file if it does not exist and contain the PID of a running process. If
     it contains a running process, assume that you should exit.
   - **Note**: This can be achieved simply using [sipid](https://github.com/cloudfoundry/sipid)'s `sipid claim` command.
1. `exec` your process, which will execute it in the context of your start script, replacing it. Use `chpst` to change
   the running user to `vcap` so your process does not run as root. This looks like
   `exec chpst -u vcap:vcap /var/vcap/packages/<package name>/bin/<executable>`.
  - Avoid using monit to change the ownership of the process. Depend on monit features sparingly, as Monit is not
    guaranteed to exist in future BOSH releases.

**Note**: The start script is executed as root. Do not assume you can only break your own process.

**Note**: If your process cannot start quickly, consider moving long-running tasks to pre-start or post-start.

##### Logging

Job logs should be put in the `/var/vcap/sys/log/<job>` directory. You can redirect the logs from your server (if
they're output on stdout and stderr) by using standard redirection:

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
**Note**: `syslog_forwarder` from the [syslog-release](https://github.com/cloudfoundry/syslog-release) should be
co-located with your job so that all logs in `/var/vcap/sys/log` are forwarded appropriately (usually to loggregator).
If your release is not using `syslog_forwarder` and still relies on `metron_agent` (from
[loggregator](https://github.com/cloudfoundry/loggregator)) then logs must be forwarded to syslog manually using the
`logger` and `tee` programs:

```
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
```

This is _ugly_ (and starts multiple processes for every line of logs). Use `syslog_forwarder`. Please.

<sub>If you must use this, you should understand it. The `exec` calls redirect `STDOUT` and `STDERR` respectively,
sending them to a subshell that calls `tee`. `tee` splits the output to 1) syslog via `logger` and 2) the BOSH log
directory (but not before appending timestamps with `awk`).</sub>

**TODO**: `syslog_forwarder` configures syslogs to be "government-compliant"
  - What does that mean?


#### Post-Start ([docs](https://bosh.io/docs/post-start.html))

Post-start is useful for custom health-checks to ensure you job has started correctly. For example, if your process
starts quickly, but takes time to discover services or connect to backends, you may wish to use post-start to query
the readiness of your job to start handling requests. If a post-start script is provided, BOSH will not consider a job
to be ready until it has exited successfully.

#### Post-Deploy ([docs](https://bosh.io/docs/post-deploy.html))

The authors have never seen this used. It may be useful. If you find it useful, please
[let us know](https://github.com/cloudfoundry/exemplar-release/pulls).

### Stopping

Job is unmonitored before any stop scripts can run, so you can safely exit in either `drain` or `monit stop` without the
job becoming listed as unhealthy.

#### Drain ([docs](https://bosh.io/docs/drain.html))

- Drain scripts are primarily for jobs that cannot be `monit stop`ed quickly (< 10 seconds? Maybe)
- For example, anything that needs to lame-duck
  - GCP: Can CPI unregister gorouter from LB?
  - Behind gorouter: Are we sending "forget me" to the router?
- Have a blocking way to tell your process to clean-up (e.g. an endpoint)
  - If you don't want to block, your script should wait-poll. Use a non-BASH language for this.
  - Drain script can run forever, so you may wish to place your own upper-bound
- When done cleaning up, exit and print 0 ("wait after draining" value)
  - If draining fails, exit non-zero

Open question: If your job needs to wait for another job to drain first, what is the best way to block on that? If you
have any ideas, please [let us know](https://github.com/cloudfoundry/exemplar-release/pulls).

#### Monit Stop

- Calls your monit stop script
- Stop ASAP (< 10 seconds)
  - If you can't, move long process to drain
- Recommended
  - If not using drain:
    - `SIGTERM`, wait 10 seconds, `SIGKILL`
    - Optionally, `SIGQUIT` to dump stacks if your runtime supports it (or you add your own support):
      - Go
      - Java
    - Use `sipid kill`!
  - If using drain:
    - Send `kill -9 $pid` to your process to kill it immediately
    - If your drain script is implemented correctly, your process may already be gone
    - Use `sipid kill`!

## Order Independence

Main goal: Process should be able to start without any dependencies (rather than quitting if a dependency is not yet
ready), even if that means returning unhealthy health checks. Process should be able to take itself out of commission
(e.g. return unhealthy healthchecks) while still serving traffic. Goals here are to only advertise readiness to
accept requests when you actually are ready, and to continue to serve in-flight requests, etc. until the advertisement
is fully stopped. Otherwise, clients will attempt to connect to you after you have died, and requests will fail.

- Starting
  - Process should come up ASAP and not depend on dependencies being up
  - Health-check should return unhealthy until enough dependencies are up such that you can fulfill requests
    - May wish to use post-start to wait for full health
- Stopping
  - If being load-balanced
    - Drain script should instruct process to mark itself as unhealthy, which will eventually cause LB to mark as
      unhealthy
    - After LB stops sending traffic, some requests may still be in flight and there still may be requests in the
      process's queue, so drain should wait for that to settle down, too
    - Similar workflow applies to non-LB dependencies. If other components rely on you, reply unhealthy until requests
      "stop" (how that's determined is up to the reader). Only consider your process ready to die when it is no longer
      receiving requests.

## Template Advice

### To ERB or not to ERB

- Do not use ERB in your scripts if you can avoid it
  - Use config files instead (and use ERB in those)
- If you _need_ to, anything in job "templates" folder can be ERBed

### BASH best practices

- Shellcheck

## Monit file

- As simple as possible (don't rely on monit always existing)
- Must specify `group vcap` for agent
  - Investigate if this is true
- Should monit files be responsible for checking things like memory usage? (e.g. CC)
  - In the futuurrrrrrre, but discourage more usage (don't rely on monit because it's not the future of BOSH)
