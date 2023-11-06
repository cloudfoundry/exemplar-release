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
    + [Pre-Start](#pre-start-docs)
    + [Monit Start](#monit-start)
      - [Logging](#logging)
    + [Post-Start](#post-start-docs)
    + [Post-Deploy](#post-deploy-docs)
  * [Stopping](#stopping)
    + [Drain](#drain-docs)
    + [Monit Stop](#monit-stop)
- [Advice for authoring `spec` files](#advice-for-authoring-spec-files)
  * [Properties](#properties)
  * [Links](#links-docs)
- [Template Advice](#template-advice)
  * [ERB](#erb)
    + [Testing](#testing)
  * [bash](#bash)
- [Monit Advice](#monit-advice)
- [Exporting Logs](#exporting-logs)
  * [Metron Agent](#metron-agent)
  * [The Ol' `tee` and `logger` Approach](#the-ol-tee-and-logger-approach)
- [Backup and Restore features](#backup-and-restore-features)

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
can use a signal in your `drain` script to mark yourself as unhealthy. After a period of time or after no more requests
are detected you can stop the process completely. If you aren't behind a load balancer then you will have to find
another way to tell upstream components about your imminent demise.


## New BOSH Job Lifecycle with BPM

BPM stands for BOSH Process Manager. You can get more information about it in the [BPM chapter](bpm-chapter) of the BOSH
documentation. The [BPM release repository](bpm-release) is also a great place where to find more information about BPM.

We provide here a [`sample-app` job](./jobs/sample-app) that shows how simple things can be when delegating daemons
management to BPM. That said, BPM doesn't provide an answer to all use cases yet. Classical BOSH `pre-start` and `drain`
scripts are still necessary for the use cases documented below.

One note about BPM `pre_start` hooks: contrarily to BOSH `pre-start`, BPM `pre_start` scripts are run at every
`monit start` or `monit stop`, i.e. even when the BOSH instance gets back up after being evicted by the underlying Cloud
infrastructure. Actually, you should put in your BPM `pre_start` what you would have put in your `start` script before
`exec`-ing your process.

[bpm-chapter]: https://bosh.io/docs/bpm/bpm/
[bpm-release]: https://github.com/cloudfoundry-incubator/bpm-release

## BOSH Job Lifecycle

Every BOSH job goes through a specific lifecycle for starting and stopping. The BOSH website has a great
[overview](https://bosh.io/docs/job-lifecycle/) of the lifecycle. You may wish to review this before reading on to
our recommendations for the individual parts of the lifecycle.

### Starting

#### Pre-Start ([docs](https://bosh.io/docs/pre-start/))

Pre-start scripts are run before BOSH hands off control to Monit, so will not necessarily run every time a job starts
(e.g. if a VM is rebooted outside of BOSH's control). However, `pre-start` scripts will run at least once for every new
release version that is deployed, and have _no timeout_ (in contrast to a `start` script). Therefore, `pre-start`
scripts can best be used for performing lengthy set-up of persistent state that must be done for a new version of a
release, such as database migrations. However, because the `pre-start` script does not necessarily run in every VM your
job will run in, do not perform any work in temporary directories such as `/var/vcap/sys/run` (see [VM Configuration
Locations][vm-config-loc] for the list of temporary directories).

[vm-config-loc]: https://bosh.io/docs/vm-config/

In general, a `pre-start` script should not be necessary and the start script should be sufficient. If you find a
`pre-start` script is necessary, keep the above caveats in mind.

#### Monit Start

The `start` script has two main responsibilities: writing the process ID (PID) to a `.pid` file and starting the main
process. All setup required for the process to run (that has not been done in the `pre-start` script) must be done at
this time. Monit places a short timeout on the `.pid` file being written, so the work done in the `start` script should
be focused on executing your process and getting it healthy quickly.

**Note**: This script will likely run many times, so ensure that it is idempotent (that it can be run repeatedly without
causing the system to enter a bad state)

The general workflow of a start script:

1. Create a log directory for log persistence in `/var/vcap/sys/log/<job name>`
   - Idempotency recommendation: Do not fail if the log directory already exists

2. Create a run directory to contain the pidfile in `/var/vcap/sys/run/<job name>`
   - Idempotency recommendation: Do not fail if the run directory already exists

3. Change ownership permissions on the `log/` and `run/` directories to user `vcap` and group `vcap`

4. Run your process with `start-stop-daemon`, which will manage your process's pidfile, ensuring multiple instances
   of the process are not run simultaneously. A recommended usage of `start-stop-daemon` looks like this:

   ```
   /sbin/start-stop-daemon \

     # Write the pidfile, and error if it exists
     # and refers to an already-running process
     --pidfile "$PIDFILE" \
     --make-pidfile \

     # Run the process as the less-privileged vcap user
     --chuid vcap:vcap \

     # Start the given process and redirect its logs
     --start \
     --exec /var/vcap/packages/paragon/bin/web \
        >> "$LOG_DIR/web.stdout.log" \
       2>> "$LOG_DIR/web.stderr.log"
   ```

   Refer to the start script of the pararagon job for a more-complete example.

**Note**: The start script is executed as root. Do not assume you can only break your own process.

**Note**: If your process cannot start quickly, consider moving long-running tasks to `pre-start` or `post-start`.

**Note for Windows Releases**: The Windows BOSH Agent does not use Monit and manages starting the script directly. If
your `pre-start` and `post-start` scripts have been written following the guidance in this document, the Windows BOSH
Agent will be able to start your process correctly.

##### Logging

Job logs should be put in the `/var/vcap/sys/log/<job>` directory. You can redirect the logs from your server (if
they're output on `stdout` and `stderr`) by using standard redirection:

```bash
exec ... \
  >> "/var/vcap/sys/log/<job>/<process>.stdout.log" \
  2>> "/var/vcap/sys/log/<job>/<process>.stderr.log"
```

If your process takes log locations as parameters, use that instead:

```bash
exec ... \
  --out-log-file "/var/vcap/sys/log/<job>/<process>.stdout.log" \
  --err-log-file "/var/vcap/sys/log/<job>/<process>.stderr.log"
```

What is important here, is that all log files _must_ end with the `.log` extention in order to be properly rotated by
BOSH. If any other extension is used for files where daemon append logs, you might end up with full disks and experience
failing nodes. For those who are interested to know more, BOSH enforces a Logrotate config
[in `/etc/logrotate.d/vcap`](https://bosh.io/docs/vm-config/#global).

For detail on forwarding logs to external locations, please see [Exporting Logs](#exporting-logs).

Logging the activity of `start`, `stop` is not critical when leveraging `start-stop-daemon` because they become really
simple. Some `pre-start` or `post-start` script are not trivial though, and need logging for debug purpose. In such
case, we recommend you prepend some time indication to the logged lines. This is useful for debugging nodes that have
been deployed for a long time, in order to distinguish old errors from recent errors.

```bash
function start_logging() {
  exec \
    > >(prepend_datetime >> /var/vcap/sys/log/<job>/<process>.stdout.log) \
    2> >(prepend_datetime >> /var/vcap/sys/log/<job>/<process>.stderr.log)
}

function prepend_datetime() {
  awk -W interactive '{ system("echo -n [$(date +%FT%T%z)]"); print " " $0 }'
}

function main() {
  start_logging
  ...
}

main "$@"
```

As a good remark, you'll note that the above code starts multiple processes for every line of logs. Yes indeed, this way
of doing is not to be applied to all cases. Here, starting multiple processes for every line of logs is acceptable
though, because scripts like `start`, `stop`, `pre- start`, `post-start` or `drain` are not supposed to be executed
often or to produce massive amounts of logs.

#### Post-Start ([docs](https://bosh.io/docs/post-start/))

`post-start` is useful for custom health-checks to ensure you job has started correctly. For example, if your process
starts quickly, but takes time to discover services or connect to its backend, you may wish to use `post-start` to query
the readiness of your job to start handling requests. If a `post-start` script is provided, BOSH will not consider a job
to be ready until it has exited successfully.

When ensuring in a `post-start` that your job's is healthy and its dependencies are met, you are responsible for
implementing the proper timeouts. Most of the time, these timeouts will need to be configurable in order for your job to
adapt to slow or loaded IaaS infrastructures. A good way to do this is to expose a timeout scale factor in your job's
configuration properties.

#### Post-Deploy ([docs](https://bosh.io/docs/post-deploy/))

`post-deploy` is useful for tasks that must occur once all instances of a deployment are started, such as checking the
health of an entire deployment, enabling a feature after a successful upgrade, or finishing database migrations for an
entire cluster.

### Stopping

Job is unmonitored before any stop scripts can run, so you can safely exit in either `drain` or `monit stop` without the
job becoming listed as unhealthy.

#### Drain ([docs](https://bosh.io/docs/drain/))

Drain scripts are optional hooks into the BOSH job lifecycle that are run before stopping the job via Monit. They are
typically used for services which must perform some work before being shut down, for example flushing a request queue or
evacuating containers from a Diego cell. As a rule of thumb, if `monit stop`-ping your job could cause dropped
connections or a lack of availability, a `drain` script should be used to prevent this. Most commonly, your `drain`
script will send a request to a drain endpoint on your process and wait for it to return rather than implementing the
drain behavior itself.

A concrete example is the [gorouter][gorouter], which has a configurable [`drain_wait`][drain-wait] parameter. When
non-zero, gorouter's drain script will instruct gorouter to report itself as unhealthy to its load-balancer with the
intent of being removed from the balanced instance group before shutting down and rejecting requests. When `monit stop`
is called, the router will already be receiving no connections, so will not drop connections when it is shut down
quickly. This is the [lame duck][lame-duck] pattern.

[gorouter]: https://github.com/cloudfoundry/gorouter
[drain-wait]: https://github.com/cloudfoundry/routing-release/blob/develop/jobs/gorouter/spec#L62-L67
[lame-duck]: https://landing.google.com/sre/book/chapters/load-balancing-datacenter.html#robust_approach_lame_duck

Drain scripts have no timeout, so should take whatever time necessary to block on any draining work. One may, however,
run the risk of writing a `drain` script that never finishes and blocks a deployment. A well-written drain script is
guaranteed to finish, such as by adding your own sensible timeout around draining work (e.g. 10 minutes).

When your draining process has completed, your drain script should output a "0" to STDOUT to inform BOSH that draining
is complete.

Open question: If your job needs to wait for another job to drain first, what is the best way to block on that? If you
have any ideas, please [let us know][contact-us].

In a `drain` script, you might need to tell the difference between your process being _temporarily_ stopped, or
_permanently_ decommissioned. This is especially relevant for cluster nodes that share a global state. A node that
permanently leaves the cluster might have to re-distribute its data to the other nodes that stay in the cluster (this is
for instance the case with Cassandra nodes) or just tell the others that it's no use waiting for it to come back
(MongoDB calls this “stepping down”).

In such case, BOSH proposed a hack-ish check on the permanent disk size being set to zero in the future. This
information is provided by the JSON document exposed by the `BOSH_JOB_NEXT_STATE` environment variable, as detailed in
the [Environment Variables](https://bosh.io/docs/drain/#environment-variables) section of the `drain` script
documentation.

You can see two examples of such scripts that follow this best practice:
- For etcd, in [cloudfoundry-incubator/cfcr-etcd-release/jobs/etcd/templates/bin/drain.erb][etcd-drain]
- For Cassandra, in [orange-cloudfoundry/cassandra-boshrelease/jobs/cassandra/templates/bin/drain][cassandra-drain]

[etcd-drain]: https://github.com/cloudfoundry-incubator/cfcr-etcd-release/blob/master/jobs/etcd/templates/bin/drain.erb
[cassandra-drain]: https://github.com/orange-cloudfoundry/cassandra-boshrelease/blob/master/jobs/cassandra/templates/bin/drain

#### Monit Stop

The `stop program` in your job's `monit` file is executed after the `drain` script (if present) has finished running. It
can also be run directly by an operator if they execute `monit stop <job>` on the machine (though operators should
always consider running `bosh stop <instance-group>/<index>` instead of using `monit stop` directly). By default, there
is a 30 second timeout on this script completing. Monit will assume scripts taking longer than this have failed. We do
not recommend changing this value if you need more time. Instead, you should do all the work necessary in your `drain`
script such that your service can shutdown quickly (where “quickly” generally means in under 10 seconds).

If you're not using `drain`, then we recommend that you send `SIGTERM` to your process, which should cause it to start
shutting down. If your process shuts down at this point then you're good to go. If for some reason your process locks up
or is unable to exit for some other reason then you should send `SIGKILL` before the timeout. If your language runtime
supports dumping the stacks of all running threads on `SIGQUIT` (Go and Java do) then you can send that signal just
before the `SIGKILL` to aid debugging why the process is stuck. If you are not using a `SIGKILL` respecting runtime then
adding this functionality to your own program normally isn't difficult.

`start-stop-daemon` can be used for this:

```
/sbin/start-stop-daemon \

  # Remove the pidfile after killing the process
  --pidfile "$PIDFILE" \
  --remove-pidfile \

  # Send SIGTERM, wait for the process to die for 20 seconds
  # If the process has not died, send SIGQUIT and wait for 1 second
  # If the process has still not died, send SIGKILL
  --retry TERM/20/QUIT/1/KILL \

  # If the process is already gone, do not error
  --oknodo \

  --stop
```

Refer to the stop script of the pararagon job for a more-complete example.

If you're using drain to kill the process then your process may already be shut down by the time that `monit stop` is
called. In this case we do not need to do anything further in the stop executable.

**Note for Windows Releases**: The Windows BOSH Agent does not use Monit and manages starting the script directly. If
your `pre-start` and `post-start` scripts have been written following the guidance in this document, the Windows BOSH
Agent will be able to stop your process correctly.

## Advice for authoring `spec` files

### Properties

The properties you include in your `spec` file create the product surface for your job from the perspective of the
operator component. Job properties can make it much easier or much harder to operate a deployment, so one should be
mindful of the operator when making decisions about properties.

- Properties should not have a “namespace” for the job itself (e.g. `my-job.port`, `my-job.hostname`), but should
  only use namespace to create property groups that the job cares about (e.g. `database.*` and `blobstore.*`).
- If a property does not need to be configured specially for every deployment and a reasonable default exists, it should
  be provided. This lets an operator have a terser deployment manifest, which is easier to generate, read, and modify.
- Include a description for every property whenever it is slightly likely that it would help with understanding. It
  can be easy for an operator to misinterpret a property and use it incorrectly.

Here is an example of properly written description, from the [ATC job of Concourse][atc-spec-example]

```yaml
  external_url:
    description: |
      Externally reachable URL of the ATCs. Required for OAuth. This will be
      auto-generated using the IP of each ATC VM if not specified, however
      this is only a reasonable default if you have a single instance.

      Typically this is the URL that you as a user would use to reach your CI.
      For multiple ATCs it would go to some sort of load balancer.
    example: https://ci.concourse.ci
```

You can find other good examples [for Cassandra](cassandra-spec-example) or [MongoDB](mongod-spec-example). Generally,
using the pipe `|` notation for the `description:` YAML property, and providing a text with lines wrapped around the
80th column, is best for readability.

[atc-spec-example]: https://github.com/concourse/concourse/blob/master/jobs/atc/spec#L78-L86
[cassandra-spec-example]: https://github.com/orange-cloudfoundry/cassandra-boshrelease/blob/master/jobs/cassandra/spec#L254-L264
[mongod-spec-example]: https://github.com/orange-cloudfoundry/mongodb-boshrelease/blob/master/jobs/mongod/spec#L80-L92

### Links ([docs](https://bosh.io/docs/links/))

Links allow jobs to provide and consume configuration that needs to be shared between jobs, which can greatly reduce
the amount of configuration required in a deployment manifest. Instead, jobs can declare which information they need
and BOSH will provide that information automatically. When links are used correctly, there are two main benefits:

- IP address information (e.g. locating servers from other jobs) does not need to be provided at all by the operator,
  so default manifests are intrinsically much more network-agnostic and portable.
- Configuration can be provided in a manifest to only one job, which will then export it as a link consumed by
  downstream jobs, making it much easier to modify that configuration and reduce human error. For example, if the port
  a server listens on is provided as a link, a change to the server's configuration will automatically propagate to all
  jobs that need to communicate with it.

If links are used whenever a job depends on configuration from another job, manifests can become much simpler and
deployments can become more reliable. When authoring your release, all properties you include in your `spec` file should
change the runtime behavior of your process and should not include IP addresses, domain names, credentials, or
certificates of other jobs. If you are including these properties, they are candidates for links instead. If the job
you depend on does not provide the information as a link, please consider submitting an issue or PR to the maintainer.

#### Overriding links

Sometimes it is necessary to allow operators to override individual properties within a link. For example, if your job
uses a link's IP information to find a dependent service, but a particular deployment may be using custom DNS for
service discovery, your job template could prefer the DNS property over the link. That template could look like this:

```yaml
<%
  require "json"

  server = nil
  if_p("database_location") do |prop|
    server = prop
  end.else do
    server = link("database").instances[0].address
  end
-%>

server: <%= server.to_json %>
```

## Template Advice

### ERB

Any template in your job can use ERB (even the `monit` files!). While this is extremely powerful it can be very
difficult to understand and maintain complex templates. Therefore, we advise avoiding any ERB in your control scripts
wherever possible. The control flow of starting and stopping your program should be deterministic and simple. All ERB
should be relegated to static configuration files so that properties can be interpolated.

### Encoding

When you inject configuration properties with the ERB syntax in shell scripts or configuration files, you should take
great care for properly encoding special caracters that are meaningful to the target syntax. Failing at doing so could
cause shell scripts to erase critical data, or mis-configuration in YAML files to cause data loss.

With Bash shell scripts, we recommend encoding properties using `shellwords`:

```bash
#!/usr/bin/env bash
<%
  require "shellwords"

  def esc(x)
    Shellwords.shellescape(x)
  end
-%>

some_property=<%= esc(p('some_property')) %> # <- Please observe that you must not quote the value here
```

With YAML config files, we recommend encoding properties with `json`:

```yaml
---
<% require "json" -%>

# The name of the cluster. This is mainly used to prevent machines in
# one logical cluster from joining another.
cluster_name: <%= p("cluster_name").to_json %> # <- Please observe that you must not quote the value here
```

For complex logic in YAML templates, you can adopt [the full-Ruby option][complex-bosh-property-templating], where you
first build a Ruby Hash for your YAML configuration, and then `YAML.dump()` it, which properly performs the expected
encoding.

Don't hesitate to [contact us][contact-us] and submit advices or best-practice encoding for other syntax schemes.

[complex-bosh-property-templating]: https://bosh.io/docs/bpm/transitioning/#complex-bosh-property-templating

#### Testing

Your templates should be simple enough to not need testing, such as by doing a simple passthrough to property parsing
in your own code, but you may find yourself with more-complex templating needs, such as when writing a release wrapping
third-party code. Testing BOSH templates is classically a hard topic, but it has been greatly simplified with a new
version of the `bosh-template` Gem, and a proper documentation of [unit testing][unit-testing] with it.

As it might still be difficult to test BOSH template rendering, you could also imagine having a simple template (e.g.
`<% p('my-properties').to_json %>`) that you would transform with a custom executable, which you can test in your
standard unit-testing workflow.

[unit-testing]: https://bosh.io/docs/job-templates/#unit-testing

### Bash

We advise you to use the `#!/usr/bin/env bash` sheebang variant.

The [ShellCheck][shellcheck] tool can be used to find common errors in your `bash` scripts. This yet another reason to
keep ERB out of your scripts!

You can also possibly use [BATS][bats] unit tests, or similar testing frameworks.

[shellcheck]: https://github.com/koalaman/shellcheck
[bats]: https://github.com/sstephenson/bats

## Monit Advice

Try to keep your Monit configuration as simple as possible. The BOSH team is planning on removing Monit soon - do not
couple yourself to it! This includes such changes as monitoring memory and changing away from the default timeout.

You must specify `group vcap` in your `monit` file because the agent [uses this tag][agent-monit-group] to find
processes it should be managing on the machine.

[agent-monit-group]: https://github.com/cloudfoundry/bosh-agent/blob/5beacb106a67e403a15e9926b6ee39006d774d31/jobsupervisor/monit_job_supervisor.go#L114

## Exporting Logs

BOSH itself does not handle forwarding logs off-system. If you have written your logs appropriately as described in
the [Logging](#logging) section, operators can choose the correct log-forwarding mechanism for their deployment using
job co-location in the deployment manifest or BOSH [addons][addons]. Operators may choose to use something like
[google-fluentd][google-fluentd] or [syslog-release][syslog-release]. If you are also responsible for providing a
deployment manifest generation tool, you may wish to provide the option to add syslog-release forwarding to all
components.

[addons]: https://bosh.io/docs/runtime-config/#addons
[google-fluentd]: https://github.com/cloudfoundry-community/stackdriver-tools#deploying-host-logging
[syslog-release]: https://github.com/cloudfoundry/syslog-release

**Backwards Compatibility Warning**: If your release is currently responsible for forwarding logs off-system, by
following this guide and removing that functionality, you are putting the onus for logging on deployment authors
(e.g. [cf-deployment][cf-deployment] or your local friendly closed-source proprietary offering) to configure logging.
If your release has traditionally done this, deployment authors may not know how to update the deployment and logs could
be lost. If you are removing this functionality, please coordinate appropriately with deployment authors and operators.

[cf-deployment]: https://github.com/cloudfoundry/cf-deployment

### Metron Agent

`metron_agent` has functionality to forward syslogs, but this has been superseded by `syslog-release`. `metron_agent`
should only be used to forward logs to [loggregator][loggregator].

[loggregator]: https://github.com/cloudfoundry/loggregator

### The Ol' `tee` and `logger` Approach

Many releases currently include complex setup to forward logs to both `/var/vcap/sys/log` as well as syslog. This
is not necessary if operators use one of the log-forwarding options mentioned above. Drawbacks of the tee and logger
approach include starting multiple processes for every line of logs, fairly subtle behavior that can easily break or
security vulnerabilities. If your release has any code like the following, please remove it and follow the
recommendations above:

```bash
DO NOT USE THIS CODE

exec > \
  >(
    tee -a >(logger -p user.info -t vcap.$(basename $0).stdout) | \
      awk -W interactive '{ gsub(/\\n/, ""); system("echo -n [$(date +\"%Y-%m-%d %H:%M:%S%z\")]"); print " " $0 }' \
      >> /var/vcap/sys/log/<job>/<process>.stdout.log
  )
exec 2> \
  >(
    tee -a >(logger -p user.error -t vcap.$(basename $0).stderr) | \
      awk -W interactive '{ gsub(/\\n/, ""); system("echo -n [$(date +\"%Y-%m-%d %H:%M:%S%z\")]"); print " " $0 }' \
      >> /var/vcap/sys/log/<job>/<process>.stderr.log
  )

DO NOT USE THIS CODE
```

<sub>If you must use this, you should understand it. The `exec` calls redirect `STDOUT` and `STDERR` respectively,
sending them to a sub-shell that calls `tee`. `tee` splits the output to 1) syslog via `logger` and 2) the BOSH log
directory (but not before appending timestamps with `awk`).</sub>


## Backup and Restore features

The BOSH Backup and Restore project (BBR) provides a framework for backing up and restoring BOSH deployments and BOSH
Directors.

For more information, go read the [BOSH Backup and Restore](https://docs.cloudfoundry.org/bbr/) chapter in Cloud Foundry
documentation.

As a BOSH Release author, you shall also be interested in reading the
[BOSH Backup and Restore Developer's Guide](https://docs.cloudfoundry.org/bbr/bbr-devguide.html).

The BBR project provides its own
[Exemplar Backup and Restore Release](https://github.com/cloudfoundry-incubator/exemplar-backup-and-restore-release) for
the purpose of demonstrating best practice in implementing the
[BBR contract](https://docs.cloudfoundry.org/bbr/bbr-devguide.html). We advise you to refer to this materials for
learning more about providing standard backup and restore features in your BOSH Releases. Here we just provide an
example [`stateful-daemon` job](./jobs/stateful-daemon) which implements very basic BBR features.

<!-- Global Links -->

[contact-us]: https://github.com/cloudfoundry/exemplar-release/issues
