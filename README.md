# Exemplar Release

## To ERB or not to ERB

- Do not use ERB if you can avoid it
  - Use config files instead
- If you need to, anything in job "templates" folder can be ERBed

## BASH best practices

- Shellcheck

## Monit file

- As simple as possible (don't rely on monit always existing)
- Must specify `group vcap` for agent
  - Investigate if this is true

## Lifecycle

[Overview](https://bosh.io/docs/job-lifecycle.html)

### Starting

#### Pre-Start ([docs](https://bosh.io/docs/pre-start.html))

- Not run every time a job starts
- Put stuff that only needs to happen once (per release version?)

#### Monit Start

- Calls your monit script with "start"
- Start ASAP
  - If you can't, move long process to post-start or pre-start
- Make run dir since it's temp and pre-start might not have created it

1. Create/chmod log dir
1. Create/chmod run dir
  - run dir is temp, will be cleared on reboot
1. Write to your pidfile
1. chpst
  - Avoid using monit for this, better to be dependent on monit as little as possible to support future BOSH evolution

Anti-recommendation:
1. ~~For extra privacy, chmod <job>/config~~
  - All jobs start as root, so this doesn't give you any extra benefit
  - Everything runs as vcap so no extra security

#### Post-Start ([docs](https://bosh.io/docs/post-start.html))

- Use for custom health-checks to ensure starting worked, system is connected to what it needs

#### Post-Deploy ([docs](https://bosh.io/docs/post-deploy.html))

### Stopping

Job is unmonitored before any stop scripts can run, so you can safely exit.

#### Drain ([docs](https://bosh.io/docs/drain.html))

- Drain scripts are primarily for jobs that cannot be `monit stop`ed quickly (< 10 seconds? Maybe)
- For example, anything that needs to lame-duck
  - GCP: Can CPI unregister gorouter from LB?
  - Behind gorouter: Are we sending "forget me" to the router?

- Take time to clean up your process, goal is for `monit stop` to succeed quickly
- Exit code for success
- Recommended
  - Have a blocking way to tell your process to clean-up (e.g. an endpoint)
    - If you don't want to block, your script should wait-poll. Use a non-BASH language for this.
    - Drain script can run forever, so you may wish to place your own upper-bound
  - When done cleaning up, exit and print 0 ("wait after draining" value)

What is the recommendation if a job needs to wait for another job to drain first?

#### Monit Stop

- Calls your monit script with "stop"
- Stop ASAP (< 10 seconds)
  - If you can't, move long process to drain
- Recommended
  - If not using drain to kill:
    - `SIGTERM`, wait 10 seconds, `SIGKILL`
    - Optionally, `SIGQUIT` to dump stacks if your runtime supports it (or you add your own support):
      - Go
      - Java
  - If using drain to kill:
    - Send `kill -9 $pid` to your process to kill it immediately
    - If your drain script is implemented correctly, your process should already be gone

## Logging

- Component logs should always be sent to /var/vcap/sys/log/<job>/<*>
- If still relying on metron_agent for syslog forwarding, forward logs to `logger` via `tee`
  - Necessary until cf-release vX.Y.Z officially recommends syslog-release
- Can pass log destinations as process arguments or redirect process output with `exec > log.txt`, but only choose one

Figure out timestamp recommendation

### syslog-release

- Looks at everything in BOSH log directory and forwards to the configure drain URL
- Configures syslogs to be government-compliant
  - What does that mean?

### Notes for cf-release

- metron actively forwards to dopplers and passively configures syslog daemon
- in current config, /var/vcap/sys/log/* is just used for operator convenience
- with syslog-forwarder, /var/vcap/sys/log/* would be what is forwarded to syslogs

### Monit FAQ

- Should monit files be responsible for checking things like memory usage? (e.g. CC)
  - In the futuurrrrrrre, but discourage more usage (don't rely on monit because it's not the future of BOSH)
