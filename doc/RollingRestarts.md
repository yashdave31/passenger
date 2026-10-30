# Rolling restarts

A rolling restart replaces an application's processes one at a time instead of stopping all of them first. The application keeps serving requests throughout, so a restart no longer causes the stall that a blocking restart does while the first new process boots.

This is an independent implementation in the open source Core. Nginx is the supported web server integration.

## Using it

Turn the directive on for an application (or a server or `http` block that contains it):

    passenger_rolling_restarts on;

The default is `off`. With it on, restarts triggered by `tmp/restart.txt` (or `always_restart.txt`) are rolling restarts. Turning the directive off restores the blocking behavior.

`passenger-config restart-app` always requests a blocking restart unless you pass `--rolling-restart`, and that flag is still rejected as an Enterprise-only feature in this tree. Use `restart.txt` to trigger a rolling restart.

## How it works

`Group::restart()` decides between the two kinds of restart. If the restart is a rolling one and the group has at least one enabled process, it calls `Group::finalizeRollingRestart()` on a helper thread. With no enabled process there is nothing to keep serving, so it does a blocking restart.

For each enabled process, newest first:

1. Spawn one replacement and wait until it is ready. The pool lock is not held while spawning.
2. Under the pool lock, attach the replacement and detach the old process. Both steps happen in one lock hold, so requests never see the group without processes.
3. The detached process drains through the existing `detachedProcessesCheckerMain()`: it is shut down once its active sessions finish, so in-flight requests are not cut off.

`restart()` switches the group to the new spawner and options while it holds the pool lock, before the helper thread starts. This stops the normal scale-up path from spawning a process on the old code that would then be attached after the roll.

## Behavior to know about

- **Mixed versions.** While a roll is in progress, old and new processes serve requests side by side. Do not use rolling restarts for changes that cannot run against both versions at once.
- **Failed deployments.** If the first replacement fails to spawn, the roll stops and the group is switched back to the previous spawner and options, so the application stays up on the old code. If a later replacement fails, the roll stops and the remaining old processes keep serving. The error is logged.
- **Process limits.** Replacing a process does not need an extra slot in the pool: detached processes do not count towards the group's or the pool's capacity, and the swap is atomic. A roll works when the group is at its maximum number of processes. During the swap the replacement process exists (and uses memory) before the old one has finished draining.
- **Overlapping restarts.** A newer restart supersedes a roll in progress. The older roll notices on its next step, drops its replacement process, and exits.
- **Shutdown.** If the group shuts down during a roll, the roll exits and drops any replacement it has not attached.

## Observing a roll

`passenger-status` shows the progress per application group, and marks each process as pending replacement (previous version) or replacement (new version):

    /app (production):
      (rolling restart: 1 of 3 processes replaced...)
      * PID: 10486   Sessions: 0       Processed: 3       Uptime: 20s
        Pending replacement (previous version)
      * PID: 10634   Sessions: 0       Processed: 0       Uptime: 1s
        Replacement (new version)

The XML output has `rolling_restarting`, `rolling_restart_total` and `rolling_restart_replaced`. The Core logs the start of a roll, each replacement, an abort and the completion at the info log level, which is above the default level (`--log-level 4` or higher shows them).

## Testing

The C++ tests are in `test/cxx/Core/ApplicationPool/PoolTest.cpp` (tests 22 to 24). They step a roll one replacement at a time using the pool debug support: the roll sends `About to replace process N` and waits for `Replace process N`, and sends `Rolling restart done` at the end. Test 24 runs 20 rolls in a row and checks the pool invariants after each. The failed-deployment and shutdown cases are not covered by C++ tests; they were verified by hand against a running Nginx.

## Installing without compiling on the server

`.github/workflows/rolling-tarball.yml` builds a tarball containing the Passenger runtime files, the agent binary and the Nginx dynamic module, using `dev/ci/package-rolling-tarball`. Pushing a tag named `rolling-*` publishes it (with its `.sha256` file) as a GitHub release; a manual run uploads it as a workflow artifact instead.

The tarball is built for one Nginx version (set in the workflow, with the SHA256 of its source tarball) on Ubuntu 22.04 x86_64. The module is built with `--with-compat`, so it loads into the distribution's Nginx of the same version without replicating its configure flags; `nginx -t` rejects a module that is not compatible. The binaries are not portable across OS releases or CPU architectures, so build on the target's OS and architecture.

To use it on a server that already runs the distribution's Nginx and Passenger packages, unpack it as root into its own directory, then point the existing configuration at it and restart Nginx (a reload does not load a different module):

- in the file that loads the Passenger module, change `load_module` to `<dir>/ngx_http_passenger_module.so`;
- change `passenger_root` to `<dir>`;
- add `passenger_rolling_restarts on;`.

Leave the distribution packages installed. Rolling back is reverting those lines and restarting Nginx. The files must not be writable by an unprivileged user, because the Nginx master process runs as root and executes them.

## Known limits

- The Core's memory grew by roughly 1 MB more over 30 consecutive rolls under load than over 30 blocking restarts (about 2.7 MB versus 1.7 MB, with the growth slowing down in both). That was not investigated further, so it is not known whether it is a leak or allocator behavior. Memory of the application processes stayed flat.
- The `passenger-config restart-app --rolling-restart` flag and the Apache `PassengerRollingRestarts` directive are still the Enterprise-only placeholders.

## Maintaining the fork

The change is kept small so upstream rebases stay cheap:

- the `passenger_rolling_restarts` entry in `src/ruby_supportlib/phusion_passenger/nginx/config_options.rb` (and the files generated from it, under `src/nginx_module/`, which `rake nginx` regenerates);
- `Options::rollingRestart` and its parsing in `Controller/InitRequest.cpp`;
- the branch in `Group::restart()` and `Group::finalizeRollingRestart()` in `Group/SpawningAndRestarting.cpp`;
- the progress counters and their output in `Group.h`, `Group/StateInspection.cpp` and `Pool/StateInspection.cpp`.

After each rebase, rebuild, run `rake test:cxx GROUPS='Core_ApplicationPool_PoolTest'`, and repeat a restart under load.
